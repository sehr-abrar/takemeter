# TakeMeter: Rocket League Discourse Classifier

A fine-tuned text classifier that evaluates discourse quality in the r/RocketLeague community, built for AI201 Project 3.

---

## Community Choice

**Community:** r/RocketLeague (with r/RocketLeagueEsports as a secondary source)

Rocket League has one of the most active and opinionated communities in competitive gaming. Players range from casual hobbyists to Grand Champion/SSL-ranked competitors, and discourse spans mechanical skill breakdowns, team strategy analysis, game health complaints, and coaching advice. This creates natural and recognizable variation in post quality — some people back claims with rank data, replay analysis, or pro match examples; others vent frustration as confident proclamations. The gap between "I've thought carefully about this" and "I'm tilted right now" is real, consistent, and immediately recognizable to anyone who spends time in the community. That gap is what makes it a good fit for a classification task.

---

## Label Taxonomy

### `analysis`
The post makes a substantive argument supported by specific, verifiable evidence — rank/MMR data, pro player statistics, mechanical explanations, match examples, or patch notes. The core claim would still hold if you stripped the emotional framing.

> **Example A:** "Since the last patch, aerial car control has changed — the deadzone adjustments affect how you feather boost during 50/50s. Watch any SSL replay pre vs post patch and the difference is clear."

> **Example B:** "In RLCS data from seasons 11–15, teams with the highest average boost differential correlated with win rate at r=0.71 — stronger than shot accuracy or save rate. Boost control is the highest-leverage team skill."

---

### `hot_take`
A bold, confident opinion stated without meaningful supporting evidence. The post asserts a claim about a player, mechanic, or game state but doesn't argue for it. The opinion might be correct, but the post is making a declaration rather than a case.

> **Example A:** "Squishy is genuinely the best player to ever touch this game and it's not even close. No one has ever had his consistency at the top level."

> **Example B:** "Psyonix completely ruined the competitive scene when they sold to Epic. The ranked system is a disaster, pro matches feel hollow, and the game's soul is gone."

---

### `rant`
An emotional outburst tied to a specific recent experience — usually a bad game, a perceived injustice in matchmaking, or a teammate behavior complaint. The post is primarily venting rather than making a generalizable argument. Little to no claim beyond "this happened to me and it sucked."

> **Example A:** "Just went 8-0 in a game while my teammate spent the entire match doing demos on our own half. Lost 9-8. Uninstalling. This community is actually braindead."

> **Example B:** "Got put into a 2v3 because someone quit in the first minute, played the whole game, nearly won, and still got -10 MMR for losing. Absolutely broken system."

---

### `tip_or_guide`
The post primarily aims to teach something — a mechanic, a rotation principle, a training approach, or a settings recommendation. Instructional in intent, regardless of whether it includes opinion.

> **Example A:** "For anyone struggling with fast aerials: the single biggest mistake is boosting through the ball instead of under it. You want to push from below at ~45°, not chase it."

> **Example B:** "If you're hardstuck platinum, the problem is almost certainly positioning, not mechanics. Watch your replays and count how many times you double-commit with your teammate."

---

## Data Collection

**Source:** The dataset was collected manually from publicly available r/RocketLeague posts using web scraping and data extraction techniques. Posts were gathered from real community discussions to capture authentic player language, including references to professional players (e.g., Jstn, Squishy, Fairy Peak), competitive ranks (Diamond, Grand Champion, SSL), and in-game mechanics (flip resets, musty flicks, speed-flip kickoffs).

**Labeling Process:** After collection, posts were manually reviewed and annotated according to the predefined label definitions. Each annotation was checked for consistency against the edge-case decision rules described in *planning.md*. Posts that fell near the boundary between two labels were either reassigned following review or retained as challenging edge-case examples to improve dataset quality and evaluation robustness.

**Label distribution:**

| Label | Count | Percentage |
|---|---|---|
| `analysis` | 50 | 24.9% |
| `hot_take` | 50 | 24.9% |
| `rant` | 50 | 24.9% |
| `tip_or_guide` | 51 | 25.4% |
| **Total** | **201** | **100%** |

**Train / Validation / Test split:** 141 / 30 / 30 (70% / 15% / 15%), stratified by label.

---

### Difficult-to-Label Examples

**Example 1: The stat-backed rant**
> "I have a 52% win rate, 350 hours, and I'm still hardstuck gold 3. This is statistically not possible unless the system is broken. I cannot be this average."

Could be `analysis` (cites specific numbers) or `rant` (emotional, accusatory). **Decision: `rant`.** The stats are deployed to validate a grievance, not to reason toward a conclusion. A skeptical reader would not be persuaded by the argument — it's frustration dressed up as data.

**Example 2: The opinion-flavored guide**
> "Shadow defense — staying between the ball and your net without committing — is the single most underrated skill in the game below Diamond. Mechanics don't win you games. Not conceding wins you games."

Could be `hot_take` (strong opinion, no evidence) or `tip_or_guide` (teaching a defensive concept). **Decision: `tip_or_guide`.** The primary intent is instructional — it's telling the reader what to do and why. The opinion framing ("most underrated") is incidental.

**Example 3: The feature request**
> "Rocket League needs a proper in-game coaching mode. The replay system is good but it takes too long to navigate for quick review. A 'highlight my worst decision this game' feature would be genuinely transformative for player development."

Could be `hot_take` (asserting an opinion about the game) or `tip_or_guide` (about player development). **Decision: `hot_take`.** The post is not teaching anything — it's advocating for a product change. The player development framing is secondary to the opinion being asserted.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace) — a distilled version of BERT, 66M parameters, well-suited for short text classification tasks.

**Training setup:**
- 3 epochs
- Learning rate: 2e-5
- Batch size: 16 (train), 32 (eval)
- Weight decay: 0.01
- Warmup steps: 50
- Best model selected by validation accuracy

**Hyperparameter decision:** Kept the default 3 epochs rather than increasing to 5. With only 141 training examples, additional epochs risked overfitting — the model would start memorizing specific phrasings in the training set rather than learning generalizable patterns. The final results suggest this concern was warranted; the model already showed signs of poor generalization on the test set.

---

## Baseline Description

**Model:** `llama-3.3-70b-versatile` via Groq API, zero-shot (no fine-tuning, no few-shot examples beyond the label definitions).

**Prompt used:**

```
You are classifying posts from the Rocket League subreddit (r/RocketLeague).
Assign each post to exactly one of the following categories.

analysis: The post makes a substantive argument supported by specific, verifiable evidence...
hot_take: A bold, confident opinion stated without meaningful supporting evidence...
rant: An emotional outburst tied to a specific recent experience...
tip_or_guide: The post primarily aims to teach something...

Respond with ONLY the label name — nothing else, no explanation, no punctuation.
```

Each test example was sent as a separate API call with temperature=0 and max_tokens=20. All 31 test examples returned parseable responses.

---

## Evaluation Report

### Overall Results

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq Llama-3.3-70b) | **0.935** |
| Fine-tuned DistilBERT | 0.516 |
| Difference | -0.419 |

Fine-tuning **regressed** relative to the zero-shot baseline by 41.9 percentage points. This is the headline finding and is discussed in depth in the Reflection section.

---

### Per-Class Metrics — Fine-Tuned Model

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 0.46 | 0.86 | 0.60 | 7 |
| hot_take | 1.00 | 0.12 | 0.22 | 8 |
| rant | 1.00 | 0.38 | 0.55 | 8 |
| tip_or_guide | 0.43 | 0.75 | 0.55 | 8 |
| **macro avg** | **0.72** | **0.53** | **0.48** | 31 |

---

### Per-Class Metrics — Zero-Shot Baseline

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 0.88 | 1.00 | 0.93 | 7 |
| hot_take | 0.88 | 0.88 | 0.88 | 8 |
| rant | 1.00 | 1.00 | 1.00 | 8 |
| tip_or_guide | 1.00 | 0.88 | 0.93 | 8 |
| **macro avg** | **0.94** | **0.94** | **0.94** | 31 |

---

### Confusion Matrix — Fine-Tuned Model

Rows = true label, columns = predicted label.

|  | → analysis | → hot_take | → rant | → tip_or_guide |
|---|---|---|---|---|
| **analysis** | 6 | 0 | 0 | 1 |
| **hot_take** | 4 | 1 | 0 | 3 |
| **rant** | 1 | 0 | 3 | 4 |
| **tip_or_guide** | 2 | 0 | 0 | 6 |

The model predicted `analysis` or `tip_or_guide` for the vast majority of examples regardless of true label. `hot_take` was almost never predicted (recall 0.12), and `rant` was predicted only 3 times total. The model collapsed toward two high-frequency surface patterns rather than learning four distinct classes.

---

### Wrong Predictions — Analysis

**#1: Rant predicted as tip_or_guide**
> "I've had three internet outages this week all during ranked games. I've lost 45 MMR to disconnections that were not my fault. There needs to be a forfeit protection system."

True: `rant` | Predicted: `tip_or_guide` | Confidence: 0.27

The post ends with a suggestion ("there needs to be a forfeit protection system"), which pattern-matches to the instructional framing of tip_or_guide posts. The model appears to have latched onto the prescriptive final sentence rather than reading the overall emotional, grievance-driven structure. This is a labeling boundary problem — the post sits at the edge between rant and hot_take itself, but the tip_or_guide prediction reveals the model is keying on surface phrasing rather than intent.

**#2: Hot_take predicted as analysis**
> "Rocket League is more mechanically demanding than every traditional sport. Coordination, reaction time, spatial reasoning, and strategic play all at once, in a physics environment humans have never evolved for."

True: `hot_take` | Predicted: `analysis` | Confidence: 0.26

This is one of the clearest failure cases. The post lists specific attributes (coordination, reaction time, spatial reasoning) which superficially resembles the enumerated evidence in analysis posts. The model cannot distinguish between "listing things" and "citing verifiable evidence" — a distinction that requires understanding the epistemic status of the claim, not just its surface structure. The very low confidence (0.26) correctly signals the model has no real conviction here.

**#3: Rant predicted as analysis**
> "I have a 52% win rate, 350 hours, and I'm still hardstuck gold 3. This is statistically not possible unless the system is broken. I cannot be this average."

True: `rant` | Predicted: `analysis` | Confidence: 0.26

This is exactly the "stat-backed rant" edge case identified in planning.md. The model did what was predicted — it saw specific numbers (52%, 350 hours) and classified it as analysis. This is a data problem: the training set needed more examples of stat-backed rants explicitly labeled as rant to teach the model that numbers alone don't make something analytical. The decision rule ("is the evidence used to reason or to validate a grievance?") is a pragmatic judgment that a small fine-tuned model on 141 examples cannot learn reliably.

---

### Sample Classifications

| Post (truncated) | True Label | Predicted | Confidence |
|---|---|---|---|
| "In RLCS data from seasons 11–15, teams with the highest average boost differential correlated with win rate at r=0.71..." | analysis | analysis | 0.61 |
| "Squishy Muffinz is the greatest player in RL history. Not Jstn, not Justin, not Fairy Peak..." | hot_take | analysis | 0.25 |
| "Just got demoted from Diamond 2 to Diamond 1 because my teammate spent three games in a row going for every single ball..." | rant | rant | 0.44 |
| "If you're struggling with fast aerials, the key is the timing of your second jump. Jump, tilt forward, THEN boost..." | tip_or_guide | tip_or_guide | 0.52 |
| "NRG will win the next World Championship. Their roster depth is unmatched and Jstn has been in the best form of his career..." | hot_take | tip_or_guide | 0.26 |

**Correct prediction explained:** The analysis post about boost differential was correctly classified with the highest confidence in this sample (0.61). It contains specific numeric evidence (r=0.71), a comparative claim, and a structured conclusion — the surface features most reliably associated with analysis in the training data. This is the clearest case in the dataset and the model handled it well.

---

## Reflection: What the Model Learned vs. What I Intended

The intended learning task was to distinguish four discourse types based on their epistemic structure — how a post reasons, not just what it says. The model instead learned a much simpler proxy: surface phrasing patterns.

The clearest evidence is the confusion matrix. `hot_take` was almost never predicted (recall 0.12), and `rant` was predicted only 3 times. The model collapsed into predicting `analysis` and `tip_or_guide` for most inputs. Why those two? Because they have the most consistent surface-level signatures in the training data: analysis posts contain numbers and named evidence; tip_or_guide posts contain imperative sentences ("try this," "practice that"). The model found these patterns and leaned on them heavily.

`hot_take` and `rant` were hardest to learn because their defining features are structural and intentional rather than lexical. A hot take and a rant can use nearly identical vocabulary. What separates them is whether the post is venting about a specific personal experience or asserting a general claim about the game — a distinction that requires understanding context, not matching words.

---

## Spec Reflection

**One way the spec helped:** The requirement to write label definitions precise enough that "two people reading them would agree on most examples" forced the development of explicit decision rules for edge cases before annotation began. The stat-backed rant rule ("is the evidence used to reason or to validate a grievance?") came directly from this pressure, and it became the most important boundary in the dataset. Without the spec's insistence on specificity, that edge case would have been labeled inconsistently across the 200 examples.

**One way implementation diverged:** The original plan envisioned collecting a larger and more diverse sample of posts from across multiple Rocket League-related communities. In practice, time constraints limited the size of the final dataset to 201 manually collected and annotated examples. This likely contributed to the fine-tuned model's difficulty learning the more nuanced distinctions between `hot_take` and `rant`, as there were relatively few examples available for each discourse pattern. If I were to continue the project, I would expand the dataset substantially and incorporate additional examples from related Rocket League discussion spaces to improve generalization.

---

## AI Usage

**Instance 1: planning.md development**

I developed the project structure, selected Rocket League as the target community, and defined the labeling approach. To speed up drafting, I used Claude to help generate an initial version of the planning.md document based on my requirements. The document included the required sections (community, labels, edge cases, data collection plan, evaluation metrics, definition of success, and AI Tool Plan). I reviewed and revised the content, refined the label definitions and edge-case rules, and finalized the four-label scheme used throughout the project.

**Instance 2: Groq classification prompt**

I designed the zero-shot classification setup and specified the output requirements for the baseline model. Claude assisted in drafting the SYSTEM_PROMPT using the label definitions from planning.md. I reviewed the generated prompt to ensure it matched the project's labeling criteria and output format requirements before using it in Section 5 of the notebook.
