# Probability Calculation Pipeline

> Technical Explanation for Data Scientists & ML Engineers

---

<p align="center">
  <img src="images/06_probability_grid.png" alt="Score Probability Matrix" width="650"/>
</p>

<p align="center"><em>The end result: a probability matrix for every possible scoreline, derived from expected goals.</em></p>

---

## Overview

This document explains how raw team statistics are transformed into match probabilities through a five-stage pipeline. Each stage adds a layer of adjustment that improves predictive accuracy. The system uses Expected Goals (xG) as its core metric and Poisson distribution for probability calculations.

---

## Stage 1: Base xG Retrieval

**What it does**: Retrieves historical Expected Goals (xG) for each team, differentiated by home and away performance.

**Why it matters**: xG captures the quality of goal-scoring opportunities created/conceded, making it a better predictor than actual goals (less noise from variance).

**Data used**:
- Offensive xG (goals expected to score) — split by home/away
- Defensive xG (goals expected to concede) — split by home/away

**Temporal weighting**: Uses a weighted average of the last 3 seasons:

| Season | Weight |
|--------|--------|
| Current | 65% |
| Previous | 25% |
| Two prior | 10% |

The current season weight is dynamically adjusted by completion percentage. If only 50% of matches have been played, the current season's influence is halved, giving more weight to historical data.

**ML Analogy**: Similar to how we might weight recent training data more heavily in online learning, but with explicit recency decay.

<p align="center">
  <img src="images/01_goals_distribution.png" alt="Goals Distribution" width="700"/>
</p>

<p align="center"><em>Real data: home teams score more (1.70 avg) than away teams (1.34 avg). Both follow Poisson distributions.</em></p>

---

## Stage 2: League Context Adjustment

**What it does**: Normalizes each team's xG considering the offensive/defensive characteristics of their league.

**Why it matters**: Leagues have different average goal rates. A team with 1.5 xG in a defensive league isn't equivalent to 1.5 xG in an attacking league.

**Mechanism**:
1. Calculate the league's home/away xG ratio
2. Use this ratio to balance the team's offensive expectation with the opponent's defensive weakness
3. Average both perspectives for the final adjusted xG

**Intuition**: This is conceptually similar to standardizing features by their distribution in ML preprocessing—here we normalize by the league's "baseline."

**Example**:
- Team A has xG 1.8 (offensive) in a league where home teams average 1.6 xG
- Team B has xG 1.8 in a league where home teams average 1.2 xG
- After adjustment, Team A's effective xG will be lower relative to their league context

<p align="center">
  <img src="images/03_team_strength.png" alt="Team Strength" width="700"/>
</p>

<p align="center"><em>Team strength visualization: attack (x-axis) vs defense (y-axis). Bottom-right = strongest teams. Color = goal difference.</em></p>

---

## Stage 3: Finishing Efficiency Offset

**What it does**: Adjusts xG based on each team's historical efficiency at converting chances into goals.

**Why it matters**: Some teams consistently outperform their xG (good finishers), others underperform. Ignoring this introduces systematic bias.

**Mechanism**:
1. Calculate a coefficient (alpha) per team using Maximum Likelihood Estimation over their historical matches
2. Apply this coefficient as an exponential multiplier to xG

**Regularization (Partial Pooling)**:
- Teams with few matches have their offset "shrunk" toward zero (neutral prior)
- More matches = more confidence in the empirical offset
- This prevents overfitting on teams with limited data

**ML Analogy**: Conceptually similar to random effects in mixed models, or how Bayesian inference shrinks estimates toward the prior with limited evidence. The formula uses:

```
alpha_final = alpha_raw × (n_matches / (n_matches + k))
```

Where `k` is a hyperparameter (typically 15) controlling shrinkage strength.

---

## Stage 4: Poisson Probability Calculation

**What it does**: Transforms adjusted xG values into probability distributions for goals, then into market probabilities.

**Why it matters**: Poisson distribution is the industry standard for modeling count events (goals) with low frequency and high variance.

**Mechanism**:
1. Each team's final xG becomes the lambda parameter of their Poisson distribution
2. Calculate probability of scoring 0, 1, 2, ... 10 goals for each team
3. Build a score matrix with all possible combinations
4. Sum relevant combinations to get market probabilities

**Markets calculated**:

| Market | Calculation |
|--------|-------------|
| Home Win | Sum of (home > away) combinations |
| Draw | Sum of (home = away) combinations |
| Away Win | Sum of (home < away) combinations |
| BTTS Yes | 1 - P(home=0) - P(away=0) + P(both=0) |
| Over 2.5 | Sum of combinations where total > 2 |
| Under 2.5 | Sum of combinations where total ≤ 2 |

**Known limitation**: Standard Poisson assumes independence between teams' goals. In reality, there's correlation (e.g., a team ahead may play more defensively). More sophisticated models (Bivariate Poisson, Dixon-Coles) address this but add complexity.

---

## Stage 5: Value Detection

**What it does**: Compares calculated probabilities against market odds to identify positive expected value opportunities.

**Why it matters**: The ultimate goal isn't predicting results—it's finding discrepancies between true probability and implied probability in the odds.

**Mechanism**:
1. Convert probability to implied odds: `implied_odds = 1 / probability`
2. Compare against market odds
3. If `implied_odds < market_odds` → value exists

**Example**:
- Calculated home win probability: 45%
- Implied odds: 1 / 0.45 = 2.22
- Market odds: 2.50
- Since 2.22 < 2.50 → **value detected**

**Interpretation**: Betting at 2.50 odds when the true probability is 45% yields +12.5% expected value.

<p align="center">
  <img src="images/04_calibration.png" alt="Model Calibration" width="550"/>
</p>

<p align="center"><em>Calibration validation: predicted probabilities align with observed frequencies. Essential for value detection.</em></p>

---

## Pipeline Flow Diagram

```
┌─────────────────────┐
│  Historical Stats   │
│  (xG by team/venue) │
└──────────┬──────────┘
           │
           ▼
┌──────────────────────────────────┐
│  1. Base xG Retrieval            │
│  Weighted average (65/25/10)     │
│  across 3 seasons                │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  2. League Context Adjustment    │
│  Normalize by league home/away   │
│  ratio                           │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  3. Finishing Offset             │
│  xG × e^α                        │
│  (MLE + partial pooling)         │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  4. Poisson Model                │
│  P(k goals) for each team        │
│  → Score matrix → Probabilities  │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  5. Value Detection              │
│  implied_odds vs market_odds     │
└──────────┬───────────────────────┘
           │
           ▼
┌─────────────────────┐
│  Output:            │
│  - Probabilities    │
│  - Value flags      │
└─────────────────────┘
```

---

## Design Rationale

**Why not use an end-to-end model (Neural Network, XGBoost)?**

1. **Interpretability**: Each stage has sporting meaning, facilitating debugging and stakeholder communication

2. **Modularity**: Individual stages can be improved without retraining everything

3. **Data efficiency**: Poisson works well with small samples; deep learning would require orders of magnitude more data

4. **Solid baseline**: This approach is the industry standard, validated by decades of use in sports analytics

**Possible extensions**:
- Incorporate additional features (injuries, weather, motivation)
- Use Dixon-Coles correction for goal correlation
- Ensemble with ML model for edge cases

---

## Key Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Season weights | 65 / 25 / 10 | Balance recency vs stability |
| Max goals modeled | 10 | P(>10) ≈ 0 for typical xG values |
| Offset bounds | [-10, 10] | Practical range (e^10 ≈ 22000x) |
| Shrinkage k | 15 | ~15 matches for 50% confidence |
| Probability decimals | 4 | Sufficient precision for odds |

---

*For complete mathematical formulas, see [Mathematical Appendix](./probability-pipeline-formulas.md)*

---

*Document prepared for public portfolio. Contains no proprietary implementation details.*
