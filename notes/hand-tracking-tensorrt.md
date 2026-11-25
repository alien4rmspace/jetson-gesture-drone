# MediaPipe hand tracking on TensorRT

Date: 2026-11-17 (PST); flight sessions 2026-11-19

- **Classification:** controlled for the worker comparison and the stream
  replay (same input, same settings, only the hand backend changed, three
  replays per backend in alternating order). Exploratory for the model checks
  and the breakdown of a MediaPipe call. Observational for the reset stall (two
  profiled replays) and for the flight sessions.
- **Question:** the [MediaPipe 1.2.3 note](mediapipe-version-comparison.md)
  ended with the hand lag coming from running the models on the CPU. If
  MediaPipe's own hand models run on the GPU with TensorRT, and the rest of
  MediaPipe's hand graph is ported around them, do the hand results stay the
  same while the lag drops?
- **Baseline A:** the MediaPipe 1.2.3 hand worker on the CPU
  (`hand-backend: mediapipe`).
- **Variant B:** `hand-backend: mediapipe-trt`, a new worker process that runs
  MediaPipe's palm detector and landmark model as TensorRT engines, with the
  rest of MediaPipe's GestureRecognizer (VIDEO mode) ported to Python and NumPy.
- **Only intended change:** the hand worker process. Crops, tracking resets,
  result handling and everything after them are shared.
- **Primary metric:** hand lag on the replay: the capture time of the first
  frame that shows a new hand result, minus the capture time of the frame it was
  computed from.
- **Correctness criteria, set before measuring:**
  - model level, engines against the TFLite models on identical crops:
    landmarks within 0.5 px at the median and 2 px at p99 (224-px crop), world
    landmarks within 1 mm and 5 mm, presence and handedness within 0.02 at p99
    and on the same side of 0.5 on at least 99% of crops, the same number of
    palms on at least 99% of crops, matched palm boxes at IoU 0.9 or more, palm
    scores within 0.02 at p99 (11 criteria);
  - worker level, against the MediaPipe worker on the same crops: same hand
    count on at least 98% of calls, landmark error at most 0.5% of the crop side
    at the median and 2% at p95, handedness equal on at least 98% of hands,
    gestures on at least 97%, false command gestures at most 1%, and at least
    97% of MediaPipe's command gestures repeated. With both engines FP32, a
    check of the port's logic: hand count and gestures equal on at least 99%;
  - stream: FPS within 1% of A or better in every pair, lower median hand lag,
    and no failed runs.

**Decision:** adopted. `mediapipe-trt` with both engines FP32 has been the
default since 2026-11-18 (private application repository, commits `16b5e6c`
and `8af0997`), and it ran in the 2026-11-19 flights. Two FP16 variants failed
criteria, and the logic check missed its 99% hand-count bar. All three are
below.

## Where a MediaPipe call's time went

Before porting anything I measured where a call's time goes: stream off, 595
consecutive hand-search crops from the 2026-11-17 test flight, three runs of
each MediaPipe variant and one LiteRT run. The first 30 calls of each MediaPipe
run were excluded, leaving 565 timed calls per run.

| Piece | Time on one core at 1728 MHz | How I measured it |
| --- | ---: | --- |
| Palm detector | 35.8 ms | LiteRT interpreter, 1 thread, pinned to one core (565 crops) |
| Landmark model, each run | 31.6 ms | Same (269 crops) |
| Gesture embedder and classifier | under 1 ms | GestureRecognizer 72.8 ms against HandLandmarker 72.5–72.8 ms per call (median) |
| The rest of MediaPipe | about 2–5 ms | What is left |

MediaPipe runs the models on one thread. Its process CPU time per call equalled
the call's wall time (ratio 0.99–1.01), and the Python API has no thread
setting. With 4 threads, LiteRT ran the same models in 10.7 and 8.9 ms.

Two settings make calls longer than one palm detection plus one landmark run:

- With up to 2 hands, MediaPipe skips the palm detector only after a call that
  returned 2 hands: 64 of 565 calls here.
- With a palm threshold of 0.25, weak palm candidates each get a landmark run
  before the presence check (0.45) drops them. Of 325 no-hand calls that
  followed a no-hand call (one pinned run), 15% took 30–40 ms (palm only), 54%
  took 70–80 ms (one rejected landmark run) and 31% took 90–110 ms (two).

The CPU clock mattered as much as anything. Unpinned, calls took 137.6–141.0 ms
at the median; pinned to one core, 72.8–75.7 ms. Unpinned, the work hops
between MediaPipe's ~13 threads one at a time, no core looks busy, and the
governor kept cores 0–3 at 806–883 MHz. Pinned, the core ran at 1728 MHz. The
clock ratio (about 2.1) roughly matches the time ratio (about 1.9). That
probably also explains the 132–158 ms medians in the earlier version note,
which ran unpinned on an idle board; that note already warned its absolute
times were high.

In the stream, one Nsight Systems replay of the same clip timed a call at
92 ms at the median (p90 133 ms, 382 calls). That report has CPU samples and
context switches for the main process only, not for the hand worker, so I can't
split the 92 ms into running and waiting for a core.

My first reading compared MediaPipe with the 4-thread LiteRT times and
concluded that most of a call was not model arithmetic. That was wrong. At
MediaPipe's one thread, the models are the call.

## Rebuilding the models in TensorRT

The palm detector (192×192 input, 2,016 anchors) and the landmark model
(224×224 input; 21 landmarks, presence, handedness and world landmarks out) come
from MediaPipe's `gesture_recognizer.task`, unchanged since 2023. I rebuild each
network layer by layer with TensorRT's network API, reading the weights straight
from the TFLite file, without ONNX or TensorFlow. Every layer's output shape is
checked against TFLite while building. FP32 engines are built with TF32 off.

Inputs: 600 palm crops spread over the clip (the crops the live pipeline sent)
and 376 hand crops, one for each distinct hand result in the flight. The
reference is TFLite with XNNPACK, the CPU path MediaPipe uses.

| Against TFLite with XNNPACK | TFLite, reference kernels | TensorRT FP16 | TensorRT FP32 |
| --- | ---: | ---: | ---: |
| Landmarks, px in the 224 crop, median / p99 / max | 0.0000 / 0.0002 / 0.0006 | 0.043 / 0.589 / 2.105 | 0.0000 / 0.0002 / 0.0007 |
| World landmarks, mm, median / p99 / max | 0.0001 / 0.0003 / 0.0005 | 0.060 / 0.575 / 1.884 | 0.0001 / 0.0003 / 0.0005 |
| Handedness on the same side of 0.5 | 100% | 99.47% (2 of 376 differ) | 100% |
| Same number of palms per crop | 100% | 97.83% | 100% |
| Lowest IoU of matched palm boxes | 1.0 | 0.578 | 1.0 |
| Criteria passed | 11 of 11 | **9 of 11** | 11 of 11 |

The FP32 engines match TFLite to about 1e-6, relative. The FP16 palm detector
failed two criteria, palm count and box IoU. Almost every failing crop involved
palms scoring 0.25–0.30, right at the live threshold. FP16 moved the raw scores
by 0.012 at the median, enough to push weak candidates across the threshold or
into a different group when overlapping boxes are merged. I didn't relax the
criteria.

GPU compute time per inference, from `trtexec`: batch 1, three 10-second runs
per engine after a 2-second warm-up, round robin, idle GPU at 1020 MHz:

| Engine | FP16, median of each run | FP32, median of each run |
| --- | ---: | ---: |
| Palm detector | 1.89–1.94 ms | 3.43–3.44 ms |
| Landmark model | 1.12–1.13 ms | 1.98–1.99 ms |

p99 was within 0.02 ms of the median in every run. This is kernel time on an
idle GPU, not the cost of a call in the stream.

## Porting MediaPipe's hand graph

The models are only part of a call. The worker also does what MediaPipe's
GestureRecognizer does around them in VIDEO mode:

1. Reuse the previous call's hand regions, and skip palm detection when it
   already has the maximum number of hands.
2. Otherwise letterbox the crop to 192×192, decode the anchors, merge
   overlapping boxes with weighted non-maximum suppression, and turn each palm
   into a hand region.
3. Add new palms that don't overlap a tracked hand.
4. Cut a rotated 224×224 crop for each region, run the landmark model, drop
   hands below the presence threshold, and work out the next call's region from
   the landmarks.
5. Remove duplicate hands.
6. Run the gesture embedder and classifier, in NumPy (about 1.7 ms per hand).

The worker speaks the old worker's protocol (a frame header on stdin, pixels in
shared memory, JSON on stdout) and builds the same per-hand result, so nothing
else in the stream changed.

Two details were easy to get wrong. I checked both against MediaPipe's own
output:

- MediaPipe sets the palm region's rotation target to `90` in a field measured
  in radians, so the target is 90 rad, not π/2. With π/2 the landmarks differed
  from MediaPipe's by 1.4 px at the median; with 90 rad, by 0.003 px.
- The next call's tracking region comes from a 12-landmark subset, not all 21.
  With all 21, tracked calls differed by 1.8 px at the median; with the subset,
  by 0.001 px.

The gesture embedder also turned out not to read its handedness and
world-landmark inputs. An older TensorRT hand backend, from May 2026, had faked
world landmarks for it, and I had suspected that hurt its gestures. It can't
have. That backend used 2020 model exports; it was removed on 2026-11-18
(commit `1b71420`).

One bug: TensorRT's Python logger printed to stdout, which carries the replies.
The client read a log line as a reply length and hung. The worker now sends
anything else written to stdout to stderr, and the client restarts a worker
whose reply length is over 1 MB instead of waiting.

## Workload

| Field | Value |
| --- | --- |
| Input | The 2-minute clip from the 2026-11-17 test flight, 1920×1080 at 30 FPS, 3,600 frames, SHA-256 `e93036e3…`, with its recorded hand-search crops |
| Worker settings | VIDEO mode, up to 2 hands, palm threshold 0.25, presence and tracking 0.45, gesture score 0.35 (the default profile) |
| Worker comparison | Real worker processes over the real shared-memory protocol. S1: frames 300–899, every frame (595 calls, no resets). S2: every 3rd frame of the clip that has a crop, about what the live worker sees (1,142 calls, 26 tracking resets set the way the live sender sets them). MediaPipe twice; the port with an FP16 landmark model once; the port with both engines FP32 once |
| Stream replay | Default stream profile with motion output off, recording on. 300 warm-up frames, then 1,200 measured (frames 300–1499), paced at 30 FPS, no profiler. Only `--hand-backend` differed. Order A B B A A B, a fresh process each time |
| Platform | Jetson Orin Nano 8 GB, JetPack 7.2.1, TensorRT 10.16.2.10, MediaPipe 1.2.3, Python 3.12.3, MAXN_SUPER power mode; clocks not locked |
| Application code | Commit `68bd977` plus the uncommitted port. The port's new source files are byte-identical to those in commits `16b5e6c` and `8af0997`. Unrelated uncommitted edits were in the tree for both backends |

## Worker comparison

Each row compares one run with MediaPipe's first run, call by call. Hands are
matched by the nearest wrist.

| Against MediaPipe run 1 | Seq. | Same hand count | Landmark error, % of crop side, median / p95 | Handedness | Gesture | False command gestures | Command gestures repeated | Failed |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| MediaPipe run 2 | S1 / S2 | 100% / 100% | 0 / 0 | 100% / 100% | 100% / 100% | 0% / 0% | 100% / 100% | none |
| Port, FP16 landmarks | S1 | 98.99% | 0.025 / 0.23 | **97.54%** | 100% | 0% | 100% of 18 | handedness |
| Port, FP16 landmarks | S2 | 98.60% | 0.023 / 0.18 | 98.38% | 99.03% | 0.65% | 97.2% of 36 | none |
| Port, both FP32 | S1 | 98.49% | 0.020 / 0.24 | 98.59% | 100% | 0% | 100% of 18 | none |
| Port, both FP32 | S2 | 98.77% | 0.018 / 0.15 | 98.71% | 99.68% | 0.32% | 100% of 36 | none |

Each comparison matched 283–317 hands.

- MediaPipe gave identical output in both runs, so every difference in the
  other rows comes from the port.
- The FP16 landmark model failed handedness on S1: 97.54% against 98%. As the
  plan said, the all-FP32 port became the candidate, and it passed all five
  criteria on both sequences.
- The logic check missed. The FP32 port gave the same hand count on 98.49% and
  98.77% of calls, against the 99% I had set. The mismatches are single calls
  or runs of up to 4 calls, in both directions (S1: 9 calls in 7 runs; S2: 14
  calls in 8 runs), and tracking agrees again after each one. That fits palm or
  presence scores that sit on a threshold and tip the other way. That is my
  reading, not a proof. I left the bar where it was and adopted the port with
  the miss stated.
- The largest landmark error, 40% of the crop side, is a call where MediaPipe
  found two hands and the port one, so the nearest-wrist match paired two
  different hands. Apart from that, 2 hands in S1 and 1 in S2 had any landmark
  more than 2% of the side off, at most 3.9%.
- Round trip per call, on an idle board and not pinned (so MediaPipe ran at low
  clocks): MediaPipe 143 / 127 ms at the median on S1 / S2, the FP32 port 13.6 /
  11.2 ms (p90 18.4 / 18.0 ms). From start to first reply: 1.3–1.4 s against
  1.0 s.

## Stream replay

| Run | Backend | Pipeline FPS | Hand lag, median / p90 / max | New results with a hand | Board CPU, 6-core mean | GPU busy | Board RAM, peak | Hand worker CPU / RSS |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| A1 | MediaPipe | 29.36 | 145.2 / 206.9 / 211.1 ms | 141 | 58.5% | 35% | 3,613 MB | 92% / 197 MB |
| B1 | Port | 29.53 | 66.8 / 72.5 / 83.4 ms | 234 | 50.2% | 40% | 3,823 MB | 19% / 596 MB |
| B2 | Port | 29.52 | 66.5 / 72.7 / 82.1 ms | 237 | 49.9% | 47% | 3,845 MB | 19% / 599 MB |
| A2 | MediaPipe | 29.45 | 150.2 / 208.6 / 219.3 ms | 139 | 58.2% | 29% | 3,729 MB | 93% / 197 MB |
| A3 | MediaPipe | 29.44 | 171.8 / 207.3 / 218.6 ms | 147 | 58.5% | 33% | 3,690 MB | 93% / 198 MB |
| B3 | Port | 29.51 | 66.7 / 72.9 / 107.1 ms | 229 | 50.0% | 46% | 3,798 MB | 19% / 588 MB |

All six runs processed 1,500 of 1,500 frames with none dropped and exited
cleanly. No port run fell back to the CPU.

Summary, median of the three run values, with the run range:

| | A: MediaPipe | B: port | Change |
| --- | ---: | ---: | ---: |
| Hand lag, median | 150.2 ms [145.2–171.8] | **66.7 ms** [66.5–66.8] | −56% |
| Hand lag, p90 | 207.3 ms [206.9–208.6] | **72.7 ms** [72.5–72.9] | −65% |
| New results with a hand, per second | 3.45 [3.41–3.61] | 5.76 [5.64–5.84] | +67% |
| Pipeline FPS | 29.44 [29.36–29.45] | 29.52 [29.51–29.53] | +0.3% |
| Board CPU, 6-core mean | 58.5% [58.2–58.5] | 50.0% [49.9–50.2] | −8.5 points |
| CPU busy × clock, summed over cores | 5.98 GHz [5.98–6.00] | 4.48 GHz [4.48–4.54] | −25% |
| GPU busy | 33% [29–35] | 46% [40–47] | +13 points |
| Board RAM, peak | 3,690 MB [3,613–3,729] | 3,823 MB [3,798–3,845] | +133 MB |
| Board input power | 12.32 W [12.32–12.36] | 12.18 W [12.16–12.20] | −0.14 W |

Every port run beat every MediaPipe run on hand lag, results per second, board
CPU and FPS, and every port run had the higher GPU load. The FPS gain is small,
and the camera's 30 FPS caps what FPS can show.

How these were measured:

- **Hand lag** comes from the recording's per-frame data file: 139–147 results
  per MediaPipe run, 229–237 per port run. Only results that contain a hand are
  logged, so the counts are lower than the number of worker replies.
- **Board CPU, GPU busy, RAM and power** come from `tegrastats` every 500 ms
  (107–109 samples per run) over each run's whole log: start-up, warm-up and
  the measured frames, not the measured window alone. GPU busy is tegrastats'
  GR3D load, a hardware counter, not traced CUDA activity. The GPU clock was not
  recorded during the replays.
- **Clocks.** The CPU governor ran lower clocks during the port's runs: cores
  4–5 sat at 1190–1267 MHz at the median, against 1728 MHz with MediaPipe. A
  busy percentage at a lower clock is less work, so the 8.5-point drop
  understates the saving. Busy fraction times clock, summed over the six
  cores, is a rough count of CPU cycles; it fell by about a quarter.
- **Hand worker CPU and RSS** are `ps` values. Its CPU figure is CPU time
  divided by the process's elapsed time, start-up included, so 92% means the
  MediaPipe worker kept about one core busy. RSS counts shared CUDA library
  pages, so the port's ~595 MB overstates what it adds. The board's peak RAM
  rose by about 130 MB (110–210 MB between neighbouring runs).
- **The stream process** used a little more CPU with the port: 174–179% of one
  core against 163–166% (`ps`, same method). After each reply, the sender
  resizes the latest crop and copies it into shared memory, and the port replies
  more often, so that is the likely cause. I did not profile it.
- **Profiler overhead:** no profiler ran in these replays. `tegrastats` and a
  `ps` sample every 2 s ran in all six; their cost was not measured.
- **Temperatures:** the GPU warmed from 48 °C to 60 °C over the six runs, in
  run order, with the two backends interleaved. Throttling was not checked.

Gestures were a sanity check, not a criterion, because results arrive at
different moments in every run. Thumb_Up showed on 56–68 hand-frames with
MediaPipe and 38–81 with the port; Open_Palm on 9–18 and 20–23. Frames showing
two hands rose from 44–81 to 110–130 of 1,200. The worker comparison found the
same hand count on 98.5–98.8% of identical crops, so this probably comes from
the port tracking from a more recent frame, but I haven't checked.

## The tracking-reset stall

When the hand crop jumps (its size changes by more than 1.5× or its centre
moves by more than 25%) or the target person changes, the sender sets a reset
flag, because the old hand regions refer to the old crop. MediaPipe has no
reset call, so the CPU worker closed its recognizer and built a new one before
reading the frame.

In a profiled replay of the same clip (Nsight Systems, frames 300–1500), the
worker normally started reading a frame 0.14 ms after the main process had
prepared it (median of 383 frames). Seven times the gap was 181–339 ms, and six
of those fell within 1.7 s, while the locked person drifted and grew in view. A
second profiled replay of the same window showed five gaps of 215–330 ms, four
of them at the same moment in the clip. The worker does nothing else between
reading the frame header and reading the image, so I attribute these gaps to the
reset; neither report has a range on the reset itself. Profiler overhead in
those captures is unknown.

Standalone timing showed why. Building a new recognizer took 31 ms at the
median, but `close()` on the running one took 223 ms (141–282 ms), waiting for
its graph threads to shut down. Pinned to one core: 27 ms and 168 ms
(139–218 ms). The stall was the close, not reloading the model. I think the
stalls cluster because each one lets the crop drift further before the next is
sent, which triggers another reset. I haven't proven that.

The fix (commit `68bd977`) builds the new recognizer, uses it at once, and
closes the old one on a background thread, with at most two closes in flight.
Every reset still starts from a fresh recognizer, so results don't change.
Checks, all outside the stream:

- On a fixed sequence of 100 crops, the old and new workers gave identical
  results.
- With crops fed at 30 FPS in the order old, new, new, old, a reset call's
  round trip was 330, 153, 124 and 374 ms at the median, against about 135 ms
  for a normal call. The longest gap between processed crops was 12, 7, 7 and
  19 frames. Each run had only 1–5 resets (5, 2, 1 and 3), so those medians rest
  on very few samples.
- In S2 of the worker comparison (26 resets, calls back to back), reset calls on
  the fixed CPU worker took 114–117 ms at the median (at most 158 ms), against
  124–128 ms for other calls.

The cluster did not reproduce outside the stream with either worker, and no
profiled replay was taken after the fix, so the fix was never measured on its
own in the stream. The three MediaPipe replays above already included it.

The port has no graph to close: a reset only clears its list of tracked hand
regions. In S2, the FP32 port's 26 reset calls took 11.0 ms at the median (at
most 16.7 ms), against 11.2 ms for other calls.

## On the drone

The stream's status line names the hand backend it actually selected; with the
engine files missing it falls back and says `mediapipe`. All eleven stream runs
logged on 2026-11-19 show `mediapipe-trt`, and the engines on the drone are the
same FP32 builds the replays used (checked by SHA-256). Five of those sessions
had the drone armed, including the flights at 17:57, 18:44 and 20:03.

Hand lag from each session's frame data file, over the whole session, on the
ground and in the air:

| Session | Backend | New results with a hand | Hand lag, median / p90 / max |
| --- | --- | ---: | ---: |
| 2026-11-19 17:33 | Port | 19 | 66.6 / 73.0 / 74.5 ms |
| 2026-11-19 17:57 | Port | 1,682 | 63.3 / 70.5 / 84.3 ms |
| 2026-11-19 18:44 | Port | 800 | 65.0 / 73.3 / 92.6 ms |
| 2026-11-19 19:37 | Port | 861 | 66.2 / 73.6 / 97.0 ms |
| 2026-11-19 20:03 | Port | 238 | 65.7 / 73.3 / 144.7 ms |
| For reference: 2026-11-17 15:32 flight | MediaPipe, before the reset fix | 337 | 183.8 / 208.3 / 604.6 ms |

The live lag matches the replay, about 66 ms. The 2026-11-17 flight was a
different session, scene and code version, so comparing with it is only
observational. Gesture accuracy in flight was not checked. False thumbs-up
readings, which also happened with MediaPipe on the CPU, were fixed after these
flights (commit `e0d3116`), and nothing compares the two backends' gestures in
flight.

Derived per-run data: [`results.json`](../data/2026-11-hand-tensorrt/results.json).

## Limits

- One recorded flight: one 1,200-frame replay window, and 595 and 1,142 crops
  for the worker comparison. Other lighting, distances, gloves and hand poses
  are untested.
- Three replays per backend. The lag, CPU and GPU differences are large against
  the run-to-run spread; the FPS difference is not.
- There is no ground truth. Agreement is measured against MediaPipe, not
  against where the hands really were.
- The 99% hand-count check missed, and the explanation (scores tipping at a
  threshold) is my reading, not a proof.
- Board CPU, GPU and RAM figures cover whole runs, start-up and warm-up
  included. Clocks were not locked, and the GPU clock during the replays was
  not recorded.
- The `trtexec` times are on an idle GPU, not next to the pose and ReID engines
  in the stream.
- Memory with the on-board language model also running was not measured.
- The reset fix was checked outside the stream only, with 1–5 resets per run.
  The raw timings of those checks were not kept, only the numbers in the commit
  message.
- The engines are built for this board and TensorRT 10.16 and must be rebuilt
  after an upgrade. The fallback to the CPU only covers missing engine files; a
  stale engine makes the worker exit and restart instead.
- In flight, only hand lag was looked at.
