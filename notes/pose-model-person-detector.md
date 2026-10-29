# Pose model as the person detector

Date: 2026-10-27 (PDT)

- **Classification:** observational. Each variant ran once on the same fixed
  replay input, with no repeated runs, so run-to-run variability is unknown.
- **Status:** analyzed from the summary recorded at the time. The per-run report
  files were not kept, so only the summarized values below are available.
- **Question:** the stream ran two GPU passes per frame. A segmentation model found
  people, then a second pass ran a pose model on each person. Does one pass of the
  pose model, used as the detector, keep the same detections, poses and person
  lock while raising throughput?
- **Baseline A:** YOLO26s-seg as the primary detector, plus a secondary DeepStream
  YOLO26s-pose pass on each detected person.
- **Variant B:** YOLO26s-pose as the primary detector. Its output rows already
  hold each person's box and 17 keypoints. The stream decodes them from the raw
  output tensor and gives each tracked detection the pose whose box matches it.
  It uses the same pose TensorRT plan as before. There is no second pass and no
  instance masks.
- **Only intended change:** which model finds people (`person-detector`).
- **Primary metric:** processed frames per second on the replay input.
- **Correctness criteria:** the share of frames with a detected person and with
  a pose, how long the person lock held, keypoint agreement on the same person
  and frame, and agreement of the pose gesture label.

## Workload

| Field | Value |
| --- | --- |
| Input | Fixed replay recording, 1280×720 H.264, 3619 frames, indoor flight with the pilot in view |
| Input SHA-256 | `56e5d61423970ccd224002eb868326f590f2ad974e5ee2be719c6a2e108a93f2` |
| Window | Whole clip: 300 warm-up frames, then 3300 measured |
| Pacing | Replay input capped at 60 FPS (the live camera runs at 30) |
| Work enabled | Person detection, NvDCF tracker with ReID embeddings for the person lock, pose; recording off |
| Hand detection | Not recorded in the summary |
| Clocks, power mode, temperatures | Not recorded |
| Application code | Private application repository, commit `8951a53` |

## Results

| Measurement | A: segmentation + pose pass | B: pose model as detector | Change |
| --- | ---: | ---: | ---: |
| Processed FPS | 38.6 | **58.7** | +52.1% |
| Frames with a person | 89.1% | 89.3% | +0.2 pp |
| Frames with a pose | 88.8% | 89.3% | +0.5 pp |
| Person lock held | 88.6% (6 breaks) | 89.2% (3 breaks) | +0.6 pp |

Relative change is `(B − A) / A × 100`; pp means percentage points. On the same
person in the same frame, keypoints differed by a median 0.4% of the box height
(about 4 px) and 3.3% at the 99th percentile. The gesture label derived from the
pose agreed on 99.9% of frames.

58.7 FPS is close to the 60 FPS input cap, so the pose detector's own ceiling
may be higher than this run shows.

## Supporting measurements

These come from separate runs and only support the explanation; they are not
part of the comparison above.

- On a profiled run of the same input (Nsight Systems, 20.3 FPS under the
  profiler), the segmentation detector took 12.4 ms of GPU time per frame. The
  pose pass took 13.0 ms per frame, at 1.2 people per frame. Together that is
  about 25 ms of GPU time per frame, which fits the baseline's 38.6 FPS ceiling.
- In isolated `trtexec` runs at batch 1, the segmentation plan took 10.1 ms and
  one pose pass 8.8 ms.

With the pose model as the detector, each frame needs one pose inference in place
of a segmentation inference plus one pose inference per person.

## Limits

- One run per variant: no repetitions or measure of variability.
- The raw per-run reports were not kept; only the summary written at the time is.
- The clip has one person at a few metres. The pose model sees the whole
  detection square at 0.59 scale, while the old second pass saw each person in a
  crop scaled up to 640 px. Keypoints of small, distant people may be less
  precise; that is untested.
- The pose detector produces no instance masks, so mask-based features are off.
- Live flights run at the camera's 30 FPS, so this throughput gain does not show
  in flight FPS. The effect on in-flight delay is in the
  [camera tilt loop note](camera-tilt-loop.md).

**Decision:** kept. The pose model has been the person detector in the
application's default profile since 2026-10-27.
