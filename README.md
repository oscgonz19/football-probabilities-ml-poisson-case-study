# Football Probability Prediction System

**[Version en Espanol](es/README.md)** | English

A case study documenting the architecture and implementation of a sports probability prediction engine that combines ML predictions with traditional Poisson-based statistical methods.

---

## Match Prediction Example

<p align="center">
  <img src="images/05_match_prediction.png" alt="Match Prediction Dashboard" width="800"/>
</p>

*Complete prediction output: 1X2 probabilities, expected goals (xG), BTTS, and Over/Under markets.*

---

## Overview

This system calculates pre-match probabilities for football matches and compares them against market odds to identify positive expected value (+EV) opportunities. It features a hybrid architecture with automatic fallback—ensuring predictions complete even when external ML services are unavailable.

### Goals Distribution Analysis

<p align="center">
  <img src="images/01_goals_distribution.png" alt="Goals Distribution" width="750"/>
</p>

*Home teams average 1.70 goals vs 1.34 for away teams—the foundation of our Poisson model.*

---

## How It Works

### 1. Team Strength Modeling

<p align="center">
  <img src="images/03_team_strength.png" alt="Team Strength Analysis" width="700"/>
</p>

Each team is positioned by offensive strength (goals scored) vs defensive strength (goals conceded). Size indicates sample size, color shows goal difference. Teams in the bottom-right quadrant (high attack, low concede) are the strongest.

### 2. Poisson Score Matrix

<p align="center">
  <img src="images/06_probability_grid.png" alt="Score Probability Matrix" width="650"/>
</p>

Using expected goals (xG), we build a probability matrix for every possible scoreline. The most likely result (1-1 at 12.4%) anchors our market calculations.

### 3. Model Calibration

<p align="center">
  <img src="images/04_calibration.png" alt="Calibration Plot" width="550"/>
</p>

*When we predict 60% probability, events occur ~60% of the time. The closer to the diagonal, the better calibrated.*

---

## Historical Performance

<p align="center">
  <img src="images/02_results_by_season.png" alt="Results by Season" width="700"/>
</p>

Four seasons of data show consistent home advantage (~44-46% home wins) with stable patterns—validating our modeling assumptions.

---

## Documentation

| Document | Description | Audience |
|----------|-------------|----------|
| [Main Case Study](football-prediction-case-study.md) | Complete portfolio case study | General |
| [Executive Summary](case-study-executive-summary.md) | High-level overview with architecture diagram | Recruiters / Managers |
| [Technical Appendix](case-study-technical-appendix.md) | Detailed technical documentation | Tech Leads / Engineers |
| [Pipeline Explained](probability-pipeline-explained.md) | How the prediction pipeline works | Data Scientists / ML Engineers |
| [Mathematical Formulas](probability-pipeline-formulas.md) | Poisson distribution and probability derivations | Statisticians / Quants |

## Visualizations

Interactive diagrams and visual explanations of the system. **[Browse all visualizations →](visualizations/README.md)**

| Visualization | Description |
|---------------|-------------|
| [System Architecture](visualizations/01-system-architecture.md) | Component overview with layer responsibilities |
| [Data Flow Pipeline](visualizations/02-data-flow-pipeline.md) | End-to-end data transformation journey |
| [ML + Poisson Fallback](visualizations/03-ml-poisson-fallback.md) | Hybrid prediction decision tree |
| [xG Weighted Calculation](visualizations/04-xg-weighted-calculation.md) | Temporal weighting methodology |
| [Poisson Score Matrix](visualizations/05-poisson-score-matrix.md) | From xG to match probabilities |
| [Value Detection](visualizations/06-value-detection.md) | Market comparison and +EV identification |

---

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

## Related Projects

| Project | Description |
|---------|-------------|
| [football-expected-goals-ml-pipeline](https://github.com/oscgonz19/football-expected-goals-ml-pipeline) | The ML prediction service that integrates with this system. Features expected goals modeling, training pipelines, and prediction API endpoints. |

## License

This case study documentation is provided for portfolio purposes.

---

*No credentials, internal URLs, or proprietary implementation details are included in this repository.*
