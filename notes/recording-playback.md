# Recordings that play at once in the browser

Date: 2026-11-20 (PST), committed 2026-11-21

- **Classification:** controlled for two desktop browser engines: the same recording in both
  layouts, served by the application's real recordings handler through an emulated link, three
  runs per layout and browser in A B B A A B order. Not tested on my phone, the PC's browser or the
  real Wi-Fi link.
- **Status:** in the code since 2026-11-21, and the existing library was converted that day. The
  automatic remux after a stream stop has run only in a bench test: no stream has run since
  2026-11-20.
- **Question:** recordings were slow to play from the drone, even on the same network. The Jetson's
  Wi-Fi link looked fine (5 GHz, −50 dBm, 468 Mbit/s transmit rate, power save off), and a
  recording needs about 12 Mbit/s. Is the file layout the cause?
- **Baseline A:** the recording as written. Since 2026-10-07 the recorder writes fragmented MP4:
  one-second fragments, with the full index at the end of the file. That is on purpose: if the
  stream is killed, every finished second still plays.
- **Variant B:** the same recording remuxed with one index at the front
  (`qtdemux ! h264parse ! mp4mux faststart=true`), with no re-encoding.
- **Only intended change:** the container layout.
- **Primary metrics:** start time (from setting the video source to the first `playing` event) and
  seek time (from jumping to 3:45 to playback running again).
- **Secondary metrics:** HTTP requests and bytes in each phase, and stalls (`waiting` events) in the
  10 s after start and the 5 s after the seek.
- **Correctness criteria, set before measuring:** B decodes to the same frames as A, both play with
  no media error, and both report a duration of about 300 s.

**Decision:** kept. Finished recordings are remuxed after each stream stop, and the recordings
service keeps its video list between starts (private application repository, commits `dbcb508`
and `7d1654b`).

## Why fragments are slow in a browser

Neither browser used the index at the end of the fragmented file. Chromium read the header of every
fragment before it showed the first frame. Firefox started at once, but on a seek it walked the
fragment headers one by one. Each step was a separate HTTP range request.

## Workload

| Field | Value |
| --- | --- |
| Recording | One 5-minute 1920×1080 H.264 recording at 30 FPS, 446 MB (about 11.9 Mbit/s), 301 fragments |
| Server | The application's recordings handler (HTTP/1.0) on the Jetson |
| Link | Emulated: one 40 Mbit/s token bucket shared by all connections, plus 5 ms before each response |
| Browsers | Firefox 153 headless (Selenium); headless Chromium (Playwright, version not recorded) |
| Each run | Fresh browser profile: start, play 10 s, seek to 225 s, play 5 s |
| Order | A B B A A B per browser, after one untimed read of both files to warm the page cache |
| Code | Private application repository `e0d3116` plus the uncommitted list cache below, which does not touch video sending |

## Results

Median of three runs, with the range. Per-run data: [`playback.json`](../data/2026-11-recording/playback.json).

| Browser and phase | A: fragmented | B: index at the front | Requests, A → B |
| --- | ---: | ---: | ---: |
| Chromium start | 5.39 s [5.38–5.40] | **0.10 s** [0.096–0.097] | 303 → 1 |
| Chromium seek to 3:45 | 0.92 s [0.91–0.92] | 0.93 s [0.925–0.928] | 1 → 1 |
| Firefox start | 0.145 s [0.139–0.174] | 0.081 s [0.067–0.084] | 4 → 1 |
| Firefox seek to 3:45 | 5.40 s [5.38–5.41] | **0.97 s** [0.968–0.974] | 192–193 → 1 |

- Chromium read 16 MB before its first frame from the fragmented file, against 0.46 MB. Once it
  had walked the headers, its seeks were as fast as on B.
- Firefox's seek read about 18 MB from the fragmented file, against 4.6 MB.
- The three runs in each cell agree within 0.04 s.

Every run reported a 300.2 s duration, no media error, and 2 stalls, both at the start, in both
layouts. B decodes to the same 9,006 frames as A (MD5 of every decoded frame), with identical
timestamps, durations, sizes and key-frame flags.

Each extra request cost about 18 ms (Chromium) to 25 ms (Firefox) here: the 5 ms delay plus the
bytes the server sent before the browser dropped the connection. That cost grows with the length
of the recording, one fragment per second, and with the real cost of a connection over Wi-Fi.
Through the website's relay, each request is two connections.

Three pre-run trials were excluded: they ran while the frame check was using the CPU, and before a
fix to the harness's statistics. Their values matched the measured runs.

## What I changed

- After a stream stop, each finished H.264 recording is queued, and the stop script remuxes the
  queue in a detached job at idle CPU and disk priority (`nice 19`, idle I/O class). If the stream
  starts again, the job stops and keeps the rest queued. The recorder still writes fragments while
  it records, so an interrupted recording is as safe as before.
- Before the remux replaces a file, it checks that the new file holds the same samples and the same
  bytes of frame data. The file keeps its modification time, which the recordings page sorts by,
  and its mode, and motion a render saved for it is carried over.
- Renders made in pieces are joined with the index at the front, and a one-piece render is remuxed
  when it finishes.

The remux took 1.4–1.7 s for the 446 MB recording (1.8 s of CPU). In a bench test of the stop
chain, the detached job handed control back to the stop script in 4 ms and remuxed the recording
in 1.7 s. Renders of 10 s from the remuxed recording matched renders from the fragmented one frame
for frame, 300 of 300, in one piece and in two, using the same saved motion.

## Converting the library

On 2026-11-21, with the stream off, I converted the existing library at idle priority: 271 files,
49.0 GB, in 398 s. 261 were remuxed and verified. The check refused 10, and I left them unchanged.
All 10 are renders from October whose last one or two samples are zero-filled or lie past the end
of the file: renders that were cut off. The remux kept one or two fewer samples, so the check did
its job. One original recording, copied first, decoded to the same 841 of 841 frames after
conversion.

## Recordings list kept between starts

The website's first Library load after starting the recordings service showed "Request cancelled
or timed out" until Refresh. The service read each video's length, size and codec by opening it
with OpenCV and kept that only in memory, so it opened all 358 videos again at every start. A
readable video took about 54 ms. The 54 that OpenCV can't open took longer, because it tries FFmpeg
and then GStreamer. The app gives up on a request after 8 s.

The service now saves the details in a file beside the recordings, keyed by each video's size and
modification time, and reads it back at start. A changed file is read again, a deleted one drops
out, and unreadable files are remembered too. At start it also lists the library once in the
background, so recordings made since the last start are read before the page asks.

| First list after a start, 358 videos | Time |
| --- | ---: |
| Before: cold disk, stream off | 32.1 s |
| Before: files already in the page cache | 15.8 s |
| After, two starts | **0.06 s** each (reading the saved file: 32–33 ms) |
| After, through the website's relay, right after starting the service | 0.07 s |

These are single timings, except the two starts. After the library was remuxed, the service read
all 358 videos again in 3.4 s, against 32 s before. I did not measure why; the index at the front
is a likely reason, since OpenCV has to find the index too.

The control server now also starts the recordings service as it starts, so the Library works after
a boot. Idle, a test copy of the service held 113 MB and used no CPU over 30 s. After the Jetson's
next boot, on 2026-11-24, the service was running, started in the same second as the control
server (private application repository, commit `58839c1`).

## Limits

- Two desktop browser engines on an emulated link. My phone (Safari and the app's player), the
  PC's browser, the real Wi-Fi link and the website's relay were not measured. I have not matched
  the original slowness to a device, or to start versus seek.
- One 5-minute recording. Longer recordings have more fragments, so the fragmented layout would be
  slower still; I did not measure that.
- The remux after a stream stop has not yet run after a real stream.
- The 10 refused renders still play the slow way.
- The list timings are single runs on one library.
