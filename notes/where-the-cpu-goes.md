# Where the stream's CPU goes

Date: 2026-11-18 (PST)

- **Classification:** exploratory. One profiled replay, no A/B. It shows where
  the CPU goes, not how much faster any rewrite would be.
- **Status:** analyzed. It led to the bilinear livestream shrink in the
  [livestream CPU note](livestream-cpu.md); nothing was rewritten.
- **Question:** which parts of the stream would gain from C++, for example a
  pybind11 module? The stream had just changed a lot: hands on TensorRT, a
  shared RGB frame ring, a faster sharpness score and browser-drawn livestream
  overlays. Until the day before, the hand worker ran MediaPipe's models on the
  CPU; it now runs them on TensorRT
  ([hand tracking note](hand-tracking-tensorrt.md)).
- **Code:** private application repository, commit `8af0997`, plus uncommitted
  flight-command and tilt-calibration edits that are not on the video path.
- **Primary measure:** CPU per process and per thread, in cores out of 6.

## Workload and capture

| Field | Value |
| --- | --- |
| Input | 2-minute 1920×1080 H.264 flight recording, the same clip as the bilinear A/B; frames 300–1,499 measured at 30 FPS, 0 dropped |
| Input SHA-256 | `e93036e39c708074d10bea26c52e7a7da05dc96c234f0e38e71cd3cd2e136800` |
| Work enabled | Default profile with motion output off: pose detector, tracker with ReID, person lock, hands on TensorRT, 1080p x264 recording in the main process, livestream 1280×720 at 20 FPS with browser-drawn overlays and the area-averaging shrink of the time. No livestream viewer was connected. |
| Profiler | Nsight Systems 2025.6.3: CUDA, NVTX and OS runtime tracing; CPU sampling and context switches for the process tree; Python sampling at 100 Hz; capture on the replay's measurement range, 40.8 s. No GPU metrics, which need root. |
| Coverage | CPU samples and scheduling events came back for the main process only (48 threads). NVTX ranges came back for the main process and both workers. Worker CPU comes from `ps` every 2 s, differenced over 10 s. The whole machine comes from tegrastats every second. |
| Platform | Jetson Orin Nano 8 GB, JetPack 7.2.1; CPU governor `schedutil`, clocks not locked (1632 MHz mean); power mode not recorded; junction 54–57 °C |

## CPU by process

| Process | Cores | Source |
| --- | ---: | --- |
| Main stream process: DeepStream pipeline, Python, 1080p recording, and the profiler's in-process cost | 1.94–2.00 | `ps`, four 10-s chunks; 2.00 from Nsight scheduling over the capture |
| Livestream encode worker | 0.71–0.72 | `ps`, four 10-s chunks |
| TensorRT hand worker | 0.19–0.26 | `ps`, four 10-s chunks |
| **Sum** | **about 2.9** | |
| Whole machine | 3.15 (2.82–3.53 per second) | tegrastats, 41 samples |

The quarter of a core between the sum and the machine is everything else: the
profiler's own processes, tegrastats, the `ps` loop, other services and the
kernel.

## Inside the main process

Scheduled CPU time per group of threads over the 40.8 s capture:

| Threads | Cores | What runs there |
| --- | ---: | --- |
| x264 encoder threads (9) | 0.74 | The 1080p recording's software H.264 encoder. Every x264 sample fell in these threads. |
| Threads without their own name (18) | 0.69 | DeepStream metadata probes (pose tensor capture and decode), TensorRT and CUDA launches, OpenCV, the recorder's sharpness score, Python |
| Frame callback thread | 0.17 | RGB frame copy, CPU frame work, person lock |
| Recording queue thread | 0.12 | Feeding the recorder |
| VPI, stream mux, decoder and other threads | 0.15 | |
| Nsight Systems threads | 0.13 | The profiler |
| **Total** | **2.00** | |

And which code, by the module on top of the stack, over 61,918 CPU samples of
the main process:

| Code | Share of samples |
| --- | ---: |
| x264 | 42.9% |
| Python interpreter | 8.7% |
| CUDA driver | 8.6% |
| C library (memory copies and the like) | 8.4% |
| Nsight injection and CUPTI (profiler) | 12.1% |
| TensorRT | 4.9% |
| OpenCV | 4.5% |
| Everything else | 9.9% |

Python bytecode is 8.7% of the process's samples, about 0.17 of a core. The
samples and the scheduled time don't agree exactly (x264 is 42.9% of the samples
but 37% of the scheduled time), so read it as roughly 0.15–0.2 of a core.

## Time per frame

Elapsed milliseconds, from completed NVTX ranges inside the capture. Indented
ranges run inside the range above them and must not be added to it. These are
elapsed times, including waits, not CPU time.

| Range | Process | N | Mean | p50 | p95 |
| --- | --- | ---: | ---: | ---: | ---: |
| Frame callback | main | 1,200 | 5.14 | 4.94 | 6.58 |
| &nbsp;&nbsp;RGB frame copy | main | 1,200 | 2.41 | 2.31 | 3.03 |
| &nbsp;&nbsp;CPU frame work | main | 1,200 | 1.31 | 1.22 | 2.08 |
| &nbsp;&nbsp;Person lock | main | 1,200 | 0.82 | 0.78 | 1.08 |
| Pose tensor capture | main | 1,200 | 1.27 | 1.14 | 1.82 |
| &nbsp;&nbsp;Tensor copy | main | 1,200 | 0.42 | 0.37 | 0.54 |
| &nbsp;&nbsp;Pose decode | main | 1,200 | 0.55 | 0.49 | 0.73 |
| Object metadata | main | 1,200 | 0.79 | 0.75 | 1.01 |
| Recording sharpness score | main, recorder thread | 1,199 | 4.53 | 3.69 | 8.98 |
| Livestream shrink (area averaging) | livestream worker | 815 | 8.65 | 8.30 | 12.34 |
| Livestream JPEG encode | livestream worker | 815 | 9.13 | 8.70 | 11.60 |
| Livestream frame data | livestream worker | 815 | 0.46 | 0.43 | 0.67 |
| Hand worker call | hand worker | 620 | 17.43 | 17.48 | 24.05 |

The frame callback takes 5.1 ms of the 33 ms between frames at 30 FPS. The
livestream worker handled 815 frames in 40.8 s, 20.0 FPS. The hand worker call
includes its GPU work and waits.

## Would C++ help?

Not much at 30 FPS:

- The biggest users are already native code: x264 (the Orin Nano has no
  hardware video encoder), OpenCV's resize, libjpeg-turbo for the livestream,
  and TensorRT and CUDA launches. Wrapping them in C++ instead of Python calls
  would not make that work smaller.
- Python bytecode in the main process is about 0.17 of a core.
- The one hot path shaped for a C++ module is the pose tensor capture, which
  copies the detector's output and decodes the poses: 1.27 ms per frame at the
  mean, about 4% of a core at 30 FPS, and part of that is a GPU-to-CPU copy. My
  first reading added the copy and decode ranges to it and got 2.2 ms, but they
  are nested inside the capture range.
- The stream already runs at the camera's 30 FPS in flight (a median of 30.0
  FPS in each of the five 2026-11-19 sessions with the drone armed), so saved
  CPU buys headroom and power, not frame rate.

The cheaper changes the profile pointed to are not C++:

- Shrink the livestream bilinearly. Done: the worker went from 80–85% to
  60–65% of a core ([livestream CPU note](livestream-cpu.md#2-shrinking-frames-bilinearly)).
- Shrink on the VIC, the Jetson's hardware image converter, instead of the CPU.
  Not tried.
- Score sharpness less often than every frame (4.5 ms per frame). Not tried.
- Encode livestream JPEGs on the hardware JPEG encoder, which the May stream
  used ([field log](field-log.md)). Not tried.

## Profiler overhead

Nsight's own threads used 0.13 of a core in the main process, and its injection
libraries and CUPTI were on top of the stack in 12.1% of the process's samples,
partly in those same threads. The A runs of the bilinear experiment ran the same
video-path code on the same clip without Nsight:

| | This profile (Nsight) | Same code without Nsight |
| --- | ---: | ---: |
| Main process (cores) | 2.00 | 1.67–1.73 (4 runs) |
| Livestream worker (cores) | 0.71 | 0.80–0.85 (4 runs) |
| Whole machine, busy × clock (GHz) | 5.16 | 4.38–4.40 (2 runs) |
| Mean core clock (MHz) | 1632 | 1438–1450 (2 runs) |
| Board power, `VDD_IN` (W) | 13.65 | 13.45–13.50 (2 runs) |

Busy × clock is each core's tegrastats load times its clock, summed over the
six cores.

The profiled run did about 18% more CPU work across the machine. The governor
raised the clocks for it, and at the higher clock the worker used less CPU time
(livestream frames were not counted in the runs without Nsight). So the
absolute numbers above are inflated by the
profiler, the main process's most, and how the CPU splits between processes
depends on the clock. These are separate runs, so this is an observational
estimate, not a measurement of the overhead.

## Data and method

[cpu-profile.json](../data/2026-11-livestream-cpu/cpu-profile.json) has the
per-process chunks, the thread groups, the sample shares, the NVTX statistics
with p99, the tegrastats averages and the comparison runs. The Nsight report
and its SQLite export stay local.

- Thread CPU: schedule-in and schedule-out events matched per thread and CPU,
  clipped to the capture, divided by its length. All events paired.
- Sample shares: the module of the top stack frame of each CPU sample. My count
  is 61,918 samples; the original experiment report says 61,904, which
  changes no share.
- NVTX: completed ranges with `0 <= start < end <= capture`; NumPy linear
  percentiles. Nesting checked range by range on the same thread.
- Worker CPU from `ps`: lifetime CPU percent times elapsed time, differenced
  over 10 s. Rounding makes it good to about ±0.01 of a core.

## Limits

- One profiled run, so no run-to-run spread for the profile itself.
- Replay, not the camera: the decoder source differs from the camera path.
- Thread-level CPU for the main process only; the workers are per process from
  `ps`.
- No GPU utilization: GPU metrics need root, and traced CUDA activity is a
  different measure, so I don't report one.
- No Python function-level breakdown: the Python samples didn't export.
- The single-core shrink and JPEG benchmarks behind the follow-up (20.9 ms
  area, 10.5 ms bilinear, 7.9 ms JPEG at 720p) ran on a synthetic frame, and
  their repetition counts weren't recorded.
- No livestream viewer was connected, so sending JPEGs to a browser is not
  included.

**Decision:** no C++ rewrite. I made the cheapest change the profile pointed
to, the bilinear livestream shrink.
