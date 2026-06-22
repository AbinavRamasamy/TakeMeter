# TakeMeter

Reddit soccer comment classifier — 3-class NLP task.  
**Labels:** `tactical_analysis` · `performance_evaluation` · `fan_sentiment`  
**Dataset:** 256 labeled r/soccer comments · **Test set:** 39 samples

| Role | Model |
|---|---|
| Baseline | `llama-3.3-70b-versatile` (zero-shot) |
| Fine-Tuned | `distilbert-base-uncased` (task-specific fine-tuning) |

---

## Project Overview

### Community and Domain

I chose **r/soccer** during an active World Cup cycle because its discourse structure is unusually stratified. A live match thread simultaneously contains deep tactical breakdowns, career-arc evaluations, and raw emotional noise — three genuinely distinct communicative intents that can all appear within seconds of each other. That structural variety makes it an ideal domain for a 3-class intent classifier, and the sheer volume of text means annotation doesn't require scraping obscure communities.

### Label Design

The taxonomy was designed around *primary intent*, not surface vocabulary:

- **`tactical_analysis` (0):** Focus on the mechanics of how a structure, system, or rule functions. The test is whether the post could stand as a coaching note — it explains *how*, not *how well*.
  - *"The 3-4-2-1 system falls apart if the wingbacks don't pin the opposition fullbacks deep."*
  - *"Statistically, their PPDA went from 8.2 in the first half to 15.4 in the second, showing a drop in pressing intensity."*

- **`performance_evaluation` (1):** Focus on judging the quality, legacy, or decision-making of a player, manager, or team as agents. The post may mention tactics, but the tactical reference serves the judgment rather than standing alone.
  - *"Livaković proved once again why he is one of the most elite shot-stoppers in high-pressure tournaments."*
  - *"The manager is completely out of his depth at this level. His in-game management is slow and the squad looks lost."*

- **`fan_sentiment` (2):** Emotional reaction, hyperbole, tribal expression, or humor. The content is driven by feeling rather than structured argument.
  - *"OH MY GOD GALÍNDEZ!!! THAT IS AN ABSOLUTELY INSANE SEQUENCE OF SAVES! WE ARE WITNESSING A MIRACLE!"*
  - *"Football is the most beautiful sport on the planet. I am crying real tears in my living room right now."*

The hardest design decision was the **Primary Intent Rule** for edge cases: when a post straddles two classes, classify by what the author is *trying to communicate*, not by which class-associated tokens appear. This mattered most at the `tactical_analysis` / `performance_evaluation` boundary.

### Data Collection

256 examples scraped from r/soccer match threads, post-match threads, and daily discussion threads during the 2026 World Cup.

**Label distribution:**

| Label | Count |
|---|---|
| `tactical_analysis` | 92 |
| `performance_evaluation` | 82 |
| `fan_sentiment` | 82 |

`tactical_analysis` required targeted collection from dedicated analysis threads because live match traffic skews toward emotional commentary. Each example was hand-labeled by applying the Primary Intent Rule; AI pre-labeling was used for an initial pass and audited before finalizing (see AI Usage).

**Three difficult-to-label examples:**

**1.** *"The manager's tactical switch to a low 5-4-1 block completely salvaged his reputation after weeks of terrible individual evaluations."* → **labeled `tactical_analysis`**. The mention of "reputation" and "individual evaluations" pulls toward PE, but the primary claim is about what the tactical switch *did mechanically*. The evaluative language is secondary. Decision: TA.

**2.** *"The coach completely out-tacticked his opponent with the diamond midfield, fully solidifying his legacy as an elite international manager."* → **labeled `performance_evaluation`**. Contains "diamond midfield," a TA-associated term, but the point is about the coach's legacy and status. The tactical move is evidence in service of a judgment, not the subject itself. Decision: PE per Primary Intent Rule.

**3.** *"An analytical review of the game indicates his drop in form is directly caused by being forced into an unsuitable inverted fullback role."* → **labeled `tactical_analysis`**. "Drop in form" reads like PE, but the argument is structural: the inverted fullback role is mechanically causing the degradation. The conclusion is about how a tactical assignment produces an outcome. Decision: TA.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` — chosen for its lightweight footprint (66M parameters vs BERT-base's 110M) and strong performance on short-text classification tasks. Given the dataset size (~217 training examples after the 39-sample test split), a smaller model was less likely to overfit than a full BERT.

**Training setup:** Standard sequence classification fine-tuning via HuggingFace Transformers. Added a linear classification head on top of the `[CLS]` token output mapping to 3 output classes. Tokenized inputs truncated to 128 tokens (all soccer comments fit well within this limit). 80/20 train/validation split within the non-test portion.

**Key hyperparameter decision — epochs:** Trained for 5 epochs rather than the typical default of 3. On an ~80-example training split per class, 3 epochs produced underfitting — validation loss was still declining. 5 epochs achieved lower validation loss without evidence of overfitting on the two well-separated classes. `performance_evaluation` validation accuracy remained low regardless of epoch count, which in hindsight reflected a data problem rather than a tuning problem.

---

## Baseline Description

**Model:** `llama-3.3-70b-versatile` (via Groq) — zero-shot, no fine-tuning, no few-shot examples.

**System prompt:**

```
You are classifying reddit posts from the r/soccer community.
Assign each post to exactly one of the following categories:

tactical_analysis: Focus on strategic mechanics, formations, analytics, or game rules.
Example: "The 3-4-2-1 system falls apart if the wingbacks don't track back properly."

performance_evaluation: Focus on legacy, individual form, or player and manager assessments.
Example: "He is a generational talent but has been struggling with his finishing this season."

fan_sentiment: Focus on live-match hype, humor, and emotional responses from supporters.
Example: "WHAT A GOAL! I can't believe we actually turned this game around!"

Respond with ONLY the label name. Do not explain your reasoning.

Valid labels:
tactical_analysis
performance_evaluation
fan_sentiment
```

**Collection:** Each of the 39 test-set comments was passed to the model as a user message. The returned label string was matched against ground truth. Only overall accuracy was logged; per-class breakdown was not captured.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy | Correct / Total |
|---|---|---|
| Baseline (Llama 3.3 70B) | **92.31%** | 36 / 39 |
| Fine-Tuned (DistilBERT) | **76.92%** | 30 / 39 |

The fine-tuned DistilBERT underperformed the zero-shot Llama baseline by **15.38%**. This is the central finding and is discussed in the error analysis below.

---

### Per-Class Metrics — Fine-Tuned Model

Derived from the confusion matrix (test set, n = 39).

| Class | Precision | Recall | F1-Score | Support |
|---|---|---|---|---|
| `tactical_analysis` | 0.70 | 1.00 | 0.82 | 14 |
| `performance_evaluation` | 1.00 | 0.31 | 0.47 | 13 |
| `fan_sentiment` | 0.80 | 1.00 | 0.89 | 12 |
| **Macro Average** | **0.83** | **0.77** | **0.73** | 39 |

> **Note:** Per-class metrics for the baseline (Llama 3.3 70B) were not captured during evaluation — only overall accuracy (92.31%) was saved in `evaluation_results.json`. A full comparison would require re-running inference with `classification_report` logging.

---

### Confusion Matrix — Fine-Tuned Model

Rows = true label, columns = predicted label.

|  | Pred: `tactical_analysis` | Pred: `performance_evaluation` | Pred: `fan_sentiment` |
|---|---|---|---|
| **True: `tactical_analysis`** | **14** | 0 | 0 |
| **True: `performance_evaluation`** | 6 | **4** | 3 |
| **True: `fan_sentiment`** | 0 | 0 | **12** |

See `confusion_matrix.png` for the heatmap rendering.

---

### Sample Classifications — Fine-Tuned Model

Five posts run through the fine-tuned model with predicted label and confidence score.

| Text | True Label | Predicted | Conf. | Correct? |
|---|---|---|---|---|
| "The 3-4-2-1 system collapses without disciplined wingback recovery runs — the back three is exposed every time." | `tactical_analysis` | `tactical_analysis` | 0.91 | ✅ |
| "He ran himself into the ground tonight. Even when his passing accuracy dropped, his leadership on the pitch was vital." | `performance_evaluation` | `tactical_analysis` | 0.40 | ❌ |
| "WE ARE GOING TO THE FINALS. I CANNOT BREATHE RIGHT NOW." | `fan_sentiment` | `fan_sentiment` | 0.97 | ✅ |
| "The coach completely got his tactical selection wrong today. Starting a slow midfield against pace was an error." | `performance_evaluation` | `tactical_analysis` | 0.39 | ❌ |
| "Belgium's golden generation ends without a trophy. A collection of world-class individuals who never became a team." | `performance_evaluation` | `performance_evaluation` | 0.72 | ✅ |

**Why the first prediction is reasonable:** The post on the 3-4-2-1 system is a clean `tactical_analysis` hit — it describes a structural dependency (wingback recovery → back-three coverage) as a mechanical truth, not a judgment about any player or coach. There is no evaluative language ("wrong," "should have," "brilliant") and the post would read identically whether the team won or lost. The model's 0.91 confidence reflects that this example sits near the center of the `tactical_analysis` distribution in training.

---

### Error Analysis

The confusion matrix reveals one dominant failure mode: **`performance_evaluation` is the model's blind spot.**

Of the 13 `performance_evaluation` test samples, the model correctly identified only 4 (recall = 0.31). All 9 errors originate from this class — 6 routed to `tactical_analysis`, 3 to `fan_sentiment`. The model never confused `tactical_analysis` with `fan_sentiment` in either direction (0 errors), meaning the poles were cleanly learned but the middle class was not.

#### Why `performance_evaluation` fails: three representative examples

**Example 1 — PE predicted as TA (conf. 0.39):**
> *"The coach completely got his tactical selection wrong today. Starting a slow midfield against pace was an error."*

The phrase "tactical selection" is literally in the text, and the model latched onto it. But this post is not analyzing *how* a tactic functions — it is judging the coach's decision-making competence. The distinction between "person made a bad tactical call" (PE) and "here is how a tactic works mechanically" (TA) is exactly the boundary the model hasn't learned. This is a **training data distribution problem**: the model almost certainly saw far more examples where "tactical" vocabulary co-occurred with `tactical_analysis` labels, so it treats that vocabulary as a decisive TA signal regardless of the framing around it.

**Example 2 — PE predicted as TA (conf. 0.49):**
> *"The substitution was a stroke of genius from the manager. Introducing a dynamic runner destabilized the defense."*

Two sentences pulling in opposite directions. Sentence one is evaluative praise of the manager → PE. Sentence two describes a tactical effect with an action-object construction ("destabilized the defense") → sounds like TA. The model likely weighted the second sentence's structure more heavily. This is a **boundary ambiguity problem**: the post genuinely contains a tactical observation embedded inside a performance judgment. The label is correct (the *primary intent* is praising the manager's decision, per the Primary Intent Rule in `planning.md`), but the model lacks enough multi-clause examples in training to learn that framing determines the label, not isolated clause content.

**Example 3 — PE predicted as FS (conf. 0.37):**
> *"The captain completely led by example tonight. When others panicked, his calmness on the ball kept everyone grounded."*

No technical vocabulary. Positive, emotional register. Praising a player's character. This structurally mirrors dozens of `fan_sentiment` training examples ("great leader," "calm under pressure" comments from live match threads). Without any analytical anchor, the model falls back on tone — and tone says FS. This is a **label definition coverage problem**: `performance_evaluation` posts that evaluate intangibles (leadership, mentality, composure) look almost identical in surface form to fan praise. The training data likely did not include enough examples of this specific sub-type of PE to teach the model the distinction.

#### What would fix this

More training examples specifically for the hard `performance_evaluation` sub-cases: (1) tactical vocabulary used evaluatively rather than analytically, and (2) character/intangible evaluations that don't read as pure fan expression. The label definition is sound — the annotation rule holds under inspection — but the training set doesn't expose the model to these edge cases often enough to learn the boundary at token level. The zero-shot Llama baseline's 92.31% accuracy confirms the labels are consistent: a model reasoning from natural language definitions gets them right. The fine-tuned model's failure is a data volume problem, not a labeling problem.

---

### Definition of Success — Revisited

Pre-project benchmarks from `planning.md`:

| Benchmark | Target | Achieved | Pass? |
|---|---|---|---|
| Macro F1 | ≥ 0.70 | 0.73 | ✅ |
| `tactical_analysis` Precision | ≥ 0.85 | 0.70 | ❌ |

The model clears the macro F1 deployment threshold but fails the use-case-specific precision target. At precision 0.70 on `tactical_analysis`, roughly 3 in 10 posts surfaced by a tactical filter would be `performance_evaluation` posts misclassified as tactics — too noisy for the browser extension use-case.

---

## What the Model Captured vs. What I Intended

The labels were designed around **communicative intent**. What the model learned was closer to **lexical identity** — which tokens reliably co-occur with each label in training.

For `tactical_analysis` and `fan_sentiment` this distinction didn't matter much: tactical posts genuinely do use structural vocabulary, and fan outbursts genuinely do use emotional register. The model's surface-token proxies happened to be accurate, producing perfect recall on both classes.

The gap shows up at `performance_evaluation` (PE). The model never built a *positive* classifier for it — it learned PE by elimination: text that isn't strongly tactical and isn't strongly emotional. When a PE post contained tactical vocabulary ("tactical selection," "poor distribution"), the TA attractor won. When it contained no technical anchor at all ("led by example"), the FS attractor won. The model overfit to **vocabulary presence** as the signal, missing the **evaluative framing** that actually defines the class.

The fix is targeted: more PE training examples for the two hard sub-cases — tactical-vocabulary-wrapped judgments and intangible character evaluations — not a label redesign.

---

## Spec Reflection

**Where the spec helped:** The Primary Intent Rule — classify by what the author is *trying to communicate*, not by which class tokens appear — was the most useful annotation guideline. It resolved most hard edge cases quickly and kept PE labels consistent. Without it, posts like "The coach got his tactical selection wrong" would have been annotated as `tactical_analysis` based on vocabulary, producing systematic noise at exactly the boundary where the model later failed.

**Where implementation diverged:** The spec called for targeted extra collection if any class was underrepresented. I applied this to `tactical_analysis` (pulling from dedicated analysis threads) but not to `performance_evaluation`, which turned out to be the harder class to collect at sufficient volume across its sub-types. In hindsight I should have run the same keyword-targeted collection for PE-specific phrasing ("wrong call," "should have," "cost us") to surface hard-case examples before training.

---

## AI Usage

**Instance 1 — Label stress-testing.** I asked Claude to generate 8–10 adversarial synthetic examples that straddled the `tactical_analysis` / `performance_evaluation` and `tactical_analysis` / `fan_sentiment` boundaries. These exposed that my original `fan_sentiment` definition would have absorbed tactical-vocabulary posts written in frustrated emotional register. For example, "The coach's tactical selection was a disaster. I can't believe he thought that would work." This is clearly a `performance_evaluation` post (judging the coach's decision), but the presence of "tactical selection" and the emotional tone would have misled annotators to label it as `tactical_analysis` or `fan_sentiment`. The AI-generated examples revealed this blind spot in my label definitions, allowing me to clarify the Primary Intent Rule and ensure that the presence of tactical vocabulary alone wouldn't override the evaluative framing. This likely improved annotation consistency and model performance on edge cases.

**Instance 2 — Annotation pre-labeling.** I sent batches of 20–30 scraped comments to Claude with the full label definitions and asked for a predicted label and one-sentence rationale per example. On unambiguous posts (clear breakdowns, clear outbursts) the assignments matched my own. On PE examples involving intangible qualities — leadership, composure, mentality — Claude's assignments were inconsistent, splitting them between `performance_evaluation` and `fan_sentiment` depending on phrasing tone. I overrode roughly 15–20% of PE labels after review. In retrospect, the pattern of AI disagreement on PE-intangible examples was a direct signal that this sub-type needed more training coverage.
