# customer-feedback-analyzer
customer-feedback-analyzer — Turns a mixed-language feedback inbox into ranked product problems, with per-language accuracy evals
# termcheck
# Multilingual customer feedback analyzer

Turns a mixed-language feedback inbox into a ranked list of product problems — without treating quiet languages as happy customers.

```bash
pip install -r requirements.txt
python -m vocx.cli data/sample_feedback.csv          # runs with no API key
export ANTHROPIC_API_KEY=...                          # then re-run for real output
```

---

## The problem

Most feedback tooling is built and evaluated in English, then applied to everything else. That works until you notice that intensity is encoded differently in different languages:

| A customer writes | Word intensity | Actual severity |
| --- | --- | --- |
| "This is the worst onboarding I have ever seen" (US English) | very high | low — they found the API key in 20 minutes |
| "Отчёт не выгружается, приходится собирать вручную" (Russian) | flat | high — a team is rebuilding a report by hand every week |
| "Só uma sugestão: o relatório não abre" (Portuguese) | friendly | high — same blocker, wrapped in politeness |
| "建议优化一下导出功能" (Mandarin) | a suggestion | high — recurring manual cleanup after every export |

A sentiment score built on English intensity cues reads the bottom three rows as calmer than the top one. The roadmap that comes out of that inbox is a roadmap of the loudest market, not the most painful problem.

## What this does

1. Reads a CSV of feedback (`id, text, lang, source, segment`).
2. Classifies each item into a theme, a sentiment, and a 0–5 **severity** score — where severity means *consequence to the customer*, never word intensity.
3. Rolls the results into a priority ranking that weights total pain and boosts issues appearing in more than one language.
4. Emits Markdown for a weekly review, or JSON for a pipeline.

Output looks like this:

```
| Theme        | Items | Mean sev | Languages      | Score |
| integrations |     2 |     4.50 | ru:1 pt:1      | 11.25 |
| performance  |     3 |     3.67 | zh:2 en:1      | 13.76 |
| onboarding   |     4 |     2.25 | en:4           |  9.00 |
```

The English-only onboarding complaints are loud and numerous. They are not the top of the list.

## How it works

```
CSV ──▶ group by language ──▶ per-language prompt ──▶ batched classify ──▶ validate ──▶ roll up
             │                       │                       │                │
             │              native calibration        8 items/call     deterministic
             │              block, not a               (see costs)      schema checks
             │              translated one
             └── one calibration block per call, never mixed
```

**Model choice.** Sonnet, not Opus. The task is classification against a written rubric, not open-ended reasoning: on the eval set the accuracy difference does not pay for the price difference. Temperature 0 — this is a measurement, and the same input should produce the same number on Monday and Thursday. Haiku was tested and drops severity accuracy in the understated languages first, which is precisely the failure this project exists to prevent.

**Batching.** Items are grouped by language so each call carries exactly one calibration block. Eight per call: below that the per-call rubric overhead dominates the token bill, above that severity scores start drifting toward the batch mean.

**Validation before judging.** Theme membership, severity range and sentiment enum are checked in code, not by a second model. Deterministic checks are free, instant, and cannot themselves hallucinate. An LLM judge is only worth paying for on things code cannot check.

## The rubric

`vocx/taxonomy.py` holds a calibration block per language, each written by a native speaker of that language rather than translated from English. Each block describes how intensity is *encoded*, so the model anchors on consequence:

> **Russian** — understated and consequence-first: severity shows up as a flat description of what broke and what it cost, with no intensifiers. A calm sentence reporting that the team switched to a manual process is a 4 or 5. Explicit anger is rare and, when present, usually signals an already-lost account.

> **Spanish** — diminutives (`un problemita`, `un detalle`) soften real blockers, especially toward a vendor the writer wants to keep a good relationship with. Conversely `un desastre` is ordinary register and is not automatically a 5.

## Eval plan

20 gold-labeled items across English, Spanish, Russian, Portuguese and Mandarin, in `evals/gold.jsonl`. The set is built as matched pairs: loud-but-minor items and quiet-but-severe items in each language, plus controls that are genuinely low severity so the rubric cannot win by simply inflating everything.

```bash
python evals/run_eval.py            # native rubrics
python evals/run_eval.py --ablate   # English rubric applied to every language
```

Three metrics:

- **Severity MAE** per language — how far off the estimate is.
- **Bias** (signed error) — negative means the system reads that language as calmer than it is. This is the number that quietly deprioritises a market.
- **Parity gap** — worst-language MAE minus best-language MAE. A blended average hides this; a system that is excellent in English and mediocre in Russian looks fine on one number and still mis-ranks the roadmap.

The `--ablate` run is the control. If the native rubrics do not beat the English-only rubric, they are decoration and should be deleted. Run both and put your own numbers here:

| Condition | Overall MAE | Parity gap | Worst-language bias |
| --- | --- | --- | --- |
| Native rubrics | _run it_ | _run it_ | _run it_ |
| English-only (ablation) | _run it_ | _run it_ | _run it_ |

Numbers are left blank deliberately. They depend on the model version you run against, and a benchmark table you cannot reproduce is worse than no table.

## Cost and latency tradeoffs

| Choice | Effect | Why this way |
| --- | --- | --- |
| Batch 8 items/call | ~6× fewer calls than one-per-item | Rubric tokens are re-sent on every call; batching amortises them |
| Group by language | Slightly more calls than one mixed batch | A mixed batch has to carry five calibration blocks or none |
| Temperature 0 | No sampling diversity | Severity is a measurement; drift between runs is a bug |
| Deterministic validation | Zero extra tokens | Catches schema breakage that a judge would rubber-stamp |

Every run prints token counts and an estimated cost. A 500-item weekly inbox is a few cents and about a minute — cheap enough that the interesting constraint is label quality, not budget.

## Failure modes

| Failure | How it shows up | Mitigation |
| --- | --- | --- |
| Model reads politeness as satisfaction | Negative bias in `pt` / `zh` on the eval | The whole point of the calibration blocks; caught by the bias metric |
| Sarcasm | "Great, another outage" scored positive | Known gap. Not solved. Sarcasm cues are language-specific and the eval set is too small to measure it honestly |
| Code-switched text | Spanglish or Runglish routed to one calibration block | Currently routed by the declared `lang` column; mixed text is a known weak spot |
| Malformed JSON from the model | Parse failure | One repair retry, then the item is surfaced as a validation warning rather than silently dropped |
| Theme drift over time | Everything lands in `other` | `other` rate is visible in the report; a rising rate means the taxonomy needs a new category |
| Quote fabrication | A quote that is not in the source | `quote_original` must be verbatim; spot-check with the source text (an automated exact-substring check is the next thing to add) |

## UX decisions

- **Markdown by default, JSON on request.** The primary reader is a PM pasting a digest into a weekly review, not a service.
- **Original-language quote first, English gloss underneath.** A translated-only quote strips the evidence a native-speaking colleague needs to check the call.
- **Severity, not sentiment, drives the ranking.** Sentiment is reported because people ask for it; it is not what the priority score is built on.
- **Validation warnings are printed, never swallowed.** A digest that quietly dropped four items is worse than one that says it dropped them.
- **Runs without an API key.** A reviewer can clone this and see the pipeline in ten seconds. Offline mode uses a keyword stub in `vocx/stub.py` — it is deliberately naive, and on the eval set it fails hardest on exactly the understated languages, which is a decent illustration of the problem this repo is about.

## What I would do next

1. Exact-substring check on every `quote_original` — cheap, deterministic, closes the fabrication hole.
2. Expand the gold set to ~50 items per language with two independent annotators, and report inter-annotator agreement. Below that, MAE differences under ~0.3 are noise.
3. Language detection instead of a declared `lang` column, with a confidence threshold that routes ambiguous items to a mixed-calibration prompt.
4. Trend view: theme priority week over week, so a rising score triggers attention before it becomes a churn call.
