# Recording off the camera's frame path

Date: 2026-11-17 (PST), with live use on 2026-11-19

- **Classification:** controlled replay for the transport comparison and the tee fix: one fixed
  input checked by SHA-256, a fresh process per run, balanced run order, three runs per variant at
  the high cap. The sharpness timings are an exploratory CPU microbenchmark. The two in-stream
  Nsight captures and the flight logs are observational.
- **Status:** the shared frame ring, the YUV recording tee and both sharpness changes have been in
  the default stream since 2026-11-17 and were flown on 2026-11-19. The CPU surface map and the
  converter bypass are opt-in and off.
- **Question:** after the [previous recording note](recording-and-overlays.md) the recording was
  clean 1080p at 30 FPS. But the recorder still took RGB frames from the frame thread and
  converted them back to I420 for x264, and when frames came faster than 30 FPS it dropped them.
  Which frame transport lets the pipeline do more, and does the recorder then keep every frame?
- **Baseline A:** frames land once in a shared RGB frame ring (below). A separate recorder process
  reads each frame by slot, converts it to I420 and encodes it with software x264. The Orin Nano
  has no hardware video encoder.
- **Variants, one switch each:**
  - **M**, CPU surface map: map the GPU's RGB surface for the CPU instead of the DLPack copy.
  - **C**, converter bypass: skip the first converter on the livestream path when on-screen
    drawing is off.
  - **Y**, YUV recording tee: branch the I420 frames to the recorder before the RGB conversion,
    and pair each one with its frame data by source timestamp.
  - **MCY**: all three.
- **Primary metric:** completed frame callbacks per second, with frames admitted at up to 240 FPS
  to show headroom. The callback is where the application handles each RGB frame on the CPU.
- **Equally important:** recorded FPS, the measured source frames found in the finished, decoded
  video, per second of the same window. More callbacks with fewer recorded frames is not a
  like-for-like win.
- **Correctness criteria, set before measuring:** every replay consumes all 600 frames with
  matching input and output frame IDs and timestamps; the CPU RGB frames are byte-identical across
  variants; every recording decodes at 1920×1080 with one frame-data row per frame, unique
  ascending source IDs and timestamps, and a sharpness score; sampled recorded frames stay within
  12/255 per channel of their source frame.

**Decision:** I kept the ring and the tee. The default profile picks the tee for clean, full-size
recordings at neutral contrast, and the RGB writer when overlays, resizing or display contrast are
asked for. The surface map and the converter bypass stay opt-in (private application repository,
commits `fa8b7d1`, `f9869bb`, `ebb8fc7`).

## The new benchmark clip

Since 2026-11-17 the fixed replay input is the first two minutes (3,600 frames) of an outdoor
GUIDED flight. The stream recorded it at native 1920×1080, 30.0 FPS on average, with no recorder
drops. Frames 0–1199 are low-altitude follow and hand gestures, 1200–1799 a climb to about 5.1 m,
and 1800–3599 higher follow and arm poses. The standard windows are in-flight (warm-up 300, then
frames 300–1499) and full (300–3599). The comparisons below used a shorter window, frames
150–599, so that each series fit 26 fresh runs.

The previous input, used in the [recording note](recording-and-overlays.md), was a 720p indoor clip
that the pipeline scaled up to 1080p. It had been recorded as processed, so its frame spacing was
uneven and it played about 1.5× fast. The new clip gives x264 real 1080p detail. Results on the two
clips are not comparable (private application repository, commit `8d9a3f6`).

## Workload

| Field | Value |
| --- | --- |
| Input | The clip above, 1920×1080 H.264 |
| Input SHA-256 | `e93036e39c708074d10bea26c52e7a7da05dc96c234f0e38e71cd3cd2e136800` |
| Window | Frames 0–149 warm-up, 150–599 measured (450 frames) |
| Admission caps | 240 FPS to show headroom; 30 FPS to match the camera |
| Work enabled | Default profile with motion output off: pose detector, tracker with ReID, person lock, hand tracking, livestream worker (no browser connected), clean 1080p recording at 30 FPS with the frame data file |
| Encoder | Software x264, 12 Mbps, 4 threads, fragmented MP4 |
| Runs | Fresh process and recording folder per run. At 240 FPS, three blocks per series in orders A M C Y MCY / MCY Y C M A / C Y A MCY M. At 30 FPS, two blocks, A Y MCY / MCY Y A. Then one separate confirmation block. |
| Platform | Jetson Orin Nano 8 GB, L4T 39.2.1, DeepStream 9.1, TensorRT 10.16.2, MAXN_SUPER power mode, `schedutil` governor, clocks not locked; temperatures about 49–64 °C across launches (tegrastats at 1 s) |
| Profiling | None attached. CPU per process was not measured in these two series. |
| Code | Private application repository `a6cfa18` plus the working-tree patch. The second series ran entirely on the final source; the first series' three blocks ran before final validation and error-handling edits. |

## Shared RGB frame ring

Before, the frame thread copied each frame out of DeepStream into a new 6.2 MB array, then copied
it again into the livestream worker's shared memory and again into the recorder's. Now a ring of
POSIX shared-memory slots (the recorder's queue size plus 5) holds the frames. The GPU-to-CPU copy
writes straight into a slot, and both workers read the frame by slot number. A slot a worker still
holds is skipped. A recording that burns overlays in draws them on its own copy, so the slot stays
clean for the other reader. That leaves one host copy per frame instead of one per reader.

Tests check that frames are handed over without copies and that pixels are preserved. I never
timed the ring on its own: it went in before the transport comparison and is part of every variant
there, the baseline included (private application repository, commit `fa8b7d1`).

## Three transports at a 240 FPS cap

Medians of three runs, with the range. Per-run data: [`frame-path.json`](../data/2026-11-recording/frame-path.json).

| Variant | Callback FPS, series 1 | Callback FPS, series 2 | Recorded FPS, series 1 | Recorded FPS, series 2 | Frames saved of 450 |
| --- | ---: | ---: | ---: | ---: | --- |
| A: RGB writer | 58.92 [56.38–58.96] | 59.18 [58.89–61.15] | 33.94 [33.45–34.04] | 34.32 [33.76–34.51] | 254–267 |
| M: surface map | **66.66** [66.57–68.19] | **66.04** [65.81–67.14] | 31.80 [30.37–32.43] | 32.03 [30.89–32.14] | 205–219 |
| C: converter bypass | 59.77 [58.64–60.14] | 58.80 [57.56–59.60] | 34.35 [33.75–34.80] | 33.64 [33.11–35.02] | 250–268 |
| Y: YUV tee | 48.46 [47.34–48.62] | 48.18 [48.01–48.43] | **48.46** [47.34–48.62] | **48.18** [48.01–48.43] | **450, every run** |
| MCY: all three | 48.54 [48.01–48.74] | 48.81 [45.02–49.53] | 48.54 [48.01–48.74] | 48.81 [45.02–49.53] | 450, every run |

Change against the baseline in the same block:

| Variant | Callback change, series 1 / 2 (median) | Recorded change, series 1 / 2 (median) | Blocks in the same direction |
| --- | ---: | ---: | --- |
| M: surface map | +15.6% / +11.6% | −6.6% / −7.2% | 6 of 6 for both |
| C: converter bypass | +2.0% / −2.3% | +1.2% / −0.4% | mixed |
| Y: YUV tee | −17.5% / −18.9% | **+42.3% / +40.3%** | 6 of 6 for both |
| MCY: all three | −17.3% / −19.0% | +43.2% / +43.5% | 6 of 6 for both |

- **The tee kept every frame.** In every run of both series it saved 450 of 450 measured frames,
  where the RGB writer saved 254–267 of them. It completed 17.5–18.9% fewer callbacks, but every
  one of them was recorded.
- **The surface map did more callbacks, and the RGB writer kept fewer of them.** It raised
  callbacks 11.6–15.6% at the median, and recorded FPS fell in all six blocks. A faster callback
  gives the separate recorder less time per frame.
- **The converter bypass showed no effect.** It moved callbacks +2.0% in the first series and
  −2.3% in the second.
- **With the tee on, the other two added nothing:** 48.5 against 48.5 callbacks per second in the
  first series, 48.8 against 48.2 in the second.

Each series ended with a separate confirmation block on the final source, one run per variant.
Both agreed with the blocks above: callbacks 58.65 and 59.07 (A), 65.53 and 66.58 (M), 58.88 and
58.67 (C), 47.91 and 47.96 (Y), 47.46 and 48.27 (MCY), and the tee saved 450 of 450 in each.

## At the camera's 30 FPS

| Callback FPS (2 runs each) | A: RGB writer | Y: YUV tee | MCY |
| --- | ---: | ---: | ---: |
| Series 1 | 29.23, 29.29 | 29.51, 29.53 | 29.59, 29.62 |
| Series 2 | 29.34, 29.38 | 29.49, 29.52 | 29.58, 29.66 |
| Tee-fix series | 29.14, 29.29 | 29.50, 29.56 | not run |

At the camera's rate every variant saved 450 of 450 measured frames in every run. The tee ran
0.1–0.4 FPS faster in every pair, which is small. So at 30 FPS the tee does not save frames. What it
saves is CPU. In the tee-fix series (next section), a sampler read `/proc` for the whole process
tree every 0.25 s:

| 30 FPS cap, cores of 6 | A: RGB writer (2 runs) | Tee (2 runs) |
| --- | ---: | ---: |
| Whole process tree | 2.80, 3.06 | **2.53, 2.54** |
| Main process | 0.70–0.72 | 1.81–1.82 |
| Recorder process | 1.42–1.62 | none |
| Hand worker | 0.66 | 0.66 |
| Encoder thread (conversion + x264) | 1.08–1.14, in the recorder | 0.71, in the main process |

The tee used 0.3–0.5 fewer cores in total. The encoder thread fell from about 1.1 to 0.71 cores
because the RGB-to-I420 conversion is gone, and the recorder process's own Python work went with
it. The main process grew by about 1.1 cores, mostly GStreamer's native threads. Coverage is the
whole process tree, every child process included; with no browser connected, the livestream worker
had nothing to send. No profiler was attached; the sampler's and tegrastats' overhead is
unknown and the same in every run. The window runs from the first FPS log line to 0.5 s before the
recording is finalized, so it includes part of the warm-up. Two runs per variant.

## The tee fix changed nothing

With the first version of the tee, callback FPS equalled recorded FPS in every run, and the frame
callback could wait up to 0.5 s for the matching YUV buffer. I read that as the encoder setting the
pace. I predicted that moving the wait to the recorder thread, and letting the tee's record queue
drop its oldest frame when full (`leaky=2`), would bring callbacks back to about 59 per second,
at the cost of some recorded frames. The plan set the bar before any run: the fixed tee within 5%
of A, and at least 10% above the unfixed tee, in every block.

Frozen copies of the source and configuration for each variant, a Latin square of three blocks at
240 FPS (A Y0 Y1 / Y0 Y1 A / Y1 A Y0), two blocks at 30 FPS:

| 240 FPS cap | Callback FPS, median [range] | Recorded FPS, median [range] | Saved of 450 |
| --- | ---: | ---: | --- |
| A: RGB writer | 58.89 [54.70–59.43] | 33.63 [32.58–34.47] | 257, 268, 261 |
| Y0: tee before the fix | 47.15 [46.46–47.87] | 47.15 [46.46–47.87] | 450 ×3 |
| Y1: tee with the fix | 47.40 [46.25–47.60] | 47.40 [46.25–47.60] | 450 ×3 |

Y1 against Y0 within each block: −0.4%, +1.0%, −1.0%. No effect. Y1 against A was consistent with
the earlier series: callbacks −13.0% to −21.5%, recorded FPS +37.5% to +46.1%.

The prediction was wrong. In a pilot run with GStreamer's queue debug log on, the record queue only
ever reported empty or not empty; it never filled. The recording branch kept up. Callback FPS
equalled recorded FPS only because nothing was dropped. I also checked that `leaky=2` takes effect
when set through the Python bindings: in a test pipeline with one branch slowed to 50 ms per buffer,
100 buffers finished in 0.21 s with it and 5.02 s without it.

I kept the fix as a robustness change, not a speedup. If the encoder does fall behind, the frame
callback can no longer stall for up to 0.5 s; a full queue drops recorded frames instead. And if
the native bridge can't be built or loaded, the stream warns and records with the RGB writer
instead of failing (private application repository, commit `86850ac`).

Two runs were excluded and kept: one pilot whose frozen copy lacked the hand worker's environment,
so the hand worker never started, and the debug-logging pilot, excluded from timing by plan. In
the first transport series, the first tee run completed no callbacks at all: the converter before
the tee failed to negotiate a format. I moved the tee ahead of that converter, kept the failed block,
and restarted the whole declared order.

## Why the tee stops near 47 FPS

I don't know. Across the three series the tee ran 47.2–48.5 callbacks per second at the 240 FPS cap
(medians), against 58.9–59.2 for the RGB writer. What the evidence says:

- **Not the encoder holding up the frame path.** The record queue never filled.
- **Not a saturated CPU overall.** The whole process tree used 3.4–4.1 of 6 cores in every run at
  the cap. That total does not rule out one busy thread. tegrastats' 1 Hz GPU load stayed below
  saturation, but tegrastats on this board does not report the VIC or the memory controller.
- **Untested:** the tee adds a second 1080p conversion out of GPU memory, to system-memory I420,
  which may compete with the RGB conversion on the same hardware.
- **Untested:** in every tee run above, the recording sharpness score (next section) ran in the
  frame callback, while the RGB writer scored frames in its own process. Standalone, the score took
  5.7–5.8 ms per frame. The gap between 58.9 and 47.4 callbacks per second is 4.1 ms per frame, and
  the surface-map result shows that time on the frame path does limit callbacks at this cap. The
  score has since left the frame path, and I have not measured the tee's ceiling again.

At 30 FPS the ceiling does not matter: every variant kept up. It matters for a 60 FPS camera mode.

## Sharpness score off the frame path

Each recorded frame carries an image sharpness score in its frame data, used for the focus status
and by the focus-map tools. At the default setting (centre 35% crop, hybrid method) it cost
5.7–5.8 ms per frame at p50 in two runs of a CPU microbenchmark (6.0 ms p95 in the first), on 40
real 1080p frames from the benchmark clip, 15 timings each. With the tee, this ran in the frame
callback, before the tilt, yaw and follow commands are published.

Two changes:

1. **Off the frame path** (private application repository, commit `7de81ea`). The frame callback
   copies only the scored crop, about 0.76 MB, into the queued request. The recorder thread scores
   it and fills in the focus status and the frame-data row. The computation is the same, split
   into two functions. The trade-offs: the focus status updates a frame or two later, and frames
   dropped from the recorder's 2-slot queue are no longer scored.
2. **Cheaper** (private application repository, commit `d8084c2`). A 3×3 Sobel or Laplacian of
   8-bit pixels stays within ±1,020 and ±2,040, so 16-bit integers hold it exactly. OpenCV's
   `norm` and `meanStdDev` then add up the squares in double precision without float64 copies of
   the image.

| Setting, p50 over 40 frames × 15 timings | float64 | int16 + OpenCV sums |
| --- | ---: | ---: |
| Default: hybrid, centre 35% | 5.66 ms | **2.15 ms** |
| Tenengrad, centre 50% | 12.2 ms | 1.86 ms |
| Laplacian (no longer computes unused Sobel gradients) | 5.51 ms | 1.24 ms |
| Whole frame | 61.7 ms | 15.4 ms |

The largest relative difference from the old score, over 40 frames, 3 methods and 3 crops, was
1.4 × 10⁻¹⁵: summation order only. Thresholds and focus maps are unchanged. A test compares against
the float64 formula, including a 1080p one-pixel checkerboard, the largest possible gradients, to
check for overflow. The microbenchmark ran in a standalone process with clocks not locked and
background load not controlled. Its script and raw timings were not kept, so these numbers come
from the commit record.

**In the stream (observational).** Two existing Nsight Systems captures on the same clip, the
standard in-flight window (warm-up 300, then 1,200 frames at 30 FPS), one capture each:

| Marker | 19:02 capture: RGB writer, float64 | 20:56 capture: tee, int16 |
| --- | ---: | ---: |
| Sharpness score, mean / p50 / p95 | 13.54 / 12.58 / 22.40 ms (n = 1,198), in the recorder process | **4.99 / 4.05 / 10.03 ms** (n = 1,199), in the main process's recorder thread |
| Frame callback, mean / p50 / p95 | 8.05 / 7.13 / 13.57 ms | 7.20 / 6.72 / 10.60 ms |

Neither capture had the score on the frame thread, and a lot changed between them: the RGB writer
process became the tee, float64 became int16, and the livestream overlay moved to the browser.
Both captures used the same Nsight settings: CUDA, NVTX and OS runtime tracing, CPU sampling of the
whole process tree, Python sampling at 100 Hz and GPU metrics at 1 kHz. Their overhead is in both
numbers and unknown. In the stream the score took about twice its standalone time (12.6 against
5.7 ms p50), which fits CPU contention and unlocked clocks, but I did not separate them. The pair shows
the score running cheaper and off the frame thread in the stream; it does not isolate the int16
change. The tee version that scored on the frame thread was never captured with Nsight.

## In use

On 2026-11-19 the stream ran 11 sessions. The pipeline built the tee in all of them, and five
included flights (the drone armed in ALT_HOLD). Pipeline FPS was 30.0 at the median in every
session; the lowest 2-second reading was 29.3. Across 12 recordings the recorder dropped 13 frames,
0–2 per recording, all already counted at the recording's first status line, about 2 s in. The
ring and both sharpness changes were in that code; the surface map and the converter bypass were
off. These are flight logs, not a comparison: no session that day used the RGB writer. The
per-session summary is in [`frame-path.json`](../data/2026-11-recording/frame-path.json).

## Limits

- Replay decodes a file instead of using the camera path. The tee and the RGB writer were never
  compared on the live camera or in flight.
- Three runs per variant per series, with a 450-frame window (15 s at 30 FPS). The two series agree
  in direction for the surface map and the tee; the converter bypass is mixed.
- No profiler in the transport series, so CPU per process comes only from the tee-fix series'
  `/proc` sampler, two runs per variant at 30 FPS. GPU, VIC and memory-controller load, end-to-end
  latency and hand inference counts were not measured. A faster callback can leave the hand worker less
  time per frame.
- Clocks were not locked and background load was not fixed: in the tee-fix series the load average
  rose from 1.3 to 7 over back-to-back runs, the same for every variant.
- The ring was never timed on its own.
- The sharpness microbenchmark's raw data was not kept, and the in-stream pair is observational.
- The tee's ~47 FPS ceiling is unexplained and has not been measured since the sharpness score
  left the frame path.
