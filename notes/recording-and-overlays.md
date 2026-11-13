# Clean 1080p recording with a frame data file

Date: 2026-11-12 (PST)

- **Classification:** exploratory. Each configuration ran once on the same fixed
  replay input, so run-to-run variability is unknown, and drop counts are known
  to vary between runs.
- **Status:** measured on the bench with the replay input; not flown yet.
- **Question:** raising the recording from 720p to 1080p at 30 FPS, with the
  overlays drawn inside the recorder, made the recorder drop frames. If the
  recorder saves the clean frame plus a small per-frame data file, and overlays
  are drawn onto a copy afterwards, does it keep up?
- **Baseline A:** the recorder draws the overlay (presentation style with the
  telemetry layer) into each 1920×1080 frame, then encodes it with software
  x264 (4 threads, 12 Mbps). The Orin Nano has no hardware video encoder.
- **Variant B:** the recorder encodes the clean frame with the same encoder
  settings and writes `<video>.frames.jsonl` beside it: line N describes frame
  N, with the tracks, poses, hand results, command, zoom, pipeline FPS, image
  sharpness and capture times the overlays are drawn from. A render tool draws
  any overlay style onto a copy later.
- **Only intended change:** where the overlay is drawn (recorder overlays off,
  frame data file on).
- **Primary metric:** frames the recorder dropped.
- **Secondary metrics:** pipeline FPS, data file size.
- **Correctness criteria:** every recorded frame gets exactly one data line,
  nothing is drawn into the recording itself, and a render draws the overlay at
  the tracked person's position on the right frame. These are automated tests
  on a short synthetic clip, not a measured comparison against live drawing.

## Workload

| Field | Value |
| --- | --- |
| Input | Fixed replay recording, 1280×720 H.264, indoor flight with the pilot in view, replayed at 30 FPS |
| Input SHA-256 | `56e5d61423970ccd224002eb868326f590f2ad974e5ee2be719c6a2e108a93f2` |
| Window | 1,800 warm-up frames, then 1,400 measured for pipeline FPS; recorder drops counted from recorder start |
| Work enabled | Default stream profile with motion output off: pose detector, tracker with ReID, person lock, MediaPipe hands, livestream, recording |
| Recording | 1920×1080 at 30 FPS (the 720p input scaled up), x264 software encoder, recorder in its own process |
| Platform | Jetson Orin Nano 8 GB, JetPack 7.2.1, TensorRT 10.16.2, MAXN_SUPER power mode; clocks not locked, temperatures not recorded |
| Application code | Runs used the working tree that became private application repository commit `c032f44`; the default profile changed in commit `de917d1` |

## Results

| Configuration (1920×1080, 30 FPS) | Recorder drops | Pipeline FPS (warmed up) |
| --- | ---: | ---: |
| Overlay drawn in the recorder, 2 encoder threads, 6 Mbps | 47 | 29.27 |
| **A:** overlay drawn in the recorder, 4 threads, 12 Mbps | 67 | 29.16 |
| **B:** clean frame + frame data file, 4 threads, 12 Mbps | **2** (both at startup) | 29.39 |
| For reference: 720p, overlay drawn in the recorder | 5 | not recorded |

Drawing the presentation overlay with its telemetry layer costs about 15 ms per
1080p frame (one offline run). At 30 FPS the recorder has 33 ms per frame for
everything, including feeding the software encoder, on a CPU already about 80%
busy across six cores. With the drawing gone it kept up.

The data file is about 3.6 KB per frame, or 6.5 MB per minute, small next to
the 12 Mbps video (90 MB per minute). The render tool runs at 30–38 FPS on an
idle Jetson, about real time. It runs at lower priority, one render at a time,
and the recordings page refuses to start one while the stream is running.

## Hand results drawn on the frame they came from

MediaPipe runs on the CPU at about 9 results per second, so by the time a hand
result is drawn live, the frame under it is newer. On the bench recording, live
hand overlays were **182 ms** old at the median (301 ms at the 90th percentile).

The data file stores each hand result with the capture time of the frame it was
computed from. The render tool puts each result back on that frame and blends
the hand points between results. This moved drawn hands 2 px at the median and
up to 278 px when a hand moved fast. There is no ground truth for where a hand
was between two results, so the blended points are an estimate. The live
overlay can't do this, because the later result doesn't exist yet when the
frame is shown.

## Livestream rate limit

The livestream's rate limit skipped any frame that arrived within 1/fps of the
last one it sent. Camera frames don't arrive exactly evenly, so a frame slightly
early was skipped and the next one sent, and the stream delivered fewer frames
than set. A fixed schedule, where each send moves the next send time forward by
1/fps, fixed it:

| Setting | Old rate limit | Fixed schedule (1280×720) |
| ---: | ---: | ---: |
| 30 FPS | 19.9–22.1 FPS | not measured |
| 20 FPS | 14.5–14.8 FPS | **19.8–20.0 FPS** |

Delivered FPS was counted from the MJPEG stream in short samples. The livestream
is now 1280×720 at 20 FPS, resized in its own encode process, and recording
stays at 1080p 30 FPS.

## The overlays

The recorder and the livestream share one overlay renderer with two layers:

- **Viewer layer:** corner brackets around each person, coloured by tracking
  state (green locked, amber acquiring, red lost), plain-word tags, the target's
  17-point skeleton, a fading tag for the current command, and the speed and
  heading sent to the drone.
- **Debug telemetry layer:** panels outside the detector's centre square with
  tracking scores, the eye gate, gestures, camera tilt, zoom, focus, sharpness,
  velocity, heading and frame rates, plus a score chip under each detection
  naming its class, faint unconfirmed tracks, eye points, and the hand crop.

Text uses bundled Noto fonts (SIL Open Font License). FreeType text is slow,
about 0.3 ms per call, so rendered text is cached as masks; that cut the
telemetry overlay from 27 to 9.6 ms per 720p frame. The livestream draws both
layers by default. Recordings are clean, and the recordings page renders a
copy with telemetry, viewer layer only, or the earlier classic style.

No images are included here yet: the only footage with the new overlays is
indoors. Field clips will follow the next outdoor flight.

## Limits

- One run per configuration. Drop counts vary between runs, so 47 against 67
  (2 against 4 encoder threads) is not a meaningful difference.
- The replay clip is 720p scaled up to 1080p, so it has less fine detail than a
  real 1080p camera frame, and x264 works harder on real detail.
- Replay decodes a file instead of using the camera path, and nothing has been
  flown with this recording mode yet.
- The recorder's own CPU use was not measured separately from the whole board.
- Overlay drawing times are single offline runs.
- Clocks were not locked and temperatures were not recorded.

**Decision:** kept. Since 2026-11-12 the default profile records clean 1080p 30
FPS video with the frame data file, and the livestream is 720p 20 FPS.
