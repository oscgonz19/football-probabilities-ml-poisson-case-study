# Football Probability Prediction System

**[Versión en Español](es/README.md)** | English

A case study documenting the architecture and implementation of a sports probability prediction engine that combines ML predictions with traditional Poisson-based statistical methods.

---

## System Overview

```mermaid
flowchart TB
    subgraph DataSources["Data Sources"]
        API1[Sports Data API]
        API2[Weather API]
        API3[ML Service]
    end

    subgraph Core["Core System"]
        ING[Data Ingestion<br/>Rate Limited]
        DB[(PostgreSQL)]
        STATS[Statistics Engine<br/>xG Weighted Averages]
    end

    subgraph Prediction["Prediction Pipeline"]
        ML_CHECK{ML Available?}
        ML_PATH[ML Predictions]
        POISSON[Poisson Fallback]
        VALUE[Value Detection]
    end

    subgraph Output["Output"]
        PROB[Probabilities<br/>1X2, BTTS, O/U]
        FLAGS[+EV Signals]
    end

    API1 & API2 --> ING --> DB --> STATS
    STATS --> ML_CHECK
    API3 -.->|Optional| ML_CHECK
    ML_CHECK -->|Yes| ML_PATH --> VALUE
    ML_CHECK -->|No/Timeout| POISSON --> VALUE
    VALUE --> PROB & FLAGS
```

---

## Key Features

- **Hybrid ML + Statistical Architecture** — ML predictions first, Poisson fallback automatically
- **Partial Response Tolerance** — Handles incomplete ML responses gracefully
- **Contract Testing** — Validates mathematical invariants (probability ranges, sum consistency)
- **Value Detection** — Compares calculated probabilities against market odds

---

## Prediction Flow

```mermaid
flowchart LR
    subgraph Input
        XG[Team xG Values]
    end

    subgraph Process
        MATRIX[Build Score Matrix<br/>11x11 probabilities]
        MARKETS[Calculate Markets]
    end

    subgraph Markets
        M1[1X2]
        M2[BTTS]
        M3[Over/Under]
    end

    subgraph Compare
        IMPLIED[Implied Odds]
        MARKET[Market Odds]
        EV{Value?}
    end

    XG --> MATRIX --> MARKETS
    MARKETS --> M1 & M2 & M3
    M1 & M2 & M3 --> IMPLIED
    IMPLIED --> EV
    MARKET --> EV
    EV -->|Yes| FLAG["+EV Flag"]
    EV -->|No| SKIP[Skip]
```

---

## ML Fallback Strategy

```mermaid
flowchart TB
    START[Prediction Request] --> CHECK{ML Service<br/>Available?}
    CHECK -->|Yes| CALL[Call ML API<br/>timeout: 5s]
    CHECK -->|No| POISSON[Poisson Calculation]

    CALL --> RESPONSE{Response<br/>Status?}
    RESPONSE -->|200 OK| VALIDATE{All Fields<br/>Present?}
    RESPONSE -->|Timeout/Error| POISSON

    VALIDATE -->|Yes| USE_ML[Use ML Predictions]
    VALIDATE -->|Partial| HYBRID[ML + Poisson<br/>for missing fields]

    USE_ML & HYBRID & POISSON --> OUTPUT[Final Probabilities]

    style OUTPUT fill:#90EE90
```

---

## Documentation

| Document | Description | Audience |
|----------|-------------|----------|
| [Main Case Study](football-prediction-case-study.md) | Complete portfolio case study | General |
| [Executive Summary](case-study-executive-summary.md) | High-level overview | Recruiters / Managers |
| [Technical Appendix](case-study-technical-appendix.md) | Detailed technical docs | Tech Leads / Engineers |
| [Pipeline Explained](probability-pipeline-explained.md) | Prediction pipeline | Data Scientists / ML Engineers |
| [Mathematical Formulas](probability-pipeline-formulas.md) | Poisson derivations | Statisticians / Quants |

## Technical Diagrams

More detailed Mermaid diagrams. **[Browse all →](visualizations/README.md)**

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Backend | Ruby on Rails |
| Database | PostgreSQL |
| HTTP Client | Faraday (retry/backoff) |
| Testing | RSpec |
| Optimization | L-BFGS-B |

---

## Related Projects

| Project | Description |
|---------|-------------|
| [football-expected-goals-ml-pipeline](https://github.com/oscgonz19/football-expected-goals-ml-pipeline) | The ML prediction service that integrates with this system |

---

*No credentials, internal URLs, or proprietary implementation details included.*
