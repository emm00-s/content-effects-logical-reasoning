# When Content Overrides Form: Logical Invariance in Small Instruction-Tuned LLMs

Code, data and results for a controlled diagnostic benchmark that tests whether small instruction-tuned language models keep the same logical judgment when the same formal problem is wrapped in different narratives.

Each of 9 propositional templates is verified with the Z3 SMT solver and rendered in 4 narrative variants (familiar, belief-violating, artificial, abstract). Models answer in a counterbalanced four-option forced-choice format (True / False / Unknown / Paradox), and the pipeline adds confidence, positional-bias and attention diagnostics on top of accuracy.

This repository accompanies:

- **MSc thesis:** *Logical Reasoning and Failure Diagnostics in Small Instruction-Tuned Language Models*, Emmanuel Giuseppe Santoro. MSc in Human-Centered Artificial Intelligence (joint programme of the Universities of Milan, Milano-Bicocca and Pavia), A.Y. 2025-2026. Supervisor: Prof. Rafael Peñaloza. Co-supervisor: Prof. Giuseppe Primiero.
- **Paper:** E. G. Santoro and R. Peñaloza, *When Content Overrides Form: Diagnosing Logical Failures in Small Instruction-Tuned Language Models*. FOUSY 2026, Workshop on Foundations of Neuro-Symbolic Logical Reasoning (co-located with TriCoLore), Brixen/Bressanone, December 2026. CEUR Workshop Proceedings, to appear.

## Key results

Models: `Qwen/Qwen2.5-3B-Instruct` and `meta-llama/Llama-3.2-3B-Instruct`, 36 puzzles x 10 seeds, temperature 0.7, top-p 1.0. 

| | Qwen2.5-3B | Llama-3.2-3B |
| --- | --- | --- |
| Sampled accuracy | 34.2% [22.2, 46.9] | 25.6% [19.7, 32.5] |
| Argmax accuracy | 34.7% [23.1, 47.2] | 28.3% [19.2, 39.2] |
| Majority-vote accuracy (per puzzle) | 13/36 | 7/36 |
| Templates with the same majority answer across all 4 variants | 4/9 | 1/9 |
| Sampled accuracy, belief-violating variant | 21.1% | 11.1% |
| Sampled accuracy, artificial variant | 46.7% | 36.7% |

Both models are close to chance overall, and both are weakest when the narrative contradicts world knowledge (belief-violating variant). Full per-model summaries are in `results/analysis_qwen3b.txt` and `results/analysis_llama32_3b.txt`; comparative figures are in `figures/behavioral/`.

![Accuracy by narrative variant](figures/behavioral/fig_02_sampled_accuracy_by_narrative_variant.png)

## Repository structure

```text
puzzles/
  templates.json        9 formal templates: axioms, query, gold label, Z3 verification code
  puzzles.json          36 narrative variants (9 templates x 4 variants)
src/                    behavioral pipeline (Python package, run in module mode)
  dataset.py            loads and validates the benchmark, joins puzzles with templates
  solvers.py            re-derives every gold label with Z3
  prompt.py             builds A/B/C/D prompts under a per-trial option permutation
  model.py              forced-choice answer distribution from any Hugging Face causal LM
  run.py                runs puzzles x seeds with systematic counterbalancing, writes a CSV
  analyze.py            accuracy, majority vote, stability, consistency, positional and confidence diagnostics
analysis/
  make_behavioral_figures.py   regenerates figures/behavioral/
  run_benchmark_colab.py       end-to-end record of the Colab workflow
attention/
  attention_run.py      attention read at the answer-prediction step, aggregated over annotated spans
  selected_spans_*.json manual span annotations (region / role / tags) for P04, P05, P07, P08, P09
results/                final behavioral CSVs and per-model text summaries
figures/behavioral/     behavioral figures (see its README)
attention_outputs/      per-template attention outputs (JSON, aggregated CSVs, figures)
docs/reproducibility.md commands to regenerate every committed artifact
```

## Setup

Python 3.10 or later.

```bash
pip install -r requirements.txt
```

All `src` scripts must be run from the repository root in module mode (`python -m src.run`), because the package uses relative imports.

`meta-llama/Llama-3.2-3B-Instruct` is gated: accept its license on Hugging Face and authenticate (`huggingface-cli login` or the `HF_TOKEN` environment variable) before running it. Full runs require a GPU.

## Usage

Validate the dataset and the formal labels (CPU, a few seconds):

```bash
python -m src.dataset   # should load 36 puzzles
python -m src.solvers   # all 9 declared gold labels must match Z3
```

Run the behavioral benchmark:

```bash
python -m src.run --model-name Qwen/Qwen2.5-3B-Instruct --seeds 10 --temperature 0.7 --top-p 1.0 --tag qwen3b
python -m src.run --model-name meta-llama/Llama-3.2-3B-Instruct --seeds 10 --temperature 0.7 --top-p 1.0 --tag llama32_3b
```

Analyze a results CSV and regenerate the figures:

```bash
python -m src.analyze results/<RESULT_FILE>.csv
python analysis/make_behavioral_figures.py
```

Run the attention analysis for one template (example: P04):

```bash
python attention/attention_run.py \
  --template-id P04 \
  --models qwen llama \
  --qwen-csv results/results_qwen3b_T0.7_topP1.0_n10_20260513_114931.csv \
  --llama-csv results/results_llama32_3b_T0.7_topP1.0_n10_20260513_154349.csv \
  --spans-json attention/selected_spans_P04.json \
  --last-n-layers 4 \
  --output-dir attention_outputs/P04
```

See [`docs/reproducibility.md`](docs/reproducibility.md) for the exact commands behind every committed file.

## Method notes

**Forced choice.** The answer distribution is read from the next-token logits at the answer step, aggregated over tokenization variants of each option letter and renormalized over A/B/C/D. Reported probabilities are temperature-scaled (T = 0.7), i.e. they describe the same distribution the sampled answer is drawn from. `max_prob` and `entropy_nats` are option-level diagnostics, not model-level uncertainty estimates.

**Counterbalancing.** For each (puzzle, seed) trial the mapping of the four labels to A/B/C/D is drawn from a deterministic schedule over all 24 permutations, with uniform coverage overall (15 trials per permutation). Coverage is not balanced within each puzzle: each puzzle sees 10 consecutive permutations, so the correct answer can sit on the same letter in up to 6 of its 10 trials. Per-puzzle measures (majority vote, stability, cross-variant consistency) should be read with this in mind.

**Attention.** Attention is averaged over heads and the last 4 layers, read at the last prompt token, and aggregated over manually annotated spans. It is a descriptive, correlational diagnostic and is not interpreted as a causal explanation of the answer. `attention_mean_per_token` is the primary cross-variant measure; `attention_sum` scales with span length and is reported only descriptively.

**Scope.** This is a small controlled diagnostic dataset, not a broad benchmark of logical reasoning.

## Citation

```bibtex
@inproceedings{santoro2026content,
  author    = {Santoro, Emmanuel Giuseppe and Pe{\~n}aloza, Rafael},
  title     = {When Content Overrides Form: Diagnosing Logical Failures in Small Instruction-Tuned Language Models},
  booktitle = {Proceedings of the Workshop on Foundations of Neuro-Symbolic Logical Reasoning (FOUSY 2026)},
  series    = {CEUR Workshop Proceedings},
  year      = {2026},
  note      = {To appear}
}
```

## AI assistance

AI coding assistants (Claude Code) were used in developing this repository. All changes were reviewed by the authors. Research design, puzzle construction, annotation and interpretation of results are the authors'.

## License

## License

The code is released under the MIT License (see `LICENSE`). The benchmark data (`puzzles/`), the span annotations (`attention/selected_spans_*.json`) and the released results and figures (`results/`, `figures/`, `attention_outputs/`) are released under CC BY 4.0.
