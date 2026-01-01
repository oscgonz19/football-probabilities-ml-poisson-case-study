# System Architecture

> High-level view of the Football Probability Prediction System

---

## Component Overview

```mermaid
flowchart TB
    subgraph External["External Services"]
        direction TB
        SPORTS[("Sports Data API<br/>Matches, Teams, Odds")]
        WEATHER[("Weather API<br/>Forecasts, Historical")]
        ML[("ML Prediction Service<br/>Neural Network Model")]
    end

    subgraph Core["Core System"]
        direction TB

        subgraph Ingestion["Data Ingestion Layer"]
            FETCH["Data Fetchers<br/>Rate Limited | Parallel | Retry"]
        end

        subgraph Storage["Persistence Layer"]
            DB[("PostgreSQL")]
            CACHE["Redis Cache<br/>API Responses"]
        end

        subgraph Processing["Processing Engine"]
            STATS["Statistics Calculator<br/>xG Aggregation"]
            PRED["Prediction Pipeline<br/>Hybrid ML + Poisson"]
            VALUE["Value Detector<br/>Odds Comparison"]
        end
    end

    subgraph Output["Outputs"]
        PROB["Probabilities<br/>1X2 | BTTS | O/U"]
        FLAGS["+EV Signals<br/>Value Opportunities"]
        LOGS["Audit Trail<br/>Decision Logs"]
    end

    SPORTS --> FETCH
    WEATHER --> FETCH
    FETCH --> DB
    FETCH --> CACHE
    DB --> STATS
    STATS --> DB
    DB --> PRED
    ML -.->|"Optional"| PRED
    PRED --> VALUE
    VALUE --> PROB
    VALUE --> FLAGS
    PRED --> LOGS
```

---

## Layer Responsibilities

### Data Ingestion Layer

```mermaid
flowchart LR
    subgraph Sources["Data Sources"]
        S1["Sports API"]
        S2["Weather API"]
    end

    subgraph Fetchers["Fetcher Components"]
        F1["Match Fetcher<br/>Scheduled + On-demand"]
        F2["Odds Fetcher<br/>Pre-match timing"]
        F3["Weather Fetcher<br/>48h forecast window"]
    end

    subgraph Features["Resilience Features"]
        R1["Rate Limiting<br/>Token bucket"]
        R2["Retry Logic<br/>Exponential backoff"]
        R3["Partial Success<br/>Continue on errors"]
    end

    S1 --> F1 & F2
    S2 --> F3
    F1 & F2 & F3 --> R1 --> R2 --> R3
```

---

### Processing Engine

```mermaid
flowchart TB
    subgraph Stats["Statistics Module"]
        direction LR
        XG["xG Calculator"]
        WEIGHT["Temporal Weighting<br/>65% / 25% / 10%"]
        OFFSET["Finishing Offset<br/>MLE + Regularization"]
    end

    subgraph Prediction["Prediction Module"]
        direction LR
        ROUTER{"ML<br/>Available?"}
        ML_PATH["ML Client<br/>HTTP + Validation"]
        POISSON["Poisson Engine<br/>Score Matrix"]
        FALLBACK["Fallback Logic"]
    end

    subgraph Value["Value Module"]
        direction LR
        IMPLIED["Implied Odds<br/>1 / probability"]
        COMPARE["Market Comparison"]
        FLAG["Flag Generator"]
    end

    XG --> WEIGHT --> OFFSET
    OFFSET --> ROUTER
    ROUTER -->|"Yes"| ML_PATH
    ROUTER -->|"No"| POISSON
    ML_PATH -->|"Timeout/Error"| FALLBACK --> POISSON
    ML_PATH --> Value
    POISSON --> Value
    IMPLIED --> COMPARE --> FLAG
```

---

## Data Flow Summary

| Stage | Input | Output | Key Logic |
|-------|-------|--------|-----------|
| **Ingestion** | API responses | Raw match data | Rate limiting, retry, partial success |
| **Storage** | Raw data | Normalized tables | Leagues, teams, matches, stats |
| **Statistics** | Historical matches | Weighted xG | 3-season weighted average |
| **Prediction** | Team xG values | Probabilities | Hybrid ML + Poisson |
| **Value Detection** | Probabilities + Odds | +EV flags | Implied vs market comparison |

---

## Key Design Decisions

### Why Hybrid Architecture?

```mermaid
flowchart LR
    subgraph Traditional["Traditional Approach"]
        T1["ML Service"] --> T2["Single Point of Failure"]
        T2 --> T3["System Down"]
    end

    subgraph Hybrid["Our Approach"]
        H1["ML Service"] --> H2{"Available?"}
        H2 -->|"Yes"| H3["ML Predictions"]
        H2 -->|"No"| H4["Poisson Fallback"]
        H3 --> H5["Always Operational"]
        H4 --> H5
    end
```

### Why Poisson as Fallback?

| Aspect | ML Model | Poisson |
|--------|----------|---------|
| Accuracy | Higher (when trained) | Industry baseline |
| Latency | ~500ms | <10ms |
| Reliability | Service-dependent | Always available |
| Interpretability | Black box | Transparent |
| Data needs | Large training set | Small sample OK |

---

*This architecture ensures the system produces predictions continuously, regardless of external service availability.*

---

[Back to Visualizations Index](./README.md)
