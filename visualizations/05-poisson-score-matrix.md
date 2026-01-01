# Poisson Score Matrix

> From xG values to match probabilities

---

## The Poisson Distribution

The Poisson distribution models the probability of a given number of events (goals) occurring in a fixed interval, given an expected rate (xG).

### Formula

```
P(k goals) = (λ^k × e^(-λ)) / k!

Where:
  λ = Expected Goals (xG)
  k = Actual goals (0, 1, 2, 3, ...)
  e = Euler's number (~2.718)
```

### Visual: Poisson Distribution Shape

```
λ = 1.5 (typical xG)

P(goals)
│
0.35┤     █
    │    ███
0.25┤   █████
    │  ███████
0.15┤ █████████
    │███████████
0.05┤█████████████
    └──┬──┬──┬──┬──┬──┬──┬──
       0  1  2  3  4  5  6  7  goals

Distribution: 22% | 33% | 25% | 13% | 5% | 1% | 0.3% | ...
```

---

## Building the Score Matrix

### Step 1: Calculate Each Team's Goal Probabilities

Given:
- **Home Team xG: 1.8**
- **Away Team xG: 1.2**

```mermaid
flowchart LR
    subgraph Home["Home Team (λ=1.8)"]
        H0["0 goals: 16.5%"]
        H1["1 goal: 29.8%"]
        H2["2 goals: 26.8%"]
        H3["3 goals: 16.1%"]
        H4["4+ goals: 10.8%"]
    end

    subgraph Away["Away Team (λ=1.2)"]
        A0["0 goals: 30.1%"]
        A1["1 goal: 36.1%"]
        A2["2 goals: 21.7%"]
        A3["3 goals: 8.7%"]
        A4["4+ goals: 3.4%"]
    end
```

### Step 2: Build the Joint Probability Matrix

Multiply independent probabilities:

```
                         AWAY GOALS
              0       1       2       3       4+
         ┌────────────────────────────────────────┐
       0 │  5.0%   6.0%   3.6%   1.4%   0.6%  │
       1 │  9.0%  10.8%   6.5%   2.6%   1.0%  │
HOME   2 │  8.1%   9.7%   5.8%   2.3%   0.9%  │
GOALS  3 │  4.8%   5.8%   3.5%   1.4%   0.5%  │
       4+│  3.3%   3.9%   2.4%   0.9%   0.4%  │
         └────────────────────────────────────────┘

Cell value = P(home=row) × P(away=col)
Example: P(1-1) = 29.8% × 36.1% = 10.8%
```

### Step 3: Color-Coded Matrix

```
                         AWAY GOALS
              0       1       2       3       4+
         ┌────────────────────────────────────────┐
       0 │  ░░░     ▓▓▓     ▓▓▓     ███     ███  │  Away wins
       1 │  ░░░     ▒▒▒     ▓▓▓     ███     ███  │
HOME   2 │  ░░░     ░░░     ▒▒▒     ▓▓▓     ███  │
GOALS  3 │  ░░░     ░░░     ░░░     ▒▒▒     ▓▓▓  │
       4+│  ░░░     ░░░     ░░░     ░░░     ▒▒▒  │  Home wins
         └────────────────────────────────────────┘

Legend:  ░░░ Home Win   ▒▒▒ Draw   ▓▓▓/███ Away Win
```

---

## Extracting Market Probabilities

### 1X2 Market (Match Result)

```mermaid
flowchart TB
    subgraph Matrix["Score Matrix"]
        ALL["All 121 cells<br/>(0-10 × 0-10)"]
    end

    subgraph Extract["Sum Regions"]
        HOME["Home Win<br/>Sum where H > A"]
        DRAW["Draw<br/>Sum where H = A"]
        AWAY["Away Win<br/>Sum where H < A"]
    end

    subgraph Result["1X2 Probabilities"]
        P1["1 (Home): 47.8%"]
        PX["X (Draw): 26.5%"]
        P2["2 (Away): 25.7%"]
    end

    ALL --> HOME & DRAW & AWAY
    HOME --> P1
    DRAW --> PX
    AWAY --> P2
```

**Visual Selection:**

```
HOME WIN cells (H > A):              DRAW cells (H = A):
    0   1   2   3   4                    0   1   2   3   4
  ┌───────────────────┐                ┌───────────────────┐
0 │                   │              0 │ █                 │
1 │ █                 │              1 │     █             │
2 │ █   █             │              2 │         █         │
3 │ █   █   █         │              3 │             █     │
4 │ █   █   █   █     │              4 │                 █ │
  └───────────────────┘                └───────────────────┘
   Sum: 47.8%                           Sum: 26.5%
```

### BTTS Market (Both Teams to Score)

```mermaid
flowchart TB
    subgraph Matrix["Score Matrix"]
        ALL["All cells"]
    end

    subgraph Extract["Identify Regions"]
        BTTS_YES["Both ≥ 1 goal"]
        BTTS_NO["Either team = 0"]
    end

    subgraph Result["BTTS Probabilities"]
        YES["Yes: 53.4%"]
        NO["No: 46.6%"]
    end

    ALL --> BTTS_YES & BTTS_NO
    BTTS_YES --> YES
    BTTS_NO --> NO
```

**Visual Selection:**

```
BTTS YES (both ≥ 1):                 BTTS NO (0 anywhere):
    0   1   2   3   4                    0   1   2   3   4
  ┌───────────────────┐                ┌───────────────────┐
0 │                   │              0 │ █   █   █   █   █ │
1 │     █   █   █   █ │              1 │ █                 │
2 │     █   █   █   █ │              2 │ █                 │
3 │     █   █   █   █ │              3 │ █                 │
4 │     █   █   █   █ │              4 │ █                 │
  └───────────────────┘                └───────────────────┘
   Sum: 53.4%                           Sum: 46.6%
```

### Over/Under 2.5 Goals

```mermaid
flowchart TB
    subgraph Matrix["Score Matrix"]
        ALL["All cells"]
    end

    subgraph Extract["Sum by Total"]
        OVER["H + A > 2.5<br/>(3+ total goals)"]
        UNDER["H + A ≤ 2.5<br/>(0-2 total goals)"]
    end

    subgraph Result["O/U Probabilities"]
        O25["Over 2.5: 56.8%"]
        U25["Under 2.5: 43.2%"]
    end

    ALL --> OVER & UNDER
    OVER --> O25
    UNDER --> U25
```

**Visual Selection:**

```
OVER 2.5 (total ≥ 3):                UNDER 2.5 (total ≤ 2):
    0   1   2   3   4                    0   1   2   3   4
  ┌───────────────────┐                ┌───────────────────┐
0 │             █   █ │              0 │ █   █   █         │
1 │         █   █   █ │              1 │ █   █             │
2 │     █   █   █   █ │              2 │ █                 │
3 │ █   █   █   █   █ │              3 │                   │
4 │ █   █   █   █   █ │              4 │                   │
  └───────────────────┘                └───────────────────┘
   Sum: 56.8%                           Sum: 43.2%
```

---

## Complete Example

### Input

| Team | xG |
|------|-----|
| Home | 1.8 |
| Away | 1.2 |

### Individual Goal Probabilities (0-5 goals)

| Goals | Home | Away |
|-------|------|------|
| 0 | 16.5% | 30.1% |
| 1 | 29.8% | 36.1% |
| 2 | 26.8% | 21.7% |
| 3 | 16.1% | 8.7% |
| 4 | 7.2% | 2.6% |
| 5 | 2.6% | 0.6% |

### Score Matrix (Most Likely Outcomes)

| Score | Probability | Cumulative |
|-------|-------------|------------|
| 1-1 | 10.8% | 10.8% |
| 1-0 | 9.0% | 19.8% |
| 2-1 | 9.7% | 29.5% |
| 2-0 | 8.1% | 37.6% |
| 0-1 | 6.0% | 43.6% |
| 2-2 | 5.8% | 49.4% |
| 3-1 | 5.8% | 55.2% |
| 0-0 | 5.0% | 60.2% |

### Final Market Probabilities

| Market | Outcome | Probability |
|--------|---------|-------------|
| **1X2** | Home (1) | 47.8% |
| | Draw (X) | 26.5% |
| | Away (2) | 25.7% |
| **BTTS** | Yes | 53.4% |
| | No | 46.6% |
| **O/U 2.5** | Over | 56.8% |
| | Under | 43.2% |

---

## Key Insight: Why This Works

```mermaid
flowchart LR
    subgraph Input["Simple Input"]
        XG["Just 2 numbers:<br/>Home xG, Away xG"]
    end

    subgraph Math["Poisson Magic"]
        DIST["Generate<br/>distributions"]
        MATRIX["Build<br/>probability matrix"]
        SUM["Sum relevant<br/>regions"]
    end

    subgraph Output["Rich Output"]
        MARKETS["6 market<br/>probabilities"]
        SCORES["All possible<br/>scorelines ranked"]
    end

    XG --> DIST --> MATRIX --> SUM --> MARKETS & SCORES
```

The Poisson model transforms **2 input values** into **complete probability distributions** for all markets.

---

## Limitations

| Limitation | Description | Mitigation |
|------------|-------------|------------|
| Independence assumption | Assumes teams' goals are independent | Dixon-Coles correction (not implemented) |
| Fixed λ | Same expected goals regardless of game state | ML model captures dynamics |
| No tactical adjustment | Doesn't model formations, playing styles | External features via ML |

---

[Back to Visualizations Index](./README.md)
