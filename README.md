# TakeMeter: An r/ufc Discourse Classifier

A fine-tuned text classifier that sorts r/ufc posts and comments by *what kind of
discourse* they are — reasoned analysis, bare opinion, a question, or a joke. The
goal is to measure discourse quality in a community where one fight produces wildly
different kinds of comments in the same thread.

> **Note on scope changes.** This project began as a **3-label, 200-example** design.
> During annotation the taxonomy was revised to **4 labels and 220 examples** (a
> `question_prompt` label was split out — see [Spec Reflection](#spec-reflection));
> `planning.md` has been updated to document that revised design and the reasoning behind
> it. This README documents the dataset and model **as actually built**.

---

## Community Choice

All data is from **r/ufc**. It is a strong fit for a discourse-quality classifier
because a single fight produces totally different kinds of comments side by side in the
same thread: technical breakdowns of *why* something happened (footwork, distance, hand
position), bare opinions with no support ("he's washed," "Ilia smokes him"), genuine
questions ("What is Gaethje's plan?"), and pure reaction (memes, one-liners). What
separates these is *how much reasoning sits behind the claim* and *what the post is
trying to do* — a real distinction, but a subtle one, so the model has to read intent
rather than keywords. The sub is also big and messy enough to source plenty of each
type.

---

## Label Taxonomy

Four mutually-exclusive labels. Every post gets exactly one; the decision rules below
handle ties. No catch-all "other" bucket is needed.

### `fight_analysis`
A post that explains a fight, fighter, matchup, strategy, judging, technique, or career
situation using **specific, checkable reasoning** — a named technique, tactical
adjustment, matchup detail, or fight history.
- *"It's leg kicks. Ilia does not check them as much as other fighters do."* (ufc_102)
- *"Rockhold's problem was that he would back out straight with his chin up and hands down when pressed and he had a hard time seeing the left from a bladed retreating position."* (ufc_135)

### `hot_take`
A strong UFC opinion, prediction, ranking, or fighter judgment with **little, vague, or
reputation-based support** — it asserts rather than argues.
- *"Topuria is gonna have a bad night."* (ufc_153)
- *"Belal is overhated, hes actually very good person"* (ufc_150)

### `reaction_meme`
A post whose main purpose is **humor, hype, sarcasm, or emotional reaction** rather than
a serious argument — even if it uses fight jargon.
- *"Wake up in the hospital. Easy."* (ufc_009)
- *"The plan is to get hit really hard, smile, ask for more, and somehow win Fight of the Night again lmao"* (ufc_042)

### `question_prompt`
A post whose main purpose is to **ask a question** — soliciting analysis, opinion, or
discussion rather than making a claim of its own.
- *"What is Gaethje's actual path to beating Ilia?"* (ufc_201)
- *"What separates a glass chin from bad defensive habits?"* (ufc_218)

### Decision rules for ambiguous posts
- Specific technical / tactical / matchup / fight-history reasoning → `fight_analysis`.
- A serious claim with little or vague support → `hot_take`.
- Mainly a joke, sarcasm, hype, or emotional reaction → `reaction_meme`, even if it uses fight language.
- The post is primarily *asking* rather than *claiming* → `question_prompt`.
- **Mixed posts → dominant purpose wins.** One checkable detail does not rescue an otherwise-vague take; one rhetorical question inside a claim does not make it a `question_prompt`.

---

## Dataset

- **Source:** r/ufc only — primarily two threads that each mix all four labels ("What
  is Gaethje's plan?" and "What MMA fighter was held back by their chin the most?"),
  plus post titles from the Best/Top/Hot/Rising pages. Pulling from more than one thread
  keeps a single matchup's vocabulary from defining a label.
- **Collection:** Public posts/comments, hand-picked into a single CSV
  (`ufc_discourse_4label_revised.csv`) with columns `id`, `text`, `label`.
- **Labeling process:** Each post was read and labeled individually against the
  definitions above. A batch was **pre-labeled with an LLM and then reviewed and
  corrected row-by-row** (the LLM's guess was never the final call — see
  [AI Usage](#ai-usage)). A running log of every post that gave pause kept the calls
  consistent.
- **Split:** The notebook performs a 70 / 15 / 15 train / validation / test split.
  Test set = 33 examples.

### Label distribution (220 examples)

| Label | Count | Share |
|---|---|---|
| `reaction_meme` | 73 | 33.2% |
| `hot_take` | 61 | 27.7% |
| `fight_analysis` | 55 | 25.0% |
| `question_prompt` | 31 | 14.1% |
| **Total** | **220** | |

No label exceeds 70%. `question_prompt` is the minority class at 14% — relevant to its
weaker performance below.

### Three genuinely difficult examples

1. **ufc_024** — *"justin is too careless and sloppy against much inferior fighters and has paid for it… hes old and these habits havent been corrected."* Could be `fight_analysis` (names a real tendency) or `hot_take`. **Decision: `hot_take`** — it names a tendency but never explains a matchup or cites evidence; it's venting, not reasoning.
2. **ufc_162** — *"Cyril Gane's best weapon is his ability to control the pace… Just kidding. He just pokes the hell out of your eyes."* Opens like `fight_analysis`. **Decision: `reaction_meme`** — the technical language is just setup for the sarcastic turn; the point is the joke.
3. **ufc_167** — *"What mma fighter was held back by their chin the most? Maybe not the absolute most but I had to pick Overeem, he was so good but his chin failed him multiple times."* Sits between `question_prompt` (it opens with a question) and `fight_analysis`. **Decision: `question_prompt`** — the post's dominant purpose is to pose the thread's question; the Overeem line is a tentative answer, not a worked argument. (This kind of post is exactly *why* `question_prompt` was split out — see Spec Reflection.)

---

## Fine-Tuning Approach

- **Base model:** `distilbert-base-uncased` (HuggingFace).
- **Platform:** Google Colab (free T4 GPU), via the starter notebook's training loop.
- **Setup:** Sequence classification head over 4 labels; 70/15/15 split; standard
  tokenization.

**Key hyperparameter decision:** I trained for **6 epochs** (double the notebook's default
of 3), keeping learning rate 2e-5 and batch size 16. The reasoning: with only ~154
training examples across 4 classes (~38 per class, and just ~22 for `question_prompt`), a
single pass per epoch exposes the model to very little signal, so 3 epochs left it
underfit. Doubling to 6 gave the model more passes to separate the subtle
`fight_analysis` ↔ `hot_take` boundary.

**In hindsight this likely backfired.** The results show the model collapsed toward
over-predicting `fight_analysis` (predicted 14× vs. 9 true) with low confidence
everywhere (0.30–0.50) — symptoms consistent with *over*-training on a tiny, overlapping
dataset: 6 epochs gave it enough passes to memorize lexical shortcuts (fight jargon →
`fight_analysis`) without learning the intent distinction. On a set this small, fewer
epochs plus more data per minority class would probably have generalized better than more
epochs on the same data.

---

## Baseline

- **Approach:** Zero-shot classification with Groq `llama-3.3-70b-versatile` — no
  task-specific training. The model is given the four label definitions (as written
  above) and instructed to output **only the label name**, which the notebook parses
  directly.
- **Why:** A zero-shot LLM tells us how hard the task is for a capable general model,
  which gives the fine-tuned numbers meaning.
- **Collection:** Run over the same locked 33-example test set used for the fine-tuned
  model.

---

## Evaluation Report

### Headline numbers (same 33-example test set)

| Model | Accuracy | Macro-F1 |
|---|---|---|
| Baseline (Groq llama-3.3-70b, zero-shot) | **0.667** | **0.67** |
| Fine-tuned DistilBERT | **0.636** | **0.62** |

**The fine-tuned model did not beat the baseline — it lost on both metrics**, ~3 points
lower on accuracy and ~5 points lower on macro-F1. This is the central finding of the
project, and the analysis below explains why. (Per the success criteria in
`planning.md`: macro-F1 0.62 is below the 0.65 "not useful" line and well under the 0.75
"genuinely useful" target. The fine-tune, as trained on this dataset, is not
deployable — and on this task a zero-shot 70B LLM with good label definitions is the
better classifier.)

### Per-class metrics — fine-tuned model

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `hot_take` | 0.80 | 0.44 | 0.57 | 9 |
| `fight_analysis` | 0.57 | 0.89 | 0.70 | 9 |
| `reaction_meme` | 0.64 | 0.64 | 0.64 | 11 |
| `question_prompt` | 0.67 | 0.50 | 0.57 | 4 |
| **Accuracy** | | | **0.636** | 33 |
| **Macro avg** | 0.67 | 0.62 | 0.62 | 33 |

### Per-class metrics — baseline (Groq llama-3.3-70b, zero-shot)

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `hot_take` | 0.47 | 0.89 | 0.62 | 9 |
| `fight_analysis` | 0.83 | 0.56 | 0.67 | 9 |
| `reaction_meme` | 0.88 | 0.64 | 0.74 | 11 |
| `question_prompt` | 1.00 | 0.50 | 0.67 | 4 |
| **Accuracy** | | | **0.667** | 33 |
| **Macro avg** | 0.79 | 0.65 | 0.67 | 33 |

A revealing contrast: the two models fail in **opposite directions**. The baseline
*over-predicts `hot_take`* (recall 0.89, precision 0.47 — it dumps borderline posts into
hot_take), while the fine-tuned model *over-predicts `fight_analysis`* (recall 0.89,
precision 0.57). Same hard boundary, resolved the opposite way. The baseline's
`question_prompt` precision is a perfect 1.00 (when it says "question," it's always
right) but recall only 0.50 — it misses the strategy-question posts that look like
analysis, the same posts the fine-tune also misroutes.

### Confusion matrix — fine-tuned model (rows = true, columns = predicted)

| True ↓ / Pred → | hot_take | fight_analysis | reaction_meme | question_prompt | **Total** |
|---|---|---|---|---|---|
| **hot_take** | **4** | 2 | 3 | 0 | 9 |
| **fight_analysis** | 0 | **8** | 1 | 0 | 9 |
| **reaction_meme** | 1 | 2 | **7** | 1 | 11 |
| **question_prompt** | 0 | 2 | 0 | **2** | 4 |
| **Pred total** | 5 | 14 | 11 | 3 | 33 |

(Supplementary image: `confusion_matrix.png`.)

The diagonal sums to 21/33 = 0.636. The single most striking column is
**`fight_analysis`, which the model predicted 14 times though only 9 posts are truly
analysis** — it over-uses `fight_analysis` as a default.

### Three specific wrong predictions, analyzed

All three are real misclassifications from the fine-tuned model's test run, chosen from
its largest error cells. Note the low confidences (0.30–0.45) — the model is barely
committing even when it commits.

1. **"If he tries to box with him it will get ugly. Justin needs to brawl"** (ufc_022) —
   True `hot_take`, **predicted `fight_analysis` (conf 0.35)**. *(true=hot_take →
   pred=fight_analysis, one of the 2 errors on the boundary `planning.md` flagged as
   hardest.)* The post names a plausible tactic ("brawl") but gives no reason *why* — no
   technique, no matchup detail, just an assertion. **Why it failed:** this is the
   `fight_analysis` ↔ `hot_take` spectrum exactly as predicted. The difference is whether
   the reasoning is *checkable*, which is semantic, not lexical; on ~38 training examples
   per class the model reads the tactical noun ("box," "brawl") and defaults to analysis.

2. **"How this hypothetical fight go? Both in their prime. Will Prates unleash hell on
   him for 5 rounds as Nate Diaz has a crazy chin and cardio?..."** (ufc_112) — True
   `question_prompt`, **predicted `fight_analysis` (conf 0.30)**. *(2 of the 4
   question_prompt errors went this way — half the questions misrouted.)* The post is
   dense with the exact vocabulary (`5 rounds`, `chin`, `cardio`, `striker`) that defines
   `fight_analysis`, and its question framing is buried mid-paragraph. **Why it failed:**
   strategy questions and strategy analysis share nearly identical token distributions;
   the only distinguishing signal is the interrogative framing, and with just ~22
   `question_prompt` training examples the model never learned to weight "is this asking?"
   over "does this contain tactical vocabulary?" The 0.30 confidence confirms it was
   nearly a coin-flip.

3. **"Jonathan Goulet seems slow"** (ufc_054) — True `hot_take`, **predicted
   `reaction_meme` (conf 0.50)**. *(true=hot_take → pred=reaction_meme, the largest single
   error cell at 3.)* A flat, low-effort opinion with no support. **Why it failed:** short,
   punchy `hot_take`s and `reaction_meme`s share surface form — both are terse one-liners
   with attitude and no fight jargon to anchor on. Stripped of tactical vocabulary, the
   model has nothing left to separate "lazy opinion" from "throwaway joke," so it guesses
   (conf 0.50). This is the mirror image of error #1: with jargon the model over-calls
   analysis; without it, it can't tell a take from a meme.

### Sample classifications

Five posts run through the fine-tuned model. The four misclassifications are real
outputs with the model's softmax confidence; the one correctly-predicted row needs its
confidence pulled from the notebook (⚠).

| Post (text) | True label | Predicted | Confidence | Correct? |
|---|---|---|---|---|
| "People who think Justin cant fight calmly go watch his fight against Tony." (ufc_003) | fight_analysis | reaction_meme | 0.35 | ✗ |
| "If he tries to box with him it will get ugly. Justin needs to brawl" (ufc_022) | hot_take | fight_analysis | 0.35 | ✗ |
| "Jonathan Goulet seems slow" (ufc_054) | hot_take | reaction_meme | 0.50 | ✗ |
| "He's probably planning on hitting him in the face. I'm no expert though" (ufc_151) | reaction_meme | fight_analysis | 0.39 | ✗ |
| "Rockhold's problem was that he would back out straight, chin up, hands down…" (ufc_135) | fight_analysis | ⚠ fight_analysis | ⚠ | ✓ (confirm) |

**Why the correct `fight_analysis` prediction is reasonable (ufc_135):** the post names a
specific, checkable mechanism — backing out straight, chin up, hands down, trouble seeing
the left from a bladed retreat. That's concrete tactical reasoning, exactly the signal
`fight_analysis` is meant to capture, and `fight_analysis` is the one class the model
learned well (recall 0.89). ⚠ *Confirm this post landed in the test split and copy its
confidence; if it wasn't in the test set, substitute any correctly-predicted
`fight_analysis` test post.*

**On confidence:** every wrong prediction above sits between 0.30 and 0.50 — the model is
barely more confident than a random guess (0.25 for 4 classes) even when it errs. That is
itself a finding: the fine-tune never built a sharp decision boundary on this small,
overlapping dataset.

---

## Reflection: what the model learned vs. what I intended

I intended the model to learn a **purpose/quality distinction** — *how much reasoning
backs a claim, and what the post is trying to do*. What it actually learned is closer to
a **vocabulary detector**: "does this post contain tactical fight language?" → predict
`fight_analysis`.

The evidence is the confusion matrix's lopsided `fight_analysis` column: the model
predicted analysis 14 times against a true count of 9, pulling in 2 `hot_take`s and 2
`question_prompt`s. Both of those error classes share the *vocabulary* of analysis
(fighter names, techniques, stances) while differing only in *intent* — a hot take
asserts without evidence; a question asks. Because intent isn't reliably encoded in
surface tokens, and because the dataset is small (~38 training examples per class, ~22
for `question_prompt`), the model fell back on the lexical signal it could learn.

The specific failure pattern: **`fight_analysis` is over-predicted at the expense of
`hot_take` (recall only 0.44) and `question_prompt` (recall 0.50)**, while
`reaction_meme` — the class whose surface form is most distinct (jokes, short hype, no
fight jargon) — held up best relative to its difficulty. That is precisely the
analysis ↔ hot_take confusion `planning.md` predicted, and adding `question_prompt`
*widened* the problem rather than fixing it, because strategy questions look lexically
identical to strategy analysis.

What would change it: (a) more training data for the two starved classes, especially
`question_prompt`; (b) feature signal that captures intent, not just tokens (e.g.
presence of evidence clauses vs. a trailing `?`); or (c) merging `question_prompt` back
into the taxonomy it was carved from, since at 14% / ~22 train examples it is too thin
for DistilBERT to learn a boundary that is mostly punctuation-deep.

---

## Spec Reflection

**One way the spec helped:** `planning.md`'s requirement to name the hardest edge case
*before* annotating forced me to articulate the `fight_analysis` ↔ `hot_take` boundary
and the "is the support checkable?" rule. That rule is what kept my annotations
consistent, and it turned out to predict the model's dominant failure mode exactly — the
confusion matrix's analysis-vs-take confusion is the same boundary the plan flagged.

**One way the implementation diverged:** the plan specified **3 labels and 200 balanced
examples**. In practice I split out a fourth label, **`question_prompt`**, and grew the
set to **220**. The trigger was annotation: a recurring class of posts (the thread-title
questions like "What is Gaethje's plan?", ufc_181/201/202…) was being force-fit into
`fight_analysis` under the original rule, even though those posts make no claim at all.
Rather than corrupt the `fight_analysis` definition, I gave questions their own label.
The trade-off, visible in the results, is that `question_prompt` arrived under-sampled
(14%, ~22 train examples) and lexically overlapping with `fight_analysis`, which is part
of why the fine-tune underperformed. In hindsight a cleaner path would have been to
either commit to `question_prompt` *and* deliberately balance it to ~25%, or keep three
labels and treat pure questions as a documented exclusion. The divergence was the right
annotation call but an incomplete data-collection follow-through.

---

## AI Usage

1. **Label stress-testing (before annotating).** I gave Claude (Opus) the label
   definitions and the `fight_analysis` ↔ `hot_take` edge case and asked for borderline
   posts. Anything I couldn't classify cleanly meant a loose definition; the "checkable
   support" rule came out of this. **What I overrode:** several of its suggested
   borderline posts I judged as clearly one label, so I discarded them rather than
   weaken the definitions.

2. **Annotation assistance (pre-labeling) — disclosed.** I used an LLM to pre-label a
   batch of unlabeled posts given the definitions, then **reviewed and corrected every
   row myself**; the model's suggestion was never the final label. ⚠ *Name the exact
   tool/model you used here (the planning draft listed "ChatGPT 5.5", which is not a real
   model name — replace with the actual one).*

3. **Failure-pattern analysis (this report).** I gave the confusion matrix and the error
   cells to Claude (Opus) and asked for the dominant pattern. It identified the
   over-prediction of `fight_analysis` and the vocabulary-vs-intent explanation; I then
   **verified each claim against the actual misclassified posts in the largest error
   cells** (ufc_022, ufc_112, ufc_054) before writing it up, rather than taking the
   pattern on faith.

---

## Repository Contents

- `planning.md` — original design document (3-label / 200-example design; see scope note above).
- `ufc_discourse_4label_revised.csv` — the labeled dataset (220 examples, 4 labels).
- `evaluation_results.json` — exported metrics (both models' accuracy, label map).
- `confusion_matrix.png` — fine-tuned model confusion matrix (image).
- `README.md` — this report.
