# Planning: r/ufc Discourse Classifier

## Community

All my data is from **r/ufc**.

r/ufc works well for this because one fight produces totally different kinds of comments, often in the same thread: breakdowns of *why* something happened (footwork, distance, hand position), bare opinions with zero support ("he's washed," "Ilia smokes him"), and pure reaction (memes, one-liners). What separates them is *how much reasoning is behind the claim* — which is a real distinction but not an obvious one, so the model has to read intent rather than keywords. The sub is also big and messy enough that I could find plenty of each type.

## Labels

Three labels — full definitions and examples are in the Label Taxonomy section below.

- **`fight_analysis`** — explains a fight, fighter, matchup, strategy, technique, or career situation with *specific* reasoning.
- **`hot_take`** — a strong opinion, prediction, ranking, or fighter judgment with *little or no* support.
- **`reaction_meme`** — mainly humor, hype, sarcasm, or emotional reaction, not a serious argument.

Every post gets exactly one label (the Decision Rules below handle ties). Between serious reasoning, bare opinion, and jokes, these cover basically all r/ufc discourse, so I don't need an "other" bucket.

## Hard Edge Cases

The real gray area is **`fight_analysis` vs `hot_take`** — both make an MMA claim, and they only differ in how good the support is, which is a spectrum. The trickiest ones are reputation-based reasons dressed up as analysis ("Belal was never that good, people only rated him 'cause he won boring fights") — there's a reason there, but it's vibes, not technique or matchup detail. A second trap is **mixed posts** that pair one checkable observation with a vague universal claim ("his leg kicks are elite, but eventually taking that much damage catches up to everyone") — one clause reads as analysis, the other as a hot take.

My rule during annotation: *does the post cite something checkable* — a specific technique, tactical adjustment, matchup detail, or fight history? If yes → `fight_analysis`; if it's vague or just intensity → `hot_take`. And for sarcasm using fight jargon, purpose wins over vocabulary — if the point is a joke, it's `reaction_meme`. I kept a log of every post I hesitated on so my calls stayed consistent.

**3 Examples:**
1. **ufc_024** — "justin is too careless and sloppy... hes old and these habits havent been corrected." Names a real tendency, but never explains a matchup or cites evidence — it's venting, not reasoning. → **`hot_take`**.
2. **ufc_162** — "Cyril Gane's best weapon is his ability to control the pace... Just kidding. He just pokes the hell out of your eyes." Starts like analysis, but the whole thing is the sarcastic turn. → **`reaction_meme`**.
3. **ufc_181** — "What is Gaethje's plan" (the thread title). A bare question, no claim — but it's explicitly asking *for* strategy reasoning, which is what the label covers. → **`fight_analysis`**.

## Data Collection Plan

- **Source:** r/ufc only — mostly two threads that each mix all three labels: "What is Gaethje's plan?" (strategy + joke replies) and "What MMA fighter was held back by their chin the most?" (durability debate), plus post titles off the Best/Top/Hot/Rising pages. Pulling from more than one thread keeps a single matchup's vocabulary from defining a label.
- **Format:** 200 examples, hand-picked, in one CSV (`id`, `text`, `label`, `notes`, `source_url`, `source_context`, `source_type`). The notebook does the 70/15/15 split.
- **Distribution:**

  | Label | Count | Share |
  |---|---|---|
  | `fight_analysis` | 67 | 33.5% |
  | `hot_take` | 67 | 33.5% |
  | `reaction_meme` | 66 | 33.0% |
  | **Total** | **200** | |

- **Balance:** Basically even (~33% each), well inside the rubric's ≥20% / ≤70% limits — no rebalancing needed. `fight_analysis` is usually the scarce one (real breakdowns are rarer than takes and memes), so I sourced it on purpose from the two reasoning-heavy threads. If it had come in under ~20% I'd have topped it up from analysis threads before splitting, but it didn't.

## Evaluation Metrics

Two models, same locked test set: a zero-shot baseline (Groq `llama-3.3-70b-versatile`, prompted with my definitions, no training) and the fine-tuned `distilbert-base-uncased`. The baseline tells me how hard the task is for a general model and makes the fine-tuned numbers mean something.

Accuracy alone isn't enough — with three roughly even classes a model could still lean on the easy ones. So:

- **Per-class precision/recall/F1** — the main thing. I care most about `fight_analysis` recall, since surfacing analysis is the whole point and missing it is the costly error.
- **Macro-F1** — weights all three classes equally; my single headline number.
- **Confusion matrix** — to see *where* it fails. I expect `fight_analysis` ↔ `hot_take` mistakes; meme confusion would mean something more broken.

## Definition of Success

- **Genuinely useful:** macro-F1 ≥ **0.75** and **`fight_analysis` recall ≥ 0.70** — it has to actually catch the analysis.
- **Deployable** (e.g. auto-flair or a "good discussion" highlighter): macro-F1 ≥ **0.80**, no class F1 below **0.65**. Lower than that and it misflags often enough to annoy people.
- **Not useful:** macro-F1 < 0.65 or analysis recall < 0.60 — worse than no filter.

These are concrete enough that I can just check the final numbers against them.

---

## AI Tool Plan

No code to generate here, so AI helps in three spots:

**1. Label stress-testing (before annotating).** I gave Claude my definitions and the `fight_analysis`↔`hot_take` edge case and asked for 8–10 borderline posts. Anything I couldn't classify cleanly meant my definitions were loose, so I tightened them first. The "checkable support" rule came out of this. *Tool: Claude (Opus).*

**2. Annotation assistance (pre-labeling).** I pre-labeled with **ChatGPT 5.5** — I gave it my label definitions and batches of unlabeled posts, it suggested one label per post, and then I read and corrected every row myself (its guess was never the final call). I'm disclosing this in my AI usage section. *Tool: ChatGPT 5.5.*

**3. Failure analysis (after evaluation).** I'll hand the LLM my wrong predictions and ask for patterns ("names a fighter → predicts analysis," "jargon in a joke → predicts analysis"), focused on the analysis↔hot_take confusions. Then I verify each pattern by pulling the actual examples before writing it up — I'm not taking its word for it. *Tool: Claude (Opus).*

---

## Label Taxonomy

### Label 1: `fight_analysis`

**Definition:** A post/comment is `fight_analysis` when it explains a fight, fighter, matchup, strategy, judging, technique, or career situation using specific reasoning.

**Clear examples:**

1. "If Gaethje tries to box with Ilia, it gets ugly. He needs to make it messy, use pressure, and force exchanges instead of letting Ilia counter cleanly."
2. "People saying Justin cannot fight calmly should rewatch the Tony fight. He showed he can stay composed when the game plan is clear."

**Uncertain example:**

> "Justin is too careless and sloppy. He's old and these habits haven't been corrected."

**Could be:** `fight_analysis` or `hot_take`
**Decision:** `hot_take`. It mentions a fighter tendency but doesn't explain the matchup or give specific evidence — it's mostly a strong negative judgment.

---

### Label 2: `hot_take`

**Definition:** A post/comment is `hot_take` when it makes a strong UFC opinion, prediction, ranking, or fighter judgment with little or no supporting reasoning.

**Clear examples:**

1. "Paddy is overrated and gets exposed by anyone in the top 10."
2. "Islam would destroy him. It would not even be competitive."

**Uncertain example:**

> "Belal was never actually that good. People only rated him because he kept winning boring fights."

**Could be:** `hot_take` or `fight_analysis`
**Decision:** `hot_take`. There's a reason, but it's vague and reputation-based — no technique, matchup detail, opponent quality, or fight evidence.

---

### Label 3: `reaction_meme`

**Definition:** A post/comment is `reaction_meme` when the main purpose is humor, hype, sarcasm, emotional reaction, or low-context entertainment rather than serious argument.

**Clear examples:**

1. "Bro is waking up in another dimension."
2. "Wake up in the hospital. Easy."

**Uncertain example:**

> "Cyril Gane controls distance beautifully. Just kidding, he jumps backward and jabs air for 25 minutes."

**Could be:** `reaction_meme` or `fight_analysis`
**Decision:** `reaction_meme`. The first sentence sounds analytical, but the point is sarcasm — the technical language is just setup for the joke.

---

## Decision Rules for Ambiguous Posts

- Specific technical, tactical, judging, matchup, or fight-history reasoning → `fight_analysis`.
- A serious UFC claim with little, vague, or reputation-based support → `hot_take`.
- Mainly a joke, meme, sarcasm, hype, or emotional reaction → `reaction_meme`, even if it uses fight language.
- **Mixed posts (one checkable detail + a vague claim) → dominant purpose wins.** A single specific detail doesn't rescue an otherwise-vague take; judge what the post is *mostly* doing. (Same tie-breaker as the sarcasm rule, so the rule set stays coherent.)
