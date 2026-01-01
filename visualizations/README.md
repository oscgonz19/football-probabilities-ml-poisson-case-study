# Visualizations

> Visual guide to the Football Probability Prediction System architecture and data flows

---

## Sample Output

<p align="center">
  <img src="../images/05_match_prediction.png" alt="Match Prediction Example" width="800"/>
</p>

*A complete match prediction showing all probability markets and expected goals.*

---

## Index

| Visualization | Description | Best For |
|---------------|-------------|----------|
| [System Architecture](./01-system-architecture.md) | High-level component overview | Understanding the big picture |
| [Data Flow Pipeline](./02-data-flow-pipeline.md) | End-to-end data transformation | Following data through the system |
| [ML + Poisson Fallback](./03-ml-poisson-fallback.md) | Hybrid prediction decision tree | Understanding resilience patterns |
| [xG Weighted Calculation](./04-xg-weighted-calculation.md) | Temporal weighting visualization | Statistics methodology |
| [Poisson Score Matrix](./05-poisson-score-matrix.md) | Probability calculation example | Mathematical intuition |
| [Value Detection](./06-value-detection.md) | Market comparison process | Business logic |

---

## Data Insights

### Goals Distribution

<p align="center">
  <img src="../images/01_goals_distribution.png" alt="Goals Distribution" width="700"/>
</p>

Home teams score more on average (1.70 vs 1.34), and both distributions follow the Poisson pattern.

### Team Strength Analysis

<p align="center">
  <img src="../images/03_team_strength.png" alt="Team Strength" width="650"/>
</p>

Attack vs Defense positioning reveals team quality at a glance.

### Score Probability Matrix

<p align="center">
  <img src="../images/06_probability_grid.png" alt="Probability Grid" width="600"/>
</p>

Every cell shows the probability of that exact scoreline occurring.

### Model Calibration

<p align="center">
  <img src="../images/04_calibration.png" alt="Calibration" width="500"/>
</p>

Well-calibrated predictions—the model's confidence aligns with actual outcomes.

---

## Quick Reference

### System Flow (Simplified)

```mermaid
flowchart LR
    A[Raw Data] --> B[xG Stats]
    B --> C{ML Available?}
    C -->|Yes| D[ML Predictions]
    C -->|No| E[Poisson Model]
    D --> F[Probabilities]
    E --> F
    F --> G[Value Detection]
    G --> H[+EV Signals]
```

---

## How to Use These Visualizations

**For Recruiters/PMs**: Start with [System Architecture](./01-system-architecture.md) for the big picture, then [ML + Poisson Fallback](./03-ml-poisson-fallback.md) to understand the resilience patterns.

**For Data Scientists**: Focus on [xG Weighted Calculation](./04-xg-weighted-calculation.md) and [Poisson Score Matrix](./05-poisson-score-matrix.md) for the statistical methodology.

**For Engineers**: [Data Flow Pipeline](./02-data-flow-pipeline.md) shows the complete data transformation sequence.

**For Business Stakeholders**: [Value Detection](./06-value-detection.md) explains how the system identifies opportunities.

---

*All diagrams use Mermaid syntax and render natively on GitHub.*
