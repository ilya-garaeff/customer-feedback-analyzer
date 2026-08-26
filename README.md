# customer-feedback-analyzer
customer-feedback-analyzer — Turns a mixed-language feedback inbox into ranked product problems, with per-language accuracy evals
# termcheck

A small, finished product: paste a glossary, a source text and a translation, get back every place the deliverable breaks its own terminology. Runs as a web page or a CLI.

```bash
pip install -r requirements.txt
python -m termcheck.server                      # http://127.0.0.1:8000, click "Load sample"
python -m termcheck.cli data/sample_source.txt data/sample_target.txt \
  -g data/glossary_ru_en.csv                    # no API key required
```

---

## Why this one

The other projects in this portfolio are tools. This is a product, and it was chosen against a filter:

| Test | This product |
| --- | --- |
| Is the pain acute and self-evident? | A client-returned deliverable over one wrong term is a real, expensive event in translation work. Nobody needs the problem explained. |
| Is there urgency, or is it nice-to-have? | Terminology gets checked at 1am before a delivery deadline, or it does not get checked. |
| Is onboarding near-zero? | Three text boxes. No account, no upload, no project setup, no CAT tool integration. |
| Is there a distribution edge? | I work as a Russian–English interpreter. I have the problem, and I know where the people who share it are. |

I built the thing I am the customer for. That is the whole thesis.

## What it checks

**Glossary compliance, per aligned segment.** If the source segment contains an approved term and the matching target segment contains no approved rendering of it, that is an error. Segment scoping matters more than it sounds: check the whole document at once and a correct rendering three sentences away silently clears a genuine miss. That exact bug appeared during development and is now what the alignment step exists to prevent.

**Internal consistency.** The same source term rendered two different approved ways in one deliverable. Both are correct in isolation; alternating between them is what a reviewer sends back.

**Numbers.** Present in source and absent in target, or the reverse. `1 500,00`, `1,500.00` and `1500` all compare equal, so formatting conventions do not generate noise.

**Script leakage.** Cyrillic left in an English target, CJK in a Latin one. Cheap to detect, embarrassing to ship.

## The design decision worth defending

The model is called once, on one question, on a subset of flags.

Everything above is regular expressions and set arithmetic. It runs in milliseconds, costs nothing, and cannot hallucinate. A changed number is a hard error that code catches perfectly; asking a language model to find it would be slower, more expensive, and *less* reliable.

The model is asked exactly one thing that code does badly:

> The approved term is **поставщик услуг**. The target says **поставщику услуг**. Same term, dative case, and the string comparison fails at the last character.

Stemming handles some of this and gets Russian aspect pairs and Mandarin measure words wrong. So flags that fail the literal pass — and only those — go to the model with a narrow question: is an acceptable inflected variant present, or is this a real deviation? A typical document sends two or three flags rather than every sentence.

The adjudication prompt is written to be unhelpful on purpose. It states that a false clear costs a client relationship and a false flag costs ten seconds, and it offers `unclear` as a first-class verdict. A model asked "is this okay?" says yes. A model told what each kind of mistake costs picks the cheaper mistake.

`--no-model` turns the model pass off entirely. Everything still runs, with more false positives, at zero cost — a real mode, not a fallback.

## Model choice

Sonnet, temperature 0. The task is a bounded linguistic judgment against a written rule, one to three times per document. Haiku was tested first because the volume argument favours it, and it accepted paraphrases as inflected variants — the exact false-clear the tool exists to prevent. Opus adds nothing measurable here: morphological variant recognition is not where extra capability shows up.

## Eval plan

Ground truth by construction. Start from a target that is known clean, inject one specific defect, check whether it is reported.

```bash
python evals/run_eval.py             # 11 cases
python evals/run_eval.py --no-model  # deterministic only, for comparison
```

Seven seeded defects — an unapproved synonym, a wrong legal term of art, a dropped term, a changed number, a dropped number, an inconsistent rendering, an untranslated fragment. Four cases that must stay clean — the unmodified text, two harmless rewordings, and one substitution of an accepted alternative, which should produce a `note` and never an error.

The clean cases carry more weight than the seeded ones. A checker that flags everything has perfect recall and is worthless, because the reviewer stops reading it in week two. False positives are what kills this category of tool, so the harness fails on a single one.

The eval also earns its keep as a regression suite: the substring bug where `provider` always looked present inside `service provider` — and made every document report a consistency problem it did not have — is now a permanent case.

## Cost and latency

| Path | Latency | Cost |
| --- | --- | --- |
| Deterministic checks | <10 ms for a 2,000-word document | 0 |
| Model adjudication | ~2 s, one call | fractions of a cent |

A working linguist checking six documents a night pays cents a month. That is the point of pushing everything possible into the free layer: the tool has to be cheap enough that using it is never a decision.

## Failure modes

| Failure | How it shows up | Mitigation |
| --- | --- | --- |
| Segment misalignment | One merged or split sentence shifts every later pair | Count mismatch is reported as a warning and the tool falls back to whole-document checking rather than producing confidently wrong segment numbers |
| Multi-word terms crossing a sentence boundary | Missed entirely | Known gap, unhandled |
| A glossary that is itself wrong | Confident enforcement of a bad term | Out of scope, and worth saying so plainly — this tool enforces a glossary, it does not validate one |
| Homographs | A term flagged in a segment where the source word means something else | Real false-positive source; the model pass catches some, not all |
| Numbers written as words | "thirty days" against "30 days" | Not handled. Would need per-language numeral parsing |
| Very long documents | Alignment drift compounds | Practical ceiling around 200 segments before manual splitting is better |

## UX decisions

- **Findings are ranked by severity, then position.** A linguist at 1am reads from the top and stops when they run out of time, so the first thing on screen has to be the thing that gets the deliverable returned.
- **The bilingual tether.** Clicking a finding highlights the term in the source *and* every candidate rendering in the target at the same time. A terminology decision is never about one side of the page, and making the reader hunt for the counterpart in the other column is where the tool would have lost its time saving.
- **Three severities, and `note` is not a bug.** Using an accepted alternative instead of the preferred term is worth knowing and is not worth blocking. Collapsing everything into "problem" is how a checker teaches people to ignore it.
- **The empty state states what was actually verified** — "every glossary term in the source has an approved rendering in the matching target segment, and the numbers agree" — rather than a green checkmark. A reviewer needs to know the scope of the reassurance.
- **`--fail-on` for CI.** So a terminology check can block a delivery pipeline rather than being a step someone remembers.
- **"Load sample" says two terms are wrong on purpose.** A demo that looks clean teaches nothing.
- **Monospace for terms throughout.** Terms are data, and in a bilingual interface an exact glyph is the thing being judged.

## What I would do next

1. **Ship it to ten interpreters and watch.** Everything below this line is a guess until that happens.
2. **`.docx` and `.sdlxliff` input.** Nobody's deliverable is a plain text file. This is the single largest gap between "works" and "used".
3. **Glossary import from TBX and CSV exports** of the tools people already keep terminology in.
4. **Term suggestion mode**: propose glossary entries from a corpus of past accepted work, so a freelancer with fifteen years of files gets a glossary without writing one.
5. **Number-word parsing** for the languages this is aimed at.
