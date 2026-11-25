# Offline renders: faster, on the GPU, 60 fps and steadier

Date: 2026-11-19, one change committed 2026-11-21 (PST)

The drone records the clean 1080p camera frame plus a data file with one line per frame. A render
tool draws the overlays onto a copy afterwards ([clean 1080p recording](recording-and-overlays.md)).
On 2026-11-19 I made renders steadier, faster and optionally 60 fps, and checked one idea that
did not work. All of it is offline work on recorded files. Nothing here changes what the drone does
in flight, and nothing here has been judged by eye in a structured way.

This note reviews existing reports. I ran nothing new for it. Where a number has no saved report
behind it, I say so.

| Change | Classification | Result |
| --- | --- | --- |
| [Renders about twice as fast](#renders-about-twice-as-fast) | Controlled on a fixed input, five changes together, 3 runs each | 30 s span **54.3 → 32.5 s** first render, **22.2 s** re-render (1.7x, 2.4x) |
| [Frame shift on the GPU, hardware decoding in the recorded colours](#frame-shift-on-the-gpu-and-hardware-decoding-in-the-recorded-colours) | Planned as controlled; the timing runs never reported | Colour error **≈2.3 → 0.36** levels mean; frame shift **≈40 → ≈6 ms** of CPU; render time change **unknown** |
| [60 fps with in-between frames](#60-fps-with-in-between-frames-that-follow-the-motion) | Exploratory micro-benchmark, 5 runs each | **29.0 vs 23.6 dB** against a plain mix; GPU warps **191.5 → 138.8 ms** per in-between frame |
| [Smoothing the up-and-down jumps](#smoothing-the-pictures-up-and-down-jumps) | Exploratory, three clips, one pass each | Frames with a jump over 5 px **32.5% → 6.1%** on a rocking flight |
| [Hands follow their pose wrist](#rendered-hands-follow-their-pose-wrist) | Exploratory, one flight | 200–300 ms after a hand result: **12.3 → 4.5 px** median error |
| [Pixhawk-gyro stabilization](#negative-result-pixhawk-gyro-stabilization-does-not-work-for-renders) | Exploratory, two flights; negative | Frames with a jump over 5 px: 25.8% / 30.5% with the gyro against 4.9% / 13.1% with vision; not built |

## What a render does

For each frame of the chosen span:

1. **Decode** the recorded H.264 frame.
2. **Measure** its vertical motion with optical flow, leaving the people out (a separate pass,
   saved and reused since this work).
3. **Smooth** the path the picture follows, then **shift** each frame onto it inside a slightly
   enlarged view.
4. **Draw** the overlays from the frame data, moved with the picture.
5. Optionally make an **in-between frame** halfway to the next one (60 fps).
6. **Convert** to YUV and **encode** with x264 in software. The Orin Nano has no hardware video
   encoder; it does have a hardware decoder (NVDEC).

Renders run at lower priority, one at a time, and the recordings page refuses to start one while
the stream runs.

## Platform and evidence

| Field | Value |
| --- | --- |
| Board | Jetson Orin Nano 8 GB (Super developer kit), MAXN_SUPER power mode, 6 CPU cores |
| Software | L4T R39.2.1 (JetPack 7.2.1), CUDA 13.2, VPI 4.1.4, OpenCV 4.6.0 (TBB, no CUDA), Python 3.12.3 |
| Encoder | x264 ultrafast, 12 Mbps (18 Mbps at 60 fps), 4 threads |
| Clocks | schedutil governor, not locked. Mean CPU clock 1.1–1.7 GHz per run in the stage breakdown. GPU clock during runs unknown |
| Temperature | 48.6 → 55 °C over the speed A/B; 49.8 → 52.1 °C over the GPU-warp A/B, no throttling seen |
| Background | Stream off. Load checked before each run; runs overlapping other work were excluded and listed. The recordings service sat idle in the background during the GPU-warp runs |
| Timing method | Wall time of the render command, or `perf_counter` timers in a script. No profiler, so no profiler overhead beyond the timers |

| Result | Private evidence | Code identity | Runs |
| --- | --- | --- | --- |
| Speed | Experiment report `render-speed-20261119` | A: commit `9913ceb`; B: the working tree that became commit `33c15a2` | 3 per variant on a 900-frame span; 1 each on a whole recording |
| GPU shift and decoding | Report `render-gpu-decode-20261119` (plan only); commit `57c3ea4` message and the decoder's code notes | A: `e1fbdbe`; B: `57c3ea4` | No timing runs; component checks |
| 60 fps | Report `20261119-interp-gpu-warp` | Benchmark on `33c15a2` unmodified plus the report's scripts; feature in `e1fbdbe` | 5 per variant, 70 frames each |
| Smoothing | Change note and design doc in the private repository | `817da2c`, re-checked at `33c15a2` and `57c3ea4` | One deterministic pass on each of 3 clips |
| Hands | Commit message and change note | `9913ceb` | One flight; sample count not recorded |
| Gyro check | My working notes only; the scripts were not kept | Not built | Two flights |

Commits are in the private application repository. Derived numbers:
[`render-speed.json`](../data/2026-11-rendering/render-speed.json) and
[`interpolation.json`](../data/2026-11-rendering/interpolation.json).

## Renders about twice as fast

**Question:** a 30-second render with smoothing took 54 s, and a 5-minute recording 7.4 minutes.
Where does the time go, and can it get near real time?

### Where the time went

Before changing anything I timed each stage on a 900-frame span (code at commit `817da2c`, two runs
per mode in AB/BA order). The timing loop was a copy of the render loop with timers, using the same
functions, not the render function itself.

| Mode (1920×1080, 900 frames) | Run 1 | Run 2 |
| --- | ---: | ---: |
| Measure pass only | 50.3 fps | 50.1 fps |
| Render, telemetry overlay, encoding | 26.5 fps | 24.3 fps |
| Render, telemetry overlay, no encoding | 30.8 fps | 30.9 fps |
| Encoder only: RGB → GStreamer `videoconvert` → x264 | 31.7 fps | 33.1 fps |
| Encoder only: I420 from OpenCV → x264 | **111.6 fps** | **113.4 fps** |

Three findings:
- the writer's RGB-to-YUV step in GStreamer ran on one thread and capped encoding at about 32 fps,
  while x264 alone ran at 112;
- decode, shift and draw ran one after another on one thread, and the process used 2.3–3.6 of 6
  cores;
- the measure pass decoded the whole video a second time.

Per frame: decode 6.2 ms, frame shift 9–12.7 ms, telemetry drawing 14–16 ms (4.8 ms for the lighter
presentation style), write 3.5–3.9 ms.

### Change

Five changes together:
1. the writer converts RGB to I420 with OpenCV on its own thread, BT.709 above 576 lines and BT.601
   below, as GStreamer chooses;
2. a reader thread decodes ahead;
3. the measured motion is saved beside the recording and reused on the next render;
4. measuring turns frames grey before shrinking them and follows corners from frame to frame;
5. long renders split into up to three consecutive pieces in separate processes, joined without
   re-encoding. Each later piece first draws the 60 frames before it without writing them, so its
   frames match one long pass.

The live recorder is unchanged.

### Plan, written before any results

- **Primary metric:** wall time of the render command (telemetry style, smoothing on) on a 30 s
  span (900 frames) of a 2026-11-17 flight. First render (no saved motion) and re-render.
- **Run plan:** 3 runs per variant, order A Bf Br Br Bf A A Bf Br (f = first render, r = re-render),
  at nice 10, background checked before each run. Governor not pinned, as the tool is used. One run
  each on a whole 5-minute recording, order Bf Br A.
- **Input identity:** the report kept only a SHA-256 prefix, `8bb2d27282079c7f`. The file was later
  remuxed in place to move its MP4 index to the front, so it no longer hashes to that.

### Results

| Input | A: `9913ceb` | B first render | B re-render |
| --- | ---: | ---: | ---: |
| 30 s span, 900 frames | 53.6, 54.6, 54.8 s (mean **54.3**) | 28.3, 36.6, 32.5 s (mean **32.5**) | 20.9, 21.2, 24.4 s (mean **22.2**) |
| Whole 5-minute recording, 9,010 frames | 444.5 s | 250.1 s | 208.3 s |

On the span that is 1.7x for a first render and 2.4x for a re-render (wall time −40% and −59%). On
the whole recording it is 1.8x and 2.1x, one run each. Every B run was faster than every A run. B's
first renders spread from 28.3 to 36.6 s, about a quarter of their mean. The board warmed from 48.6
to 55 °C and the later runs were slower, which fits heat, but with clocks unpinned I can't separate
the two.

### Correctness, against criteria set before measuring

1. **Same frames out.** 900 and 9,010 frames, timestamps every 33.3 ms from 0 to 300.3 s. Pass.
2. **Pieces match one pass.** As written, "PSNR ≥ 40 dB between a 3-piece and a 1-piece render",
   it failed: 33.7 dB minimum, 36.3 dB median. Two lossy x264 encodes of the same frames differ that
   much on their own, so I had set the criterion wrong. After seeing that failure I replaced it
   with a check before encoding: 900 of 900 frames byte-identical.
3. **New measuring matches old.** Per-frame motion differed by 0.03–0.14 px at the median (pass),
   but by 1.1 / 1.9 / 3.2 px at p99 on three clips against a 2 px limit (fail on one), and the
   smoothed paths differed by up to 39 px against 3 px (fail). All of it sits on a few frames
   moving 40–80 px, smeared by the rolling shutter, where RANSAC picks a different answer from
   nearly the same corners. Turning frames grey first, otherwise the same arithmetic (0.007 px
   median), moved the path by 22 px on its own. The outcome check passed: frames with a jump over
   5 px went 6.1 → 5.7%, 2.7 → 3.5% and 0.6 → 0.7%, within 1 point. The single largest jump on
   the rocking clip rose from 37 to 76 px.
4. **Test suite:** 2,291 passed.

### Which change did what

These come from stage timings, not separate A/B runs. OpenCV's YUV conversion took about 11 ms of
CPU per frame against 30 ms in GStreamer. Measuring ran at 71 instead of 50 frames a second. Saved
motion loads in 0.1–0.5 s instead of a measure pass. With motion saved, 3 pieces took 20.7 s
against 27.0 s for one, one run each.

What is left is CPU. One render process uses about 111 ms of CPU per frame, and OpenCV's frame
shift alone takes 37–41 ms of it (8–10 ms of wall time across 6 threads). CPU summed across the
parallel piece processes was not measured.

## Frame shift on the GPU and hardware decoding in the recorded colours

Two changes, planned as one controlled A/B (report `render-gpu-decode-20261119`, baseline commit
`e1fbdbe`, the same span and run order as above). The plan and its criteria were written before any
results. **The timing runs never reported, so the effect on render time is unknown.** What exists are
component measurements and checks, recorded with commit `57c3ea4`.

### Frame shift on the GPU

The smoothing shift now runs as a VPI perspective warp on CUDA. It costs 5–6 ms of CPU per 1080p
frame (5–8 ms of wall time) against 37–41 ms of CPU for `cv2.warpAffine`, with the same output
within 4 levels (mean 0.06). The gain is CPU, not wall time per frame: OpenCV already spread the
warp over six threads. With three pieces running at once, CPU is the limit. VPI only offers a zero
border, which is never sampled, because the margin keeps every output pixel inside the recorded
frame. Without VPI, or with `--no-gpu`, it falls back to OpenCV.

### Decoding in the recorded colours

The recordings are H.264 labelled BT.709. OpenCV's decoder reads the YUV correctly but converts it
to RGB with the BT.601 matrix. Every render before this shifted colours: pure red (200, 40, 40) came
back as (185, 23, 41), and on footage the mean error was about 2.3 levels (p99 7–8).

Renders now decode on NVDEC through GStreamer, take the decoder's own NV12 output, and convert it
with the BT.709 matrix: within 0.36 levels of an exact conversion on average (p99 3). One trap: asking
`nvvidconv` for I420 instead re-converted the BT.709 YUV to BT.601 YUV (pure red Y 80, U 112 came out
Y 92, U 104) while still labelling it BT.709.

Hardware decoding did **not** save CPU for rendering, compared with OpenCV plus a colour fix. The
RGB frames cost 28–32 ms of CPU each, because the colour conversion runs on the CPU, against
22–25 ms for OpenCV with a BT.601-to-BT.709 correction (within 1.5 levels mean, p99 4–5). Two runs
each. The CPU cost of the old, uncorrected OpenCV decode was not measured. I kept the more faithful path. The
measure pass reads the decoder's Y plane directly: 11 ms of CPU per frame against 21 ms.

### Checks

- On the Jetson: decoded frames start on the frame asked for and run to the end; decoded colours
  match the recorded ones; a failing hardware decoder falls back to OpenCV.
- Smoothing outcome, frames with a jump over 5 px on the three clips: 6.3 / 2.9 / 0.7% against
  5.7 / 3.5 / 0.7% before, within the 1-point criterion.
- Full test suite on 2026-11-21: 2,337 passed.

The hypothesis was about 111 → 75 ms of CPU per frame and a third off render time. The shift saves
about 34 ms of CPU per frame, and decoding for rendering costs 3–10 ms more than corrected OpenCV
would. The net effect on a whole render is untested.

## 60 fps with in-between frames that follow the motion

**Problem:** fast moves look stuttery at 30 fps in renders. The frames are already evenly timed:
camera timestamps sit on a 33.3 ms grid, and their ±6 ms variation is stamping noise. Mixing two
neighbouring frames would only show two see-through copies of anything that moved.

**Change (commit `e1fbdbe`):** `--interpolate`, or the recordings page's **60 fps** box, off by
default. After each frame comes one made halfway to the next. OpenCV's DIS optical flow (MEDIUM
preset) runs both ways on half-size grey copies. Each frame is warped halfway along it at full
1080p, and the two are mixed 50/50. The in-between overlay is drawn from the frame data on either
side: boxes and skeletons halfway, hands re-timed, text from the earlier frame.

### Choosing the mix

Test: rebuild frame k+1 from frames k and k+2, and compare it with the real frame by PSNR. 70
triplets from 7 s (210 frames) of a 2026-11-19 flight, already smoothed, no overlays (input SHA-256
`c1196853b2845615e2637b3040804ec6a58c9f4cf6869347b13668a04a19051e`).

| Method | Mean PSNR | Median | Range |
| --- | ---: | ---: | ---: |
| Follow the motion, mix weighted by flow agreement (full size) | 28.66 dB | 28.49 dB | 17.47–38.74 dB |
| Follow the motion, mix weighted by flow agreement (half size) | 28.66 dB | 28.49 dB | 17.47–38.75 dB |
| **Follow the motion, plain 50/50 mix** | **29.00 dB** | **28.71 dB** | 18.15–38.84 dB |
| Plain mix, no motion | 23.56 dB | 23.32 dB | 18.03–36.18 dB |

The plain 50/50 mix beat agreement weighting on all 70 triplets, by 0.34 dB on average (0.01 to
1.87 dB). Following the motion beat a plain mix on 65 of 70 triplets, by 5.4 dB on average; on the
other 5 it lost by up to 1.0 dB.

That choice also made it cheaper. My CPU prototype took 438 ms of wall time per in-between frame
(1,058 ms of CPU, 20 triplets): flow 120.5 ms, agreement weighting 161.7 ms, a NumPy blend 90.5 ms,
image warps 41.4 ms and smaller steps. Dropping the weighting and the NumPy blend took it to about
191 ms.

### GPU warps

- **Baseline A:** the two full-size image warps with `cv2.remap`, flow maps upsampled to 1080p.
- **Variant B:** VPI `dynamic_remap` on CUDA, given the half-size maps; upload and readback counted.
- **Same in both:** the flow and the plain mix.
- **Runs:** 5 per variant, order A B B A A B B A A B, 70 in-between frames per run, 5 warm-up frames
  not counted. Exclusion rule set beforehand (another process over half a core, or a VPI error);
  nothing excluded.
- **Criteria set beforehand:** B within 0.2 dB of A, and A-to-B mean difference ≤ 1 level.

| Run | A-01 | B-01 | B-02 | A-02 | A-03 | B-03 | B-04 | A-04 | A-05 | B-05 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Mean ms per in-between frame | 190.5 | 138.8 | 140.3 | 190.8 | 187.8 | 140.3 | 137.2 | 193.8 | 194.5 | 137.4 |
| Process CPU ms per frame | 693 | 503 | 512 | 691 | 686 | 510 | 505 | 706 | 696 | 500 |

Mean of run means: **A 191.5 ms** (range 187.8–194.5), **B 138.8 ms** (range 137.2–140.3), −27.5%.
Every B run was faster than every A run. Process CPU fell 694 → 506 ms per in-between frame (−27%).
That CPU covers every thread of the one process; GPU time is not in it, and GPU utilization was not
measured. The target I set, B at most half of A, was not met: once the mix was plain, the warps were
only about a quarter of the time.

Quality: leave-one-out PSNR 29.00 dB in both. A and B frames differ by 0.07 levels on average, and by
8 levels at the 99.9th percentile of the worst frame, at moving edges where the two resample
slightly differently.

The flow is now about 120 of the 139 ms. I kept the MEDIUM preset and full-resolution warps on
purpose; a cheaper flow is the next lever if renders need to be faster. The Orin's optical flow
hardware ran dense flow through VPI in about 29 ms per 1080p pair, but every later call from Python
read back the first call's result, so I don't trust it yet.

### In a whole render

One run each, outside the A/B: a 7 s span took 38 s at 60 fps against 6.4 s at 30 fps in one
process. A whole 83 s flight took 277 s at 60 fps with three processes. One pass and two pieces
match to about 37 dB on every frame, the encoders' own difference, including the in-between frame
at the join.

### Limits

- The test frames are two frames apart, twice the motion a real in-between frame bridges, so the
  test is harder than real use. PSNR measures closeness to the real frame, not how motion looks.
- One 7 s clip from one flight.
- Motion blur stays, and a jump the camera made between two frames becomes two half-jumps.

## Smoothing the picture's up-and-down jumps

**Problem:** the camera's tilt servo is commanded in whole degrees and moves at full speed. Each
one-degree step jumped the picture about 15 px at 1080p (median of 171 isolated steps in the tilt
timing log, p90 27 px) within one or two frames, about 0.11 s after the write. Hard pitching adds
bigger jumps. In recordings it looked like jitter.

**Change (commit `817da2c`), in renders only:**
1. **Measure.** Lucas-Kanade optical flow on corners at 480 px wide, checked backwards, with
   everyone's box from the frame data left out. A shift-scale-rotation fitted with RANSAC gives the
   vertical motion at the frame centre, so people moving and zoom don't count. Frames with fewer
   than 12 agreeing corners stay as recorded.
2. **Smooth.** Of all paths within 6% of the frame height of the measured one, take the one that
   bends least: a Hodrick-Prescott filter with a margin, period 1.5 s. It is solved with accelerated
   projected gradient descent in NumPy, about 1 s for 5 minutes of video. The render can look ahead,
   so a ramp starts before the move does.
3. **Shift** each frame onto that path inside a view enlarged about 14%, so no edge shows. Boxes,
   skeletons and hands move with the picture; panels stay put.

A first version the same day blurred the path with a 0.15 s Gaussian inside a 4% margin. Moves
bigger than the margin leaked through it.

### Results

Vertical motion of the plain frames, measured with phase correlation, a different method from the
smoother's. One deterministic pass per clip. Each cell is recorded → first version → path solver.

| Clip | Frames | Frames with a jump over 5 px | p99 jump | Largest |
| --- | ---: | --- | --- | --- |
| 2026-11-19 indoor flight, ±25° pitch rocking | 2,070 | 32.5% → 9.3% → **6.1%** | 37.2 → 20.2 → 14.3 px | 103 → 117 → 37 px |
| 2026-11-19, 61 one-degree commands, out of focus | 900 | 20.1% → 4.9% → **2.7%** | 29.6 → 16.1 → 10.5 px | 58 → 41 → 27 px |
| 2026-11-17 flight, 58 one-degree commands | 900 | 16.7% → 0.4% → **0.6%** | 24.2 → 3.4 → 2.5 px | 127 → 46 → 43 px |

A "jump" here is the change in vertical motion from one frame to the next. On the rocking clip,
the path solver at the old 4% margin gave 6.5%, so most of the gain is the solver, not the larger
margin. The later measuring changes moved these to 5.7 / 3.5 / 0.7% (commit `33c15a2`) and
6.3 / 2.9 / 0.7% (commit `57c3ea4`).

The two measures disagree on what is left. The render's own optical flow finds 0.2% of frames over
5 px on the rocking clip. They agree on calm frames (correlation 0.974, median difference 0.5 px):
where the raw picture moves less than 15 px a frame, 88% of that clip, phase correlation finds
1.2% over 5 px. The rest is on the fast frames of hard pitch swings, smeared and skewed by the
rolling shutter. There phase correlation's match score drops from 0.89 to about 0.5, and the two
methods disagree by more than 5 px on 3.6% of frames, so neither is trustworthy there.

Limits: moves too fast for the margin still come through, as gently as the margin allows. Dark,
plain or very blurred frames can't be measured and stay as recorded. The 14% enlargement costs some
field of view and a little sharpness. Yaw, roll and rolling-shutter skew are not corrected. Before
the speed-up, smoothing doubled render time.

## Rendered hands follow their pose wrist

**Problem:** in renders of a 2026-11-19 flight, the drawn hands drifted off the person whenever the
picture moved. Hand results arrive about 14 times a second, with gaps. Between results the render
held the last one still for up to 0.3 s, or slid it in a straight line toward the next, while the
pose and boxes updated every frame. In one quick pitch the person moved 200 px down the frame in
0.2 s, and the hands ended up over the face.

**Change (commit `9913ceb`):** on each hand result's own frame, the render ties the hand to the
nearest confident pose wrist within one hand's size, else to its person's box. On the frames after,
the hand moves with that anchor.

**Check:** leave each hand result out, draw its frame from the other results, and measure the
distance to the result that was left out.

| Time since the last hand result | Before (median) | After (median) |
| --- | ---: | ---: |
| 100–200 ms | 8.7 px | **4.1 px** |
| 200–300 ms | 12.3 px (p90 104) | **4.5 px** (p90 11.5) |

Frames more than 15 px off fell from 13% to 5%. I watched a 7 s preview before committing, and the
hands looked better.

Limits: one flight, and the number of hand results in the test was not recorded. The analysis script
was not kept, so the numbers can't be re-checked from saved files. About 8% of the hand results on
that flight are false hands at the edge of the hand crop; this change doesn't remove them. The live
overlay is unchanged.

## Negative result: Pixhawk-gyro stabilization does not work for renders

**Question:** instead of measuring motion from the picture, could the flight controller's gyro drive
the smoothing (rotation-only stabilization, with some filter tuning)?

**Data:** the Pixhawk's own log had the IMU at 15 Hz and attitude at 6 Hz, too slow for 30 fps
video. The 50 Hz attitude messages the Jetson receives are saved in the tilt timing log, which was on
for the two flights below.

**Method:** a full 3D rotation model from the attitude and the servo commands (servo scale 0.92,
measured on the bench), turned into an image motion with the lens's field-of-view table. Fitted lags:
attitude 37.5–40 ms, servo 110–115 ms. I then simulated the smoother fed by each motion source and
scored the result with phase correlation.

| Flight | Frames | Recorded | Vision (optical flow) | Gyro + servo commands |
| --- | ---: | ---: | ---: | ---: |
| 2026-11-19 rocking clip, ±25° pitch | 2,070 | 32.5% | **4.9%** | 25.8% |
| Second 2026-11-19 flight, ±29° pitch, ±18° roll | 3,750 | 35.6% | **13.1%** | 30.5% |

Frames with a jump over 5 px. The gyro predicts the drone's own motion well: roll to image rotation
R² 0.78 and 0.90, slope 0.99 on the first flight. On first-flight frames without a servo step, its
vertical motion was off by 2.4 px rms where the motion itself was 7.3 px rms. It predicts the servo
steps badly: 8.9 px rms off where they moved 15.8 px rms.
And 89% of the jumps vision leaves behind sit at servo steps.

**Why:** the servo moves the camera relative to the airframe, and the Pixhawk's IMU is on the
airframe, so it never sees the step. Only the command is known, and isolated one-degree steps with
the drone still landed 105 / 126 / 164 ms (p10 / p50 / p90, 55 steps) after the write. That spread
is about two frames, so the command can't put the step on the right frame. The gyro filter settings
are flight-control filters tuned by AutoTune, not a render setting. Using the gyro only for the 20
frames vision could not measure gave 13.3% against 13.1%.

**Decision:** not built; renders keep the vision measure. The gyro could help as a fallback on dark
or featureless frames, or for roll. Real rolling-shutter correction would need fast IMU logging and
sub-millisecond time sync.

Limits: the "vision" column comes from this check's own simulation of the smoother. On the rocking
clip it gave 4.9% where the render's measured result is 6.1%, so compare within the table only.
These numbers are from my working notes; the scripts were not kept.

## Limits

- Several results are single passes or single runs: the whole-recording timings, the 60 fps
  whole-render timings, the smoothing clips and the hand check.
- Clocks were not locked. GPU clocks during runs are unknown. CPU summed across parallel piece
  processes was not measured.
- The GPU shift and hardware decoding have no end-to-end timing.
- Two criteria in the speed A/B failed as written. I replaced one after seeing the failure; the
  outcome check that replaced the other was set beforehand.
- The hand check and the gyro check have no saved analysis scripts.
- None of the visual results has been judged by eye in a structured way; the metrics measure
  motion and closeness to real frames.

**Decision:** kept. Since commit `57c3ea4` (2026-11-21), renders smooth the tilt jumps by default,
shift frames on the GPU, decode in the recorded colours, split long spans across up to three
processes, reuse saved motion, move held hands with their wrist, and offer 60 fps as an option.
