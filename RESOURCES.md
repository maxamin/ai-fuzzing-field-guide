# Resources

## Tools
- [AFL++](https://github.com/AFLplusplus/AFLplusplus) and [LibAFL](https://github.com/AFLplusplus/LibAFL): the main research-grade fuzzing platforms
- [OSS-Fuzz-gen](https://github.com/google/oss-fuzz-gen): LLM harness generation at scale
- [Fuzz Introspector](https://github.com/ossf/fuzz-introspector): reachability and blocker analysis
- [Fuzz4All](https://github.com/fuzz4all/fuzz4all): LLM-as-generator universal fuzzing
- [ChatAFL](https://github.com/ChatAFLndss/ChatAFL): LLM-guided protocol fuzzing
- [AFLNet](https://github.com/aflnet/aflnet): stateful network fuzzing
- [syzkaller](https://github.com/google/syzkaller): kernel fuzzing (the KernelGPT target)
- [CASR](https://github.com/ispras/casr): crash triage
- [Echidna](https://github.com/crytic/echidna), [Medusa](https://github.com/crytic/medusa), [ItyFuzz](https://github.com/fuzzland/ityfuzz): smart contract fuzzers

## Papers
- Fuzz4All (ICSE 2024), arXiv:2308.04748
- TitanFuzz (ISSTA 2023), arXiv:2212.14834
- ChatAFL (NDSS 2024)
- KernelGPT, arXiv:2401.00563
- Evaluating Fuzz Testing (CCS 2018)
- SoK: Prudent Evaluation Practices for Fuzzing (IEEE S&P 2024)

## Learning
- [The Fuzzing Book](https://www.fuzzingbook.org)
- [AFL++ docs](https://aflplus.plus/docs/)
- [AIxCC](https://aicyberchallenge.com): finalist cyber reasoning systems and write-ups

## More tools
- [CASR](https://github.com/ispras/casr): crash reports, clustering and exploitability hints (`casr-afl`, `casr-libfuzzer`, `casr-cli`)
- [rr](https://rr-project.org): record and replay debugging for root-cause analysis
- [ProFuzzBench](https://github.com/profuzzbench/profuzzbench): network protocol fuzzing benchmark with Docker targets
- [Magma](https://hexhive.epfl.ch/magma/): ground-truth bug benchmark for fuzzers
- [FuzzBench](https://github.com/google/fuzzbench): coverage and bug benchmarking service
- [Atheris](https://github.com/google/atheris), [Jazzer](https://github.com/CodeIntelligenceTesting/jazzer), [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz): language-specific fuzzing engines
- [Grammarinator](https://github.com/renatahodovan/grammarinator): ANTLR-grammar-based test generation
- [Gambit](https://github.com/Certora/gambit): Solidity mutation generator for checking invariant suites
- [Foundry](https://github.com/foundry-rs/foundry): `forge` invariant and fuzz testing for Solidity
- [Buttercup](https://github.com/trailofbits/buttercup): Trail of Bits' open-sourced AIxCC cyber reasoning system

## More papers
- Lyu et al., "MOpt: Optimized Mutation Scheduling for Fuzzers", USENIX Security 2019
- Böhme et al., "Coverage-based Greybox Fuzzing as Markov Chain" (AFLFast), CCS 2016
- Aschermann et al., "REDQUEEN: Fuzzing with Input-to-State Correspondence", NDSS 2019
- Fioraldi et al., "LibAFL: A Framework to Build Modular and Reusable Fuzzers", CCS 2022
- Hazimeh et al., "Magma: A Ground-Truth Fuzzing Benchmark", SIGMETRICS 2021
- Vargha and Delaney, "A Critique and Improvement of the CL Common Language Effect Size Statistics", 2000 (the Â12 statistic)
- Deng et al., "Large Language Models are Edge-Case Fuzzers" (FuzzGPT), ICSE 2024

## Blogs and write-ups
- Google Security Blog posts on AI-powered fuzzing with OSS-Fuzz-gen (2023–2024)
- AIxCC final results and team write-ups (DEF CON 33, August 2025): <https://aicyberchallenge.com>

## Companion books in this wiki
- **synthetic-vulnerability-datasets**: contamination-free ground truth for evaluating fuzzers
- **automated-vulnerability-repair**: what happens after a crash is triaged
- **llm-smart-contract-vuln-research**: invariant and property work for contracts
