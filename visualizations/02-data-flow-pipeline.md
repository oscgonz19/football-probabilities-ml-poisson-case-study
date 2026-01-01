# Data Flow Pipeline

> Complete journey of data from raw API responses to actionable predictions

---

## Historical Data Overview

<p align="center">
  <img src="../images/02_results_by_season.png" alt="Results by Season" width="700"/>
</p>

*Four seasons of data showing consistent patterns: ~44-46% home wins, ~21-25% draws, ~29-35% away wins.*

---

## End-to-End Flow

```mermaid
flowchart TB
    subgraph Phase1["Phase 1: Data Collection"]
        direction LR
        API["Sports Data API"]
        MATCHES["Match Schedule"]
        TEAMS["Team Stats"]
        ODDS["Market Odds"]
    end

    subgraph Phase2["Phase 2: Enrichment"]
        direction LR
        WEATHER["Weather Data"]
        HISTORY["Historical xG"]
    end

    subgraph Phase3["Phase 3: Transformation"]
        direction TB
        XG_CALC["xG Calculation"]
        LEAGUE_ADJ["League Adjustment"]
        OFFSET["Finishing Offset"]
    end

    subgraph Phase4["Phase 4: Prediction"]
        direction TB
        ML_TRY["Try ML Service"]
        POISSON["Poisson Model"]
        PROBS["Probabilities"]
    end

    subgraph Phase5["Phase 5: Analysis"]
        direction LR
        VALUE["Value Detection"]
        FLAGS["Signal Generation"]
    end

    API --> MATCHES & TEAMS & ODDS
    MATCHES --> Phase2
    WEATHER --> Phase3
    HISTORY --> Phase3
    TEAMS --> XG_CALC
    XG_CALC --> LEAGUE_ADJ --> OFFSET
    OFFSET --> Phase4
    ML_TRY -.-> PROBS
    POISSON --> PROBS
    PROBS --> VALUE --> FLAGS
```

---

## Phase Details

### Phase 1: Data Collection

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant F as Fetcher
    participant API as Sports API
    participant DB as Database

    S->>F: Trigger daily run
    F->>API: GET /matches (today + 7 days)
    API-->>F: Match list

    loop For each match
        F->>API: GET /match/{id}/odds
        API-->>F: Odds data
        F->>DB: Upsert match + odds
    end

    F->>API: GET /teams/stats
    API-->>F: Team statistics
    F->>DB: Update team stats

    Note over F,DB: Rate limited at 10 req/sec
```

**Key Points:**
- Scheduled daily before first matches
- Fetches 7-day window for advance processing
- Rate limited to respect API constraints
- Partial failures don't block entire run

---

### Phase 2: Enrichment

```mermaid
flowchart LR
    subgraph Inputs
        M["Match"]
        STADIUM["Stadium Location"]
    end

    subgraph WeatherFetch["Weather Integration"]
        GEO["Geocode Stadium"]
        FORECAST["Get Forecast"]
        HISTORICAL["Get Historical<br/>(for past matches)"]
    end

    subgraph Output
        RISK["Risk Flag"]
        TEMP["Temperature"]
        WIND["Wind Speed"]
        PRECIP["Precipitation"]
    end

    M --> GEO --> FORECAST
    STADIUM --> GEO
    GEO --> HISTORICAL
    FORECAST --> RISK & TEMP & WIND & PRECIP
    HISTORICAL --> RISK & TEMP & WIND & PRECIP
```

**Weather Thresholds:**
| Condition | Threshold | Effect |
|-----------|-----------|--------|
| Wind | > 50 km/h | High risk flag |
| Rain | > 10 mm/h | High risk flag |
| Temperature | < -5C or > 35C | High risk flag |

---

### Phase 3: Transformation

```mermaid
flowchart TB
    subgraph Raw["Raw Inputs"]
        XG_OFF["Offensive xG<br/>(goals expected to score)"]
        XG_DEF["Defensive xG<br/>(goals expected to concede)"]
        VENUE["Home/Away Split"]
    end

    subgraph Weighting["Temporal Weighting"]
        S1["Current Season<br/>Weight: 65%"]
        S2["Previous Season<br/>Weight: 25%"]
        S3["Two Seasons Ago<br/>Weight: 10%"]
        COMPLETION["Season Completion<br/>Adjustment"]
    end

    subgraph Adjustment["Adjustments"]
        LEAGUE["League Context<br/>Normalize by league avg"]
        FINISHING["Finishing Offset<br/>MLE coefficient"]
    end

    subgraph Final["Final xG"]
        HOME_XG["Home Team xG"]
        AWAY_XG["Away Team xG"]
    end

    XG_OFF & XG_DEF --> S1 & S2 & S3
    S1 & S2 & S3 --> COMPLETION
    VENUE --> COMPLETION
    COMPLETION --> LEAGUE --> FINISHING
    FINISHING --> HOME_XG & AWAY_XG
```

---

### Phase 4: Prediction

```mermaid
flowchart TB
    subgraph Input
        HOME["Home xG: 1.8"]
        AWAY["Away xG: 1.2"]
    end

    subgraph Decision["Prediction Path"]
        TRY_ML{"ML Service<br/>Available?"}
        CALL_ML["Call ML API<br/>timeout: 5s"]
        VALIDATE{"Response<br/>Valid?"}
        USE_ML["Use ML Predictions"]
        FALLBACK["Poisson Fallback"]
    end

    subgraph Poisson["Poisson Calculation"]
        MATRIX["Build Score Matrix<br/>0-10 goals each team"]
        SUM["Sum Probabilities"]
    end

    subgraph Output
        P1X2["1X2: 48% / 27% / 25%"]
        BTTS["BTTS: 58% / 42%"]
        OU["O/U 2.5: 62% / 38%"]
    end

    HOME & AWAY --> TRY_ML
    TRY_ML -->|"Yes"| CALL_ML
    TRY_ML -->|"No"| FALLBACK
    CALL_ML --> VALIDATE
    VALIDATE -->|"Yes"| USE_ML
    VALIDATE -->|"Partial/Error"| FALLBACK
    USE_ML --> Output
    FALLBACK --> MATRIX --> SUM --> Output
```

---

### Phase 5: Analysis

```mermaid
flowchart LR
    subgraph Calculated["Our Probabilities"]
        P_HOME["Home: 48%"]
        P_DRAW["Draw: 27%"]
        P_AWAY["Away: 25%"]
    end

    subgraph Implied["Implied Odds"]
        I_HOME["1/0.48 = 2.08"]
        I_DRAW["1/0.27 = 3.70"]
        I_AWAY["1/0.25 = 4.00"]
    end

    subgraph Market["Market Odds"]
        M_HOME["2.10"]
        M_DRAW["3.40"]
        M_AWAY["4.50"]
    end

    subgraph Comparison["Comparison"]
        C_HOME["2.08 < 2.10<br/>+EV: 1%"]
        C_DRAW["3.70 > 3.40<br/>No value"]
        C_AWAY["4.00 < 4.50<br/>+EV: 12.5%"]
    end

    subgraph Flags["Value Flags"]
        F_HOME["HOME: Minor value"]
        F_AWAY["AWAY: Strong value"]
    end

    P_HOME --> I_HOME --> C_HOME --> F_HOME
    P_DRAW --> I_DRAW --> C_DRAW
    P_AWAY --> I_AWAY --> C_AWAY --> F_AWAY
```

---

## Data State at Each Phase

| Phase | Data Format | Volume | Latency |
|-------|-------------|--------|---------|
| Collection | JSON from API | ~100 matches/day | ~5 min |
| Enrichment | Joined records | Same | ~2 min |
| Transformation | Calculated xG | Same | ~1 min |
| Prediction | Probabilities | Same | ~30 sec (Poisson) |
| Analysis | Value flags | Subset with value | Instant |

---

[Back to Visualizations Index](./README.md)
