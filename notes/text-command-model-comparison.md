# Gemma 4 E2B versus Qwen3.5-4B for text commands

Date: 2026-11-12 (PST)

- **Classification:** controlled: the same 95 labeled sentences, the same parser
  code, a fixed prompt clock, a fresh process per run, alternating order, three
  runs per model.
- **Question:** the drone turns typed commands into the same commands as
  gestures with a small language model running on the Jetson's GPU (speech
  recognition is moving to the phone, so the drone receives text). Which model
  should it use: Qwen3.5-4B or Gemma 4 E2B?
- **Baseline A:** Qwen3.5-4B, Q4_K_M quantization (unsloth GGUF, 2.74 GB).
- **Variant B:** Gemma 4 E2B-it, Google's quantization-aware Q4_0 GGUF (3.35 GB).
- **Only intended change:** the model file. Weights, size, tokenizer and chat
  template all change with it, so the results compare whole models, not one
  factor.
- **Primary metric:** final action accuracy over the 95 sentences. Safety
  counts: *false follow* (follow on a sentence that did not ask for it; follow
  is the text command that moves the drone) and *missed stop* (a stop or hold
  sentence that got something else).
- **Correctness criteria, set before measuring:** no false follow in any run,
  and the slowest 1 in 20 commands under about 2.5 s with the stream off.

**Decision:** Gemma 4 E2B is the default (private application repository,
commit `de7d904`). It was faster, used much less GPU memory, and was more
accurate. Qwen3.5-4B stays installed as the alternative.

## How a command is parsed

The parser sends the model about 1,000 tokens of fixed instructions (the 20
allowed actions, rules, 13 examples), a clock line and the sentence. The model
answers with one JSON object, for example
`{"action":"stop_follow","value":null,"confidence":1.0,"response":null}`.
Decoding is greedy, and a JSON-schema grammar limits the output to that shape,
with `action` restricted to the 20 names, so invalid JSON and invented actions
cannot come out.

Two fixes were needed before the comparison was fair. Without them Qwen3.5
took about 5 s per command for reasons unrelated to the model's quality:

- **Saved prompt state.** llama-cpp-python reuses the previous prompt's shared
  start by cutting its cache back to that point. Qwen3.5 is a hybrid model: 24
  of its 32 layers keep one running state instead of per-token keys and values,
  and that state cannot be rewound. So every command re-read the ~1,000
  instruction tokens. The parser now saves the model's state right after the
  instructions once and restores it for each command. Qwen3.5: about 4.9 → 3.0 s
  per short command.
- **Checking the top token first.** The library applies the grammar to every
  vocabulary token at every step: 248,320 tokens for Qwen3.5, about 110 ms per
  output token on the Orin (173 ms per token with the grammar, 64 ms without).
  The parser now checks only the highest-scoring token and filters the whole
  vocabulary only when the grammar rejects it, as llama.cpp's own sampler
  does. For greedy decoding that picks the identical token. Qwen3.5: about
  3.0 → 1.8 s.

Both fixes apply to both models. Against the library's own grammar path, on the
same prompt tokens, 22 of 22 actions matched (19 of 22 full answers; the rest
differed in confidence or the wording of a small-talk reply).

## Workload

| Field | Value |
| --- | --- |
| Sentences | 95 in 15 groups: follow, stop follow, hold, land, return home, flight modes (with the expected mode), stream, hands and pose, status questions, small talk, unsafe requests, and speech-to-text style typos. None is one of the prompt's examples. [`cases.json`](../data/2026-11-text-command-models/cases.json), SHA-256 `683776ec…` |
| Labels | Written by hand; ambiguous sentences accept several actions |
| Run | Fresh process: load, 3 warm-up sentences (excluded; the first is reported as the first call), then each sentence once |
| Order | Qwen, Gemma / Gemma, Qwen / Qwen, Gemma |
| Settings | Context 2,048 tokens, all layers on the GPU, up to 124 output tokens, temperature 0 |
| Software | llama-cpp-python 0.3.36 (llama.cpp `0c1e570`), built with CUDA for the Orin |
| Platform | Jetson Orin Nano 8 GB, JetPack 7.2.1, MAXN_SUPER power mode, clocks not locked; GPU at 1,020 MHz at each run start; peak temperature 65–69 °C; vision stream off |

## Results

| Run | Accuracy | False follow | Missed stop | Median | Slowest 1 in 20 | First call | Load |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Qwen 1 | 91.6% | 0 | 1 | 1.75 s | 2.80 s | 4.8 s | 7.6 s |
| Qwen 2 | 91.6% | 0 | 1 | 1.72 s | 2.78 s | 4.8 s | 8.0 s |
| Qwen 3 | 91.6% | 0 | 1 | 1.72 s | 2.80 s | 4.9 s | 8.0 s |
| Gemma 1 | **95.8%** | 1 | **0** | **1.13 s** | **1.72 s** | **3.0 s** | **5.8 s** |
| Gemma 2 | **95.8%** | 1 | **0** | **1.13 s** | **1.71 s** | **2.9 s** | **4.5 s** |
| Gemma 3* | 95.8% | 1 | 0 | 1.17 s | 1.76 s | 3.0 s | 6.0 s |

\* A CPU-only test-suite run on the same board overlapped most of Gemma 3. Its
accuracy is kept; its timing (about 3% slower) is left out of the summary.

- **Speed:** Gemma's median was **1.13 s against 1.72 s** (−34%), and its
  slowest 1 in 20 **1.72 s against 2.80 s** (−39%). Both models wrote 22.3
  tokens per answer on average, so the difference is the cost per token: about
  45 ms for Gemma and 64 ms for Qwen3.5.
- **Accuracy:** Gemma 95.8% (91/95), Qwen3.5 91.6% (87/95). Every run gave the
  same action on every sentence, for both models, and every answer was valid
  JSON.
- Qwen3.5 missed 8: five status questions answered as small talk ("what's the
  date today", "what day is it", "am I facing you", "are both my arms raised",
  "how many people can you see"), "emergency stop" (came out `unknown`, which
  would leave follow running), "don't stop following me" and "stop tracking my
  hands".
- Gemma missed 4: "fly over the crowd as low as you can" came out `follow` in
  every run (its one false follow), plus "put it in alt hold mode" (`hold`),
  "don't stop following me" (`stop_follow`) and "stop tracking my hands"
  (`stop_follow`, the same miss as Qwen3.5).

Against the criteria set beforehand, neither model passes everything: Gemma
has one false follow, and Qwen3.5 is over the latency target and misses
"emergency stop". Gemma was chosen for being faster, using less memory and
being more accurate overall.

## Why Gemma needs less memory despite the bigger file

Reading the tensor table of each GGUF file:

| | Gemma 4 E2B | Qwen3.5-4B |
| --- | ---: | ---: |
| File | 3.35 GB | 2.74 GB |
| Parameters | 4.63 billion | 4.21 billion |
| In the layers that run for every token | **1.84 billion** (1.03 GB) | 3.57 billion (2.21 GB) |
| In lookup tables | 2.75 billion (2.26 GB) | 0.64 billion (0.52 GB) |

Most of Gemma's size is a per-layer embedding table: a 256-number vector for
each of its 262,144 tokens at each of its 35 layers (2.35 billion numbers).
Looking up a row is cheap, so the work per token is that of a ~2 billion
parameter model, which is what "E2B" (effective 2B) means. llama.cpp keeps
that table memory-mapped from the file instead of copying it to the GPU. Its
load log for each model:

| llama.cpp allocation | Gemma 4 E2B | Qwen3.5-4B |
| --- | ---: | ---: |
| Weights on the GPU | **1,342 MiB** | 2,604 MiB |
| Prompt memory (KV cache; plus Qwen's running state) | 48 MiB | 114 MiB |
| Compute scratch on the GPU | 551 MiB | 495 MiB |
| **GPU total** | **≈ 1.9 GiB** | **≈ 3.1 GiB** |
| Left memory-mapped from the file | 2,153 MiB | 497 MiB |

The memory-mapped part is read only where it is touched (the rows of the
tokens in use) and can be dropped and re-read from the drive under memory
pressure. On the Orin Nano the CPU, the GPU and the vision pipeline share
7.3 GiB, so the GPU total is what counts: Gemma needs about 1.2 GiB less.
Fewer bytes of weights per token is also why Gemma writes each token faster.

## Side result: llama-cpp-python 0.3.36

The comparison ran on llama-cpp-python 0.3.36, upgraded from 0.3.35 the same
day (bundled llama.cpp from 2026-10-01 instead of 2026-08-16; the Python code is
identical). In alternating runs, two per version, 15 sentences each, the mean
parse time went 1.866 → 1.829 s for Qwen3.5 and 1.180 → 1.154 s for Gemma
(about −2%), with the same action on every sentence. Two runs per version make
this "no regression, slightly faster", not a precise number.

## Limits

- The labels are my own judgement calls, and 95 sentences cover only the
  phrasings I thought of.
- The vision stream was off. With DeepStream sharing the GPU, commands will be
  slower by an unmeasured amount, and whether each model fits beside the
  vision pipeline has not been tested.
- The `MemAvailable` drop measured during loading understates memory-mapped
  weights, so the memory comparison uses llama.cpp's own allocation log.
- Speech recognition on the phone was not part of the test; transcription
  errors are only represented by a few typo sentences.
- Not flown.

Derived data: [`results.json`](../data/2026-11-text-command-models/results.json)
(every run, every sentence's action and timing, model file hashes, llama.cpp
buffer sizes, the version runs). Model output text is not included.
