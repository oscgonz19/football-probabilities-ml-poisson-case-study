# Football Probability Prediction System: Technical Appendix

> Case Study - Detailed Technical Documentation

---

## 1. System Overview

The prediction system calculates match probabilities using a hybrid approach that combines external ML predictions with traditional statistical methods. The architecture prioritizes availability through automatic fallback, validates all outputs against strict invariants, and handles partial or malformed responses gracefully.

### Core Design Principles

- **Fail-safe by default**: Any external service failure triggers fallback to internal calculations
- **Validate everything**: All probabilities are checked against mathematical invariants before persistence
- **Log with context**: Every operation includes identifiers for debugging without exposing sensitive data
- **Test at boundaries**: Unit tests cover logic; regression tests validate external contracts

---

## 2. Prediction Flow Architecture

### 2.1 High-Level Flow

```
Match Input
    │
    ▼
┌─────────────────────┐
│  Climate Filter     │ → Flags high-risk weather conditions
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  xG Assignment      │ → Assigns pre-match expected goals
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐     ┌─────────────────────┐
│  ML Prediction      │────▶│  Response Validator │
│  (if enabled)       │     │  (checks structure) │
└─────────┬───────────┘     └─────────┬───────────┘
          │                           │
          │ failure/timeout           │ invalid
          ▼                           ▼
┌─────────────────────┐     ┌─────────────────────┐
│  Traditional Calc   │◀────│  Partial Fill       │
│  (Poisson-based)    │     │  (hybrid response)  │
└─────────┬───────────┘     └─────────────────────┘
          │
          ▼
┌─────────────────────┐
│  Invariant Check    │ → Validates sums, ranges, consistency
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Value Detection    │ → Compares vs market odds
└─────────┬───────────┘
          │
          ▼
    Persist Results
```

### 2.2 Fallback Strategy

The system implements a three-tier fallback strategy:

| Tier | Condition | Action |
|------|-----------|--------|
| 1 | ML service returns valid complete response | Use ML predictions directly |
| 2 | ML service returns partial response | Use ML for available markets, fill missing with Poisson |
| 3 | ML service fails/times out/unavailable | Use Poisson calculations for all markets |

**Fallback is automatic and requires no operator intervention.**

---

## 3. Validation Invariants

All probability outputs must satisfy these invariants before persistence:

### 3.1 Range Invariants

```
For all probability values P:
  0.0 ≤ P ≤ 1.0
```

### 3.2 Sum Invariants

```
1X2 Market:
  |P(home) + P(draw) + P(away) - 1.0| < ε
  where ε = 0.05 (tolerance for rounding)

BTTS Market:
  |P(yes) + P(no) - 1.0| < ε

Over/Under Market (for each threshold):
  |P(over) + P(under) - 1.0| < ε
```

### 3.3 xG Sanity Checks

```
For expected goals values:
  0.0 ≤ xG < 10.0  (upper bound: no team realistically expects 10+ goals)
```

### 3.4 Validation Pseudocode

```python
def validate_probabilities(match_data):
    errors = []

    # Range checks
    prob_fields = ['prob_home', 'prob_draw', 'prob_away',
                   'prob_btts_yes', 'prob_btts_no',
                   'prob_over_25', 'prob_under_25']

    for field in prob_fields:
        value = match_data.get(field)
        if value is not None:
            if not (0.0 <= value <= 1.0):
                errors.append(f"{field} out of range: {value}")

    # Sum checks
    TOLERANCE = 0.05

    sum_1x2 = sum([match_data.get('prob_home', 0),
                   match_data.get('prob_draw', 0),
                   match_data.get('prob_away', 0)])
    if abs(sum_1x2 - 1.0) > TOLERANCE:
        errors.append(f"1X2 sum invalid: {sum_1x2}")

    sum_btts = sum([match_data.get('prob_btts_yes', 0),
                    match_data.get('prob_btts_no', 0)])
    if abs(sum_btts - 1.0) > TOLERANCE:
        errors.append(f"BTTS sum invalid: {sum_btts}")

    # xG sanity
    for xg_field in ['home_xg', 'away_xg']:
        xg = match_data.get(xg_field)
        if xg is not None and not (0.0 <= xg < 10.0):
            errors.append(f"{xg_field} unreasonable: {xg}")

    return errors
```

---

## 4. Fallback Implementation

### 4.1 Predict with Fallback Pseudocode

```python
def predict_with_fallback(match):
    # Step 1: Assign pre-match xG (always required)
    assign_prematch_xg(match)

    # Step 2: Check if ML predictions are enabled
    if not ml_service_enabled():
        return calculate_traditional(match)

    # Step 3: Attempt ML prediction
    try:
        response = ml_client.predict(
            match_id=match.id,
            home_team=match.home_team,
            away_team=match.away_team,
            match_date=match.date,
            league=match.league
        )

        # Step 4: Validate response structure
        if not is_valid_response(response):
            log_warning(f"Invalid ML response for match {match.id}")
            return calculate_traditional(match)

        # Step 5: Check for partial response
        result = extract_ml_predictions(response)
        missing_markets = find_missing_markets(result)

        if missing_markets:
            log_warning(f"Partial ML response, filling: {missing_markets}")
            traditional = calculate_traditional(match)
            result = merge_predictions(result, traditional, missing_markets)

        return result

    except TimeoutError:
        log_warning(f"ML timeout for match {match.id}, using fallback")
        return calculate_traditional(match)

    except ServiceError as e:
        log_warning(f"ML error for match {match.id}: {e}, using fallback")
        return calculate_traditional(match)

    except Exception as e:
        log_error(f"Unexpected error for match {match.id}: {e}")
        return calculate_traditional(match)


def is_valid_response(response):
    """Check response has minimum required structure"""
    if not isinstance(response, dict):
        return False

    probs = response.get('probs')
    if not probs:
        return False

    # Must have at least 1X2 probabilities
    required = ['home', 'draw', 'away']
    for key in required:
        if key not in probs or not isinstance(probs[key], (int, float)):
            return False

    return True


def find_missing_markets(result):
    """Identify which markets need fallback calculation"""
    missing = []

    if result.get('prob_btts_yes') is None:
        missing.append('btts')

    if result.get('prob_over_25') is None:
        missing.append('over_under_25')

    if result.get('prob_over_15') is None:
        missing.append('over_under_15')

    return missing
```

---

## 5. Partial Response Handling

The system tolerates incomplete ML responses by design:

| ML Response Contains | Action |
|---------------------|--------|
| 1X2 only | Accept 1X2, calculate BTTS and O/U traditionally |
| 1X2 + BTTS | Accept both, calculate O/U traditionally |
| 1X2 + O/U 2.5 only | Accept both, calculate BTTS and O/U 1.5 traditionally |
| Empty/malformed | Log warning, use full traditional calculation |

**Rationale**: This allows the ML service to evolve independently. New markets can be added to the ML response without requiring code changes in the consumer.

---

## 6. Contract Testing Approach

### 6.1 Test Pyramid

```
         ┌─────────────┐
         │  Regression │  ← Real HTTP calls to live service
         │   Tests     │     (run on-demand, not in CI)
         └──────┬──────┘
                │
         ┌──────┴──────┐
         │ Integration │  ← Mocked HTTP with realistic fixtures
         │   Tests     │     (run in CI)
         └──────┬──────┘
                │
         ┌──────┴──────┐
         │   Unit      │  ← Pure logic tests
         │   Tests     │     (run in CI)
         └─────────────┘
```

### 6.2 Regression Test Contract

Regression tests validate against a live ML service instance:

```python
# Contract: 1X2 probabilities
assert response.prob_home is not None
assert response.prob_draw is not None
assert response.prob_away is not None
assert 0.0 <= response.prob_home <= 1.0
assert 0.0 <= response.prob_draw <= 1.0
assert 0.0 <= response.prob_away <= 1.0
assert abs(response.prob_home + response.prob_draw + response.prob_away - 1.0) < 0.05

# Contract: BTTS probabilities
assert response.prob_btts_yes is not None
assert response.prob_btts_no is not None
assert abs(response.prob_btts_yes + response.prob_btts_no - 1.0) < 0.05

# Contract: xG values
assert response.home_xg >= 0.0
assert response.home_xg < 10.0
assert response.away_xg >= 0.0
assert response.away_xg < 10.0
```

### 6.3 Test Isolation

- **Regression tests** run only with explicit flag (`RUN_REGRESSION=1`)
- **CI pipeline** runs unit and integration tests only (no external dependencies)
- **Regression failures** don't block deploys but trigger alerts

---

## 7. Operational Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| ML service downtime | Medium | Low | Automatic fallback to Poisson; no manual intervention needed |
| ML service returns invalid data | Low | Medium | Strict validation before persistence; falls back on failure |
| ML service latency spike | Medium | Low | 15-second timeout; falls back immediately on timeout |
| Network partition | Low | Low | Retry with exponential backoff (2 attempts); then fallback |
| ML contract changes | Medium | Medium | Partial response tolerance; regression tests catch breaking changes |
| Weather API unavailable | Low | Low | Matches processed without weather; flagged for later update |
| Database connection issues | Low | High | Standard Rails connection pooling; retry logic |
| Rate limit exceeded on data API | Medium | Medium | Adaptive rate limiting; backoff on 429 responses |

### 7.1 Monitoring Recommendations

- **Alert on**: Fallback rate > 20% over 1 hour (indicates ML service issues)
- **Alert on**: Validation failure rate > 1% (indicates data quality issues)
- **Dashboard**: Show prediction source distribution (ML vs traditional)
- **Log analysis**: Track latency percentiles by prediction source

---

## 8. Performance Characteristics

| Operation | Typical Latency | Notes |
|-----------|-----------------|-------|
| ML prediction (success) | 200-500ms | Network-bound |
| ML prediction (timeout) | 15,000ms | Configured ceiling |
| Traditional calculation | 5-20ms | CPU-bound, parallelizable |
| Full match processing | 50-100ms | Including DB writes |
| Batch processing (100 matches) | 10-30s | With 4 parallel workers |

---

## 9. Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| Application | Ruby on Rails | Web framework, job orchestration |
| Database | PostgreSQL | Persistent storage with constraints |
| HTTP Client | Faraday | External API calls with middleware |
| Testing | RSpec, WebMock | Unit/integration testing |
| Optimization | L-BFGS-B | Finishing offset calibration |
| Numerical | Numo::NArray | Array operations for statistics |

---

*Document prepared for public portfolio. Contains no credentials, internal URLs, or proprietary implementation details.*
