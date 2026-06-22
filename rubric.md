Project 3: TakeMeter
Total Points: 25pts + 4pts bonus

Required Features
3pts	Label Taxonomy
1	2–4 labels are defined in complete sentences with clear boundaries between them.
1	Each label has 2 example posts included.
1	Definitions are specific enough that two people reading them would likely agree on most cases — not one-word descriptions or vague adjectives.
4pts	Annotated Dataset
1	Dataset has at least 200 labeled examples.
1	README documents the data source, labeling process, and label distribution (count per label).
1	At least 3 genuinely difficult examples are described, including the labeling decision made for each.
1	No single label accounts for more than 70% of the dataset.
2pts	Fine-Tuning Pipeline
1	README names the base model and the training platform (e.g., Colab, local, HuggingFace AutoTrain).
1	At least one key training decision is described and justified — e.g., number of epochs, learning rate, or batch size, with reasoning or observation (not just "I used the default").
2pts	Baseline Comparison
1	README describes the baseline approach — the prompt used and how results were collected.
1	Evaluation report shows performance metrics for both the fine-tuned model and the baseline on the same test set.
5pts	Evaluation Report and Error Analysis
1	At least one per-class metric is reported for the fine-tuned model (precision, recall, or F1 — not just overall accuracy).
1	A confusion matrix or equivalent table is included showing where the model confuses labels.
1	At least 3 specific wrong predictions are analyzed — each with an explanation tied to the data, the label boundary, or the model's behavior (not just "it got it wrong").
2	A reflection on the gap between what the model captured and what was intended — describes a specific failure pattern (a label pair, a post type, a distributional issue) rather than a generic observation like "it needs more data."
4pts	planning.md
1	Community choice is explained — not just named, but with reasoning for why it's a good fit for this task.
1	Labels are defined with examples and the hardest anticipated edge case is named and addressed.
1	Data collection plan and evaluation metrics are described, with reasoning for the chosen metrics.
1	A specific definition of "good enough" performance is stated (a concrete threshold, not just "it should work well"); AI Tool Plan covers at least one intended use: label stress-testing, annotation assistance, or failure pattern analysis.
3pts	Demo Video
1	Demo or source shows 3–5 posts being classified by the fine-tuned model with label and confidence visible; one correct prediction is narrated/explained with reasoning for why it was correct.
1	Demo or source shows one incorrect prediction with an explanation of what went wrong and why — not just "it got it wrong."
1	README includes an evaluation report summary surfacing the key metrics.
2pts	AI Usage and Spec Reflection
1	Section describes at least 2 specific instances of AI tool use: what the student directed the AI to do, and what they revised or overrode. Any AI assistance used during annotation is disclosed (e.g., using an LLM to pre-label examples that were then reviewed and corrected).
1	Spec reflection is present and substantive — describes one way the spec helped guide the work and one way implementation diverged from it and why.
