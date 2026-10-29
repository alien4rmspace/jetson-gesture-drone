# Performance engineering

Measured performance work on the drone's vision pipeline, from the first CPU YOLO stream to the current DeepStream pipeline. The [README](../README.md) has the project overview. Each section links to a note with the full method, sample counts and limits.

Measurements come from one Jetson Orin Nano. Many are short operational samples rather than repeated controlled trials, and each note says which comparisons are observational. Stream FPS includes every enabled stage (camera, inference, hand and pose processing, overlays, encoding), and the camera's 30 FPS cap limits what a live stream FPS result can show.

## Measured results

| Problem | Change I made | Observed result |
| --- | --- | --- |
| CPU inference limited the YOLOv8n stream | Moved inference to PyTorch CUDA, then exported a TensorRT FP16 engine | **8.5 → 23.7 → 30.0 FPS** in the application stream; 30 FPS was the camera ceiling. |
| CPU color conversion added cost to the browser/hand path | Earlier optimization: produced BGR frames directly on the GPU, removing the CPU’s BGRx-to-BGR conversion | **3.10 → 1.61 ms** for CPU frame mapping/color preparation in short live samples. |
| Frame callbacks prepared compressed images for an asynchronous hand worker | Standardized the later pipeline on RGB, moved hand transfer to shared memory, and updated output encoders | Callback **p95: 16.66 → 13.25 ms**; main-process CPU across six cores **21.77% → 19.68%** in two 30-second captures. Worker CPU and total-system savings were not measured. |
| Copying GPU frames created a temporary CPU buffer and then a second NumPy copy | Copied directly into the final NumPy array, using a 2D CUDA copy for padded rows | Mean frame-copy time **6.17 → 3.16 ms**; callback **p95: 19.09 → 13.37 ms**; main-process CPU **28.06% → 23.78%** in a later pair of 30-second captures. |
| TensorRT builder settings had an unknown payoff | Built separate YOLO26s detector and pose plans at optimization levels 3 and 5, then benchmarked each | Level 5 gave **+8.9% detector** and **+6.9% pose** throughput in isolated 10-second `trtexec` runs. |
| CPU-only local Qwen responses were slow | Built `llama-cpp-python` with CUDA and offloaded supported layers | One command prompt fell from **14.3 → 4.52 s**; a conversational sample fell from **14–18 → 6.44 s**. |
| A segmentation pass plus a pose pass on every person limited throughput | Used the YOLO26s pose model as the only person detector: one pass gives boxes and keypoints | **38.6 → 58.7 FPS** on a fixed replay clip with input capped at 60 FPS; same people found, lock held 88.6% → 89.2%. One run each. |
| Frames queued before the camera-tilt controller saw them | Followed from the single-pass detector: the pipeline now keeps up with the 30 FPS camera | Frame-in to tilt decision **202 → 35–43 ms** (p50); camera tilt loop delay **0.63 → 0.24–0.27 s**. Separate flights, observational. |
| Faster camera tilt overshot and hunted because of the loop delay | Added a Smith predictor that subtracts tilt already commanded but not yet visible, then doubled the tilt speed | Person more than 0.15 of the frame off centre **5.3% → 1.3–3.4%** of frames over four flights, with no hunting. Observational. |

## RGB and shared-memory frame processing

While reviewing the frame pipeline with Nsight Systems, I noticed frames were being converted back and forth between RGB and BGR: RGB from DeepStream became BGR for CPU processing **to match OpenCV’s default color convention**, then returned to RGB for MediaPipe. I standardized CPU frame processing on RGB to eliminate the redundant conversions and replaced JPEG hand-frame transfer with raw RGB shared memory. This reduced the work needed to pass frames between processes by removing JPEG encoding and decoding.

The September 28 Nsight Systems comparison measured the synchronous frame callback and the separate MediaPipe recognition call before and after the RGB conversion. All latency values are milliseconds.

| Measurement | Before: BGR + JPEG — p50 / p95 / p99 (ms) | After: RGB + shared memory — p50 / p95 / p99 (ms) | Relative change — p50 / p95 / p99 |
| --- | ---: | ---: | ---: |
| Frame callback | 7.63 / 16.66 / 20.12 | **6.53 / 13.25 / 18.60** | −14.4% / −20.5% / −7.6% |
| Hand submission in the camera callback | 2.77 / 3.63 / 4.78 | **0.38 / 0.65 / 0.79** | −86.1% / −82.0% / −83.4% |
| MediaPipe recognition | 77.28 / 122.33 / 126.76 | 83.03 / 119.92 / 129.38 | +7.4% / −2.0% / +2.1% |

Relative change is `(after − before) / before × 100`, calculated from unrounded data. Negative latency changes mean faster; positive changes mean slower.

Average utilization over each approximately 30-second capture:

| Measurement | Before: BGR + JPEG | After: RGB + shared memory | Relative change | Scope |
| --- | ---: | ---: | ---: | --- |
| Main-process CPU utilization | **21.77%** | **19.68%** | −9.6% | Scheduled CPU time divided by capture time and six CPU cores. |

Main-process CPU decreased by **9.6% relative**. CPU scheduling coverage excludes the separate MediaPipe, recording, and browser workers, so this does not establish total application CPU savings.

The callback became shorter, while recognition latency showed no consistent improvement. Frame-callback latency excludes background completion and is not camera-to-browser latency.

The [full RGB comparison](rgb-shared-memory-comparison.md) includes sample counts, stage timings, measurement methods, and the derived JSON data. Scene activity was not held identical, and several pipeline components changed together.

### Visual benchmarking log

**Before — BGR conversion and JPEG hand-frame preparation**

Capture: `vision-20260928-123919.nsys-rep`.

![Nsight timeline before RGB standardization: an 11.924 ms Frame Processing range contains RGB-to-BGR conversion and JPEG hand-frame encoding.](../assets/nsight/vision-20260928-123919-bgr.png)

This frame's purple `Frame Processing` range spans **11.924 ms**. Within it,
`RGB to BGR Conversion` takes **4.745 ms**, and `Process CPU Frame` takes
**5.883 ms**, including **2.656 ms** of `Hand Frame Encoding`.

**After — RGB frames and shared-memory hand transfer**

Capture: `vision-20260928-133010.nsys-rep` (a later RGB run).

![Nsight timeline with RGB and shared memory: a 7.687 ms Frame Processing range contains RGB Frame Copy and a shorter hand-submission section.](../assets/nsight/vision-20260928-133010-rgb.png)

This frame's purple `Frame Processing` range spans **7.687 ms**. `RGB Frame Copy`
takes **4.261 ms**, and `Process CPU Frame` takes **2.788 ms**. Within the latter,
`Process Hand Frame` takes **537 µs (0.537 ms)**, including **391.5 µs (0.3915 ms)**
of `Hand Frame Snapshot` (timings supplied from the expanded Nsight ranges).
JPEG hand-frame encoding has been replaced by this snapshot/submission work;
preparation and recognition continue in the sender thread and MediaPipe process.

These screenshots show individual frames at different timeline zoom levels.
Their durations are examples, not averages or percentiles, and nested ranges
should not be added together. The table above compares captures `123919` and
`130901`; the after screenshot illustrates the RGB path in the later `133010`
capture and is not the source of that table's statistics.

## Direct-to-NumPy frame copy

After standardizing the pipeline on RGB, I noticed that the frame-copy helper still copied GPU pixels into a temporary CPU buffer, then copied them again into a NumPy array. I changed it to allocate the final NumPy array first and copy directly into its memory. Padded image rows use `cudaMemcpy2D` so the copy skips padding while preserving the pixels.

The September 28 captures at **18:27:31 (before)** and **18:57:12 (after)** both used RGB/shared-memory processing with the CPU frame path enabled. This is a later comparison with its own baseline.

| Measurement | Before: temporary buffer + NumPy copy — p50 / p95 / p99 (ms) | After: direct NumPy copy — p50 / p95 / p99 (ms) | Relative change — p50 / p95 / p99 |
| --- | ---: | ---: | ---: |
| Frame callback | 9.63 / 19.09 / 29.67 | **7.22 / 13.37 / 17.13** | −25.0% / −30.0% / −42.3% |
| RGB frame copy | 5.25 / 11.79 / 16.30 | **2.74 / 5.91 / 9.01** | −47.8% / −49.9% / −44.7% |

Relative change is `(after − before) / before × 100`, calculated from unrounded data. Negative latency changes mean faster; positive changes mean slower.

| Measurement | Before: temporary buffer + NumPy copy | After: direct NumPy copy | Relative change |
| --- | ---: | ---: | ---: |
| Average main-process CPU utilization across six cores | 28.06% | **23.78%** | −15.2% |
| Mean RGB frame-copy time (ms) | 6.17 | **3.16** | −48.8% |

Observed mean frame-copy time fell **48.8%**, and frame-callback p50/p95/p99 fell **25.0% / 30.0% / 42.3%**. Main-process CPU fell **4.28 percentage points**, or **15.2% relative**. The after capture recorded **900 `cudaMemcpy2D` calls**, confirming that the new path ran.

CPU measurements exclude the separate workers, and callback latency is not camera-to-browser latency. Scene and PTZ conditions were not controlled, so these runs do not isolate the change's causal effect. The [full direct-to-NumPy comparison](direct-numpy-copy-comparison.md) includes sample counts, methods, limitations, and derived data.

### Frame-copy visual benchmarking log

**Before — temporary CPU buffer followed by a NumPy copy**

Capture: `vision-20260928-182731-cpu-frames.nsys-rep`.

![Nsight timeline before the direct-to-NumPy change: RGB Frame Copy takes 3.970 ms, with cudaMemcpy visible in the CUDA API row.](../assets/nsight/vision-20260928-182731-temporary-copy.png)

This frame's `RGB Frame Copy` range spans **3.970 ms**, and the full `Frame Processing` callback spans **9.578 ms**. `Process CPU Frame` takes **3.152 ms**. The CUDA API row shows `cudaMemcpy`; the subsequent CPU-to-CPU NumPy copy is included in the frame-copy range but does not appear as a CUDA API call.

**After — copy directly into the final NumPy array**

Capture: `vision-20260928-185712-cpu-frames.nsys-rep`.

![Nsight timeline after the direct-to-NumPy change: RGB Frame Copy takes 3.129 ms, with cudaMemcpy2D visible in the CUDA API row.](../assets/nsight/vision-20260928-185712-direct-numpy.png)

This frame's `RGB Frame Copy` range spans **3.129 ms**, and the full `Frame Processing` callback spans **6.373 ms**. `Process CPU Frame` takes **2.497 ms**. The CUDA API row shows `cudaMemcpy2D`, which copies the padded source rows into the final NumPy array.

Both screenshots come from the same captures used in the table above: `182731` before and `185712` after. They illustrate individual frames at different timeline zoom levels, not averages or percentiles. Nested ranges should not be added together.

## TensorRT optimization level 3 vs 5

I created a benchmarking script that automatically builds and compares FP16-capable YOLO26s pose and detector TensorRT plans across builder optimization levels. On the same Orin Nano with TensorRT 10.16.2, the script built level 3 and level 5 plans, ran each through trtexec for 10 seconds after a 1-second warm-up, collected latency and throughput metrics, and selected the faster plans locally. Build time is a one-time cost; the other columns measure isolated engine inference.

| Model | Level | Throughput (qps) | Mean latency (ms) | GPU mean / p99 (ms) | Build time (s) |
| --- | ---: | ---: | ---: | ---: | ---: |
| Pose | 3 | 146.832 | 7.198 | 6.804 / 7.085 | 440.30 |
| Pose | **5** | **156.991** | **6.791** | **6.364 / 6.543** | 1,101.53 |
| Detector | 3 | 155.401 | 6.824 | 6.429 / 6.714 | 38.34 |
| Detector | **5** | **169.218** | **6.312** | **5.904 / 6.184** | 175.52 |

Level 5 increased throughput by 6.9% for pose (146.8 → 157.0 qps) while reducing mean latency by 0.407 ms (5.7%). For detection, throughput increased by 8.9% (155.4 → 169.2 qps) while mean latency decreased by 0.512 ms (7.5%). GPU mean inference time improved by 0.440 ms (6.5%) for pose and 0.525 ms (8.2%) for detection. These gains came at approximately 2.5× and 4.6× longer one-time build times, respectively. Both level 5 plans were selected by the comparison script. The [full benchmark note](tensorrt-builder-comparison.md) records the configuration and limits.

## Pose model as the person detector

The stream used to find people with YOLO26s segmentation, then ran a second DeepStream pass with YOLO26s pose on each person. Profiling put that at about 25 ms of GPU time per frame. The pose model's own output rows already hold each person's box and 17 keypoints, so I made it the only person detector. The stream now decodes the keypoints from the raw output tensor and matches them to each tracked detection.

| Measurement (fixed replay clip, input capped at 60 FPS) | Segmentation + pose pass | Pose model as detector |
| --- | ---: | ---: |
| Processed FPS | 38.6 | **58.7** (+52.1%) |
| Frames with a person / with a pose | 89.1% / 88.8% | 89.3% / 89.3% |
| Person lock held | 88.6% (6 breaks) | 89.2% (3 breaks) |

Keypoints for the same person in the same frame differed by a median 0.4% of the box height, and the derived gesture label agreed on 99.9% of frames. This was one run per variant; the raw reports were not kept, and distant people are untested. The [full note](pose-model-person-detector.md) lists the workload and limits.

## Camera tilt loop

The camera's tilt servo keeps the person vertically centred, but a tilt move only shows up in a later frame. With one detector pass, frames stopped queueing in the pipeline. In flights, a frame reached the tilt decision in 35–43 ms (p50) instead of 202 ms, and the whole loop delay measured 0.24–0.27 s instead of 0.63 s. The delay still makes a controller that keeps correcting what it sees overshoot and hunt. So I added a Smith predictor: each frame, the controller subtracts the tilt it has already commanded but the frame cannot show yet. In simulation, that stopped twice-as-fast settings from hunting, and in four flights the person was more than 0.15 of the frame off centre in 1.3–3.4% of frames, against 5.3% before. These are separate flights in different places, so the comparisons are observational. The [camera tilt loop note](camera-tilt-loop.md) has the per-session timings, the simulation and the limits.
