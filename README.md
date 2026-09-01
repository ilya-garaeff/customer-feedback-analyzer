# Multilingual customer feedback analyzer

Classifies a mixed-language feedback CSV into themes with a 0–5 severity score, then ranks themes by total severity rather than volume. Written to test one specific idea: that a severity rubric written natively per language scores understated languages more accurately than an English rubric applied to everything.

## Run

```bash
pip install -r requirements.txt

make serve    # http://127.0.0.1:8000, click "Load sample"
make demo     # CLI, no API key needed
make verify   # 13 assertions
make eval     # scores the classifier against evals/gold.jsonl
make ablate   # same eval with the English rubric forced onto every language

export ANTHROPIC_API_KEY=...   # then re-run for real model output
```

## What the code does

`vocx/taxonomy.py` holds a 13-theme list, an anchored 0–5 severity scale, and a calibration paragraph per language (en, es, ru, pt, zh). Severity is defined as consequence to the customer, not word intensity.

`vocx/analyze.py` groups items by language so each API call carries exactly one calibration block, batches 8 per call, and parses the response into `Verdict` objects. `Verdict.validate()` checks theme membership, severity range and sentiment enum in code — no judge model. `report()` ranks themes by `count × mean_severity`, with a 1.25× multiplier for themes appearing in more than one language.

`vocx/cli.py` outputs Markdown or JSON and prints token usage and estimated cost.

`vocx/stub.py` is a keyword classifier used when no API key is present, so the pipeline and the eval harness run on a fresh clone.

## Interface

`make serve` runs a FastAPI app with one endpoint. Paste a CSV, get the ranked theme list.

Each theme draws two bars: how much is said about it, and what it costs. Where the second runs well past the first, a quiet theme is outranking a loud one, and the page says so in words rather than leaving you to read the pixels. That divergence is the only reason this tool ranks on severity instead of volume, so it is the thing the interface is built around.

Clicking a theme swaps the side rail to that theme's examples, original language first with the English gloss underneath — a native-speaking colleague needs the original to check the call. Items that failed schema validation are listed rather than dropped.

## Where things are

| File | Lines | What it holds |
| --- | --- | --- |
| `vocx/taxonomy.py` | ~90 | Theme list, severity anchors, the five calibration blocks, prompt builder |
| `vocx/analyze.py` | ~160 | Load, batch, classify, validate, roll up, render Markdown |
| `vocx/cli.py` | ~40 | Argument parsing, output |
| `vocx/server.py` | ~55 | FastAPI app, `POST /api/analyze` |
| `vocx/web/index.html` | ~230 | Full UI including the divergence bars |
| `vocx/llm.py` | ~130 | Anthropic wrapper, usage/cost accounting, offline replay |
| `vocx/stub.py` | ~70 | Offline keyword classifier |
| `evals/gold.jsonl` | 20 items | Gold theme + severity labels across five languages |
| `evals/run_eval.py` | ~110 | Per-language MAE, signed bias, theme accuracy, parity gap, ablation switch |
| `tests/smoke.py` | ~70 | 13 assertions across the pipeline, validation, ranking and the API |

## Evals

`evals/gold.jsonl` has 20 hand-labeled items, four per language, built as matched pairs: loud-but-minor items and quiet-but-severe items in each language, plus genuinely low-severity controls so a rubric cannot win by inflating everything.

`run_eval.py` reports three numbers per language — severity MAE, signed bias (negative means the system reads that language as calmer than it is), and theme accuracy — plus the parity gap between the best and worst language.

`--ablate` overwrites every calibration block with the English one. That is the control condition. **I have not yet run either condition against a live model, so I have no results to report.** The harness works and the offline stub scores against it; the comparison that would justify the per-language rubrics is the next thing to run, not something already established.

## Limits

- 20 eval items is too few to distinguish MAE differences below roughly 0.3. Real conclusions need ~50 per language and a second annotator.
- Language is read from a `lang` column, not detected. Code-switched text is routed to one calibration block and handled badly.
- `quote_original` is not verified to be a verbatim substring of the input.
- `BATCH_SIZE = 8` is a guess, not a tuned value.
- Sarcasm is unhandled and unmeasured.

## Model

Sonnet at temperature 0, configurable via `--model` and the `MODEL` env var. Temperature 0 because severity is meant to be a repeatable measurement. Cost estimates in `Usage` use public list prices and can be overridden with `PRICE_IN` / `PRICE_OUT`.
