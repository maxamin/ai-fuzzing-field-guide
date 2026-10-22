# Learned Mutation and Scheduling

Every fuzzer makes two decisions over and over: **which seed** to mutate next and **how** to mutate it. Both are bandit problems, and machine learning has attacked them for years, with mixed results. This chapter separates what works from what makes good paper figures.

## Scheduling as a multi-armed bandit

Treat each mutation operator (or each seed) as an arm and reward it for new coverage. MOpt (USENIX Security 2019) used particle swarm optimisation over AFL operators and is now an option in AFL++. Simple bandits (UCB1, Thompson sampling) work well because:

- rewards are sparse and non-stationary, so use a sliding window;
- the per-decision overhead must be nanoseconds.

```python
# Thompson sampling over mutators: reward = found new edge
import random
alpha = [1]*K; beta = [1]*K
def pick():
    return max(range(K), key=lambda i: random.betavariate(alpha[i], beta[i]))
def update(i, new_cov):
    if new_cov: alpha[i] += 1
    else:       beta[i]  += 1
```

## Neural program smoothing (and why to be careful)

NEUZZ trained a neural network to approximate the program's branch behaviour and used gradients to choose which bytes to mutate. Later independent re-evaluations (e.g. "Revisiting Neural Program Smoothing for Fuzzing", ESEC/FSE 2023) found that the advantages largely vanished under fair benchmarking against modern AFL++. That is the theme of this whole field: **check claims against a strong, well-configured baseline.**

## Where learned models do help

1. **Byte-importance from taint or cmplog** is cheaper and usually better than neural gradients. Learn from it, don't replace it.
2. **Input-to-state correspondence** (REDQUEEN) solves most magic-byte problems without ML.
3. **LLM plateau analysis**: give the model a function with an uncovered branch plus the nearest covered input, and ask for an input that satisfies the condition. It acts like a slow, sometimes-wrong concolic engine that handles semantics solvers cannot (string formats, checksums described in comments).
4. **Mutator synthesis**: ask a model to write a *custom mutator* (AFL++ custom mutator API, or LibAFL `Mutator` trait) for a format. The model runs once; the mutator runs a billion times.

## LibAFL as a research substrate

LibAFL is a Rust library for building fuzzers out of components: `Scheduler`, `Mutator`, `Feedback`, `Executor`. To test a learned scheduler you implement one trait and keep everything else identical to the baseline. That is how you get a fair comparison.

```rust
// sketch: a scheduler that delegates to an external policy
impl<S> Scheduler<I, S> for BanditScheduler { /* next(), on_add(), on_evaluation() */ }
```

## Power schedules you already have

Before building a learned scheduler, know what AFL++ already does. Its power schedules (`-p fast`, `explore`, `exploit`, `coe`, `lin`, `quad`, `rare`, `mmopt`, `seek`) come from AFLFast and later work. They assign more energy to seeds that exercise rarely hit paths. `-L 0` turns on MOpt. CMPLOG (`-c`) and `AFL_USE_CMPLOG`-built binaries handle most magic-value problems. A learned scheduler that does not beat `-p fast` with CMPLOG on the same seeds has not shown anything.

## Designing an LLM plateau breaker

A plateau breaker is an asynchronous helper that watches the campaign and intervenes only when coverage stops growing. A workable design:

1. **Detect** a plateau: no new edges for T minutes (30 is a reasonable start for a 24-hour run). Read `last_find` from AFL++'s `fuzzer_stats`.
2. **Select** a blocker: from a coverage report, pick an uncovered branch whose parent block *is* covered. Fuzz Introspector calls these "blockers" and ranks them by how much code sits behind them.
3. **Prompt** with the function source, the condition, and the covered input closest to it (hex for binary, text otherwise). Ask for a Python script that writes an input satisfying the condition.
4. **Check**: run the script, execute the output on a coverage build, and confirm the target branch is now covered. Discard if not.
5. **Inject** surviving inputs into the queue (AFL++ picks up files dropped into a `-F` foreign sync directory).

The model never touches the fast loop and every suggestion is verified by execution. Budget the helper, for example at most one call per ten minutes, and log every prompt and response.

## Worked example: Thompson sampling on AFL++ custom mutators

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

AFL++ supports Python custom mutators through `AFL_PYTHON_MODULE`. That is too slow for production, but fine for learning how a bandit behaves on a small target.

```python
# bandit_mut.py  -- load with AFL_PYTHON_MODULE=bandit_mut PYTHONPATH=.
import random
K = 3
alpha, beta = [1]*K, [1]*K
last = None

def init(seed):
    random.seed(seed)

def _flip(b):
    if b: b[random.randrange(len(b))] ^= 1 << random.randrange(8)
    return b
def _dup(b):
    if b:
        i = random.randrange(len(b)); b[i:i] = b[i:i+8]
    return b
def _magic(b):
    if b:
        i = random.randrange(len(b)); b[i:i+4] = random.choice([b"\xff\xff\xff\xff", b"\x00\x00\x00\x80"])
    return b
OPS = [_flip, _dup, _magic]

def fuzz(buf, add_buf, max_size):
    global last
    last = max(range(K), key=lambda i: random.betavariate(alpha[i], beta[i]))
    return OPS[last](bytearray(buf))[:max_size]
```

AFL++'s Python API does not hand the mutator a coverage reward per call, so a real implementation reads reward from a side channel, such as periodic diffs of `fuzzer_stats` or queue file names containing `+cov`. Do this the honest way: run the bandit and a uniform-random version of the same three operators, ten trials each, on the same Magma target, and compare.

```bash
AFL_PYTHON_MODULE=bandit_mut PYTHONPATH=. AFL_CUSTOM_MUTATOR_ONLY=1 \
  afl-fuzz -i seeds -o out-bandit -- ./target @@
```

Expected result on most targets: the bandit converges on one operator quickly, and its advantage over uniform choice is small. That is a useful lesson. The fixed overhead of a Python mutator usually costs more than the bandit gains, which is why MOpt lives in C inside AFL++.

## Common pitfalls

- **Weak baselines.** Comparing against AFL 2.52b or AFL++ without CMPLOG inflates every learned technique.
- **Stationary bandits.** Operator usefulness changes as the corpus grows. Use a sliding window or discounting.
- **Hidden overhead.** Python mutators, IPC to a model server, or GPU inference can halve exec/s. Report exec/s next to coverage.
- **Unverified model inputs.** A plateau-breaker suggestion that does not cover the target branch on replay is noise. Drop it.
- **Short runs.** Learned schedulers often look good at one hour and fade by 24.

## Exercises

1. Implement Thompson sampling over AFL++ custom mutators and run it against the default on three Magma targets with 10 trials each.
2. Find an uncovered branch in a target. Ask a model to produce a satisfying input from source, and compare against running with CMPLOG for 10 minutes.
3. Ask a model to write an AFL++ custom mutator for PNG chunks that keeps CRCs valid.

## Further reading

- MOpt: Lyu et al., USENIX Security 2019
- NEUZZ: She et al., IEEE S&P 2019; and the 2023 re-evaluation, Nicolae et al., ESEC/FSE 2023
- REDQUEEN: Aschermann et al., NDSS 2019
- LibAFL: Fioraldi et al., CCS 2022, <https://github.com/AFLplusplus/LibAFL>
- AFL++ custom mutators: <https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/custom_mutators.md>
