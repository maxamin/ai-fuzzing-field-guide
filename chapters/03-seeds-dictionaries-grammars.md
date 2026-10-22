# Seeds, Dictionaries and Grammars from Models

The quickest win from LLMs in fuzzing is often not harnesses but **better starting material**. A fuzzer that starts from 50 diverse, valid inputs gets deep much sooner than one starting from `"AAAA"`.

## Seed corpus generation

Prompt pattern:

> You are generating a fuzzing seed corpus for {format}. Produce {N} small, *valid* files that each exercise a *different* feature: {feature list from spec TOC}. Output each as a separate code block with a filename.

Then filter mechanically:

1. Validate with the reference parser and drop invalid files (or keep a small share of near-valid ones on purpose).
2. Run the corpus through the target with coverage and keep only seeds that add new edges (`afl-cmin`, or `-merge=1` in libFuzzer).
3. Minimise each one (`afl-tmin`).

For binary formats, ask the model to **write a generator script** (Python with `struct`) rather than emitting bytes. Models make arithmetic mistakes on offsets and lengths, while scripts are checkable and reproducible.

## Dictionaries

AFL++ and libFuzzer dictionaries list tokens such as keywords, magic numbers and delimiters. A model given the spec or the parser's source can produce a reasonable dictionary in seconds. Cross-check it against values extracted mechanically:

```bash
# tokens the binary actually compares against
strings -n 3 ./target | sort -u > strings.txt
# AFL++ can also auto-extract compare operands with CMPLOG / AUTODICT
```

Keep the union, drop tokens longer than 128 bytes, and escape non-printables as `\xNN`.

## Grammar inference

Grammar-based fuzzers (Nautilus, Grammarinator, Gramatron, and AFL++'s grammar mutator) need a context-free grammar. Writing one by hand for a real-world format takes days. Models can draft one from:

- the specification (RFC, ISO, or an ANTLR grammar of a related language),
- example files,
- the parser source code (a recursive-descent parser *is* a grammar).

Validation loop: generate 1,000 strings from the drafted grammar, parse them with the real parser, and feed the rejection messages back. Iterate until acceptance is above about 90%. Then **deliberately relax** the grammar (optional fields, unbounded repetition) because bugs live just outside the valid language.

## LLM as the generator itself (Fuzz4All style)

Instead of a grammar, the model produces inputs directly, guided by an evolving prompt ("generate a program that uses feature X in an unusual way"). It is expensive per input but very effective for **compilers, interpreters, query engines and anything whose input is code**, where validity is semantic, not just syntactic.

Cost control:

- Batch: ask for 20 inputs per call.
- Cache: dedupe by hash before execution.
- Mix: alternate LLM-generated inputs with cheap mutation of the best ones.

## Choosing between seeds, dictionaries, grammars and generation

These four techniques sit on a spectrum from cheap and generic to expensive and precise. Start at the cheap end and move up only when coverage plateaus.

| Technique | Model cost | Effect on fuzzer | Best for |
|---|---|---|---|
| Seed corpus | one-off, small | better starting points | any format with a reference parser |
| Dictionary | one-off, tiny | splices tokens into mutations | text protocols, keyword-heavy formats |
| Grammar | one-off, medium, iterative | structure-preserving mutation | languages, config formats, query DSLs |
| Model as generator | ongoing, large | semantically valid inputs | compilers, interpreters, SQL engines |

A useful rule: if `afl-cmin` keeps fewer than 10% of model seeds, the model is producing near-duplicates and you should change the feature list in the prompt, not ask for more files.

## Worked example: seeds and a dictionary for a TOML parser

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

The target is any TOML library with a libFuzzer or AFL++ harness. The reference validator here is Python's standard `tomllib` (Python 3.11+).

**Step 1 – feature list.** Take the TOML specification's headings: basic strings, multi-line literal strings, integers with underscores, hex/octal/binary integers, floats with exponents, `inf`/`nan`, offset date-times, local dates, arrays, inline tables, arrays of tables, dotted keys.

**Step 2 – ask for a generator, not files.** Prompt: "Write a Python script that writes one small valid TOML file per feature in this list into `seeds/`, plus three files that combine features."

**Step 3 – validate mechanically.**

```python
# validate.py
import pathlib, tomllib, shutil
ok = pathlib.Path("seeds_ok"); ok.mkdir(exist_ok=True)
for p in pathlib.Path("seeds").iterdir():
    try:
        tomllib.loads(p.read_text())
        shutil.copy(p, ok / p.name)
    except (tomllib.TOMLDecodeError, UnicodeDecodeError) as e:
        print(f"reject {p.name}: {e}")
```

Expect a few rejections (models often get date-time edge cases wrong). Keep two or three rejected files in a separate `near_valid/` folder: they exercise error paths.

**Step 4 – minimise against the target.**

```bash
afl-cmin -i seeds_ok -o seeds_min -- ./toml_fuzz_afl @@
for f in seeds_min/*; do afl-tmin -i "$f" -o "tmin/$(basename $f)" -- ./toml_fuzz_afl @@; done
```

**Step 5 – dictionary.** Ask the model for tokens, then union with mechanically extracted ones:

```text
# toml.dict (AFL++ / libFuzzer format)
kw_true="true"
kw_false="false"
kw_inf="inf"
kw_nan="nan"
arr_tbl="[["
arr_tbl_end="]]"
ml_basic="\"\"\""
ml_lit="'''"
dt="1979-05-27T07:32:00Z"
hex="0x"
```

**Step 6 – compare.** Run three configurations for one hour each, at least five trials: empty seed, model seeds, model seeds plus dictionary (`afl-fuzz -x toml.dict`). Compare final edge counts from `fuzzer_stats` (`edges_found`). Expect the biggest jump from seeds; dictionaries matter more once the corpus saturates.

## Grammar quality checks

When a drafted grammar reaches high acceptance, check two more things before trusting it. First, **coverage of the grammar itself**: count how many production rules were used by at least one of the 1,000 samples. Unused rules are either unreachable (a bug in the grammar) or too deep for the generator's depth limit. Second, **coverage of the parser**: measure parser line coverage from the samples. A grammar that is 99% accepted but covers 30% of the parser is too narrow.

## Common pitfalls

- **Letting the model emit binary files directly.** Lengths and checksums will be wrong. Ask for a script and run it.
- **Keeping every model seed.** Large redundant corpora slow every queue cycle. Always run `afl-cmin` or libFuzzer `-merge=1`.
- **Over-valid grammars.** A grammar tuned to 100% acceptance stops exercising error handling, where many bugs live.
- **Huge dictionary entries.** AFL++ ignores or truncates very long tokens, and big dictionaries dilute useful ones. Keep entries short and specific.
- **Assuming the spec matches the implementation.** Validate against the target parser, not only the reference one, and look at disagreements: they are often bugs.

## Exercises

1. Generate a seed corpus for a TOML parser with and without a model. Compare edge coverage after 1 hour.
2. Infer a grammar for a small DSL from its parser source. Report the acceptance rate after each refinement round.
3. Measure how much of a model-generated dictionary already appears in `strings` output.

## Further reading

- AFL++ dictionaries and CMPLOG docs: <https://github.com/AFLplusplus/AFLplusplus/tree/stable/docs>
- Nautilus: Aschermann et al., NDSS 2019
- Grammarinator: <https://github.com/renatahodovan/grammarinator>
- Fuzz4All: arXiv:2308.04748
- The Fuzzing Book, "Fuzzing with Grammars": <https://www.fuzzingbook.org/html/Grammars.html>
