# Planning: r/ufc Discourse Classifier

## Community

All my data is from **r/ufc**.

r/ufc works well for this because one fight produces totally different kinds of posts/comments, often in the same thread: breakdowns of *why* something happened, bare opinions with little support, pure jokes/reactions, and discussion prompts that ask the community a question without making a clear claim. What separates them is intent: is the post explaining, asserting, reacting, or asking? That is a real distinction, but not an obvious keyword match, so the model has to read the purpose of the text.

## Labels

Four labels — full definitions and examples are in the Label Taxonomy section below.

- **`fight_analysis`** — explains a fight, fighter, matchup, strategy, technique, durability issue, or career situation with *specific* reasoning.
- **`hot_take`** — a strong opinion, prediction, ranking, or fighter judgment with *little or no* support.
- **`reaction_meme`** — mainly humor, hype, sarcasm, emotional reaction, or a low-context title/fragment that depends on image/thread context.
- **`question_prompt`** — mainly asks a genuine question or opens a discussion, without giving a serious argument.

Every post gets exactly one label. The important rule is that **form alone does not decide the label**: a question can still be a joke, and a short fighter-name/title can be `reaction_meme` if the missing image/thread context is doing most of the work.

## Hard Edge Cases

The hardest boundary is still **`fight_analysis` vs `hot_take`**. Both can make an MMA claim. The difference is whether the post gives specific support: technique, tactical adjustment, matchup detail, fight history, opponent quality, durability pattern, or a named example. If it only says someone is washed, overrated, terrible, doomed, or would win easily, that is `hot_take`.

A new gray area is **`question_prompt` vs `hot_take`**. If the post mainly asks the community a question, it is `question_prompt`. If it asks a question but immediately gives a clear unsupported answer, the dominant purpose may still be `hot_take`.

Another gray area is **`question_prompt` vs `reaction_meme`**. A joke phrased as a question, like “What if I punch really hard?”, is still `reaction_meme`. A real discussion prompt, like “What is Gaethje's plan?”, is `question_prompt`.

For low-context post titles, I used this rule: if the text is only a fighter name, short phrase, or title that probably relies on a picture/meme not captured in the CSV, I treat it as `reaction_meme` unless it clearly makes a serious UFC claim.

**Examples:**

1. **ufc_024** — “justin is too careless and sloppy... hes old and these habits havent been corrected.” It names a tendency, but it mostly vents a negative judgment without a specific matchup explanation. → **`hot_take`**.
2. **ufc_162** — “Cyril Gane's best weapon is his ability to control the pace... Just kidding. He just pokes the hell out of your eyes.” It starts like analysis, but the point is the joke. → **`reaction_meme`**.
3. **ufc_181** — “What is Gaethje's plan.” This is a bare discussion prompt, not an analysis answer. → **`question_prompt`**.
4. **ufc_082** — “Rico couldn't recover from that knockdown... Rico's known for his chin.” This cites a specific durability/fight-history reason. → **`fight_analysis`**.

## Data Collection Plan

- **Source:** r/ufc only — mostly threads that mix analysis, takes, memes, and question prompts.
- **Format:** 220 examples in one CSV (`ufc_discourse_4label_revised.csv`: `id`, `text`, `label`). The notebook does the 70/15/15 split.
- **Important relabeling change:** I added `question_prompt`, then went back and collected more question/discussion-prompt examples (taking it from ~11 to 31), and tightened `fight_analysis` so simple strategy fragments without reasons no longer automatically count as analysis.
- **Final distribution (as built):**

  | Label | Count | Share |
  |---|---:|---:|
  | `reaction_meme` | 73 | 33.2% |
  | `hot_take` | 61 | 27.7% |
  | `fight_analysis` | 55 | 25.0% |
  | `question_prompt` | 31 | 14.1% |
  | **Total** | **220** | **100.0%** |

- **Balance note:** No label exceeds 70%, so the dataset clears the rubric's hard imbalance limit. `question_prompt` is still the minority at 14.1% — below the ~20% rule of thumb. I added rows to get it from 5.5% to 14.1%, but it remains the thinnest class, which (as the evaluation confirmed) is part of why the four-class fine-tune underperformed: with only ~22 training examples, the model could not learn the question-vs-analysis boundary, which is mostly punctuation-deep. To push it past 20% I'd collect more genuine discussion prompts or apply class weighting before a final run.

## Evaluation Metrics

Two models, same locked test set: a zero-shot baseline and the fine-tuned `distilbert-base-uncased`. The baseline tells me how hard the task is for a general model and makes the fine-tuned numbers meaningful.

Accuracy alone is not enough, especially now that `question_prompt` is smaller. So:

- **Per-class precision/recall/F1** — the main diagnostic.
- **Macro-F1** — the headline number, because it weights all four labels equally.
- **Confusion matrix** — to see whether the model confuses `fight_analysis` with `hot_take`, or misreads questions/jokes as analysis.

I care most about `fight_analysis` recall and `question_prompt` recall. Missing analysis defeats the purpose of the classifier, and missing question prompts means the new label is not actually being learned.

## Definition of Success

- **Genuinely useful:** macro-F1 ≥ **0.75**, `fight_analysis` recall ≥ **0.70**, and `question_prompt` recall ≥ **0.60** after adding enough question examples.
- **Deployable**: macro-F1 ≥ **0.80**, no class F1 below **0.65**.
- **Not useful:** macro-F1 < **0.65**, analysis recall < **0.60**, or the model collapses into one label in the confusion matrix.

## Label Taxonomy

### Label 1: `fight_analysis`

**Definition:** A post/comment is `fight_analysis` when it explains a fight, fighter, matchup, strategy, judging, durability issue, technique, or career situation using specific reasoning.

**Clear examples:**

1. “If Gaethje tries to box with Ilia, it gets ugly. He needs to make it messy, use pressure, and force exchanges instead of letting Ilia counter cleanly.”
2. “People saying Justin cannot fight calmly should rewatch the Tony fight. He showed he can stay composed when the game plan is clear.”

**Uncertain example:**

> “He needs to use the clinch and kicks a lot imo.”

**Could be:** `fight_analysis` or `hot_take`  
**Decision:** `hot_take`. It gives a strategy opinion, but no reason. Under the stricter taxonomy, bare strategy suggestions are not enough for `fight_analysis`.

---

### Label 2: `hot_take`

**Definition:** A post/comment is `hot_take` when it makes a strong UFC opinion, prediction, ranking, or fighter judgment with little or no supporting reasoning.

**Clear examples:**

1. “Paddy is overrated and gets exposed by anyone in the top 10.”
2. “Islam would destroy him. It would not even be competitive.”

**Uncertain example:**

> “Belal was never actually that good. People only rated him because he kept winning boring fights.”

**Could be:** `hot_take` or `fight_analysis`  
**Decision:** `hot_take`. There is a reason, but it is vague and reputation-based — no technique, matchup detail, named fight evidence, or specific pattern.

---

### Label 3: `reaction_meme`

**Definition:** A post/comment is `reaction_meme` when the main purpose is humor, hype, sarcasm, emotional reaction, or low-context entertainment rather than serious argument. This also includes very short meme/title fragments that rely on an attached image or missing thread context.

**Clear examples:**

1. “Bro is waking up in another dimension.”
2. “Wake up in the hospital. Easy.”
3. “Prime Andrei Arlovski.”

**Uncertain example:**

> “Cyril Gane controls distance beautifully. Just kidding, he jumps backward and jabs air for 25 minutes.”

**Could be:** `reaction_meme` or `fight_analysis`  
**Decision:** `reaction_meme`. The first sentence sounds analytical, but the point is sarcasm.

---

### Label 4: `question_prompt`

**Definition:** A post/comment is `question_prompt` when its main purpose is to ask a genuine question or open a discussion, without giving a serious argument.

**Clear examples:**

1. “What is Gaethje's plan?”
2. “Was Belal Muhammad ever really that good?”
3. “How does Ilia plan on countering this?”

**Uncertain example:**

> “How would this fight go? WW wins easily.”

**Could be:** `question_prompt` or `hot_take`  
**Decision:** `hot_take`. The answer (“WW wins easily”) is the dominant claim; the question is just rhetorical framing. It's asserting a winner, not opening a genuine discussion.

## Decision Rules for Ambiguous Posts

- Specific technical, tactical, judging, matchup, durability, or fight-history reasoning → `fight_analysis`.
- A serious UFC claim with little, vague, or reputation-based support → `hot_take`.
- Mainly a joke, meme, sarcasm, hype, emotional reaction, or low-context image/title fragment → `reaction_meme`.
- Mainly a genuine question or discussion prompt with no serious answer/argument → `question_prompt`.
- A question mark does not automatically mean `question_prompt`; joke questions stay `reaction_meme`, and question+argument posts are labeled by the dominant purpose.
- Mixed posts → dominant purpose wins. A single specific detail does not rescue an otherwise vague take, and technical words do not rescue a joke.

## AI Tool Plan

There is no code to generate in this project, so AI assists in three specific places:

**1. Label stress-testing (before annotating).** I gave Claude (Opus) my label
definitions and the `fight_analysis` ↔ `hot_take` edge case and asked for borderline
posts. Anything I couldn't classify cleanly meant a loose definition, so I tightened it
first — the "checkable support" rule came out of this. I discarded its suggestions that I
judged clearly one label rather than weaken the definitions. *Tool: Claude (Opus).*

**2. Annotation assistance (pre-labeling) — disclosed.** I used an LLM to pre-label a
batch of unlabeled posts given the definitions, then reviewed and corrected every row
myself; its guess was never the final call. This is disclosed in the README's AI usage
section. *Tool: ChatGPT*

**3. Failure analysis (after evaluation).** I gave the confusion matrix and wrong
predictions to Claude (Opus) and asked for the dominant error pattern, then verified each
claim against the actual misclassified posts before writing it up. This surfaced the
"jargon → predicts fight_analysis" / "question with tactical vocabulary → predicts
analysis" pattern. *Tool: Claude (Opus).*
