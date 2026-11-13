# MediaPipe 0.10.18 versus 1.2.3 for hand gestures

Date: 2026-11-12 (PST)

- **Classification:** controlled for the isolated test (identical inputs,
  alternating order, three pairs); observational for the end-to-end replay (one
  run per version).
- **Question:** the hand worker used MediaPipe 0.10.18 (November 2024), the last
  0.10 release with Linux aarch64 wheels. The 1.x releases ship them again. Does
  1.2.3 change the hand results, or make hand recognition faster?
- **Baseline A:** MediaPipe 0.10.18.
- **Variant B:** MediaPipe 1.2.3 (2026-11-11). The worker code and the gesture
  model file are the same; the model is unchanged since 2023.
- **Only intended change:** the MediaPipe package, along with the dependency
  versions it installs (NumPy and OpenCV among them).
- **Primary metric:** median time per recognition call.
- **Correctness criteria:** landmark distance between versions on the same
  crop, gesture label agreement, and the number of hands found.

## Workload

| Field | Value |
| --- | --- |
| Input | 900 hand crops cut from the fixed replay recording (SHA-256 `56e5d614…`) with the stream's own crop transforms |
| Worker options | Video mode, up to 2 hands, detection 0.25, presence and tracking 0.45, gesture score 0.35 |
| Order | A B A B A B, the first 30 calls of each run excluded |
| Device | MediaPipe on the CPU (its GPU delegate is not available from Python on this platform), otherwise idle board |
| Platform | Jetson Orin Nano 8 GB, JetPack 7.2.1, MAXN_SUPER power mode; clocks not locked |

## Results

| Pair | A: 0.10.18 median per call | B: 1.2.3 median per call | Difference |
| ---: | ---: | ---: | ---: |
| 1 | 156.1 ms | 132.0 ms | −24.1 ms |
| 2 | 151.2 ms | 141.3 ms | −9.9 ms |
| 3 | 158.2 ms | 152.6 ms | −5.6 ms |

1.2.3 was faster in every pair, but the gap shrank from 24 to 6 ms, so this does
not establish a speed difference.

The results agreed: landmarks differed by 0.22 px at the median (1.27 px at the
90th percentile, in crop pixels), gesture labels agreed on 99.4% of crops, and
the versions found 685 and 684 hands. Each version gave the same output on every
repeat.

End to end on the replay, one run each (A / B): 8.6 / 8.4 hand results per
second, hand lag 195 / 199 ms at the median (302 / 308 ms at the 90th
percentile), pipeline 29.06 / 29.13 FPS, recorder drops 4 / 4. No practical
difference.

## Limits

- On an idle board the CPU clocks drop, so the absolute times here are higher
  than inside the running stream. Compare within the table only.
- One replay per version.
- Only one recording's hands; other lighting, distances and hand poses are
  untested, although the model file is the same.

**Decision:** adopted 1.2.3 on 2026-11-12 (private application repository,
commit `beadb6e`). It is an upgrade for support, not for speed: the hand lag
comes from running the models on the CPU. A TensorRT port of MediaPipe's own
current hand models is the likely next step. An earlier TensorRT hand backend
(May 2026) used older 2020 model exports without world landmarks and recognised
closed fists less reliably, so it was set aside. The port has not been started.
