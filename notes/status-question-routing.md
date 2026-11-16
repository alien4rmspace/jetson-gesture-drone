# Status questions: the model picks the readings, code says the numbers

Date: 2026-11-15 (PST)

- **Classification:** controlled: the same 132 labeled sentences, a fixed prompt clock, a fresh
  process per run, alternating order, three runs per variant, the vision stream off.
- **Question:** the drone answers typed questions about its own state ("how is the GPS?", "how
  many amps are we drawing?") with a small language model on the Jetson. Is it better to let the
  model choose *which* readings a question asks for, and have code read and say them?
- **Baseline A:** the model chose between four "inspect" actions (battery, resources, scene,
  pose). Keyword patterns then picked the topics to answer, with keyword backups for GPS and
  battery.
- **Variant B:** one `status` action with a `fields` list the model fills in, restricted by the
  output grammar to 16 reading names. Code reads each reading live and writes the sentence.
- **Variant C:** B with one field renamed (`lock` became `person_lock`); see "A field name is part
  of the prompt" below.
- **Only intended change:** how status questions are classified and routed. B also changes the
  action list, the JSON schema, the prompt's rules and examples, and the server routing, so the
  A-to-B result is their combined effect.
- **Primary metric:** status accuracy. A status sentence is right only when the readings the drone
  would answer are exactly one of the labeled sets. A missing reading or an extra one read aloud
  both count as wrong.
- **Correctness criteria, set before measuring:**
  - command accuracy at most 1 sentence lower;
  - no false follow or false takeoff, and no missed stop;
  - 100% valid JSON;
  - median latency within +10%.

**Decision:** B is the default (private application repository, commit `92eeb7a`). C scored better overall but broke one question B always got right, which a rule set before the runs forbids, so it was not adopted.

## Why the model does not read the data out

The obvious design is to give the model all the telemetry and let it answer. Three things
measured earlier in this project argued against it:

- **Numbers.** When the model phrased the battery answer, it took about 7 s and said
  "3.905666666666667 V". Code writes the same answer in 0.01 s.
- **Made-up or stale answers.** One model answered the flight mode from earlier conversation
  instead of the drone; another said the motors were "operating normally" with no data.
- **Size.** The telemetry snapshot alone is about 1,000 tokens. The context is 2,048 tokens, and
  the snapshot changes every second, so unlike the fixed instructions it cannot be pre-read once.

So the model only gets the *names* of the readings, each with a one-line description:

```text
- battery: battery voltage, current, amps, power draw or charge left
- gps: GPS fix, satellites, HDOP or GPS lock
- armed: whether the motors are armed
- motors: motor throttle outputs and how hard the motors work
...
```

The model answers `{"action":"status","fields":["altitude","armed"],"confidence":0.9}`. The
JSON-schema grammar has two alternatives: commands keep their old four-key shape, so they cost no
extra tokens, and the status form allows 1 to 4 known names. Code drops readings another one
already gives (the motors answer includes arming), reads the sources (a cached telemetry service,
stream statistics, the process list) and writes one sentence. Keyword matching remains only as a
backup for GPS and battery questions the model calls unknown.

## How it was scored

Comparing action names no longer works when one version says `inspect_resources` and the other
`status`. The comparison tool therefore:

- runs each variant from frozen copies of its source files, in its own process;
- after the model's answer, runs that variant's real routing code with the answer functions
  replaced by recorders, which note which readings would be answered without fetching anything;
- scores status sentences on those readings, and every other sentence on its action, which is also
  wrong if any reading is answered.

## Workload

| Field | Value |
| --- | --- |
| Sentences | 132 in 19 groups. 34 are status questions: 12 earlier "inspect" questions relabeled with readings, plus 22 new ones. The new ones cover GPS, amps, arming, flight mode, follow-mode state, person lock, memory and two-reading questions. The rest are commands, small talk, unsafe requests and typos. None is a prompt example. [`cases.json`](../data/2026-11-status-questions/cases.json) |
| Labels | Written by hand. Each status sentence lists every acceptable set of readings. |
| Model | Gemma 4 E2B, Q4_0 GGUF, all layers on the GPU, greedy decoding, context 2,048 tokens |
| Run | Fresh process: load, 3 warm-up sentences, then each sentence once |
| Order | A B, B A, A B; then B C, C B, B C (a first B-C attempt was stopped when the video stack was started mid-run and is excluded) |
| Platform | Jetson Orin Nano 8 GB, MAXN_SUPER power mode, vision stream off |

## Results: A versus B

Every sentence got the same answer in all three runs of each variant, so the table shows one run's
counts (the latency spread across runs was under 2%).

| | A: inspect actions + keywords | B: status + fields |
| --- | ---: | ---: |
| Status questions exactly right (34) | 26 | **30** |
| Commands right (98) | 92 | 91 |
| Turned follow mode on when not asked | 4 | **0** |
| False takeoff / missed stop | 0 / 0 | 0 / 0 |
| Valid JSON | 100% | 100% |
| Median / slowest 1 in 20 | 1.18 / 1.74 s | **1.10 / 1.52 s** |
| Output tokens / prompt tokens | 22.0 / 1,278 | 20.5 / 1,472 |

The biggest effect is on safety. A sent "are you following me right now?", "is follow mode on
right now?" and "are you locked onto me?" to the **follow command**, which turns follow mode on;
follow is the one text command that moves the drone. B answers all three as questions.

A's other status misses:
- "how many amps are we drawing" read out the whole RAM/CPU report;
- "is the drone armed right now" came back as unknown;
- "which flight mode is it in" was answered as small talk;
- "check the battery and the GPS" gave only the battery.

B was faster although its prompt is 194 tokens longer. The fixed instructions are pre-read once,
and a status reply is shorter than the old four-key object.

B lost one command: "what is the takeoff altitude" now reads the current altitude instead of
answering as small talk. It only reads data; nothing moves.

## A field name is part of the prompt

B's four remaining misses all added a reading nobody asked for:
- "how many satellites do you have" and "what's the HDOP" came back as `gps` + `lock`;
- "what is the RAM usage" added CPU;
- "how many amps are we drawing" added the motors.

The first two have a clear cause. The `gps` description says "GPS lock", and the next field down
was named `lock` (meaning locked onto a person). With a constrained output, the model chooses
between names, so the names and their one-line descriptions are the whole interface. C renames
that field to `person_lock`, described as "whether you are locked onto a person (tracking them)".
It was run against B the same way, with the same criteria plus one more: no status question B gets
right in every run may become wrong.

| | B: `lock` | C: `person_lock` |
| --- | ---: | ---: |
| Status questions exactly right (34) | 30 | 31 |
| Commands right (98) | 91 | 92 |
| False follow / false takeoff / missed stop | 0 / 0 / 0 | 0 / 0 / 0 |
| Median / slowest 1 in 20 | 1.11 / 1.53 s | 1.12 / 1.68 s |

The rename fixed the two satellite and HDOP questions. It also changed things it was not aimed at:
- the command "stop tracking my hands" went from stop follow to the right action, disable hands;
- "what's the power draw right now", right in every B run, came back as battery + motors.

That last change broke the pre-set rule, so C was not adopted, although it was 2 sentences better
overall. Writing the rule down first is what keeps a better total from explaining away a new miss.
The run also shows how sensitive a 2B model is to its prompt: renaming one field moved a command
classification elsewhere in the prompt. So every prompt edit needs the whole evaluation.

## Limits

- The labels are hand-written, and 22 of the 34 status questions were written for this change.
- One model was tested; the alternative model (Qwen3.5-4B) was not run with this prompt.
- The vision stream was off, so latency has no GPU contention.
- "The motors sound loud" is still classified as the hold command, in every variant. Hold is not
  executed from text.
- Not flown. Answers were checked live on the bench with the drone disarmed: about 1.5 s per
  question through the stream's HTTP endpoint.
