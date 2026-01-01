# Football Probability Prediction System

**[Versión en Español](es/README.md)** | English

A case study documenting the architecture and implementation of a sports probability prediction engine that combines ML predictions with traditional Poisson-based statistical methods.

---

<p align="center">
  <img src="images/05_match_prediction.png" alt="Match Prediction Dashboard" width="800"/>
</p>

<p align="center"><em>Sample output: 1X2 probabilities, expected goals (xG), BTTS, and Over/Under markets for a single match.</em></p>

---

## Overview

This system calculates pre-match probabilities for football matches and compares them against market odds to identify positive expected value (+EV) opportunities. It features a hybrid architecture with automatic fallback—ensuring predictions complete even when external ML services are unavailable.

**Key Features:**

- **Hybrid ML + Statistical Architecture** — ML predictions first, Poisson fallback automatically
- **Partial Response Tolerance** — Handles incomplete ML responses gracefully
- **Contract Testing** — Validates mathematical invariants (probability ranges, sum consistency)
- **Value Detection** — Compares calculated probabilities against market odds

---

## How It Works

<table>
<tr>
<td width="50%">

### Team Strength Modeling

Each team is positioned by offensive strength (goals scored) vs defensive strength (goals conceded). Teams in the bottom-right quadrant are the strongest.

</td>
<td width="50%">

<img src="images/03_team_strength.png" alt="Team Strength" width="400"/>

</td>
</tr>
<tr>
<td width="50%">

### Poisson Score Matrix

Using expected goals (xG), we build a probability matrix for every possible scoreline. Each cell shows the probability of that exact result.

</td>
<td width="50%">

<img src="images/06_probability_grid.png" alt="Score Matrix" width="400"/>

</td>
</tr>
<tr>
<td width="50%">

### Model Calibration

When we predict 60% probability, events occur ~60% of the time. The closer to the diagonal, the better calibrated the model.

</td>
<td width="50%">

<img src="images/04_calibration.png" alt="Calibration" width="400"/>

</td>
</tr>
</table>

---

## Documentation

| Document | Description | Audience |
|----------|-------------|----------|
| [Main Case Study](football-prediction-case-study.md) | Complete portfolio case study | General |
| [Executive Summary](case-study-executive-summary.md) | High-level overview with architecture diagram | Recruiters / Managers |
| [Technical Appendix](case-study-technical-appendix.md) | Detailed technical documentation | Tech Leads / Engineers |
| [Pipeline Explained](probability-pipeline-explained.md) | How the prediction pipeline works | Data Scientists / ML Engineers |
| [Mathematical Formulas](probability-pipeline-formulas.md) | Poisson distribution and probability derivations | Statisticians / Quants |

## Technical Diagrams

Mermaid-based architecture and flow diagrams. **[Browse all →](visualizations/README.md)**

| Diagram | Description |
|---------|-------------|
| [System Architecture](visualizations/01-system-architecture.md) | Component overview with layer responsibilities |
| [Data Flow Pipeline](visualizations/02-data-flow-pipeline.md) | End-to-end data transformation |
| [ML + Poisson Fallback](visualizations/03-ml-poisson-fallback.md) | Hybrid prediction decision tree |
| [Poisson Score Matrix](visualizations/05-poisson-score-matrix.md) | From xG to match probabilities |

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Backend | Ruby on Rails |
| Database | PostgreSQL |
| HTTP Client | Faraday (retry/backoff) |
| Testing | RSpec |
| Optimization | L-BFGS-B (finishing offset calibration) |

---

## Related Projects

| Project | Description |
|---------|-------------|
| [football-expected-goals-ml-pipeline](https://github.com/oscgonz19/football-expected-goals-ml-pipeline) | The ML prediction service that integrates with this system. Features expected goals modeling, training pipelines, and prediction API endpoints. |

---

## License

This case study documentation is provided for portfolio purposes.

---

*No credentials, internal URLs, or proprietary implementation details are included in this repository.*
