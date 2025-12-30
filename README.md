# Football Probability Prediction System

A case study documenting the architecture and implementation of a sports probability prediction engine that combines ML predictions with traditional Poisson-based statistical methods.

## Overview

This system calculates pre-match probabilities for football matches and compares them against market odds to identify positive expected value (+EV) opportunities. It features a hybrid architecture with automatic fallback—ensuring predictions complete even when external ML services are unavailable.

## Documentation

| Document | Description | Audience |
|----------|-------------|----------|
| [Main Case Study](football-prediction-case-study.md) | Complete portfolio case study | General |
| [Executive Summary](case-study-executive-summary.md) | High-level overview with architecture diagram | Recruiters / Managers |
| [Technical Appendix](case-study-technical-appendix.md) | Detailed technical documentation | Tech Leads / Engineers |
| [Pipeline Explained](probability-pipeline-explained.md) | How the prediction pipeline works | Data Scientists / ML Engineers |
| [Mathematical Formulas](probability-pipeline-formulas.md) | Poisson distribution and probability derivations | Statisticians / Quants |

## Key Features

- **Hybrid ML + Statistical Architecture**: Attempts ML predictions first, falls back to Poisson calculations automatically
- **Partial Response Tolerance**: Handles incomplete ML responses gracefully
- **Contract Testing**: Validates mathematical invariants (probability ranges, sum consistency)
- **Weather Integration**: Factors in climate conditions for risk flagging
- **Value Detection**: Compares calculated probabilities against market odds

## Tech Stack

- Ruby on Rails
- PostgreSQL
- Faraday (HTTP client with retry/backoff)
- RSpec (testing)
- L-BFGS-B optimization (finishing offset calibration)

## Architecture

```
Data Sources (Sports API, Weather API, ML Service)
                    │
                    ▼
            Ingestion Layer
                    │
                    ▼
         PostgreSQL Storage
                    │
                    ▼
        Statistics Processing
                    │
                    ▼
    Prediction Pipeline (ML → Poisson Fallback)
                    │
                    ▼
    Output: Probabilities + Value Flags
```

## License

This case study documentation is provided for portfolio purposes.

---

*No credentials, internal URLs, or proprietary implementation details are included in this repository.*
