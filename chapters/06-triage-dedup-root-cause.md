# Triage, Deduplication and Root Cause with AI

A long campaign produces thousands of crashing inputs. The work that matters is turning them into a handful of correctly prioritised, well-explained bug reports. That is mostly reading, which models can help with, as long as the evidence comes from tools.

## Pipeline

```
crashes ──► reproduce (3x, fresh process) ──► sanitizer report
         ──► stack-hash bucketing (top N frames)
         ──► minimise input (afl-tmin / -minimize_crash)
         ──► LLM: classify, explain, draft report
         ──► human: confirm, severity, disclose
```

## Deduplication

Stack-hash bucketing over the top 3–5 frames is the baseline. It over-splits (inlining differences) and under-splits (one bad length check reached from many callers). Improvements:

- Bucket by **the first frame inside the project**, not inside libc or the allocator.
- For heap bugs, include the **allocation and free stack** from ASan.
- Use a model to merge buckets: give it two reports and ask whether they share a root cause, *requiring it to cite lines*. Accept merges only with a cited shared faulty line.

## Classification prompts that work

Give the model the sanitizer output, the minimised input as hex, and the source of the top three project frames. Ask for structured output:

```json
{
  "bug_class": "heap-buffer-overflow-read",
  "root_cause_line": "src/parse.c:412",
  "attacker_control": "length field at offset 6 controls read size",
  "exploitability_notes": "read only; info leak possible",
  "confidence": "medium",
  "evidence": ["ASan: READ of size 64", "len unchecked against buf_size at L405"]
}
```

Reject outputs whose `root_cause_line` does not exist or is not in the stack. This one mechanical check removes most hallucinated explanations.

## Root cause with record/replay

`rr` records an execution so you can step backwards. A useful agent loop: the model proposes a watchpoint ("where was `len` last written?"), the tool runs `rr` with `reverse-continue`, and the result goes back to the model. The model plans, the debugger provides ground truth.

## Patch suggestion, carefully

AIxCC systems generated patches automatically. For your own triage, treat patches as **explanations**: a minimal patch that makes the crash go away *and* keeps the test suite passing is strong evidence the root cause is understood. Do not ship it without review. Models like to "fix" by adding a bounds check at the crash site rather than at the faulty length calculation.

## Severity and exploitability signals from tools

Models are tempted to guess severity from a bug-class name. Prefer signals produced by tools:

| Signal | Tool | What it suggests |
|---|---|---|
| WRITE vs READ | ASan report | writes are usually more severe |
| Access size and offset from allocation | ASan "located N bytes after" | small fixed offsets vs attacker-sized overflows |
| Heap-use-after-free with alloc/free stacks | ASan | lifetime bug; check if the freed object holds pointers |
| Reachability from a public entry point | reproduce through CLI or parser API | real attack surface |
| Exploitability class | CASR (`casr-san`) | rough triage (EXPLOITABLE / PROBABLY_EXPLOITABLE / NOT_EXPLOITABLE) |
| Fixed in later version | bisect with `git bisect run` | already known; check advisories |

Feed these signals to the model as facts, and ask it to write the explanation, not to decide the facts.

## Worked example: from a crash directory to a draft report

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

Assume an AFL++ run left crashes in `out/default/crashes/` and you have an ASan build `./target_asan` that takes a file argument.

**Step 1 – reproduce three times in fresh processes.**

```bash
mkdir -p repro
for c in out/default/crashes/id:*; do
  ok=0
  for i in 1 2 3; do ./target_asan "$c" 2> /tmp/asan.txt >/dev/null || ok=$((ok+1)); done
  [ "$ok" -eq 3 ] && cp "$c" repro/
done
```

Flaky crashes go to a separate list: they often mean uninitialised memory or global state, which is still interesting but needs MSan or a different harness.

**Step 2 – bucket with CASR.** `casr-afl` reads the AFL++ output directory (including the target command line recorded in `fuzzer_stats`), runs each crash, writes `.casrep` reports and clusters them:

```bash
casr-afl -i out -o casr-out
casr-cli casr-out/cl1/*.casrep | head    # inspect one report
```

`casr-out/` then contains clusters (`cl1/`, `cl2/`, …) each with reports and inputs. On a typical campaign, thousands of crashing inputs collapse to tens of clusters.

**Step 3 – minimise one input per cluster.**

```bash
afl-tmin -i casr-out/cl1/<input> -o min/cl1.bin -- ./target_asan @@
```

**Step 4 – prompt for structured output.** Provide the `.casrep` (or raw ASan text), `xxd min/cl1.bin | head -20`, and source of the top three project frames. Ask for the JSON shown earlier.

**Step 5 – mechanical checks.**

```python
import json, os, re
r = json.load(open("cl1.json"))
path, line = r["root_cause_line"].rsplit(":", 1)
assert os.path.exists(path), "hallucinated file"
assert int(line) <= sum(1 for _ in open(path)), "line out of range"
stack = open("cl1.asan.txt").read()
assert os.path.basename(path) in stack, "root cause not in stack: needs human review"
```

The last check is a heuristic: real root causes are sometimes outside the stack (for example, where a length was computed). Route those cases to a human rather than rejecting them.

**Step 6 – human confirmation and disclosure.** A person reads the report, confirms severity, and files it through the project's security policy.

## Common pitfalls

- **Triaging from non-reproducible crashes.** Always reproduce in a fresh process before spending model or human time.
- **Over-trusting stack hashes.** Inlining and different optimisation levels change frames. Bucket on symbolised, project-internal frames.
- **Letting the model decide severity.** Use sanitizer facts and reachability. The model writes; tools decide.
- **Sending proprietary crash data to external APIs.** Check data-handling agreements, or use a local model for confidential code.
- **Fixing at the crash site.** The faulty computation is often upstream. Check patches against a variant of the input that reaches the same bug by another path.

## Exercises

1. Take 200 crashes from a campaign. Compare stack-hash buckets with model-merged buckets against your manual ground truth.
2. Build the JSON-schema classifier above with the line-existence check, and measure the rejection rate.
3. Use `rr` to find where a corrupted length originates. Then try the agent loop.

## Further reading

- AddressSanitizer: <https://github.com/google/sanitizers>
- rr: <https://rr-project.org>
- CASR (crash triage tool): <https://github.com/ispras/casr>
- AIxCC finalist CRS write-ups: <https://aicyberchallenge.com>
