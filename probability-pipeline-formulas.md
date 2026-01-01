# Probability Pipeline: Mathematical Formulas

> Compact Reference for Statisticians & Quants

---

## Visual Overview

<p align="center">
  <img src="images/06_probability_grid.png" alt="Score Probability Matrix" width="600"/>
</p>

<p align="center"><em>The Poisson score matrix: each cell shows P(home scores i, away scores j). This is the core output of the mathematical pipeline below.</em></p>

---

## 1. Weighted xG Average

```
xG_weighted = Σ(wᵢ × xGᵢ) / Σwᵢ

where:
  w = [65, 25, 10]  (current season, -1, -2)

  w₀_adjusted = w₀ × completion_ratio
  completion_ratio = matches_played / expected_matches
```

If a season lacks valid data (`xG = 0` or `null`), its weight is redistributed proportionally to seasons with data.

---

## 2. League Context Adjustment

```
ratio = league_xG_home_avg / league_xG_away_avg

xG_home = (xG_for_home × ratio + xG_against_away_opponent × (1/ratio)) / 2
xG_away = (xG_for_away × (1/ratio) + xG_against_home_opponent × ratio) / 2
```

This balances offensive expectation with defensive weakness of the opponent, normalized by league averages.

---

## 3. Finishing Offset

### 3.1 Log-Likelihood (Poisson)

```
ℓ(α) = Σᵢ [-λᵢ + gᵢ × ln(λᵢ) - ln(gᵢ!)]

where:
  λᵢ = xGᵢ × e^α
  gᵢ = actual goals in match i
  α  = finishing offset (parameter to optimize)
```

### 3.2 Optimization

```
α_raw = argmax ℓ(α)
        α ∈ [-10, 10]

Method: L-BFGS-B (quasi-Newton with bounds)
```

### 3.3 Partial Pooling (Regularization)

```
α_final = α_raw × (n / (n + k))

where:
  n = number of team's matches
  k = 15 (shrinkage hyperparameter)
```

**Behavior**:
- When `n → 0`: `α_final → 0` (neutral prior)
- When `n → ∞`: `α_final → α_raw` (trust empirical data)

### 3.4 Application

```
xG_final = xG_adjusted × e^(α_final)
```

---

## 4. Poisson Distribution

<p align="center">
  <img src="images/01_goals_distribution.png" alt="Goals Distribution" width="650"/>
</p>

<p align="center"><em>Real data: goals follow Poisson distributions with λ_home ≈ 1.70 and λ_away ≈ 1.34.</em></p>

### 4.1 Probability Mass Function

```
P(X = k | λ) = (e^(-λ) × λ^k) / k!

where:
  X = number of goals
  λ = xG_final (expected goals)
  k ∈ {0, 1, 2, ..., 10}
```

### 4.2 Team Distributions

```
P_home(k) = P(X = k | λ_home)
P_away(k) = P(X = k | λ_away)
```

### 4.3 Score Matrix

```
M[i,j] = P_home(i) × P_away(j)

     Away Goals
      0      1      2      3    ...
Home:
  0   M[0,0]  M[0,1]  M[0,2]  M[0,3]
  1   M[1,0]  M[1,1]  M[1,2]  M[1,3]
  2   M[2,0]  M[2,1]  M[2,2]  M[2,3]
  ...

Note: Assumes independence between home and away goals
```

---

## 5. Market Probabilities

### 5.1 Match Result (1X2)

```
P(Home Win) = Σ M[i,j]  ∀ i > j
P(Draw)     = Σ M[i,i]  ∀ i
P(Away Win) = Σ M[i,j]  ∀ i < j

Verification: P(H) + P(D) + P(A) = 1
```

### 5.2 Both Teams To Score (BTTS)

```
P(BTTS Yes) = 1 - P_home(0) - P_away(0) + M[0,0]
P(BTTS No)  = 1 - P(BTTS Yes)

Alternative form:
P(BTTS Yes) = Σ M[i,j]  ∀ i > 0, j > 0

Verification: P(Yes) + P(No) = 1
```

### 5.3 Over/Under Total Goals

```
P(Over N)  = Σ M[i,j]  ∀ (i + j) > N
P(Under N) = Σ M[i,j]  ∀ (i + j) ≤ N

Example - Over 2.5:
P(Over 2.5) = 1 - [M[0,0] + M[1,0] + M[0,1] + M[2,0] + M[1,1] + M[0,2]]
            = 1 - Σ M[i,j] where i + j ≤ 2

Verification: P(Over N) + P(Under N) = 1
```

---

## 6. Value Detection

### 6.1 Implied Odds

```
implied_odds = 1 / P_calculated
```

### 6.2 Value Condition

```
value_exists = (implied_odds < market_odds)

Equivalent:
value_exists = (P_calculated > 1 / market_odds)
```

### 6.3 Expected Value

```
EV = (P_calculated × market_odds) - 1

Example:
P = 0.45, odds = 2.50
EV = (0.45 × 2.50) - 1 = 0.125 = +12.5%
```

### 6.4 Edge (Advantage)

```
edge = P_calculated - P_implied
     = P_calculated - (1 / market_odds)

Example:
edge = 0.45 - (1/2.50) = 0.45 - 0.40 = 0.05 = 5%
```

---

## 7. Complete Pipeline Pseudocode

```python
def calculate_match_probabilities(match):
    # 1. Get weighted xG (3 seasons)
    home_xg = weighted_average(
        team=match.home_team,
        seasons=last_3_seasons,
        weights=[65, 25, 10],
        adjust_for_completion=True
    )
    away_xg = weighted_average(match.away_team, ...)

    # 2. League context adjustment
    ratio = match.league.xg_home_avg / match.league.xg_away_avg
    home_xg = (home_xg * ratio + opponent_against * (1/ratio)) / 2
    away_xg = (away_xg * (1/ratio) + opponent_against * ratio) / 2

    # 3. Finishing offset
    home_offset = get_finishing_offset(match.home_team)  # MLE + shrinkage
    away_offset = get_finishing_offset(match.away_team)
    home_xg *= exp(home_offset)
    away_xg *= exp(away_offset)

    # 4. Poisson distributions
    home_dist = [poisson_pmf(k, home_xg) for k in range(11)]
    away_dist = [poisson_pmf(k, away_xg) for k in range(11)]

    # 5. Score matrix
    matrix = [[home_dist[i] * away_dist[j]
               for j in range(11)]
               for i in range(11)]

    # 6. Market probabilities
    prob_home = sum(matrix[i][j] for i in range(11) for j in range(i))
    prob_draw = sum(matrix[i][i] for i in range(11))
    prob_away = sum(matrix[i][j] for i in range(11) for j in range(i+1, 11))

    prob_btts_yes = sum(matrix[i][j] for i in range(1,11) for j in range(1,11))
    prob_btts_no = 1 - prob_btts_yes

    prob_over_25 = sum(matrix[i][j] for i in range(11) for j in range(11) if i+j > 2)
    prob_under_25 = 1 - prob_over_25

    # 7. Value detection
    value_home = (1/prob_home) < match.odds_home_win
    value_draw = (1/prob_draw) < match.odds_draw
    value_away = (1/prob_away) < match.odds_away_win

    return {
        'probabilities': {...},
        'value_flags': {...}
    }
```

---

## 8. Parameter Summary

| Parameter | Value | Mathematical Role |
|-----------|-------|-------------------|
| Season weights | [65, 25, 10] | Weighted average coefficients |
| Max k (goals) | 10 | Truncation of infinite Poisson sum |
| Offset bounds | [-10, 10] | Constrained optimization domain |
| Shrinkage k | 15 | Regularization strength in partial pooling |
| Tolerance ε | 0.05 | Acceptable deviation from sum = 1.0 |

---

## 9. Known Limitations

1. **Independence assumption**: Standard Poisson assumes home and away goals are independent. Correlation exists in practice.

2. **Static xG**: Does not account for in-match dynamics (red cards, injuries, tactical changes).

3. **No draw inflation**: Poisson slightly underestimates draws. Dixon-Coles correction addresses this.

4. **Finishing offset stability**: Small sample sizes can produce volatile offsets despite regularization.

---

*Document prepared for public portfolio. Contains standard statistical formulas only.*
