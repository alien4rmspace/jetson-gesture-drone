# Livestream CPU: overlays drawn by the browser, frames shrunk bilinearly

Date: 2026-11-17 and 2026-11-18 (PST); iPhone follow-up 2026-11-19

The livestream is a 1280×720 MJPEG stream at 20 FPS. An encode worker process
makes it from the 1920×1080 pipeline frames: for each frame it shrank the
frame, drew the overlay and encoded a JPEG. I made two changes to the worker,
each measured on a fixed replay clip with alternating runs:

1. The browser draws the overlay. The worker sends the clean JPEG together with
   the data that frame's overlay is drawn from.
2. The worker shrinks frames bilinearly instead of by area averaging.

The second came out of a CPU profile of the whole stream, which has its own
note: [where the stream's CPU goes](where-the-cpu-goes.md).

## 1. Overlays drawn by the browser

- **Classification:** controlled replay, small: two runs per variant in A B B A
  order. The drawing moves to the viewing device, and its cost there was not
  measured.
- **Status:** measured on the bench. It was in the running code during the
  2026-11-19 flights (see [Used live](#used-live)), but I have not seen it work
  on a phone.
- **Question:** the worker drew the presentation overlay with its telemetry
  into every livestream JPEG, about 10 ms per 720p frame in an earlier offline
  timing. Text that changes every frame (scores, rates) missed the text cache
  and went through FreeType again each time. The text was also squeezed through
  JPEG compression, and every viewer got the same telemetry. If the worker sends
  the clean frame plus the overlay's data, and the browser draws it, how much
  worker time goes away, with the same frame rates?
- **Baseline A:** the worker draws the overlay into each livestream JPEG.
- **Variant B:** the worker encodes the clean frame. The overlay data, about
  2.3 KB of JSON, goes in the same multipart part as an `X-Frame-Data` header on
  `/stream.mjpg?data=1`. The page reads the stream with `fetch()` and draws frame
  and overlay on a canvas, with a JavaScript port of the Python overlay renderer,
  at the screen's pixel density and with the same bundled fonts. Each viewer can
  hide or show the telemetry. Private application repository, commit `848f1c4`.
- **Only intended change:** one command-line flag (`stream-overlays-in-browser`),
  on one frozen copy of the source.
- **Primary metric:** the worker's own per-frame timer for its overlay step
  (elapsed ms). Hypothesis written before the runs: about 10 ms in A, under 1 ms
  in B.
- **Correctness criteria, set before the runs:** every replay completes with
  matching input and output frame hashes; every B part carries frame data that
  parses, with frame size 1280×720, and no A part does; livestream within 1 FPS
  of its 20 FPS setting and pipeline within 1 FPS of each other.

Sending the data in the same part as its frame means the two can't drift apart,
as they could on a separate WebSocket.

### Workload

| Field | Value |
| --- | --- |
| Input | Fixed replay recording, 1280×720 H.264, indoor flight with the pilot in view, replayed at 30 FPS; the pipeline runs at 1920×1080 |
| Input SHA-256 | `56e5d61423970ccd224002eb868326f590f2ad974e5ee2be719c6a2e108a93f2` |
| Window | 1,950 warm-up frames, then 1,200 measured; all metrics from 70 to 100 s after the first livestream frame, a window fixed before the runs |
| Work enabled | Default profile with motion output off: pose detector, tracker with ReID, person lock, MediaPipe hands on the CPU (the default that day), livestream 1280×720 at 20 FPS, 1080p recording |
| Viewer | A small Python client on the Jetson read `/stream.mjpg?data=1` in both variants and checked every part. Nothing drew the overlay on the Jetson in B. |
| Measurements | Worker timers averaged every 2 s (600 frames per run in the window); whole-machine CPU from `/proc/stat` and main-process CPU every second; tegrastats every second |
| Code | A frozen copy of the working tree: base commit `a6cfa18` plus uncommitted work from other sessions (frame transport and recorder changes), identical in A and B. The change was committed later that evening as `848f1c4`. |
| Platform | Jetson Orin Nano 8 GB, JetPack 7.2.1; power mode not recorded; clocks not locked (1687–1714 MHz mean in the window); junction 52–60 °C |

### Results

| Run (order) | Worker overlay step (ms) | JPEG encode (ms) | Machine CPU (cores of 6) | Busy × clock (GHz) | Board power (W) | Livestream FPS | Pipeline FPS | JPEG (KB) |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| A-01 (1) | 18.86 | 8.75 | 3.87 | 6.67 | 12.72 | 20.0 | 29.48 | 113.3 |
| B-01 (2) | **8.23** | 9.20 | 3.71 | 6.32 | 12.65 | 20.0 | 29.47 | 94.0 |
| B-02 (3) | **8.23** | 9.15 | 3.70 | 6.37 | 12.63 | 20.0 | 29.43 | 94.0 |
| A-02 (4) | 19.05 | 8.83 | 3.86 | 6.64 | 12.79 | 20.0 | 29.44 | 113.3 |

Machine CPU is busy CPU time over wall time from `/proc/stat`, for the whole
machine. Busy × clock is each core's tegrastats load times its clock, summed
over the six cores, averaged over 30 one-second samples. Board power is the
`VDD_IN` rail. A KB is 1,024 bytes.

- **Worker step:** 18.9–19.1 → 8.2 ms per frame, the same in both pairs, about
  10.7 ms less. The hypothesis of "under 1 ms" did not hold as written: in this
  code the timer starts before the 1080p-to-720p shrink, so both variants
  include it. Afterwards, on the idle board, serializing the frame data took
  0.18 ms (median) and the shrink 4.5 ms on six OpenCV threads. So B's 8.2 ms is
  essentially the shrink, and the 10.7 ms difference is the drawing.
- **CPU and power:** the machine was 0.16 of a core less busy (3.86–3.87 →
  3.70–3.71 cores). At 20 FPS, 10.7 ms per frame is 0.21 of a core of worker
  time. Here the clocks stayed near their 1728 MHz top in all four runs, so the
  CPU percentages can be compared directly: busy × clock fell 4.7% and board
  power 0.12 W (0.9%). The main process stayed at 1.91–1.92 cores; the drawing
  was all in the worker.
- **Size:** without the drawn text and panels, the JPEGs are 17% smaller
  (113.3 → 94.0 KB). With 2.3 KB of data per frame, the livestream
  needs about 15% less bandwidth: 18.6 → 15.8 Mbit/s at 20 FPS.
- **Unexplained:** JPEG encoding was 0.4 ms slower in B in both pairs. It is
  small and I don't know why.
- **Correctness:** every B part carried frame data that parsed with frame size
  1280×720 (2,138 of 2,138 and 2,140 of 2,140 parts), and no A part carried any.
  Livestream 20.0 FPS and pipeline 29.4–29.5 FPS in all four runs.

I also checked the page in a real browser engine, in a separate run without
timing. Headless Firefox on the real stream page showed the canvas in place of
the image with the full overlay, and **Hide Telemetry** left only the viewer
layer. At a device pixel ratio of 2, the canvas was 1342×755 pixels for a box
671 CSS pixels wide. Side by side with the Python renderer on the same frame
and data, the text, panels and shapes matched; canvas lines anti-alias slightly
thinner. Tests check that the JavaScript renderer draws the same text, and the
same shapes in the same places, as the Python one, with and without telemetry.
Recordings and renders still use the Python renderer, so the two have to be
changed together.

### Follow-up: readable on an iPhone

On 2026-11-19 the browser-drawn livestream didn't work for me. In the 17:33
session I saw nothing in it, although the stream ran at 30 FPS, the encode
worker never restarted, the recording was normal and a browser reached the
page; which device I was looking on wasn't recorded. Later, the iPhone app I'm
building showed "Video connection interrupted".

The page and the app read `/stream.mjpg?data=1` with `fetch()`. Several other
projects report that WebKit, which every iPhone browser and web view uses, fails
a `fetch()` of `multipart/x-mixed-replace` and only lets an `<img>` show that
type. I did not reproduce this, and I haven't confirmed it caused the 17:33
blank.

The fix (private application repository, commit `0386e9e`) sends
`/stream.mjpg?data=1` as `application/octet-stream`, with the same bytes. The
page and the app split the parts themselves, by each part's headers and
`Content-Length`, so they never needed the type. `/stream.mjpg` without
`?data=1`, which an `<img>` shows, keeps `multipart/x-mixed-replace`. A test
checks the new type. With a stand-in drone sending it, the app's browser
preview (Chromium) drew the overlay. It has not been tried on an iPhone. If the
app can't read the data stream, it shows the plain video, so an iPhone with an
older drone still gets video, without overlays.

## 2. Shrinking frames bilinearly

- **Classification:** controlled replay: two A B B A blocks and one control
  block, 4 A, 8 B and 2 C runs, all valid. Two earlier A runs failed to start
  and are kept.
- **Status:** measured on the bench. It was in the running code during the
  2026-11-19 flights; its CPU effect on the camera and its picture on a phone
  were not checked.
- **Question:** the [CPU profile](where-the-cpu-goes.md) put the worker at
  about 0.7 of a core, mostly the 1080p-to-720p shrink (8.7 ms per frame, area
  averaging) and the JPEG encode (9.1 ms). On one thread pinned to one core at
  1728 MHz, shrinking a synthetic 1080p frame took 20.9 ms by area averaging
  (`cv2.INTER_AREA`) and 10.5 ms bilinear (`cv2.INTER_LINEAR`), with a mean
  pixel difference of 0.35 of 255. Does bilinear save CPU in the real pipeline?
- **Baseline A:** area averaging.
- **Variant B:** bilinear. Recordings keep area averaging. Private application
  repository, commit `ab84511`.
- **Control C:** the B code, imported from a copied source tree the way the A
  runs imported theirs, to rule out the copy itself.
- **Only intended change:** one constant in the encode worker.
- **Primary metric:** the worker's CPU, in percent of one core.
- **Correctness criteria:** a test checks that the livestream shrinks
  bilinearly with its overlay coordinates scaled to match, and that recordings
  keep area averaging (full suite: 2,056 passed); every replay valid, with no
  dropped frames.

### Workload

| Field | Value |
| --- | --- |
| Input | 2-minute 1920×1080 H.264 flight recording; frames 300–1,499 measured, replayed at 30 FPS |
| Input SHA-256 | `e93036e39c708074d10bea26c52e7a7da05dc96c234f0e38e71cd3cd2e136800` |
| Work enabled | Default profile with motion output off: pose detector, tracker with ReID, person lock, hands on TensorRT, 1080p x264 recording inside the main process, and the livestream worker with browser-drawn overlays (it shrinks, serializes the frame data and encodes). No viewer was connected; the worker still worked through the livestream frames, as its CPU shows. |
| CPU measurement | `/proc/<pid>/stat` every second for the main process and the worker, plus each main-process thread in the last two blocks. Window: the seconds the worker was more than 15% busy, minus the first and last (48–50 s per run). No profiler. |
| Clocks and power | tegrastats every second in the last block |
| Code | Commit `8af0997` plus uncommitted flight-command and tilt-calibration edits, which are not on the video path |
| Platform | Jetson Orin Nano 8 GB, JetPack 7.2.1; CPU governor `schedutil`, clocks not locked; power mode not recorded; junction 57–60 °C mean |

### Results

| Block | Runs in order | Worker (% of a core) | Main process (% of a core) | Pipeline FPS |
| --- | --- | --- | --- | --- |
| First (its A runs failed to start) | B1 / B2 | 64.7 / 65.1 | 195.5 / 197.7 | 29.34 / 29.33 |
| A/B | A1 / B3 / B4 / A2 | 82.1 / **62.7** / **62.5** / 85.0 | 168.1 / 190.1 / 191.8 / 173.3 | 29.20 / 29.36 / 29.33 / 29.22 |
| Control | C1 / B5 / B6 / C2 | 60.8 / 59.7 / 60.2 / 61.1 | 187.0 / 185.9 / 187.0 / 187.0 | 29.37 / 29.35 / 29.35 / 29.34 |
| Clocks | A3 / B7 / B8 / A4 | 80.5 / **60.7** / **60.7** / 81.4 | 167.4 / 186.6 / 187.7 / 168.2 | 29.28 / 29.31 / 29.33 / 29.26 |

The worker's per-second readings had a standard deviation of 2–6 points within
a run.

The worker fell from 80.5–85.0% to 59.7–65.1% of a core, in every pair: about
21 points. That is what the single-core benchmark predicts: 10.4 ms less per
frame at 20 FPS is 0.21 of a core. The control matched B, so the copied source
tree did not change anything.

Pipeline FPS was 0.03–0.16 FPS higher in the B runs than in the A runs of the
same block, every time. The difference is small and the replay is paced at
30 FPS, so I don't read much into it.

### Why CPU percent hid the saving

The main process's CPU rose by about as much as the worker's fell: in the two
A/B blocks, from 167–173% to 187–192% of a core. Added together, the stream
used the same CPU time in A and B there: 248–258% against 247–254% of a core.
And it did the same work. Twelve of the 14 recordings were byte-identical after
their first 276 bytes, which hold the file's creation times, and those twelve
include A, B and C runs. The other two (one A, one B) are 28 bytes longer; I
didn't look into why. In the clocks block, every main-process thread was about
10–15% busier in B: the x264 encoder threads 82–83% → 94–95% of a core, the
unnamed threads 47% → 52%, and the frame queue thread 15% → 16–17%.

What changed was the clock. The CPU governor (`schedutil`) sets the clock from
the load. With less load it ran the cores slower, so the main process's
unchanged work took more CPU time. tegrastats in the clocks block:

| Run | Load summed over cores (%) | Mean core clock (MHz) | Busy × clock (GHz) | Board power, `VDD_IN` (W) | CPU/GPU/CV rail (W) |
| --- | ---: | ---: | ---: | ---: | ---: |
| A3 | 301 | 1450 | 4.40 | 13.45 | 6.10 |
| B7 | 304 | 1247 | **3.82** | **13.21** | 5.83 |
| B8 | 304 | 1245 | **3.82** | **13.22** | 5.85 |
| A4 | 303 | 1438 | 4.38 | 13.50 | 6.12 |

Each row averages 48–49 one-second samples. Busy × clock fell 13%, board power
about 0.25 W (1.9%), and the CPU/GPU/CV rail 4%, with the same inference work
on the GPU.
Measured by CPU percent alone, this change would have looked like it did
nothing. Since then I log clocks and power next to CPU time when I compare CPU
on this board.

I haven't compared the two livestreams by eye. Bilinear at a 1.5× shrink can
shimmer slightly on fine patterns, and the 0.35/255 mean difference was measured
on a synthetic frame, not camera footage. Recordings are not affected.

## Used live

The stream logs and the repository history for 2026-11-19 show:

- Both changes were in the running code in every session that day: the
  repository was at `ab84511` or a later commit, and the default profile turns
  browser-drawn overlays on. The drone was armed in five of the eight evening
  sessions, which include that evening's flights, so both ran on the drone in
  flight.
- Nothing measured the livestream's CPU on the camera, before or after either
  change.
- In three sessions a browser on the local network downloaded the overlay
  fonts, which only the browser-drawn page asks for, and dropped those
  downloads part way (broken pipes in the log). The logs don't show whether
  video or overlays appeared.
- The iPhone fix was in the code from 19:42, for the last two sessions. Nothing
  in the logs shows a phone reading the stream after that.

## Data and method

Per-run derived data:
[browser-overlay-ab.json](../data/2026-11-livestream-cpu/browser-overlay-ab.json)
and [bilinear-resize-ab.json](../data/2026-11-livestream-cpu/bilinear-resize-ab.json).
Raw logs, telemetry and recordings stay local.

- Worker step and JPEG times are the worker's own timers, reported every 2 s
  and averaged weighted by frame count.
- Worker and main-process CPU are differences in `utime + stime` over the
  window, divided by the window length.
- Busy × clock is the sum over cores of load ÷ 100 × clock, averaged over the
  window's one-second tegrastats samples. tegrastats reports each core's load
  over the last second and its clock at the sample, so this estimates CPU
  cycles per second rather than measuring them.
- In the overlay runs, tegrastats times are whole seconds and were lined up
  with the viewer's clock to within about 1 s.

## Limits

- Replay, not the camera. Each experiment used one clip, and the two used
  different clips and hand backends, so their numbers don't add into one
  before-and-after.
- Two runs per variant for the overlay; four A against eight B for the shrink.
- Nothing measured the viewing device, where the decoding and drawing now
  happen.
- With no viewer connected in the shrink runs, livestream frames were not
  counted. I expect the same count in both variants, since the worker had time
  to spare in both, but it is not verified.
- Clocks were not locked and the power mode was not recorded. Busy × clock
  varied a lot from second to second (standard deviation 0.6–0.9 GHz), but the
  run means agree within each variant.
- CPU coverage: the shrink runs measured the main process and the worker; the
  overlay runs measured the main process and the whole machine. The hand
  worker is only in the whole-machine numbers.
- "In the running code" comes from the repository history. Uncommitted edits
  in the working tree at the time can't be ruled out.

**Decision:** kept both. Since 2026-11-17 the default profile draws livestream
overlays in the browser, and since 2026-11-18 the livestream shrinks frames
bilinearly. The iPhone fix is in, untested on an iPhone.
