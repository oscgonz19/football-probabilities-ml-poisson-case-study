# xG Weighted Calculation

> How historical Expected Goals are combined into match predictions

---

## Goals Distribution in Real Data

<p align="center">
  <img src="../images/01_goals_distribution.png" alt="Goals Distribution" width="700"/>
</p>

*Home teams score more on average (1.70 vs 1.34). These distributions follow the Poisson pattern we use for predictions.*

---

## What is xG (Expected Goals)?

Expected Goals measures the quality of goal-scoring opportunities. Unlike actual goals (which have high variance), xG captures the underlying performance.

```mermaid
flowchart LR
    subgraph Reality["Actual Goals (High Variance)"]
        G1["Match 1: 0 goals"]
        G2["Match 2: 4 goals"]
        G3["Match 3: 1 goal"]
        G4["Match 4: 3 goals"]
        AVG1["Average: 2.0"]
    end

    subgraph XG["Expected Goals (Stable)"]
        X1["Match 1: 1.5 xG"]
        X2["Match 2: 2.1 xG"]
        X3["Match 3: 1.8 xG"]
        X4["Match 4: 1.9 xG"]
        AVG2["Average: 1.83 xG"]
    end

    style AVG2 fill:#ccffcc
```

**xG is a better predictor** because it smooths out the luck factor.

---

## Temporal Weighting

We don't use simple averages. Recent data matters more than old data.

### Weight Distribution

```
Season Weights:

Current Season   ████████████████████████████████████  65%
Previous Season  ██████████████░░░░░░░░░░░░░░░░░░░░░░  25%
Two Prior        ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  10%

```

### Why These Weights?

```mermaid
mindmap
  root((65 / 25 / 10))
    Current Season 65%
      Most relevant
      Current form
      Same squad
      Recent tactics
    Previous Season 25%
      Still valuable
      Core players likely same
      Tactical continuity
    Two Seasons Ago 10%
      Baseline only
      Many changes likely
      Provides stability
```

---

## Season Completion Adjustment

The current season's weight adjusts based on how much of the season has been played.

### Formula

```
Effective Weight = Base Weight × Season Completion %

Example at mid-season (50% complete):
  Current Season: 65% × 0.50 = 32.5% effective
  Previous Season: 25% (unchanged)
  Two Seasons Ago: 10% (unchanged)
  Remaining weight redistributed proportionally
```

### Visual: Weight Progression Through Season

```
Start of Season (0% complete):
Current   ░░░░░░░░░░░░░░░░░░░░  0% (no data yet)
Previous  ██████████████████████████████████  71%
Two Prior ██████████░░░░░░░░░░  29%

Mid-Season (50% complete):
Current   █████████████████░░░  48%
Previous  ███████████████░░░░░  37%
Two Prior ██████░░░░░░░░░░░░░░  15%

End of Season (100% complete):
Current   ████████████████████████████████████  65%
Previous  ██████████████░░░░░░░░░░░░░░░░░░░░░░  25%
Two Prior ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  10%
```

---

## Calculation Example

### Team: Manchester Example FC

**Raw xG Data:**

| Season | Home xG (avg) | Away xG (avg) | Matches |
|--------|--------------|---------------|---------|
| Current (50% done) | 2.1 | 1.4 | 19 |
| Previous | 1.9 | 1.3 | 38 |
| Two Prior | 1.7 | 1.2 | 38 |

### Step 1: Apply Season Completion

```mermaid
flowchart LR
    subgraph Weights["Adjusted Weights"]
        W1["Current: 65% × 50% = 32.5%"]
        W2["Previous: 25% → 37.1%*"]
        W3["Two Prior: 10% → 14.8%*"]
    end

    NOTE["*Redistributed proportionally<br/>to sum to ~84.4%"]
```

Normalization factor: 32.5 + 37.1 + 14.8 = 84.4% → scale to 100%
- Current: 32.5 / 84.4 = 38.5%
- Previous: 37.1 / 84.4 = 44.0%
- Two Prior: 14.8 / 84.4 = 17.5%

### Step 2: Calculate Weighted Average

```
Home xG Calculation:

Current (2.1):   ████████████████████  × 38.5% = 0.81
Previous (1.9):  ██████████████████████████  × 44.0% = 0.84
Two Prior (1.7): ████████████  × 17.5% = 0.30
                 ─────────────────────────────────
                 Total: 1.95 xG
```

### Step 3: Result

| Venue | Weighted xG |
|-------|-------------|
| Home | 1.95 |
| Away | 1.31 |

---

## Home vs Away Split

Why do we track home and away separately?

```mermaid
flowchart TB
    subgraph HomeAdvantage["Home Advantage Effect"]
        H1["Crowd Support"]
        H2["Familiar Pitch"]
        H3["No Travel Fatigue"]
        H4["Tactical Familiarity"]
    end

    subgraph Impact["Statistical Impact"]
        I1["Home teams score<br/>~0.4 more xG on average"]
        I2["Home teams concede<br/>~0.2 less xG on average"]
    end

    subgraph Treatment["Our Approach"]
        T1["Track Home xG separately"]
        T2["Track Away xG separately"]
        T3["Match: Use team's home<br/>vs opponent's away"]
    end

    HomeAdvantage --> Impact --> Treatment
```

---

## Match Prediction Flow

```mermaid
flowchart TB
    subgraph TeamA["Home Team"]
        A_HOME["Home Offensive xG: 1.95"]
        A_AWAY_DEF["Away Defensive xG: 1.42"]
    end

    subgraph TeamB["Away Team"]
        B_AWAY["Away Offensive xG: 1.31"]
        B_HOME_DEF["Home Defensive xG: 1.18"]
    end

    subgraph Calculation["For This Match"]
        CALC_A["Home Expected Goals =<br/>f(A offensive, B defensive)"]
        CALC_B["Away Expected Goals =<br/>f(B offensive, A defensive)"]
    end

    subgraph Result["Match Prediction"]
        RES_A["Home xG: 1.72"]
        RES_B["Away xG: 1.15"]
    end

    A_HOME & B_HOME_DEF --> CALC_A --> RES_A
    B_AWAY & A_AWAY_DEF --> CALC_B --> RES_B
```

---

## Why This Approach Works

| Benefit | Explanation |
|---------|-------------|
| **Recency bias** | Current form weighted 65% at full season |
| **Early season stability** | Historical data fills gaps when current data sparse |
| **Home/away awareness** | Captures venue-specific performance |
| **Smooth transitions** | Gradual weight shifts prevent sudden changes |
| **Noise reduction** | xG more stable than actual goals |

---

## Key Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Current weight | 65% | Strong recency preference |
| Previous weight | 25% | Meaningful historical context |
| Two prior weight | 10% | Baseline stability |
| Min matches for full weight | ~30 | Statistical significance |
| Completion factor | 0-100% | Scales current season contribution |

---

[Back to Visualizations Index](./README.md)
