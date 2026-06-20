# TakeMeter: Rocket League Discourse Classifier

---

## 1. Community

**Chosen community:** [r/RocketLeague](https://www.reddit.com/r/RocketLeague/) (and the Rocket League competitive subreddit r/RocketLeagueEsports as a secondary source)

Rocket League has one of the most active and opinionated online communities in competitive gaming. Players range from casual hobbyists to Grand Champion/SSL-ranked competitors, and discourse covers everything from mechanical skill breakdowns to team strategy to broader game health complaints. This creates natural variation in post quality, some people back claims with rank data, replay analysis, or pro match examples; others vent frustration as confident proclamations. The gap between "I've thought about this" and "I'm tilted right now" is real, consistent, and recognizable to anyone who spends time in the community. That gap is exactly what makes it a good fit for a classification task.

---

## 2. Labels

### Label 1: `analysis`
**Definition:** The post makes a substantive argument supported by specific, verifiable evidence: rank/MMR data, pro player statistics, mechanical explanations, match examples, or patch notes. The core claim would still hold if you stripped the emotional framing.

**Example A:**
> "Since the hitbox standardization changes, Octane's corner clears have measurably worse angles than Fennec because of the slight nose-up orientation at speed. If you watch Jstn's positioning replays pre- vs post-patch, he shifted toward Fennec around this exact time. The hitbox difference is only ~2%, but on aerial redirects it compounds."

**Example B:**
> "People complaining about solo queue MMR loss don't realize that the system penalizes you less if your team-average MMR was below yours. Check your tracker — your uncertainty rating probably spiked last season because you got promoted too fast and the system is correcting. That's not broken, that's how TrueSkill-style systems work."

---

### Label 2: `hot_take`
**Definition:** A bold, confident opinion stated without meaningful supporting evidence. The post asserts a claim, about a player, mechanic, or game state, but doesn't argue for it. The opinion might be correct, but the post is making a declaration rather than a case.

**Example A:**
> "Squishy is genuinely the best player to ever touch this game and it's not even close. No one has ever had his consistency at the top level. RLCS should retire his jersey number or whatever the esports equivalent is."

**Example B:**
> "Psyonix completely ruined the competitive scene when they sold to Epic. The ranked system is a disaster, pro matches feel hollow, and the game's soul is gone. Change my mind."

---

### Label 3: `rant`
**Definition:** An emotional outburst tied to a specific recent experience — usually a bad game, a perceived injustice in matchmaking, or a teammate behavior complaint. The post is primarily venting rather than making a generalizable argument. Little to no claim beyond "this happened to me and it sucked."

**Example A:**
> "Just went 8-0 in a game while my teammate spent the entire match doing demos on our own half. Lost 9-8. Uninstalling. This community is actually braindead."

**Example B:**
> "Whoever at Psyonix decided to put diamond lobbies in the same casual pool as gold needs to be fired. I just want to chill and I'm getting stomped by smurfs every single game. Not fun."

---

### Label 4: `tip_or_guide`
**Definition:** The post primarily aims to teach something — a mechanic, a rotation principle, a training approach, a settings recommendation. It's instructional in intent, regardless of whether it includes opinion.

**Example A:**
> "For anyone struggling with air dribbles: the single biggest mistake I see is boosting through the ball instead of under it. You want to push from below at ~45°, not chase it. Practice the musty flick first to build the feel, then add the carry. 10 minutes in freeplay per day for two weeks made a bigger difference than any playlist grind."

**Example B:**
> "If you're hardstuck platinum, the problem is almost certainly positioning, not mechanics. Watch your replays and count how many times you double-commit with your teammate. Fix that one thing and you'll climb before you need to learn ceiling shots."

---

## 3. Hard Edge Cases

**The primary edge case:** Posts that include a specific stat or mechanic reference but are clearly written from a tilted emotional state — what I'm calling the "stat-backed rant."

**Example:**
> "I've been hardstuck Diamond 3 for 200 hours and my win rate is literally 49.8%. That is statistically impossible unless the matchmaking is rigged. Psyonix is running a casino, not a ranked system."

This could be `analysis` (cites specific data) or `rant` (emotionally charged, accusatory, no real argument). 

**Decision rule:** If the evidence cited is used to support a coherent, generalizable argument that would convince a skeptical reader, label it `analysis`. If the evidence is deployed to validate a grievance, cited to prove a feeling rather than to reason toward a conclusion, label it `rant`. The above example is a `rant`: the stat is used to justify frustration, not to argue a real point about matchmaking design.

A secondary edge case is posts that blend `hot_take` with `tip_or_guide` (e.g., "Octane is the only correct choice for ranked, here's why you should switch"). **Decision rule:** If the primary purpose is to change behavior/teach, label it `tip_or_guide`. If the primary purpose is to assert a position, label it `hot_take`.

---

## 4. Data Collection Plan

**Sources:**
- r/RocketLeague (primary): posts and top-level comments from the past 6 months, filtered to text posts and comment threads with >10 upvotes
- r/RocketLeagueEsports: pro-scene discussion threads for `analysis` examples
- Reddit search for specific flairs ("Discussion", "Gameplay Question", "Rant")

**Target distribution (200 examples minimum):**

| Label | Target Count | Notes |
|---|---|---|
| `analysis` | 50 | Hardest to find, may need to search pro scene subreddit |
| `hot_take` | 60 | Most common type in r/RL |
| `rant` | 60 | Very abundant, will need to cap to avoid imbalance |
| `tip_or_guide` | 50 | Seasonal post cycles (new season, ranked reset) are good sources |

**If a label is underrepresented after 150 examples:** I'll do a targeted keyword search (e.g., searching "because" or "data" to surface analytical posts, or "uninstalling" / "teammates" to surface rants). If `analysis` is still under 40 after targeted search, I'll lower the minimum to 40 and document the imbalance.

**Data collection approach:** Scrape using the Reddit JSON API (no auth required for public posts) or manually copy-paste into a CSV. Each example will include: the post text (truncated to ~280 characters if long), subreddit, post type (top-level post vs. comment), and my label.

---

## 5. Evaluation Metrics

**Primary metrics:**

- **Per-class F1 score** — because the classes are not equally important and a model that ignores `analysis` (the hardest and most meaningful label) should not score well overall
- **Macro-averaged F1** — treats all four classes equally, appropriate here since each label represents a meaningfully different discourse type
- **Confusion matrix** — to see which labels the model conflates (my hypothesis: `hot_take` and `rant` will be confused most often)

**Why accuracy alone is insufficient:** With four classes and potential imbalance toward `hot_take` and `rant`, a model predicting those two labels constantly could achieve ~60% accuracy while completely missing `analysis` and `tip_or_guide`. Macro F1 forces the model to perform across all classes.

**Baseline comparison:** I'll compare the fine-tuned DistilBERT to zero-shot Llama-3.3-70b on the same test set using the same metrics.

---

## 6. Definition of Success

A classifier is **genuinely useful** for community moderation or flair auto-tagging if:

- Macro F1 ≥ 0.70 on the test set
- Per-class F1 for `analysis` ≥ 0.65 (this is the hardest class and the most valuable to identify correctly)
- The fine-tuned model outperforms the zero-shot baseline by at least 10 percentage points in macro F1

**Acceptable for deployment:** Macro F1 ≥ 0.65 with no single class below F1 = 0.50. Below that threshold, the model is unreliable enough that it would generate user complaints if used for auto-flair or content surfacing.

**If the model beats 95% accuracy:** Audit test set for leakage and check whether labels collapsed (e.g., whether the test set accidentally over-represents easy examples).

---

## 7. AI Tool Plan

### Label stress-testing
Before annotating, I'll give Claude my four label definitions and the edge case description and ask it to generate 10–15 posts that sit at the boundary between `hot_take`/`rant` and `analysis`/`hot_take`. If I can't classify the generated posts cleanly using my definitions, I'll revise the definitions before touching the real data. This is faster than discovering ambiguity 100 examples in.

### Annotation assistance
I'll use Claude to pre-label the first 50 examples as a calibration check, not to replace my labeling, but to surface cases where my intuition and the AI's diverge. Any disagreement becomes a candidate for my "hard cases" section. For the remaining 150, I'll label independently and note which examples felt uncertain. I'll track which examples were pre-labeled in a `prelabeled` column in the CSV (value: 1 or 0) for transparency.

### Failure analysis
After generating predictions on the test set, I'll give Claude the full list of wrong predictions (post text + true label + predicted label) and ask it to identify systematic patterns before I write up the evaluation. I'll verify any pattern it proposes by manually counting instances, I won't accept a pattern claim I can't verify in the data myself.

---

## Stretch Features (Planned for future)

- [ ] **Inter-annotator reliability** — will ask a friend who plays Rocket League to label 30 examples independently and compute Cohen's kappa
- [ ] **Deployed interface** — simple Gradio app that accepts a post and returns label + confidence score
- [ ] **Error pattern analysis** — systematic review of whether short posts, sarcastic posts, or stat-backed rants are disproportionately misclassified

*(This section will be updated before starting each stretch feature.)*
