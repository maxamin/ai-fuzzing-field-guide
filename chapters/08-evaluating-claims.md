# Evaluating AI Fuzzing Claims Honestly

AI fuzzing papers and product pages are full of large percentage improvements. Many do not survive contact with a fair benchmark. This chapter is a checklist for your own experiments and for reading other people's.

## The Klees et al. checklist (still the standard)

"Evaluating Fuzz Testing" (CCS 2018) set the rules. In short:

- **Multiple trials**, at least 10 and preferably 20, per configuration.
- **Long enough runs**: 24 hours for coverage claims. Many techniques win at 1 hour and lose at 24.
- **Statistical tests**: Mann-Whitney U for differences, Vargha-Delaney Â12 for effect size.
- **Ground-truth bugs**, not crash counts. Use Magma or known CVEs.
- **Same seeds** and **same baseline configuration** (e.g. AFL++ with CMPLOG, not vanilla AFL 2.52b).

## AI-specific pitfalls

1. **Training data contamination.** If the target and its bugs are older than the model's training cutoff, the model may *remember* the crashing input or the fixing commit. Use targets or bugs that appeared after the cutoff, or synthetic bugs.
2. **Uncounted compute.** An LLM-assisted run uses GPU time. Compare at equal cost, or at least report the cost.
3. **Human-in-the-loop leakage.** Prompts tuned on the evaluation targets are a form of overfitting. Freeze prompts before evaluation.
4. **Nondeterminism.** Model outputs vary between runs and API versions. Record model ID, temperature and date, and cache responses for reproducibility.
5. **Harness bugs counted as findings.** See Chapter 2. Report how many "bugs" were confirmed by maintainers.

## Benchmarks

| Benchmark | Use |
|---|---|
| Magma | ground-truth bugs in real libraries |
| FuzzBench | coverage comparisons, as a service |
| ProFuzzBench | network protocols |
| UniFuzz | real-world programs, CVE-based |
| SmartBugs / DeFiHackLabs | smart contracts (see the companion books) |

## A reporting template

```
Target set:     12 Magma targets, commit <hash>
Baseline:       AFL++ 4.x, CMPLOG + dict, same seeds
Treatment:      + LLM plateau breaker (model <id>, temp 0.2), max 1 call / 10 min
Trials:         20 x 24 h each
Compute:        CPU core-hours equal; LLM spend $X total
Metrics:        bugs triggered (Magma), median time-to-bug, edge coverage
Stats:          Mann-Whitney U, A12
Contamination:  bugs introduced after <model cutoff>? yes/no per bug
```

## Reading a paper or product claim

When you read "our AI fuzzer finds 3x more bugs", ask in this order:

1. **Bugs or crashes?** If crashes, ignore the number until deduplication is described.
2. **Confirmed by whom?** Maintainer acknowledgement, CVE assignment or a fixing commit is strong. "Potential vulnerabilities" is weak.
3. **Against which baseline, configured how?** Look for AFL++ version, CMPLOG, dictionaries and identical seeds.
4. **How long and how many times?** One 24-hour run per configuration is an anecdote.
5. **At what cost?** Total tokens or dollars, and whether the baseline got the equivalent CPU.
6. **Could the model have seen the answer?** Check target and bug dates against the model's training cutoff.

Schloegel et al. reviewed well over a hundred fuzzing papers from top venues and found that many fell short on these practices, especially trial counts, statistical testing and reproducibility. Expect the same in AI-fuzzing work, and hold your own write-ups to the higher bar.

## Worked example: a 10-trial comparison with statistics

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

You have a baseline (AFL++ with CMPLOG) and a treatment (the same plus an LLM plateau breaker). The aim is a defensible statement about coverage.

**Step 1 – run trials.** Use identical seeds, cores pinned with `AFL_NO_AFFINITY` unset, and one core per trial. Run 10 trials per configuration for 24 hours. Script it so every trial is launched the same way.

```bash
for t in $(seq 1 10); do
  timeout 24h afl-fuzz -i seeds -o runs/base/$t -c ./target_cmplog -- ./target @@ &
done
```

**Step 2 – collect final metrics.** AFL++ writes `fuzzer_stats` with `edges_found`. For bug-based metrics on Magma, use Magma's survival reports instead.

```bash
for d in runs/*/*; do
  echo "$(dirname $d | xargs basename),$(grep edges_found $d/default/fuzzer_stats | awk '{print $3}')"
done > edges.csv
```

**Step 3 – test.**

```python
import csv
from statistics import median
from scipy.stats import mannwhitneyu
rows = list(csv.reader(open("edges.csv")))
base = [int(v) for k, v in rows if k == "base"]
treat = [int(v) for k, v in rows if k == "treat"]
u, p = mannwhitneyu(treat, base, alternative="two-sided")
a12 = sum((t > b) + 0.5 * (t == b) for t in treat for b in base) / (len(treat) * len(base))
print(f"median base={median(base)} treat={median(treat)} p={p:.4f} A12={a12:.2f}")
```

**Step 4 – interpret.** A12 is the probability that a random treatment run beats a random baseline run. Values near 0.5 mean no practical difference; Vargha and Delaney's conventional thresholds put 0.56, 0.64 and 0.71 at small, medium and large effects. A significant p-value with A12 of 0.58 is a small effect, and it may not justify the model cost.

**Step 5 – report** with the template above, including model ID, prompt version, total spend and every trial's raw number in an appendix.

## Common pitfalls

- **Reporting means of skewed distributions.** Fuzzing outcomes are heavy-tailed. Report medians and intervals.
- **Cherry-picked targets.** Choose the benchmark set before running, and report every target, including losses.
- **Unequal compute.** If the treatment uses an extra machine for model orchestration, give the baseline an extra core.
- **Uncached model outputs.** Without saved responses, nobody (including you) can rerun the experiment. Store prompts and outputs.
- **Contaminated ground truth.** Old CVEs in well-known projects are likely in training data. Prefer bugs introduced after the cutoff or synthetic bugs (see **synthetic-vulnerability-datasets**).

## Exercises

1. Pick an AI-fuzzing paper and check it against the checklist. Which items are missing?
2. Run any two fuzzer configurations for 1 h and 24 h on the same target. Does the ranking flip?
3. Design a contamination-free benchmark by injecting synthetic bugs into a recently released library.

## Further reading

- Klees et al., "Evaluating Fuzz Testing", CCS 2018
- Magma: Hazimeh et al., SIGMETRICS 2021, <https://hexhive.epfl.ch/magma/>
- FuzzBench: <https://github.com/google/fuzzbench>
- Schloegel et al., "SoK: Prudent Evaluation Practices for Fuzzing", IEEE S&P 2024
