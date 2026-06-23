
# TakeMeter — r/soccer Discourse Classifier

A fine-tuned text classifier that evaluates discourse quality in r/soccer Reddit comments, categorizing posts as **analysis**, **hot_take**, or **reaction**.

---

## Community

I chose r/soccer because it's one of the most active sports communities on Reddit and the discourse varies enormously — you get detailed tactical breakdowns right next to someone losing their mind over a last-minute goal. That range makes it a good fit for a classification task because the differences between post types are real and meaningful, not manufactured.

---

## Label Taxonomy

**analysis** — the post builds a structured argument using stats, tactics, historical context, or reasoning. The claim is supported, not just stated.
- *"United's pressing has completely broken down this season — their PPDA is 12.3, worst in the top half of the table, compared to 8.1 last year under the same system."*
- *"Rodri's value isn't in what you see, it's in what doesn't happen. City concede 0.4 fewer xG per 90 when he plays vs. when he doesn't — that's elite."*

**hot_take** — a bold or controversial opinion stated without real supporting evidence. The post asserts something confidently but doesn't back it up.
- *"Mbappe is already better than peak Messi, it's not even close."*
- *"The Premier League is the worst-coached top league in Europe. The tactics are embarrassing."*

**reaction** — an immediate emotional response to something that just happened. No argument, just a feeling.
- *"THAT SAVE. THAT SAVE. I CANNOT."*
- *"Bro how did that not go in. I'm done. Season over."*

---

## Data Collection

- **Source:** r/soccer, collected via the PullPush Reddit API from comment threads including match threads, daily discussion posts, and transfer news threads
- **Size:** 208 examples (after removing questions and low-quality comments)
- **Labeling process:** Comments were pre-labeled using Claude with label definitions as the prompt, then manually reviewed and corrected. All pre-labeled rows are flagged in the `pre_labeled` column of the CSV.
- **Label distribution:**
  - hot_take: 80 (38%)
  - analysis: 68 (33%)
  - reaction: 60 (29%)

**3 difficult-to-label examples:**

1. *"Pep's won the lot and he still thinks it's a trophy, I'd believe him over any redditor tbh"* — Could be analysis (logical argument using Pep's credentials) or hot_take (assertive, no structure). → **hot_take**. The reasoning is a shortcut, not a real argument.

2. *"R9 is overrated overall is my hot take, people put him in the GOAT conversation when he's had neither the peak or longevity"* — References specific criteria (peak, longevity) which feels like analysis, but nothing is backed up with data. → **hot_take**.

3. *"This is A BIT like Fergie dragging those makeshift United squads to titles. Our season has been so cursed with injuries that if we had a lesser quality manager, we would for sure not make CL semis"* — Emotional tone but builds a structured argument using a historical parallel. → **analysis**.

---

## Fine-Tuning Approach

- **Base model:** `distilbert-base-uncased` (HuggingFace)
- **Training setup:** 3 epochs, learning rate 2e-5, batch size 16 (default settings)
- **Train/val/test split:** 70% / 15% / 15% (145 / 31 / 32 examples)
- **Key hyperparameter decision:** Kept the default 3 epochs. In hindsight, the small dataset size (145 training examples) was the main limiting factor — more epochs or a larger dataset would likely improve results significantly.

---

## Baseline

The zero-shot baseline used Groq's `llama-3.3-70b-versatile` with the following prompt structure:

```
You are classifying comments from r/soccer on Reddit.
Assign each comment to exactly one of the following categories.

analysis: the post builds a structured argument using stats, tactics, historical context, or reasoning.
hot_take: a bold or controversial opinion stated without real supporting evidence.
reaction: an immediate emotional response. No argument, just a feeling or exclamation.

Respond with ONLY the label name.
```

All 32 test examples returned parseable responses (32/32).

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq) | **0.781** |
| Fine-tuned DistilBERT | **0.375** |

Fine-tuning regression: 0.406 — the fine-tuned model performed significantly worse than the baseline.

### Per-Class Metrics — Baseline

| Label | Precision | Recall | F1 |
|---|---|---|---|
| analysis | 0.89 | 0.73 | 0.80 |
| hot_take | 0.69 | 0.92 | 0.79 |
| reaction | 0.86 | 0.67 | 0.75 |

### Per-Class Metrics — Fine-Tuned Model

| Label | Precision | Recall | F1 |
|---|---|---|---|
| analysis | 0.33 | 0.09 | 0.14 |
| hot_take | 0.38 | 0.92 | 0.54 |
| reaction | 0.00 | 0.00 | 0.00 |

### Confusion Matrix (Fine-Tuned Model)

|  | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|---|---|---|---|
| **True: analysis** | 1 | 10 | 0 |
| **True: hot_take** | 1 | 11 | 0 |
| **True: reaction** | 1 | 8 | 0 |

The model predicted hot_take for nearly every example. It never once predicted reaction correctly.

### 3 Wrong Predictions Analyzed

**1.** Text: *"Swedish flag so the song gets overrated. It's like English player tax"*
True: hot_take → Predicted: analysis (confidence: 0.35)
The word "overrated" appears frequently in analysis posts in the training data, likely causing the model to associate it with analysis rather than hot_take. The confidence is also very low (0.35), showing the model had no strong signal.

**2.** Text: *"Yeah it's the Derek Jeter phenomenon. People overrate both because they were charismatic captains of popular teams. Jeter won 5 gold gloves despite being statistically the worst defensive player in MLB history"*
True: analysis → Predicted: hot_take (confidence: 0.35)
This is a genuinely analytical post with a historical comparison and a specific stat. The model failed to recognize it as analysis, likely because the word "overrate" dominated and the training data didn't have enough examples of this style.

**3.** Text: *"That save. Absolute worldie."*
True: reaction → Predicted: hot_take (confidence: 0.35)
A clear reaction post that the model labeled as hot_take. Since the model learned to predict hot_take for everything, even obvious reactions got mislabeled. The model never learned the reaction class at all.

### Sample Classifications

| Text | True Label | Predicted | Confidence |
|---|---|---|---|
| "Boss we're not doing this… community shield isn't a trophy, it's a dinner plate" | hot_take | hot_take | 0.38 |
| "That's it if Pep gets to claim community shields then I can too" | reaction | hot_take | 0.36 |
| "Salah's underlying numbers this season are staggering — 0.71 xG+xA per 90" | analysis | hot_take | 0.36 |
| "Absolutely incredible. What a final chapter to one of the greatest managers" | reaction | hot_take | 0.35 |
| "Kevin DeBruyne remains the most overrated footballer of all time." | hot_take | hot_take | 0.38 |

The one correct example ("community shield isn't a trophy") is a short, punchy opinion with no evidence — exactly what the model learned to classify as hot_take. It got it right for the wrong reason: it just defaulted to hot_take.

---

## Reflection: What the Model Learned vs. What I Intended

The model didn't learn to distinguish between analysis, hot_take, and reaction. It learned to predict hot_take for almost everything.

The root cause is the training set size — 145 examples spread across 3 classes gives DistilBERT very little signal. The model defaulted to the majority class (hot_take, 38% of training data) because that was the safest bet with so little data to learn from.

There's also a label boundary problem. The analysis/hot_take boundary is genuinely hard — both involve opinions about football, and the difference is subtle (evidence vs. no evidence). Even with more data, this boundary would likely be the model's weakest point.

The zero-shot baseline (Groq) actually handled this task better because a large language model already understands the difference between structured argument and bare assertion. Fine-tuning a small model on 145 examples couldn't compete with that general language understanding.

---

## Spec Reflection

**One way the spec helped:** The requirement to define a decision rule for edge cases before annotating forced me to think carefully about the analysis/hot_take boundary early. Having that rule written down made annotation faster and more consistent.

**One way implementation diverged:** The spec assumes fine-tuning will improve on the baseline. In this case it didn't — the dataset was too small for DistilBERT to learn the task. A real deployment would need significantly more labeled data (500+ examples per class) before fine-tuning would be worth it.

---

## AI Usage

1. **Pre-labeling the dataset:** Claude was used to pre-label all 208 comments using the label definitions from planning.md as the prompt. Every label was manually reviewed and corrected before use. A `pre_labeled` column in the CSV flags all AI-assisted rows.

2. **Error pattern analysis:** After fine-tuning, the wrong predictions were reviewed to identify patterns. The main pattern identified — model defaulting to hot_take — was verified by looking at the confusion matrix, which showed reaction was never predicted correctly.

---

## Demo Video

https://drive.google.com/file/d/1bEHr6vHedBAALIvd8QfjnhyOlQKjSr6t/view?usp=sharing
