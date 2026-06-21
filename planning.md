# TakeMeter — planning.md

---

## Community & Domain Analysis

### Community Selection
I chose **r/soccer**, specifically focusing on the high-volume traffic generated during tournament matchdays (like the ongoing World Cup). 

### Why It Fits
This community is a perfect fit for a text classification task because its discourse is incredibly varied in intent, substance, and tone. When a match is live, thousands of users comment simultaneously. People can immediately differentiate between someone breaking down a tactical shift and someone just typing a reactionary meme. This stark, structural difference in how arguments are presented makes it ideal for natural language processing.

---

## Taxonomy & Labels

### 1. tactical_analysis (0)
* **Definition:** A post or comment focused strictly on structural mechanics, team formations, technical rule implementations, advanced match statistics, or organizational frameworks of the sport.
* **Example 1:** "The structural transition from a 4-3-3 in possession to a 3-4-2-1 out of possession is the only reason Denmark didn't concede five goals today."
* **Example 2:** "The referee applied the new IFAB clarification on handball perfectly here; the arm was extended making the body unnaturally bigger, and it wasn't a deflection."

### 2. performance_evaluation (1)
* **Definition:** A post or comment evaluating the legacy, psychological state, individual quality, or long-term career trajectory of players, teams, or managers without technical tactical breakdown.
* **Example 1:** "If Belgium goes out in the group stage, De Bruyne's international career ends on the lowest possible note, completely overshadowing his golden generation status."
* **Example 2:** "Croatia’s aging midfield simply couldn't handle the physical intensity of a relentless 90-minute counter-press from a younger side."

### 3. fan_sentiment (2)
* **Definition:** A post or comment driven entirely by immediate emotion, live-match hyperbole, superficial complaints, humor, or tribal fan expression.
* **Example 1:** "WE ARE SUCKING ALL THE WAY TO THE FINALS! UNREAL!"
* **Example 2:** "Are you kidding me?! How did he miss that open net?! My grandmother could have scored that."

---

## Hard Edge Cases

### The Ambiguity
The most common edge case occurs when a user utilizes tactical vocabulary to express a purely visceral emotional reaction. For example: *"Pure anti-football from the opposition. Just parked the bus for 90 minutes and prayed for a draw. Horrible to watch."* The phrase "parked the bus" refers to a low-block defensive tactic, which makes it seem like `tactical_analysis`. 

### Handling Rules
During annotation, this will be handled via a **Primary Intent Rule**. If a post mentions a tactical element but wraps it entirely in hyperbole and emotional complaints without breaking down *how* the structure functioned mechanically, it will be classified under `fan_sentiment`. To fall under `tactical_analysis`, the comment must prioritize structural execution over subjective frustration.

---

## Data Collection Plan

### Source Location
Data will be scraped and sampled directly from active World Cup Match Threads, Post-Match Threads, and Daily Discussion hubs on [r/soccer/new](https://www.reddit.com/r/soccer/new/).

### Collection Strategy
The target is a minimum of 200 total examples, aiming for roughly 65–80 examples per label to keep the dataset balanced. 

### Mitigation Strategy
If `tactical_analysis` is significantly underrepresented after collecting 200 random comments (as match threads lean heavily toward live hype), I will pivot my collection plan to search exclusively for tactical keywords or pull text directly from dedicated tactical analysis threads rather than live match streams.

---

## Evaluation Metrics

### Selected Metrics
I will evaluate the model using **Macro-Averaged F1-Score** alongside a **Confusion Matrix**, rather than relying solely on Accuracy.

### Why They are Right
Because live Reddit data naturally skews heavily toward low-effort emotional commentary (`fan_sentiment`), a model could easily achieve high baseline accuracy just by predicting the majority class. Macro-averaged F1-score averages the precision and recall of each class independently, forcing the model to perform well on the scarcer, highly substantive `tactical_analysis` posts to get a high score. The confusion matrix will pinpoint exactly where the boundaries are bleeding—such as whether tactical rants are being misclassified as pure sentiment.

---

## Definition of Success

### Deployment Benchmark
For this classifier to be genuinely useful as a community filtering tool (e.g., a browser extension that allows users to filter out live match noise to find smart discussion), it needs to achieve a **Macro F1-score of at least 0.70**. 

### Good Enough Threshold
For real-world deployment, I would accept a configuration where `tactical_analysis` has exceptionally high **Precision (0.85+)**, even if its Recall is lower. It is much better for a user filtering for high-quality tactical discussion to miss a few good posts (lower recall) than to have their curated feed flooded with raw emotional screaming misclassified as tactics (poor precision).

---

## AI Tool Plan

### 1. Label Stress-Testing
I will provide my label definitions and edge-case criteria to an LLM, prompting it to generate 5–10 adversarial, boundary-blurring test posts that specifically straddle the line between `tactical_analysis` and `fan_sentiment`. If the model surfaces synthetic examples that cannot be cleanly resolved via the Primary Intent Rule, I will tighten the linguistic boundaries in my definitions before starting the 200-example annotation phase.

### 2. Annotation Assistance
I will use an LLM to pre-label batches of scraped comments to accelerate the workflow. Every automated assignment will be audited and corrected before being pushed into the definitive training partition, so no mistakes are made.

### 3. Failure Analysis
Post-evaluation, all validation inputs that yield mismatched predictions will be exported into a structured JSON log. This failure log will be passed to an LLM with instructions to extract latent patterns, semantic biases, or common vocabulary triggers that caused the model to slip up. I will independently verify the AI's structural error hypotheses by cross-referencing the flagged tokens against the project's original annotation boundaries.
