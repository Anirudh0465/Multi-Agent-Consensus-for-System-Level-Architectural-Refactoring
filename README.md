# Multi-Agent Consensus for System-Level Architectural Refactoring

A multi-agent consensus framework using **Gemma 4** to automate complex, multi-file code refactoring — submitted to the [Gemma 4 Developer Agent Paper Track](https://www.kaggle.com/competitions/gemma-4-developer-agent-paper-track) on Kaggle.

## Problem

Single-pass LLM coding tools handle small edits well, but system-level refactors that span many files often break imports, interfaces, and tests. A model sees one slice of the codebase and makes changes that are locally plausible but globally inconsistent.

## Approach

Instead of trusting one model's output, we run a committee of three **Gemma 4 12B-it** agents that must reach consensus before a change is accepted:

| Agent | Role |
|---|---|
| **Architect** | Reads the repository, proposes a step-by-step refactoring plan, decides which files change and how |
| **Developer** | Implements the plan as concrete multi-file patches |
| **QA** | Runs real tests (`pytest`) and static checks (`compileall`), then reviews the architecture |

The Developer ⇄ QA loop repeats until QA signs off or a retry limit (3 rounds) is hit. Only a patch that passes both automated tests and architectural review is accepted.

```
Repo → Architect → Developer ⇄ QA → Accepted Patch
                   (retry loop)
```

## Experiments

We compare a **single-agent baseline** (one Gemma call, no feedback) against the **multi-agent committee** on four benchmark refactoring tasks:

| Task | Description | Min Files |
|---|---|---|
| `payments_split` | Split monolithic payment handler into DB, logic, routing layers | 3 |
| `inventory_layers` | Split inventory manager into storage, logic, command layers | 3 |
| `report_pipeline` | Split report processor into parse, validate, aggregate, render stages | 4 |
| `notifier_strategy` | Replace if/elif dispatch with strategy pattern (one class per channel) | 2 |

Each benchmark includes behavior-based tests that must pass after refactoring. The same QA step judges both methods.

### Metrics

- **Success rate**: tests pass AND code was split into the required number of files AND reviewer approved
- **Test pass rate** after refactor
- **Compilation/import success rate**
- **Structural errors** (broken imports, interface mismatches)
- **Cost**: tokens and time per run
- **Failure analysis**: categorized as `ok`, `tests_failed`, `compile_error`, `too_few_files`, `no_parseable_files`, or `review_not_approved`

## Repository Structure

```
├── .gitignore
├── README.md
├── LICENSE                              # Apache 2.0
└── gemma-4-multi-agent-consensus-demo.ipynb   # Main notebook (runs on Kaggle)
```

## How to Run

### On Kaggle (recommended)

1. Open the [public Kaggle notebook](#) *(link to be added)*
2. Attach **Gemma 4 12B-it** (Transformers variant) via **Add Input → Models**
3. Set accelerator to **GPU T4 ×2** and turn **Internet** on
4. **Run All** — cell 5 sanity-checks the benchmarks, cell 6 does a quick committee run, cell 7 runs the full experiment

### Locally

```bash
git clone https://github.com/Anirudh0465/Multi-Agent-Consensus-for-System-Level-Architectural-Refactoring.git
cd Multi-Agent-Consensus-for-System-Level-Architectural-Refactoring
pip install torch transformers accelerate bitsandbytes pytest pandas
```

You'll need a GPU with ≥16 GB VRAM and the Gemma 4 model weights.

## Key Results

Evaluation over **24 independent experiment runs** (4 tasks × 3 runs × 2 methods) on 2× NVIDIA T4 GPUs:

### Overall Performance (Mean over 12 runs per method)

| Method | Success Rate | Compiles | Tests Pass | Structural Errors | Mean Rounds | Mean Tokens | Mean Runtime |
|---|---|---|---|---|---|---|---|
| **Baseline** | 83.3% (10/12) | 100% | 83.3% | 0.0 | 1.00 | 2,203 | 151.5s (~2.5 min) |
| **Committee** | **91.7% (11/12)** | **100%** | **91.7%** | **0.0** | **1.42** | 9,661 | 388.6s (~6.5 min) |

### Task-by-Task Success Breakdown (3 runs each)

| Benchmark Task | Target Files | Baseline Success | Committee Success | Key Observation |
|---|---|---|---|---|
| `payments_split` | 3 | 66.7% (2/3) | **100.0% (3/3)** | Baseline failed run 1 (`tests_failed` due to return contract); Committee succeeded in all 3 runs. |
| `inventory_layers` | 3 | 100.0% (3/3) | 100.0% (3/3) | Both methods handled storage/logic/CLI separation reliably. |
| `report_pipeline` | 4 | 66.7% (2/3) | 66.7% (2/3) | 4-stage pipeline; Committee recovered across multi-round feedback (run 1 resolved after 24k tokens), with 1 run exhausting rounds. |
| `notifier_strategy` | 2 | 100.0% (3/3) | 100.0% (3/3) | Strategy pattern refactor completed cleanly across all runs. |

### Failure Analysis

| Method | `ok` (Success) | `tests_failed` | `compile_error` | `too_few_files` | `review_not_approved` |
|---|---|---|---|---|---|
| Baseline | 10 | **2** | 0 | 0 | 0 |
| Committee | **11** | **1** | 0 | 0 | 0 |

### Key Takeaways

1. **Compilation vs. Runtime Behavior**: Both baseline and committee achieved 100% compilation and zero syntax errors across all 24 runs. Single-pass LLMs easily produce code that compiles, but fail silently on behavioral contracts.
2. **Multi-Agent Error Recovery**: In `payments_split` (run 1) and `report_pipeline` (run 0 and run 1), the Developer ⇄ QA feedback loop detected broken test assertions and repaired them across multiple rounds (mean rounds: 1.42).
3. **The Compute Trade-off**: The multi-agent consensus loop uses ~4.4× more tokens and ~2.6× more runtime than the single-agent baseline, directly exchanging compute budget for higher architectural fidelity and semantic correctness.

## Technical Details

- **Model**: Gemma 4 12B-it, 4-bit quantized (NF4 + double quantization)
- **Hardware**: 2× NVIDIA T4 GPUs on Kaggle (model split across both)
- **Inference**: ~5–9 min per committee run, ~1–6 min per baseline run
- **Token cost**: Committee uses ~4–5× more tokens than baseline

## Limitations

- Benchmark repositories are small (50–100 lines of source). Results may not generalize to large codebases.
- Multi-agent runs cost significantly more tokens than single-agent.
- "Consensus" is iterative review (Developer ⇄ QA), not multi-voter. The Architect plans once.
- Tests check behavior, not that behavior is *identical* — subtle regressions may pass.
- Results are from a pilot study with limited runs, not a large-scale statistical comparison.

## License

This project is licensed under the [Apache License 2.0](LICENSE).

## Acknowledgements

- [Google DeepMind](https://deepmind.google/) for the Gemma 4 model family
- [Kaggle](https://www.kaggle.com/) for GPU compute and the competition platform