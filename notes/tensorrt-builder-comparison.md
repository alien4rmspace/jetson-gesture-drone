# TensorRT builder optimization level 3 versus 5

Measured on 2026-09-22 with an NVIDIA Jetson Orin Nano and TensorRT 10.16.2. Both models were static batch-one YOLO26s ONNX exports built as FP16-capable TensorRT plans with a 3,071 MiB workspace setting. Each builder level used its own timing cache. `trtexec` loaded each plan and measured inference for at least 10 seconds after a 1,000 ms warm-up, with data transfers enabled. The same GPU and benchmark options were used for all four plans.

| Model | Builder level | Build time | Throughput | Mean latency | Mean GPU compute | P99 GPU compute |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Pose | 3 | 440.30 s | 146.832 qps | 7.198 ms | 6.804 ms | 7.085 ms |
| Pose | 5 | 1,101.53 s | 156.991 qps | 6.791 ms | 6.364 ms | 6.543 ms |
| Detector | 3 | 38.34 s | 155.401 qps | 6.824 ms | 6.429 ms | 6.714 ms |
| Detector | 5 | 175.52 s | 169.218 qps | 6.312 ms | 5.904 ms | 6.184 ms |

Level 5 improved throughput by **6.9% for pose** and **8.9% for detection** relative to level 3. Mean GPU compute time fell by about 6.5% and 8.2%, respectively. The one-time build cost increased about 2.5× for pose and 4.6× for detection. All four builds and benchmark runs completed successfully. The build logs contained a weakly typed network deprecation warning and an expected first-use timing-cache miss; no benchmark error was recorded.

The benchmark ran each engine alone. It does not include camera capture, DeepStream scheduling, MediaPipe, overlay work, browser encoding, or other concurrent GPU work. Results are a single run per plan, and actual instantaneous GPU clocks were not captured, so a repeated, alternating-order benchmark would improve confidence if a small difference mattered. The comparison script selected level 5 for both plans, with end-to-end stream validation treated separately. The [follow-up below](#follow-up-2026-11-12-the-pose-engine-the-stream-was-using) found that the pose engine the stream loaded was not the level 5 plan.

Representative commands used the following form:

```bash
trtexec --onnx=model.onnx --saveEngine=model_opt5.engine \
  --fp16 --builderOptimizationLevel=5 --memPoolSize=workspace:3071 \
  --timingCacheFile=timing-opt5.cache --skipInference

trtexec --loadEngine=model_opt5.engine --warmUp=1000 --duration=10
```

The original generated engines and logs remain local to the Jetson; they are device-specific binary artifacts and are not distributed here.

## Follow-up, 2026-11-12: the pose engine the stream was using

- **Classification:** controlled for the isolated benchmark (two alternating
  runs per engine); observational for the stream replay (one run per engine).
- **Finding:** the pose engine the stream actually loaded, built on 2026-09-22
  with its builder level not recorded, took 8.1 ms of GPU compute per
  inference. That is slower than both plans above, so the level 5 plan had not
  been deployed. How the slower build became the deployed file was not traced.
- **Change:** rebuilt the stream's pose engine as FP16 at level 5 with a
  2,048 MiB workspace (TensorRT 10.16.2.10, JetPack 7.2.1). The previous engine
  was kept for rollback.
- **Correctness criteria:** the same people, lock state and keypoints on the
  fixed replay recording, compared frame by frame against the previous engine.

`trtexec --loadEngine` with a 1,000 ms warm-up and 20 s duration, the two
engines run alternately, twice each:

| Pose engine | GPU compute (median) | Throughput |
| --- | ---: | ---: |
| Previous (2026-09-22, level not recorded) | 8.12–8.14 ms | 122 qps |
| **Rebuilt at level 5** | **6.53–6.55 ms** | **154–155 qps** |

On the fixed replay recording, aligned by source frame, the rebuilt engine gave
the same lock state on 100% of frames and the same people on 99.7%, with
keypoints 0.22 px apart at the median (0.36 px at the 90th percentile) on
1920×1080 frames. Two replays with the same engine are identical, so these
small differences come from the new engine's arithmetic, not from replay noise.

Whole stream with the default profile on the replay input, tegrastats every
0.5 s over 64 steady seconds, one run per engine:

| Measurement | Previous engine | Level 5 engine |
| --- | ---: | ---: |
| GPU load: mean (median, p90) | 43% (42%, 68%) | **34% (29%, 59%)** |
| CPU, six cores at 1,728 MHz | 77% | 81% |
| Board power | 13.7 W | 13.3 W |
| Pipeline FPS | 29.1 | 29.1 |

GPU load fell by about a fifth. FPS stayed the same because the replay is paced
at the camera's 30 FPS. The CPU, at about 80%, is the tighter resource in the
default run; the pose engine runs on the GPU, so the 4-point CPU difference is
most likely run-to-run variation. Limits: one replay per engine, the replay
decodes a file instead of using the camera path, and clocks were not locked.

**Decision:** kept. The stream has used the level 5 pose engine since
2026-11-12. This repository's earlier claim that the deployment used level 5
was based on the script's selection, not on the file the stream loaded.
