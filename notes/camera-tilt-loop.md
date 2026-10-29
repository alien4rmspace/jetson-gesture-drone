# Camera tilt loop: shorter delay and a Smith predictor

Dates: 2026-10-16 to 2026-10-28 (PDT)

The camera's tilt servo keeps the tracked person vertically centred. The
controller decides from the person box in each frame, and a tilt move only shows
up in a later frame. Throughout that delay the controller still sees the old
error and keeps correcting it. That makes faster settings overshoot and hunt.
Earlier, at 40 deg/s with a narrow slow-down zone, the tilt cycled 20–30 degrees
every 2.5 s with the drone on the ground. The application then ran a cautious 20
deg/s with a wide slow-down zone, which started moving slowly and let the person
drift toward the frame edge.

This note covers two changes:

1. The delay itself fell after the [pose model became the person
   detector](pose-model-person-detector.md).
2. A Smith predictor let the tilt run twice as fast without hunting.

## 1. Delay through the pipeline

- **Classification:** observational. These are separate flights. The detector,
  tracker settings, tilt settings, place, light and the person's movement all
  differ, so the comparison does not isolate one cause.
- **Source:** the application's tilt timing log. It records a per-frame timestamp
  when the frame enters DeepStream's `nvstreammux` (mux), when detection and
  tracking metadata is ready, and when the tilt controller decides. It also
  records every tilt write. Derived per-session values are in
  [`data/2026-10-tilt-loop/sessions.json`](../data/2026-10-tilt-loop/sessions.json).
- **Loop delay** is the lag that best aligns tilt writes with the opposite motion
  of the person box, measured at the controller's decision time. It assumes the
  person is roughly still while the camera tilts. The analyzer counts an estimate
  as confident at r ≤ −0.5 with at least 40 moving pairs.

| Session (start) | Detector | Processed FPS | Mux → metadata p50 / p99 (ms) | Metadata → decision p50 (ms) | Mux → decision p50 / p99 (ms) | Loop delay (r) |
| --- | --- | ---: | ---: | ---: | ---: | --- |
| 2026-10-16 13:18 | Segmentation + pose pass | 23.4 | 161.5 / 195.2 | 40.0 | 202.1 / 240.9 | 0.63 s (−0.33, below the bar) |
| 2026-10-27 19:54 | Pose model | 30.0 | 20.8 / 40.3 | 13.8 | 35.0 / 62.5 | 0.27 s (−0.60) |
| 2026-10-28 15:22 | Pose model | 30.0 | 25.8 / 37.7 | 15.7 | 41.8 / 58.8 | 0.26 s (−0.51) |
| 2026-10-28 16:00 | Pose model | 30.0 | 27.1 / 48.2 | 15.9 | 43.3 / 71.9 | 0.25 s (−0.51) |
| 2026-10-28 16:04 | Pose model | 30.0 | 25.7 / 37.7 | 15.3 | 41.3 / 59.1 | 0.24 s (−0.47, just below the bar) |

On 2026-10-16 the pipeline processed 23.4 frames per second from a 30 FPS
camera, so frames waited inside it: 162 ms between the muxer and metadata is
about 3.8 frame intervals. With one pose pass per frame, the pipeline keeps up
with the camera and frames no longer queue. Metadata is ready 21–27 ms after the
muxer, and the controller decides 35–43 ms after it (p50). The full loop, from a
tilt write to the box moving the other way, measured 0.24–0.27 s on all four
later sessions. That was 0.63 s on 2026-10-16, at low confidence, and an earlier
recording-based estimate gave about 0.45 s.

Limits: mux timestamps exclude sensor exposure and camera buffering before the
muxer. The 2026-10-16 delay estimate is below the analyzer's confidence bar. The
sessions differ in many ways besides the detector.

## 2. Smith predictor

The controller keeps its tracking tilt angle (without the separate pitch
stabilization offset) for the last moments. For each frame it subtracts the tilt
it has already commanded but the frame cannot show yet:

```text
predicted error = error − tilt moved within the delay × camera degrees per servo degree / vertical field of view
```

The vertical field of view comes from a zoom-to-field-of-view table and the frame
shape. The predictor's delay setting is the measured 0.27 s. The settings went
from 20 deg/s with a 0.30 slow zone to 40 deg/s with a 0.10 slow zone.

### Simulation

A closed-loop simulation in the application drives the real tilt controller and
motion profile against a camera that shows each servo move after a set delay. It
compares settings; it does not predict exact flight numbers. Servo scale is 0.69
camera degrees per servo degree, as measured on the bench, and values are the
worst of three noise seeds.

| Case | 20 deg/s, slow zone 0.30, no predictor | 40 deg/s, slow zone 0.10, no predictor | 40 deg/s, slow zone 0.10, predictor |
| --- | --- | --- | --- |
| Still person, delay 0.45–0.65 s, view 30–40° | settles | **hunts**: 13–19 direction changes | settles, 0 direction changes |
| Still person 0.30 of the frame off centre, true delay 0.27 s: time to within 0.10 | 0.77–1.45 s | 0.60–0.77 s, overshoot up to 0.15 of the frame | **0.64–0.85 s**, no overshoot |
| Person drifting down 4°/s for 3 s, true delay 0.27 s: furthest off centre | 0.16–0.17 | 0.11–0.13 | 0.12–0.14 |

The predictor stayed free of hunting with a true delay 0.2 s longer than
assumed, a field of view 20–25% off, and a true servo scale of 1.0 against an
assumed 0.69. With that servo scale, the old 20 deg/s settings themselves hunted
at a narrow view and long delay. A predictor delay set too long (0.45 s against a
true 0.27 s) made the tilt sluggish at wider views: 1.2–1.3 s to re-centre at
40–52 degrees of vertical view.

### Flights

- **Classification:** observational. The baseline is one flight on 2026-10-16
  with the old settings and the segmentation detector; the others use the
  predictor and the pose model. Values come from `drone_ai.camera.tilt_framing`
  on the same tilt timing log.

| Session | Tilt settings | Person more than 0.15 of the frame off centre | Edge contact, boxes up to 0.8 of the frame | Quick tilt reversals |
| --- | --- | ---: | ---: | ---: |
| 2026-10-16 13:18 | 20 deg/s, slow 0.30, no predictor | 5.3% | 3.1% | 0.4/min |
| 2026-10-27 19:54 | 40 deg/s, slow 0.10, predictor delay 0.45 s | 1.3% | 0.6% | 0.2/min |
| 2026-10-28 15:22 | predictor delay 0.27 s | 2.7% | 1.3% | 1.3/min |
| 2026-10-28 16:00 | predictor delay 0.27 s | 1.5% | 0.7% | 1.1/min |
| 2026-10-28 16:04 | predictor delay 0.27 s | 3.4% | 1.0% | 1.9/min |

Edge contact counts a box within 1% of the top or bottom of the frame. Boxes
taller than 0.8 of the frame are left out, because they touch an edge whatever
the tilt does. A quick reversal is a change of tilt direction within 1 s of the
previous move.

Quick reversals rose on 2026-10-28, but they were reactions, not hunting. 31 of
the 36 followed a drone pitch change of at least 2 degrees within 1.5 s, and all
36 followed the person box moving. The only short back-and-forth runs (2–3 s)
came while the drone rocked −10 to +5 degrees in pitch under stick input, and
while raised arms moved the box centre during gestures.

**Decision:** kept. Since 2026-10-28 the application's default profile runs 40
deg/s, a 0.10 slow zone and the predictor at 0.27 s.

## Limits

- The flight comparisons are observational, against one baseline flight in a
  different place and with a different detector.
- The simulation models the servo and camera delay simply; it ranks settings.
- Pitch stabilization of the camera shares the servo. In simulation it was
  unchanged at the bench-measured 0.11 s servo delay and somewhat worse at 0.2 s.
- Tilt writes wait while the zoom lens moves: the camera's controller takes one
  command at a time. A later change gives tilt the controller between zoom steps;
  it is not covered here.
