# Just Go Mid and Fight: Which Tier 1 Region in League Has the Most Action Packed Games?

**By Matthew Do & Amrith Krishnakumar** | DSC 80 Final Project

---

## Introduction

League of Legends is one of the world's most watched esports, with four major "tier 1" professional leagues: the **LCK** (Korea), **LPL** (China), **LEC** (Europe), and **LCS** (North America). The LCK and LPL are known for dominating the international stage. Each league differs in play style.

**Central Question:** *Among the four tier 1 leagues, which has the most "action packed" games and is that difference statistically significant?*

We define "action" as **Kills Per Minute (KPM)**: total kills in a game divided by its duration. As KPM tells us how many fights are actually significant and lead to a change in gold.
This analysis uses the **[Oracle's Elixir 2022 LoL Esports Dataset](https://oracleselixir.com/)** which recorded every professional match played in 2022. After filtering to the four main tier 1 leagues, the dataset contains **21,624 rows** covering **1,802 unique games**.

### Relevant Columns

| Column | Type | Description |
|--------|------|-------------|
| `gameid` | nominal | Unique ID for each match |
| `league` | nominal | Region (LCK, LPL, LEC, LCS) |
| `side` | nominal | Which side the team played (Blue / Red) |
| `kills` | quantitative | Kills by this player or team in the game |
| `gamelength` | quantitative | Match duration in seconds |
| `result` | binary | Outcome: 1 = win, 0 = loss |
| `firstblood` | binary | Did the team secure first blood? |
| `firstdragon` | binary | Did the team secure first dragon? |
| `firsttower` | binary | Did the team secure first tower? |
| `golddiffat15` | quantitative | Gold differential vs. opponent at 15 minutes |
| `xpdiffat15` | quantitative | XP differential vs. opponent at 15 minutes |
| `csdiffat15` | quantitative | CS (creep score) differential vs. opponent at 15 minutes |
| `killsat15` | quantitative | Team kills at 15 minutes |
| `deathsat15` | quantitative | Team deaths at 15 minutes |
| `assistsat15` | quantitative | Team assists at 15 minutes |
| `datacompleteness` | nominal | `'complete'` or `'partial'` (some leagues omit time snapshots) |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

The raw dataset mixes **player rows** and **team summary rows**. We split these into two DataFrames:
- `df_players`: individual player statistics
- `df_teams`: team level aggregates used for game level analysis

We then:
1. Converted `gamelength` from seconds to minutes (`game_duration_min`).
2. Built a **game level DataFrame** (`game_kills`) with one row per game by grouping `df_teams` on `gameid` and summing both teams' kills. This avoids double counting.
3. Dropped the 0 games with missing `game_duration_min` or `kills` (none were dropped).
4. Computed `kpm = total_kills / game_duration_min`.

The head of the cleaned game level DataFrame is shown below:

| gameid | total_kills | game_duration_min | league | kpm |
|--------|-------------|-------------------|--------|-----|
| 8401-8401_game_1 | 19 | 22.75 | LPL | 0.8352 |
| 8401-8401_game_2 | 30 | 24.07 | LPL | 1.2465 |
| 8402-8402_game_1 | 20 | 31.55 | LPL | 0.6339 |
| 8402-8402_game_2 | 19 | 37.85 | LPL | 0.5020 |
| 8402-8402_game_3 | 27 | 31.67 | LPL | 0.8526 |

### Univariate Analysis

<iframe
  src="assets/kpm_distribution.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

The distribution of KPM across all 1,802 tier 1 games is roughly bell shaped and slightly right skewed centered around 0.75 kills per minute. Most games fall between 0.4 and 1.2 KPM, with a small amount of high action games exceeding 1.5 KPM. This range suggests meaningful variation in combat intensity both within and across leagues.

### Bivariate Analysis

<iframe
  src="assets/kpm_by_league.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

The box plot reveals clear differences in KPM across leagues. LPL games have the highest median KPM at 0.82 and widest spread, consistent with the LPL's reputation for having the most aggressive playstyle. LEC is second at 0.78, LCS third at 0.72, and LCK has the lowest median at 0.69. The medians are meaningfully separated, suggesting these differences are not simply noise.

### Interesting Aggregates

The table below summarizes action metrics by league. LPL leads in both mean and median KPM as well as average total kills while also having the shortest average game duration that is consistent with an aggressive playstyle that ends games quickly.

| League | Games | Mean KPM | Median KPM | Mean Total Kills | Mean Duration (min) |
|--------|-------|----------|------------|-----------------|---------------------|
| LCK    | 467   | 0.700    | 0.688      | 23.099          | 33.668              |
| LPL    | 786   | 0.839    | 0.817      | 26.240          | 31.557              |
| LEC    | 243   | 0.804    | 0.780      | 26.490          | 33.221              |
| LCS    | 306   | 0.733    | 0.719      | 23.964          | 33.026              |

---

## Assessment of Missingness

### NMAR Analysis

We believe the **`url`** column is likely **NMAR** (Not Missing At Random). Roughly 54% of team level rows are missing a `url`. Whether a broadcast URL exists and remains accessible depends on broadcaster and streaming platform decisions such as whether the VOD was published, and whether it was later removed for copyright or streaming rights reasons. These decisions correlate with a match's viewership and commercial value as a dramatic playoff series is far more likely to remain publicly linked than a random regular season game.

### Missingness Dependency

We analyzed whether the missingness of **`golddiffat15`** (missing for 43.6% of team rows — all LPL games) depends on other columns.

**Test 1: Does missingness depend on `league`?**

We used **Total Variation Distance (TVD)** as the test statistic comparing the league distribution of rows where `golddiffat15` is missing vs. not missing. With 1,000 permutations:

- Observed TVD: **1.0000**
- p-value: **0.0000**

We reject the null hypothesis. The missingness of `golddiffat15` is **MAR on `league`** as LPL's data provider does not record 15 minute snapshots so every LPL row is missing this column while every LCK, LCS, and LEC row has it.

<iframe
  src="assets/missingness_dist.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

The observed TVD of 1.0 falls far outside the null distribution (all permuted TVDs are near 0) confirming that the dependency is not due to chance.

**Test 2: Does missingness depend on `result` (win/loss)?**

We used the **absolute difference in missing proportions** between winners and losers as the test statistic:

- Missing rate for winners: 43.62%
- Missing rate for losers: 43.62%
- Observed |difference|: **0.000000**
- p-value: **1.0000**

We fail to reject the null hypothesis. The missingness of `golddiffat15` **does not depend on** `result`. Games are equally likely to have partial data regardless of which team won. Data completeness is determined by the LPL's recording policy and not by game outcomes.

---

## Hypothesis Testing

**Null Hypothesis (H₀):** The mean KPM is the same across all four tier 1 regions (LCK, LPL, LEC, LCS). Any observed differences in group means are due to random chance.

**Alternative Hypothesis (Hₐ):** At least one tier 1 region has a meaningfully different mean KPM from the others.

**Test Statistic:** The variance of the four group mean KPMs. A large variance indicates that the region's average action levels are spread far apart. We use a permutation test with 10,000 shuffles of the `league` labels to build a null distribution.

**Significance Level:** α = 0.05

**Results:**
- Observed variance of group means: **0.004054**
- p-value: **≈ 0.0000**

**Conclusion:** We reject H₀. The differences in mean KPM across tier 1 regions are statistically significant and are extremely unlikely to be due to chance. LPL's notably higher KPM at 0.839 and LCK's lower KPM at 0.7 drive the variance. This does not prove causation as factors like patch version, team compositions, and the current meta could all contribute but the evidence strongly suggests region and cultural identity is associated with systematic differences in fighting frequency.

<iframe
  src="assets/hypothesis_test.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

The observed variance of 0.004054 falls far outside the null distribution confirming statistical significance.

---

## Predicting Wins

**Problem:** We are predicting whether a team **wins** (`result = 1`) or **loses** (`result = 0`) a professional LoL match using only information available at the **15 minutes**.

**Type:** Binary Classification

**Response Variable:** `result`: the single most important outcome of any match. Predicting it from mid game simulates the kind of real time win probability model used by analysts and broadcast tools.

**Features**:

| Feature | Type | Description |
|---------|------|-------------|
| `golddiffat15` | quantitative | Team gold advantage at 15m |
| `xpdiffat15` | quantitative | Team XP advantage at 15m |
| `csdiffat15` | quantitative | Team CS advantage at 15m |
| `killsat15` | quantitative | Team kills at 15m |
| `deathsat15` | quantitative | Team deaths at 15m |
| `assistsat15` | quantitative | Team assists at 15m |
| `firstblood` | binary | Did the team secure first blood? |
| `firstdragon` | binary | Did the team secure first dragon? |
| `firsttower` | binary | Did the team secure first tower? |
| `side` | nominal | Blue or Red side |

**Evaluation Metric:** **Accuracy**: every game produces exactly one winner and one loser so classes are perfectly balanced (50/50). Accuracy is therefore a fair metric with no class imbalance bias. A naive classifier that always predicts "win" would score exactly 50% and our model will beat this.

**Data used:** `df_teams` filtered to `datacompleteness == 'complete'` rows only. All LPL games are `'partial'` as they are missing the 15 minute snapshot columns and must be excluded.

---

## Baseline Model

**Model:** Logistic Regression in a single sklearn `Pipeline`

**Features (3):**
- `golddiffat15`: quantitative, standardized with `StandardScaler`
- `xpdiffat15`: quantitative, standardized with `StandardScaler`
- `side`: nominal (Blue/Red), one hot encoded with `OneHotEncoder(drop='first')`

The two quantitative features are the most directly informative 15 minute signals as a gold lead and an XP lead both strongly correlate with map control and snowball potential. `side` is included because Blue side has structural advantages such as first pick priority that can affect win rate independently of a team's performance.

**Performance:**
- Train accuracy: **71.75%**
- Test accuracy: **74.69%**
- Naive baseline: **50.00%**

The baseline achieves a meaningful 24 point improvement over the naive baseline using only 2 features, but it leaves a lot on the table by ignoring kills, objectives, and CS differentials that are also known at 15 minutes.

---

## Final Model

**New Features Engineered:**

1. **`kill_diff_at15`** = `killsat15 − deathsat15`: the net kill advantage at 15 minutes. Raw kill and death counts are informative individually, but their difference captures the *direction* of the early game more cleanly. A team with 3 kills and 1 death is usually in a better position than one with 3 kills and 3 deaths.

2. **`resource_adv_at15`** = `golddiffat15 + xpdiffat15`: a composite of the two biggest resource leads. Gold buys items and XP shows scaling and abilities. Their sum approximates total snowball potential. Teams with a large combined lead have opportunity to continue to extend their lead.

These features are grounded in the data generating process: Most league matches are decided before 20 minutes and the 15 minute resource state encodes most of the information about who has early control.

**Additional features added**: `csdiffat15`, `killsat15`, `deathsat15`, `assistsat15`, `firstblood`, `firstdragon`, `firsttower`.

**Model:** `RandomForestClassifier`: chosen because win probability at 15 minutes is nonlinear. A gold lead only matters greatly when combined with objectives such as dragons, towers and baron. Random Forests capture these interaction effects naturally which logistic regression cannot.

**Hyperparameter Tuning (GridSearchCV, 5 fold CV):**

| Hyperparameter | Values searched | Best value |
|----------------|-----------------|------------|
| `n_estimators` | 100, 200 | **200** |
| `max_depth` | 5, 10, None | **5** |
| `min_samples_split` | 2, 5 | **5** |

Best CV accuracy: **71.38%** (on training data)

**Performance:**
- Final Model Test accuracy: **75.18%**
- Baseline Test accuracy: **74.69%**
- Improvement: **+0.49 percentage points**

The final model improves on the baseline by adding more informative features and using a model that can capture feature interactions. The best `max_depth=5` and `min_samples_split=5` indicate that moderate regularization generalizes best to unseen games. I feel like we aren't able to improve the model as much because there are many other bigger issues that cause an incorrect prediction. For example, if a team drafts a more early game focused team composition and so they are inherently going to have a gold lead early versus a late game team composition but have good odds of losing the game later on.

---

## Fairness Analysis

**Groups:**
- Group X: teams that played on **Blue side**
- Group Y: teams that played on **Red side**

Blue side historically has structural advantages in LoL such as having first pick priority and an easier camera maneuvering perspective. We ask: does our model predict equally well for both sides or does it systematically advantage one over the other?

**Evaluation metric:** Accuracy

**Null Hypothesis (H₀):** The model is fair and its accuracy for Blue side and Red side teams are roughly the same and any difference is due to random chance.

**Alternative Hypothesis (Hₐ):** The model is unfair and accuracy differs between Blue side and Red side teams.

**Test statistic:** |accuracy_blue − accuracy_red|  
**Significance level:** α = 0.05

**Results:**
- Accuracy for Blue side: **73.53%**
- Accuracy for Red side: **76.85%**
- Observed |difference|: **0.0332**
- p-value: **0.4890** (1,000 permutations)

**Conclusion:** We fail to reject H₀. The observed accuracy gap of 3.32 percentage points between Blue and Red side is well within the range of random variation (p = 0.489). We cannot conclude that the model is unfair with respect to which side a team played on.

<iframe
  src="assets/fairness_dist.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

The observed difference (red) falls near the center of the null distribution confirming that the gap is not statistically significant.
