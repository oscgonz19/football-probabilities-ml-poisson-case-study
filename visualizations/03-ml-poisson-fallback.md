# ML + Poisson Fallback

> Hybrid prediction architecture for maximum resilience

---

## The Problem with Single-Path Dependencies

```mermaid
flowchart LR
    subgraph Fragile["Traditional ML-Only Architecture"]
        A["Match Data"] --> B["ML Service"]
        B -->|"Success"| C["Predictions"]
        B -->|"Timeout"| D["No Predictions"]
        B -->|"Error"| D
        B -->|"Partial"| D
        D --> E["System Blocked"]
    end

    style D fill:#ffcccc
    style E fill:#ff9999
```

**Result:** Any ML service issue stops the entire prediction pipeline.

---

## Our Solution: Hybrid Architecture

```mermaid
flowchart TB
    subgraph Input["Input"]
        MATCH["Match with xG Data"]
    end

    subgraph Primary["Primary Path: ML Service"]
        TRY["Attempt ML Prediction"]
        CALL["HTTP Request<br/>Timeout: 5s"]
        RESPONSE{"Response<br/>Status?"}
    end

    subgraph Validation["Response Validation"]
        FULL{"All fields<br/>present?"}
        PARTIAL["Log Warning<br/>Fill Missing"]
    end

    subgraph Fallback["Fallback Path: Poisson"]
        POISSON["Calculate via<br/>Poisson Distribution"]
    end

    subgraph Output["Output"]
        PROBS["Final Probabilities<br/>Always Available"]
    end

    MATCH --> TRY --> CALL
    CALL --> RESPONSE
    RESPONSE -->|"200 OK"| FULL
    RESPONSE -->|"Timeout"| POISSON
    RESPONSE -->|"5xx Error"| POISSON
    RESPONSE -->|"Network Error"| POISSON
    FULL -->|"Yes"| PROBS
    FULL -->|"No"| PARTIAL --> PROBS
    PARTIAL -.->|"Missing fields"| POISSON
    POISSON --> PROBS

    style PROBS fill:#ccffcc
```

---

## Decision Tree

```mermaid
flowchart TB
    START["Start Prediction"]
    Q1{"ML Service<br/>Configured?"}
    Q2{"Connection<br/>Successful?"}
    Q3{"Response<br/>Status 2xx?"}
    Q4{"All Required<br/>Fields Present?"}
    Q5{"Fields Within<br/>Valid Range?"}

    ML_SUCCESS["Use ML Predictions"]
    PARTIAL["Use Partial ML +<br/>Poisson for Missing"]
    POISSON["Use Poisson Only"]

    LOG1["Log: ML unavailable"]
    LOG2["Log: Connection failed"]
    LOG3["Log: API error"]
    LOG4["Log: Partial response"]
    LOG5["Log: Validation failed"]

    START --> Q1
    Q1 -->|"No"| POISSON
    Q1 -->|"Yes"| Q2
    Q2 -->|"No"| LOG2 --> POISSON
    Q2 -->|"Yes"| Q3
    Q3 -->|"No"| LOG3 --> POISSON
    Q3 -->|"Yes"| Q4
    Q4 -->|"No"| LOG4 --> PARTIAL
    Q4 -->|"Yes"| Q5
    Q5 -->|"No"| LOG5 --> POISSON
    Q5 -->|"Yes"| ML_SUCCESS

    style ML_SUCCESS fill:#ccffcc
    style PARTIAL fill:#ffffcc
    style POISSON fill:#cce5ff
```

---

## Failure Scenarios

### Scenario 1: ML Service Timeout

```mermaid
sequenceDiagram
    participant P as Prediction Pipeline
    participant ML as ML Service
    participant POISSON as Poisson Engine

    P->>ML: POST /predict (timeout: 5s)
    Note over ML: Processing takes > 5s
    ML--xP: [Timeout]
    P->>P: Log warning
    P->>POISSON: Calculate fallback
    POISSON-->>P: Probabilities
    P->>P: Mark source = "poisson_fallback"
```

### Scenario 2: ML Service Returns Error

```mermaid
sequenceDiagram
    participant P as Prediction Pipeline
    participant ML as ML Service
    participant POISSON as Poisson Engine

    P->>ML: POST /predict
    ML-->>P: 500 Internal Server Error
    P->>P: Log error with context
    P->>POISSON: Calculate fallback
    POISSON-->>P: Probabilities
    P->>P: Mark source = "poisson_fallback"
```

### Scenario 3: Partial ML Response

```mermaid
sequenceDiagram
    participant P as Prediction Pipeline
    participant ML as ML Service
    participant POISSON as Poisson Engine

    P->>ML: POST /predict
    ML-->>P: 200 OK (missing BTTS fields)
    P->>P: Validate response
    P->>P: Log partial warning
    P->>POISSON: Calculate BTTS only
    P->>P: Merge ML (1X2, O/U) + Poisson (BTTS)
    P->>P: Mark source = "hybrid"
```

---

## Comparison: ML vs Poisson

| Aspect | ML Predictions | Poisson Fallback |
|--------|---------------|------------------|
| **Accuracy** | Higher (when model current) | Industry baseline |
| **Latency** | ~500ms | <10ms |
| **Availability** | Service-dependent | 100% |
| **Features Used** | All available | xG only |
| **Interpretability** | Black box | Fully transparent |
| **Resource Cost** | GPU inference | CPU only |

---

## Why Both Approaches?

```mermaid
mindmap
  root((Hybrid<br/>Architecture))
    ML Strengths
      Pattern recognition
      Multi-feature learning
      Non-linear relationships
      Historical context
    Poisson Strengths
      Always available
      Mathematically sound
      Low latency
      No dependencies
      Industry standard
    Combined Benefits
      Maximum uptime
      Best of both worlds
      Graceful degradation
      Independent evolution
```

---

## Real-World Benefits

### Uptime Comparison

```
Traditional ML-Only:
    ML Up     ████████████████████░░░░ ~85% uptime
    Predictions ████████████████████░░░░ ~85% coverage

Hybrid ML + Poisson:
    ML Up     ████████████████████░░░░ ~85% uptime
    Poisson   ████████████████████████ 100% uptime
    Predictions ████████████████████████ 100% coverage
```

### Quality Distribution

```
When ML is available (85% of time):
    Source: ML predictions
    Quality: Optimal

When ML is unavailable (15% of time):
    Source: Poisson fallback
    Quality: Good baseline (industry standard)

Overall: Never blocked, always producing predictions
```

---

## Implementation Highlights

**Error Handling:**
- Timeouts: Configurable per-endpoint (default 5s)
- Retries: Exponential backoff (1s, 2s, 4s) max 3 attempts
- Circuit breaker: After 5 consecutive failures, skip ML for 60s

**Logging:**
- Every decision logged with match ID
- Fallback reasons tracked for metrics
- Response times recorded for monitoring

**Validation:**
- Probability ranges: 0.0 to 1.0
- Mass check: P(home) + P(draw) + P(away) ≈ 1.0
- xG sanity: 0.0 to 10.0 (reject outliers)

---

[Back to Visualizations Index](./README.md)
