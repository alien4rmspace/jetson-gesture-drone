# Turning gesture commands off by voice

Date: 2026-11-16 (PST)

- **Classification:** controlled: the same labeled sentences, a fixed prompt clock, a fresh process
  per run, alternating order, three runs per variant, the vision stream off.
- **Question:** the drone takes commands from hand and body gestures and from typed or spoken text.
  I wanted to be able to say "turn off gestures" (while carrying something, or with people waving
  nearby) and "gestures back on". Can the small language model on the Jetson learn the switch from
  a one-line prompt change, without losing anything it already does?
- **Baseline B:** the prompt adopted the day before ([status questions](status-question-routing.md)).
  It had `enable_hands` / `disable_hands` actions with no rule, and nothing executed them.
- **Variant G:** the actions renamed `enable_gestures` / `disable_gestures`, plus one rule line and
  one example.
- **Only intended change:** those names, the rule and the example. The stream side (the switch
  itself) is tested with unit tests, not in this benchmark.
- **Primary metric:** accuracy on 20 on/off sentences.
- **Correctness criteria, set before measuring:**
  - no sentence B gets right in every run may become wrong;
  - other commands at most 1 sentence lower;
  - no false follow, false takeoff, missed stop, or switch toggled by a question or remark;
  - 100% valid JSON;
  - median latency within +10%.

**Decision:** G is the default (private application repository, commit `a6cfa18`). It failed the first criterion on one sentence, and
I adopted it anyway, knowingly; see "One sentence broke again".

## What the switch does

The model only classifies the sentence. Code on the stream holds one flag, on at every start.
While it is off:
- **Gestures that stop still work:** open palm, crossed arms and two open palms (stop follow). They
  can only stop the drone, and an emergency stop should not depend on a setting.
- **Gestures that move the drone are ignored:** follow, movement, movement mode and twirl. A gesture
  movement already running is stopped on the next update, and the video overlay shows "gestures off".
- **Follow mode is left as it is.** "Stop following" still ends it.

Neither direction asks for confirmation, because neither moves the drone. Hand and pose detection
keep running, since the stop gestures need them.

## Why the prompt had to change

A pilot on B showed the model had no idea what to do with these sentences. "Disable gesture
commands" became a flight-mode change, "turn off gestures" and "gestures off" went to the pose
detector, "stop listening to my gestures" became *stop following*, and "ignore my hand signals" got
a chat reply saying it would, while nothing changed. That last one is the worst kind of failure: a
confident answer with no action behind it.

G names the actions after what they do and adds one rule: turn off, disable, stop or ignore gesture
commands, gesture control, hand gestures or hand signals means `disable_gestures`; turning them
back on means `enable_gestures`; the pose actions are only for the pose detector.

## Workload

| Field | Value |
| --- | --- |
| Sentences | 150 in 20 groups: the 132 from the status-question benchmark plus 18 new ones. 20 are on/off sentences: 16 new, plus 4 older hand-switch sentences relabelled. The other 2 new ones are a question ("are gestures enabled") and a remark ("my hands are cold"), which must not flip the switch. None is a prompt example. [`cases.json`](../data/2026-11-gesture-switch/cases.json) |
| Labels | Written by hand. 12 of the 16 new on/off sentences were in the pilot on B; 4 were written after it. |
| Model | Gemma 4 E2B, Q4_0 GGUF, all layers on the GPU, greedy decoding, context 2,048 tokens |
| Run | Fresh process: load, 3 warm-up sentences, then each sentence once |
| Order | B G, G B, B G |
| Platform | Jetson Orin Nano 8 GB, MAXN_SUPER power mode, vision stream off |

## Results

Every sentence got the same answer in all three runs of each variant. Full per-run and per-sentence
data: [`results.json`](../data/2026-11-gesture-switch/results.json).

| | B | G |
| --- | ---: | ---: |
| On/off sentences right (20) | 4 | **18** |
| Status questions exactly right (34) | 30 | 30 |
| Other commands right (96) | 90 | 90 |
| Switch flipped by a question or remark | 0 | 0 |
| False follow / false takeoff / missed stop | 0 / 0 / 0 | 0 / 0 / 0 |
| Valid JSON | 100% | 100% |
| Median / slowest 1 in 20 | 1.16 / 1.56 s | 1.15 / 1.55 s |
| Prompt tokens | 1,472 | 1,570 |

G's two remaining on/off misses:
- "you can watch my hand signals again" reads the gesture list instead of switching on;
- "stop tracking my hands" still becomes *stop following*, as in every earlier version.

Neither one toggles anything by mistake. The prompt grew by 98 tokens without a slower median,
because the fixed part of the prompt is pre-read once. The slowest single reply got longer
("what can you do", 3.1 to 4.6 s), because the chat answer itself was longer: 97 output tokens
against 63.

## One sentence broke again

The status total stayed at 30, but two sentences swapped:
- "what is the RAM usage" was fixed (memory only; B also added CPU);
- "what's the power draw right now", right in every B run, now also reads the motor line.

That is the same sentence the `person_lock` rename broke the day before, from an unrelated edit.
Two different one-line changes flipping the same sentence suggest it sits close to a tie for this
model, so almost any prompt edit can tip it.

The rule "no sentence that was right may become wrong" therefore failed again. I adopted G anyway,
knowingly: 14 more on/off sentences right and one fixed status question, against one extra
read-only reading on one question. The rule is still the right one to write before measuring,
because it stops a better total from hiding a new miss. But a sentence on a tie will block almost
every change. The honest ways out are to fix that sentence on purpose and measure the fix, or to
waive the rule in the open, as here. Relabelling it after seeing the result is not one of them.

## Live check

On the running drone (disarmed, on the bench), through the same text endpoint the phone app and the
control page use:
- "turn off gestures" switched them off in about 2 s, and the video overlay changed to "gestures off";
- "ignore my hand signals" answered that they were already off;
- "turn gestures back on" and "gestures on" switched them back on and then said "already on".

One sentence behaved differently live: "are gestures enabled" read out the lock, follow, people and
pose readings, where every benchmark run had picked the gesture-list action. The benchmark fixes
the clock in the prompt so every run sees identical text; the live prompt carries the real time,
and the server's settings may differ. That can tip a borderline sentence. I could not confirm the
cause on the drone, because a second copy of the model did not fit in memory beside the running
stream. Either way nothing was toggled. But the drone cannot yet *say* whether gestures are on,
because there is no reading for it.

## Limits

- The labels are hand-written, and 12 of the 16 new on/off sentences were also used in the pilot
  that motivated the rule (on B, not G).
- One model was tested; Qwen3.5-4B was not run with this prompt.
- The vision stream was off, so latency has no GPU contention.
- The switch is not saved: every stream start turns gestures back on.
- A deterministic benchmark shows the answers are stable for that exact prompt. The live prompt
  differs (the clock), and one borderline sentence answered differently live.
- "Stop gestures still work while off" is covered by unit tests; it was not tried live (nobody in
  view).
- Not flown.
