# Candidate Selection

## Purpose
Candidate selection is the deterministic step that decides which raw monitoring signals are worth carrying forward into:
- detail enrichment
- report generation
- AI summarization

This step exists to reduce noise before the AI layer. The model should not analyze all raw telemetry rows. It should analyze a smaller set of findings that already look operationally relevant.

Current implementation reference:
- `Implementation/project/AiMonitoring/Production/ProductionCandidateSelector.cs`
- `Implementation/project/AiMonitoring.Tests/Production/ProductionCandidateSelectorTests.cs`

## Position In The Flow
The current production flow is:

1. load config and KQL
2. collect raw telemetry
3. select candidates
4. enrich candidates with detail queries
5. build deterministic report
6. add AI summaries and final AI narrative

Candidate selection is the first step that adds operational judgment.

## Mental Model

### Purpose
Decide which raw telemetry groups deserve attention before enrichment and AI.

### Inputs
- raw 14-day telemetry rows
- category-specific grouping keys
- simple historical metrics computed from those rows

### Processing
1. group raw rows into one logical issue
2. discard groups that do not exist today
3. compute simple metrics from the remaining group
4. apply deterministic rules
5. assign one or more `FocusReason` values

### Outputs
- candidate rows only
- each candidate carries both:
  - the key identity of the issue
  - the reason why it was selected

### What This Step Does Not Do
- it does not enrich details yet
- it does not explain probable cause
- it does not classify known vs actionable
- it does not generate the final report

## Core Idea
Each category starts from 14-day telemetry grouped by a logical issue key.

The selector then:
- keeps only groups that have a current-day row
- computes simple statistics from the 14-day history
- applies category-specific rules
- assigns a `FocusReason`

If no rule matches, the issue is dropped.

The 4 key metrics used repeatedly are:
- `latest`: the current-day count
- `count`: how many days are represented in the group
- `median`: the middle historical count
- `threshold`: the 75th percentile of historical counts

This makes the algorithm simple, explainable, and easy to test.

## Algorithm Family
This candidate-selection logic is not one famous named algorithm.

The best way to describe it is:

**a hybrid rule-based anomaly selection algorithm with robust statistical thresholds**

It combines:
- simple statistics over a recent history window
- explicit business/operational rules

So it is mathematical, but it is not a machine-learning model and not a single canonical detection algorithm like:
- ARIMA
- Isolation Forest
- EWMA
- CUSUM
- Prophet

It is closer to:
- window-based anomaly detection
- quantile-threshold heuristics
- novelty detection
- domain-specific policy rules

## What Is Statistical Vs What Is Business Rule

### Statistical parts
These parts come from the observed telemetry history itself:
- `medianCount`
- `thresholdCount` as the 75th percentile
- comparing `latestCount` to `thresholdCount`
- comparing today’s value to its recent empirical distribution

This is why the selector can detect that something is unusual relative to recent history.

### Business-rule parts
These parts are operational choices made by the monitoring designers:
- `failRate > HighFailureRateThreshold`, currently `0.6`
- backend spike requires `latestCount > 10`
- frontend exceptions are always kept if they happened today
- user-impact rules like `Affects X users`

These rules are not learned from data.
They encode what the team currently considers worth surfacing.

## Why This Is Rigorous Even Without ML
This algorithm is still rigorous because:
- it uses explicit measurable inputs
- it applies deterministic rules
- its thresholds are explainable
- the same input always produces the same output
- every candidate can be justified with concrete metrics

That makes it very suitable for monitoring, where explainability and operational trust matter a lot.

In other words:
- it is **not predictive AI**
- it is **not statistical modeling in the academic sense**
- but it is still a disciplined anomaly-selection mechanism

## Practical Name To Use
If we need to name this approach in documentation or presentation material, good names would be:

- `Robust Threshold-Based Candidate Selection`
- `Heuristic Anomaly Selection With Domain Rules`
- `Deterministic Candidate Selection Using Quantiles And Business Rules`

Those names are more accurate than calling it “AI” or “machine learning”.

## Step By Step: From Raw Data To Candidate

This is the generic transformation that happens before we even look at category-specific rules.

### Step 1: Start from raw 14-day telemetry rows
Example raw rows for one failure-like pattern:

| Name | AppRoleName | ResourceId | days_in_past | failedCount | failRate |
|---|---|---|---:|---:|---:|
| GET /api/orders | MainInterface | resource-a | 0 | 4 | 0.8 |
| GET /api/orders | MainInterface | resource-a | 1 | 1 | 0.1 |

At this point, these are only raw rows. They are not yet candidates.

### Step 2: Group rows into one logical issue
The raw rows are grouped by the category key.

For failures, the group key is:
- `Name`
- `AppRoleName`
- `ResourceId`

So the two rows above now become one logical issue:
- `GET /api/orders` on `MainInterface` / `resource-a`

### Step 3: Keep only groups that exist today
The selector looks for a current-day row:
- `days_in_past == 0`

If a group has no current-day row, it is dropped immediately.

Why:
- candidate selection is about what deserves attention **today**

### Step 4: Compute simple metrics
For the kept group, the selector computes:
- `latest`
  - current-day count
- `count`
  - number of observed days
- `median`
  - middle count value
- `threshold`
  - 75th percentile

For the failure example above:
- `latest = 4`
- `count = 2`
- `median = 2.5`
- `threshold = 3.25`
- `latestRate = 0.8`

### Where these values come from
Using the same failure example:

| Name | AppRoleName | ResourceId | days_in_past | failedCount | failRate |
|---|---|---|---:|---:|---:|
| GET /api/orders | MainInterface | resource-a | 0 | 4 | 0.8 |
| GET /api/orders | MainInterface | resource-a | 1 | 1 | 0.1 |

The selector derives the metrics like this:

- `latestCount = 4`
  - take the row where `days_in_past == 0`
  - read `failedCount`

- `latestRate = 0.8`
  - take the same current-day row
  - read `failRate`

- `observationCount = 2`
  - count how many rows exist in the grouped issue
  - here we have:
    - day 0
    - day 1
  - so the count is `2`

- `counts = [1, 4]`
  - take the `failedCount` values from the group
  - sort them ascending

- `medianCount = 2.5`
  - there are 2 sorted values: `[1, 4]`
  - for an even number of values, the selector averages the two middle values
  - `(1 + 4) / 2 = 2.5`

- `thresholdCount = 3.25`
  - this is the 75th percentile of `[1, 4]`
  - the current implementation uses linear interpolation
  - position = `(n - 1) * 0.75`
  - with `n = 2`, position = `(2 - 1) * 0.75 = 0.75`
  - lower index = `0`, upper index = `1`
  - fraction = `0.75`
  - threshold = `1 + (4 - 1) * 0.75 = 3.25`

So these values are not arbitrary:
- `observationCount = 2` comes from the number of grouped rows
- `latestCount = 4` and `latestRate = 0.8` come from the current-day row
- `thresholdCount = 3.25` comes from the 75th percentile calculation over the grouped counts

This logic comes directly from the selector implementation:
- `Implementation/project/AiMonitoring/Production/ProductionCandidateSelector.cs`

### Step 5: Apply category-specific rules
Now the selector asks:
- does this metric profile match one or more rules?

If no rule matches:
- drop the group

If one or more rules match:
- keep the group
- assign `FocusReason`

### Step 6: Emit the candidate
The output is no longer raw telemetry.

It becomes a candidate row such as:

| Name | LatestCount | ThresholdCount | FocusReason |
|---|---:|---:|---|
| GET /api/orders | 4 | 3.25 | High failure rate, Failure increase |

This candidate is then passed to:
- detail enrichment
- deterministic report generation
- AI summarization

## Worked Example: Failure Candidate End To End

### Raw input
| Name | AppRoleName | ResourceId | days_in_past | failedCount | failRate |
|---|---|---|---:|---:|---:|
| GET /api/orders | MainInterface | resource-a | 0 | 4 | 0.8 |
| GET /api/orders | MainInterface | resource-a | 1 | 1 | 0.1 |

### Grouping
One logical issue:
- `GET /api/orders`
- `MainInterface`
- `resource-a`

### Metrics
- `latestCount = 4`
- `latestRate = 0.8`
- `observationCount = 2`
- `thresholdCount = 3.25`

### Rule evaluation
- `latestRate > 0.6` -> yes
- `observationCount == 1` -> no
- `latestCount > thresholdCount` -> yes

Expanded:
- `0.8 > 0.6` -> yes
- `2 == 1` -> no
- `4 > 3.25` -> yes

### Candidate output
- keep it
- `FocusReason = High failure rate, Failure increase`

That is the full logic path from raw rows to a surfaced candidate.

## Why This Step Exists

### 1. Noise reduction
Raw telemetry contains many signals that are technically true but not worth reporting every day.

Examples:
- a small number of stable failed requests
- low, unchanged backend exceptions
- expected patterns that are not operationally urgent

Without candidate selection, the report would be noisy and AI would waste effort summarizing low-value findings.

### 2. Separation of facts and interpretation
Candidate selection is deterministic.
It decides:
- what is suspicious
- why it is suspicious

AI comes after that and adds:
- explanation
- probable cause
- suggested action
- final narrative

### 3. Cost control
If every raw issue were summarized by AI:
- token usage would increase
- latency would increase
- results would become harder to control

Candidate selection reduces the volume before prompting.

### 4. Operational consistency
The same telemetry input should produce the same candidate set every time.

That is why this step should stay rule-based.

## How The Current Algorithm Works

## Category 1: Failures

### Telemetry source
Failures come from failed request telemetry.

Current raw model:
- `AppFailureFinding`

Grouping key:
- `Name`
- `AppRoleName`
- `ResourceId`

### Metrics computed
For each grouped issue:
- `latestCount`
- `latestRate`
- `medianCount`
- `thresholdCount`
- `observationCount`

### Rules
A failure becomes a candidate if at least one of these is true:
- `latestRate > HighFailureRateThreshold`
  - reason: `High failure rate`
- `observationCount == 1`
  - reason: `New failure`
- `latestCount > thresholdCount`
  - reason: `Failure increase`

Current value:

```text
HighFailureRateThreshold = 0.6
```

This means a failure rate must be strictly greater than 60 percent to trigger `High failure rate`.
Exactly `0.6` does not trigger the rule by itself.

### Example: kept
Raw grouped telemetry:
- `GET /api/orders`
- day 0: `4` failures, fail rate `0.8`
- day 1: `1`

Why it is kept:
- fail rate is above `HighFailureRateThreshold`

Focus reason:
- `High failure rate`

### Example: dropped
Raw grouped telemetry:
- `GET /api/products`
- day 0: `2` failures, fail rate `0.02`
- day 1: `2`

Why it is dropped:
- fail rate is low
- not new
- latest count is not above threshold

### Why this category uses these rules
For request failures, rate matters more than raw count alone.

`4/5` failures is much more interesting than `4/1000`.

That is why the failure category includes `latestRate`, unlike the exception categories.

## Category 2: Backend Exceptions

### Telemetry source
Backend exception telemetry.

Current raw model:
- `AppExceptionFinding`

Grouping key:
- `OperationName`
- `Title`
- `ResourceId`

### Metrics computed
For each grouped issue:
- `latestCount`
- `medianCount`
- `thresholdCount`
- `observationCount`

### Rules
A backend exception becomes a candidate if at least one of these is true:
- `observationCount == 1`
  - reason: `New exception`
- `latestCount > 10` and `latestCount > thresholdCount`
  - reason: `Exception increase`

### Example: kept because new
Raw grouped telemetry:
- `GenerateInvoice / NullReferenceException`
- day 0: `2`

Why it is kept:
- first occurrence in the 14-day window

Focus reason:
- `New exception`

### Example: kept because of spike
Raw grouped telemetry:
- `GenerateInvoice / NullReferenceException`
- day 0: `25`
- day 1: `2`
- day 2: `3`

Why it is kept:
- latest count is above `10`
- latest count is also above the 75th percentile

Focus reason:
- `Exception increase`

### Example: dropped
Raw grouped telemetry:
- `RefreshToken / TimeoutException`
- day 0: `2`
- day 1: `1`
- day 2: `2`

Why it is dropped:
- not new
- not above `10`
- not clearly abnormal

### Why this category is stricter than frontend
Backend exceptions tend to be noisier and often include low-volume technical events that are not user-visible.

So the current design requires stronger evidence before surfacing them.

## Category 3: Frontend Exceptions

### Telemetry source
Browser-side exceptions from frontend App Insights.

Current raw model:
- `FrontendExceptionFinding`

Grouping key:
- `ProblemId`
- `ResourceId`

### Metrics computed
For each grouped issue:
- `latestCount`
- `medianCount`
- `thresholdCount`
- `observationCount`
- `impactedUsers`

### Rules
Frontend exceptions are intentionally treated more aggressively.

A frontend exception is always kept if it happened today.

The selector always assigns at least one reason:
- `latestCount > 100`
  - `High volume`
- `latestCount > 10`
  - `Moderate volume`
- `latestCount > 1`
  - `Low volume`
- otherwise
  - `Single occurrence`

Additional reasons:
- `observationCount == 1`
  - `New exception`
- `impactedUsers > 1`
  - `Affects X users`
- `latestCount > 10` and `latestCount > thresholdCount`
  - `Exception increase`

### Example: single occurrence but still kept
Raw grouped telemetry:
- `TypeError`
- day 0: `1`
- users: `1`

Why it is kept:
- frontend policy keeps all current-day browser exceptions

Focus reason:
- `Single occurrence`
- `New exception`

### Example: medium-volume user-facing issue
Raw grouped telemetry:
- `BrowserAuthError`
- day 0: `12`
- day 1: `2`
- impacted users: `4`

Why it is kept:
- current-day issue
- count > `10`
- user impact > `1`
- above threshold

Focus reason:
- `Moderate volume`
- `Affects 4 users`
- `Exception increase`

### Why frontend is treated differently
Even one browser exception can represent:
- a broken page
- a blocked user action
- a UI regression

So the current design favors visibility over suppression for frontend issues.

## Category 4: Performance

### Telemetry source
Performance regression findings already filtered by KQL.

Current raw model:
- `PerformanceFinding`

### Current behavior
Performance findings are not re-filtered in the selector.

They are only ordered by:
- descending `PercentIncrease`

### Why
The KQL query already acts as a first-level filter.

Current KQL logic selects performance findings when either:
- `count_today > 100` and `percentIncrease > 50`
- or a slower long-baseline condition is met

So by the time the selector sees them, they are already candidates.

### Example
KQL-selected telemetry:
- `GetAllTools`
- `count_today = 131`
- `percentIncrease = 111.3`

Why it stays:
- already selected by KQL
- then sorted to appear before weaker regressions

## Strengths Of The Current Algorithm

### 1. Explainable
Every candidate has explicit reasons.

### 2. Cheap
It uses simple grouping and threshold logic.

### 3. Testable
The behavior is easy to express with unit tests.

### 4. Good pre-filter for AI
It reduces telemetry volume before summarization.

### 5. Category-aware
It does not force the same threshold logic on all signal types.

## Current Limitations

### 1. Fixed thresholds
Some thresholds are hardcoded:
- failure rate `> HighFailureRateThreshold`, currently `0.6`
- backend spike `> 10`
- frontend moderate volume `> 10`
- frontend high volume `> 100`

These may not fit every project or every service.

### 2. No service-specific sensitivity
All services within a category use the same thresholds.

In reality:
- some endpoints are naturally noisy
- some services are more critical than others

### 3. Weak use of history
The algorithm uses:
- median
- 75th percentile

But it does not use:
- weekday patterns
- seasonality
- deployment-aware baselines

### 4. Frontend policy may over-surface noise
Keeping all current-day frontend exceptions is good for visibility, but may become noisy in larger systems.

### 5. Performance logic is split
Performance candidate logic is mostly in KQL, while other categories are mostly in C#.

That split is acceptable, but it means the selection logic is not fully centralized.

## Suggested Enhancements

These are not decisions yet. They are discussion points.

### Enhancement 1: Externalize thresholds
Move category thresholds to config.

Example:
- failure high rate threshold
- backend spike minimum count
- frontend volume buckets

Benefits:
- easier reuse
- easier tuning per environment

### Enhancement 2: Service-level sensitivity
Allow per-service or per-domain overrides.

Example:
- auth endpoints more sensitive
- noisy maintenance endpoints less sensitive

Benefits:
- fewer false positives
- more relevant prioritization

### Enhancement 3: Severity scoring
Instead of only storing `FocusReason`, compute a numeric score as well.

Example inputs to score:
- count
- rate
- user impact
- recency
- threshold gap

Benefits:
- easier sorting
- easier escalation levels
- easier report prioritization

### Enhancement 4: Known-issue suppression before AI
Some known patterns could be downgraded before report generation, not only inside AI.

Examples:
- repeated `499`
- repeated `401` under expected behavior

Benefits:
- reduces noise earlier
- reduces unnecessary AI usage

### Enhancement 5: Better historical baselining
Replace simple 75th percentile logic for some categories with richer baselines.

Examples:
- compare weekday to weekday
- use rolling windows
- deployment-aware resets

Benefits:
- better anomaly quality

### Enhancement 6: Performance selection consistency
Decide whether performance candidate selection should remain:
- mostly in KQL
or
- be mirrored in C# for symmetry and easier explanation

Benefits:
- clearer ownership of the rule set

## Recommendation For V1
For the first production-ready version, keep the current algorithm philosophy:
- deterministic
- explainable
- category-specific
- easy to test

But strongly consider these two early improvements:
- externalized thresholds
- severity scoring

Those two changes would increase reuse without making the algorithm much more complex.

## Short Summary
Candidate selection is the gatekeeper between:
- raw telemetry
and
- actionable monitoring findings

Its job is not to explain issues.
Its job is to decide:
- what deserves attention
- why it deserves attention
- what should move to the enrichment and AI steps
