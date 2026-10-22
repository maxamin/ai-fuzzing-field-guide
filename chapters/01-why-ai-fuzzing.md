# Why AI Fuzzing, and Why Now

Coverage-guided fuzzing (AFL++, libFuzzer, honggfuzz) is one of the most productive bug-finding techniques ever built. Its weak points have been stable for a decade:

1. **Harnesses are written by hand.** OSS-Fuzz projects often cover only a small slice of a library's API because someone has to write every `LLVMFuzzerTestOneInput`.
2. **Structured inputs defeat blind mutation.** Bit flips rarely produce valid SQL, valid PDF cross-reference tables or a valid TLS handshake state machine.
3. **Plateaus.** Campaigns stall on magic values, checksums and deep state, and nobody knows why until a human reads the coverage report.
4. **Triage is the real cost.** Ten thousand crashes, forty root causes, three security bugs.

Large language models (LLMs) are useful exactly where this human effort sits. They read APIs and documentation, write plausible code, know common file formats and protocols, and can summarise a stack trace. They are bad at the thing fuzzers do well: executing millions of inputs per second with no false confidence.

The productive design pattern is therefore **"LLM in the slow loop, fuzzer in the fast loop."**

```
  slow loop (minutes)                fast loop (microseconds)
 ┌──────────────────────┐           ┌───────────────────────┐
 │ LLM: write harness,  │  seeds,   │ AFL++/libFuzzer:       │
 │ grammar, dictionary, │ ────────► │ mutate, execute,       │
 │ explain plateau      │ ◄──────── │ measure coverage       │
 └──────────────────────┘  coverage └───────────────────────┘
                          + crashes
```

Never put a model call on the per-execution path. One inference is around a million executions you are not running.

## What changed between 2022 and 2026

- **Google OSS-Fuzz-gen** showed LLM-written harnesses adding real coverage to OSS-Fuzz projects and finding new bugs, including bugs in projects fuzzed for years. Coverage *increase* of existing projects is the honest metric they report, and it is the one you should use too.
- **Fuzz4All** (ICSE 2024) used an LLM as both the input generator and mutation engine for compilers and interpreters in many languages, a domain where grammar-based fuzzers previously needed one grammar per language.
- **TitanFuzz / FuzzGPT** targeted deep learning libraries (PyTorch, TensorFlow) by having a model write API call sequences, an area where traditional fuzzers struggle because the "input" is a program.
- **ChatAFL** (NDSS 2024) used an LLM to extract protocol message grammars and state transitions from RFC knowledge for network protocol fuzzing.
- **KernelGPT** generated syzkaller syscall specifications (syzlang) for Linux drivers that lacked them.
- The DARPA **AI Cyber Challenge (AIxCC)** ran 2023–2025, and its finalist cyber reasoning systems combined classic fuzzing, static analysis and LLM agents for patching. Several were open-sourced after the 2025 final. They are the best public reference architectures available.

## What this book covers

| Chapter | Problem attacked |
|---|---|
| 2 | Harness synthesis |
| 3 | Seeds, dictionaries, grammars |
| 4 | Stateful protocol fuzzing |
| 5 | Learned mutation and scheduling |
| 6 | Triage, deduplication, root cause |
| 7 | Smart contract fuzzing with AI |
| 8 | Evaluating claims honestly |

## The economics of a model call

The slow-loop rule is easiest to defend with arithmetic. Suppose a target runs at 5,000 executions per second on one core, and an LLM call with a large prompt takes about 20 seconds end to end. That one call costs roughly 100,000 executions if the fuzzer waits for it. If the fuzzer keeps running while the model thinks (the right design), the call costs only money and a little orchestration CPU.

So the question for every AI component is not "is the model clever?" but "does one model output buy more new coverage than the executions or money it costs?" A harness that unlocks 2,000 new edges is worth hundreds of calls. A model that picks the next mutation is worth nothing, because the fuzzer makes that decision millions of times per hour.

| AI task | Frequency | Run asynchronously? | Typical payoff |
|---|---|---|---|
| Harness synthesis | once per target function | yes | large: new reachable code |
| Seed / dictionary / grammar | once per format | yes | medium to large early in a campaign |
| Custom mutator code | once per format | yes | medium, compounding |
| Plateau breaking | every N minutes of stall | yes | small, occasionally decisive |
| Crash triage and reports | once per bucket | yes | saves human hours |
| Per-input mutation decisions | per execution | no | negative |

## Where it has worked in public

Two kinds of evidence are worth trusting: bugs that maintainers confirmed and fixed, and coverage gains measured by the same infrastructure before and after. Google's security blog reported in late 2024 that OSS-Fuzz-gen harnesses had found around two dozen new vulnerabilities, including an out-of-bounds issue in OpenSSL (CVE-2024-9143), in code that existing human-written harnesses did not reach. In each case the model wrote the harness, and the sanitizer and the maintainer decided whether the crash was real.

The AIxCC final at DEF CON in August 2025 is the other reference point. Teams' systems found and patched synthetic and real bugs in open-source C and Java projects. The scoring rewarded only findings that came with a reproducing input (a proof of vulnerability) and patches that passed functionality tests. That is this book's house principle: **the model proposes, the checker decides.**

## Worked example: measuring your fuzzing gap before adding AI

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

Before adding any model, measure what your existing harnesses miss. This uses the OSS-Fuzz helper with a real project (`libpng` here; any integrated project works).

```bash
git clone --depth 1 https://github.com/google/oss-fuzz && cd oss-fuzz
python3 infra/helper.py build_image libpng
python3 infra/helper.py build_fuzzers --sanitizer address libpng
python3 infra/helper.py run_fuzzer libpng libpng_read_fuzzer -- -max_total_time=300
```

Step 1 gives you a baseline: libFuzzer prints lines such as `#1048576 pulse cov: 2361 ft: 9012 corp: 512/... exec/s: 3495`. Write down `cov` (edges) and `exec/s`.

Step 2 generates a Fuzz Introspector report, which lists functions and how much of each is reached:

```bash
python3 infra/helper.py introspector libpng --seconds=300
# open build/out/libpng/introspector-report/inspector/fuzz_report.html
```

Step 3: sort the "all functions" table by cyclomatic complexity and filter for "reached by fuzzers = 0". These are your candidate targets for Chapter 2.

Step 4: compute the budget. At 3,500 exec/s, one 20-second blocking call costs about 70,000 executions. Run model calls in a separate process so the cost is zero executions.

The output of this exercise is a short table of target functions, their complexity, and today's coverage. Any AI technique you add later is measured against that table.

## Common pitfalls

- **Putting the model on the hot path.** Any per-input inference turns a 5,000 exec/s fuzzer into a 0.1 exec/s one. Keep models asynchronous.
- **Measuring crashes instead of bugs.** Crash counts inflate with duplicates and harness bugs. Count deduplicated, maintainer-confirmed root causes.
- **Skipping the baseline.** If you never ran AFL++ with CMPLOG and a dictionary, you cannot attribute gains to the model.
- **Trusting model explanations without reproduction.** A crash explanation is only credible when the input reproduces under a sanitizer in a fresh process.
- **Ignoring cost.** Record tokens and dollars per campaign so a later reader can judge whether the gain was worth it.

## A note on ethics

Everything here is meant for software you own or are authorised to test, open-source projects (report through their security policy), and bug bounty scopes. Fuzzers find real vulnerabilities, so use coordinated disclosure.

## Exercises

1. Pick an OSS-Fuzz project. Use its introspector report to find the five largest functions that are not reachable from any harness.
2. Estimate the cost of one LLM call in executions forgone for a target running at 5,000 exec/s.
3. Sketch where the slow loop and fast loop sit in a fuzzer you already use.

## Further reading

- OSS-Fuzz-gen: <https://github.com/google/oss-fuzz-gen>
- Fuzz4All: Xia et al., "Fuzz4All: Universal Fuzzing with Large Language Models", ICSE 2024, arXiv:2308.04748
- Deng et al., "Large Language Models Are Zero-Shot Fuzzers" (TitanFuzz), ISSTA 2023, arXiv:2212.14834
- Meng et al., "Large Language Model guided Protocol Fuzzing" (ChatAFL), NDSS 2024
- Yang et al., "KernelGPT", arXiv:2401.00563
- AIxCC: <https://aicyberchallenge.com>
- The Fuzzing Book: <https://www.fuzzingbook.org>
