# LLM Harness Synthesis

A fuzz harness is a small program that turns a byte buffer into calls on the target API. Writing good ones takes knowledge of the API, its preconditions, and how to clean up. That makes it a good fit for code-generating models, provided you close the loop with a compiler and a fuzzer.

## The generate–compile–run–refine loop

```
target function + context
        │
        ▼
  LLM drafts harness ──► compile ──fail──► feed errors back (≤ N rounds)
        │                  │ ok
        │                  ▼
        │            run 60 s fuzz smoke test
        │                  │
        │      crash at t≈0?──yes──► likely harness bug: feed back
        │                  │ no
        │                  ▼
        └──────── coverage delta > 0? ──► keep, else discard
```

Each gate removes a category of junk:

| Gate | Removes |
|---|---|
| Compiles | hallucinated functions, wrong headers |
| No instant crash | harness misuse (NULL contexts, wrong sizes, double free) |
| Coverage increase over existing harnesses | redundant harnesses |
| Survives 24 h without only-harness crashes | subtle API misuse |

## Context selection is the whole game

Models write better harnesses when the prompt contains:

- The **function signature and its doc comment**.
- **Existing call sites** from tests or examples. This is the strongest single signal. Search the repo for callers and include two or three.
- **Type definitions** for every struct parameter, up to depth two.
- **An existing harness from the same project** as a style example (build flags, includes, init calls).
- Explicit **rules**: "do not call `exit`", "free everything you allocate", "use `FuzzedDataProvider`".

Tools like Fuzz Introspector expose reachability and complexity data that tells you which functions to target first: high cyclomatic complexity, not yet reached, and accepting pointer-plus-length inputs.

## Distinguishing harness bugs from target bugs

This is the most important skill. An LLM harness that passes a 4-byte buffer to a function that requires 16 produces an out-of-bounds read *in the target*. The stack trace looks like a real bug.

Heuristics:

1. **Crash within seconds on nearly any input:** suspect the harness.
2. **The crash violates a documented precondition** (e.g. "`buf` must be at least `HEADER_LEN`"): harness bug.
3. **Reproduce through a second path:** can the same input reach the crash through the CLI tool or a public parser entry point? If yes, it is real.
4. **Ask a second model, adversarially:** "Here is the harness and the crash. Argue that this is a harness misuse." Treat a persuasive argument as a signal for human review, not a verdict.

## Minimal harness template to give the model

```c
#include <stdint.h>
#include <stddef.h>
#include "target.h"

int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    if (size < MIN_SIZE) return 0;          // respect preconditions
    target_ctx *ctx = target_new();
    if (!ctx) return 0;
    target_parse(ctx, data, size);          // function under test
    target_free(ctx);                       // no leaks: leaks are findings
    return 0;
}
```

## Beyond C: other ecosystems

- **Rust:** `cargo fuzz` with `arbitrary::Arbitrary` derives. Models are good at writing `#[derive(Arbitrary)]` structs that describe an API call sequence.
- **Python:** Atheris. Fuzzing native extensions is especially valuable.
- **Java:** Jazzer, which also detects injection-style bugs, not only memory errors.
- **Go:** native `testing.F` fuzzing. Models can convert table-driven tests into fuzz targets almost mechanically.

## How OSS-Fuzz-gen structures the loop

OSS-Fuzz-gen is the best public reference. It takes a benchmark YAML file that names a project and a target function, builds a prompt from Fuzz Introspector data (signature, cross-references, type definitions), asks a model for a harness, builds it in the project's OSS-Fuzz container, runs it for a short period, and records whether it compiled, whether it crashed, and the line coverage it reached. Failed builds are fed back with the compiler errors for a few fix-up rounds. Newer versions add agent-style steps where the model can inspect files in the project before writing the harness.

Two lessons from that design carry over to any in-house setup:

1. **Build in the real build environment.** A harness that compiles on your laptop but not with the project's flags and sanitizers is worthless. Use the OSS-Fuzz Docker image, or your CI image.
2. **Score on coverage reached by the new harness that existing harnesses did not reach.** Raw coverage rewards harnesses that duplicate existing ones.

## Worked example: a three-round harness loop with libFuzzer

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

This walk-through targets a small parser in a project you control. The commands are the same for any C library.

**Step 1 – gather context.** Collect the target signature, doc comment, and two callers:

```bash
grep -rn "cfg_parse_buffer(" src/ tests/ | head -5
ctags -x --c-kinds=s include/cfg.h | grep cfg_ctx
```

**Step 2 – prompt.** Give the model the template above plus the context and rules: use `FuzzedDataProvider` only if C++, never call `exit`, free all allocations, respect `CFG_MIN_LEN`.

**Step 3 – compile gate.**

```bash
clang -g -O1 -fsanitize=fuzzer,address,undefined -Iinclude \
    harness.c src/*.c -o cfg_fuzzer 2> build.log || cat build.log
```

If it fails, send `build.log` (trimmed to the first 40 lines) back to the model. Cap this at three rounds; after that the context is usually wrong, not the code.

**Step 4 – smoke run.**

```bash
mkdir -p corpus && ./cfg_fuzzer corpus -max_total_time=60 -print_final_stats=1
```

Expected healthy output: `cov` keeps growing for the first seconds, `exec/s` in the thousands, no crash. A crash at `#2` or `#3` with an empty or tiny input almost always means the harness breaks a precondition.

**Step 5 – coverage delta.** Merge the new corpus into the existing project corpus and compare:

```bash
./existing_fuzzer -merge=1 merged/ old_corpus/ corpus/ 2>&1 | grep "MERGE-OUTER"
```

`MERGE-OUTER: N new files with K new features added` tells you if the harness reached anything the old corpus did not. Better still, build both with `-fprofile-instr-generate -fcoverage-mapping` and compare `llvm-cov report` line counts.

**Step 6 – keep or discard.** Keep the harness only if it adds coverage and survives a longer run (hours) without harness-only crashes.

## Common pitfalls

- **Prompting with the signature only.** Without call sites, models guess initialisation order and produce harnesses that crash immediately.
- **Letting the model "fix" crashes by adding try/catch or early returns.** That hides target bugs. A harness may only enforce documented preconditions.
- **Global state leaks between iterations.** libFuzzer runs inputs in one process. A harness that never resets a global cache produces irreproducible crashes. Check reproduction with a single-input run.
- **Measuring compile rate as success.** Compiling is the first gate, not the metric. Many compiling harnesses reach almost nothing.
- **Fuzzing with optimisations that remove the bug.** Keep `-O1` or `-O2` with sanitizers and avoid `-DNDEBUG` when assertions guard invariants.
- **Accepting the model's crash verdict.** Reproduce under ASan in a fresh process and check against preconditions before filing.

## Exercises

1. Take a library with an existing OSS-Fuzz integration. Pick one unreached function and generate three harnesses with different context sets (signature only, plus call sites, plus types). Compare compile rate and coverage.
2. Write a "harness critic" prompt that flags precondition violations. Measure it on 20 crashes you have triaged by hand.
3. Convert a Go table-driven test into a `testing.F` fuzz target with a model, then run it.

## Further reading

- OSS-Fuzz-gen: <https://github.com/google/oss-fuzz-gen>
- Fuzz Introspector: <https://github.com/ossf/fuzz-introspector>
- cargo-fuzz: <https://github.com/rust-fuzz/cargo-fuzz>
- Atheris: <https://github.com/google/atheris>
- Jazzer: <https://github.com/CodeIntelligenceTesting/jazzer>
