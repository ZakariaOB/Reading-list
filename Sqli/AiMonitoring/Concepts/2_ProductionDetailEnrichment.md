# Detail Enrichment

## Purpose
Detail enrichment is the deterministic step that turns a **flagged candidate** into a **report-ready evidence row**.

Candidate selection tells us:
- this issue deserves attention
- why it deserves attention

Detail enrichment then asks:
- what concrete evidence do we need to explain the issue?

This step exists to make the later report and AI layers work on useful evidence instead of only coarse signals.

Current implementation reference:
- [ProductionDetailQueryBuilder.cs](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Implementation/project/AiMonitoring/Production/ProductionDetailQueryBuilder.cs)
- [ProductionDetailCollector.cs](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Implementation/project/AiMonitoring/Production/ProductionDetailCollector.cs)
- [ProductionDetailQueryBuilderTests.cs](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Implementation/project/AiMonitoring.Tests/Production/ProductionDetailQueryBuilderTests.cs)

## Position In The Flow
The current production flow is:

1. load config and KQL
2. collect raw telemetry
3. select candidates
4. enrich candidates with detail queries
5. build deterministic report
6. add AI summaries and final AI narrative

Detail enrichment is the bridge between:
- **selection**
and
- **reporting / AI**

## Mental Model

### Purpose
Take a candidate and gather the minimum useful evidence needed to explain it later.

### Inputs
- `ProductionCandidateSnapshot`
- monitored-resource settings from `MonitoringSettings`
- one-day detail KQL templates

### Processing
1. take one candidate
2. build a one-day detail query specific to that candidate
3. run the query against Azure Monitor
4. map the returned rows into typed detail models
5. carry the original `FocusReason` into the detail row

### Outputs
- `ProductionDetailSnapshot`
  - `FailureDetails`
  - `BackendExceptionDetails`
  - `FrontendExceptionDetails`

### What This Step Does Not Do
- it does not decide whether an issue is important
- it does not explain probable cause
- it does not classify known vs actionable
- it does not write the final report narrative

## Why This Step Exists

### 1. Candidates are too coarse
A candidate tells us that something is suspicious, but it does not contain enough evidence to explain the issue clearly.

Example:
- `GET /api/orders`
- fail rate `0.8`
- focus reason `High failure rate`

This tells us **that** the issue matters, but not:
- which result codes dominate
- whether this looks like auth, client disconnect, or backend failure

### 2. AI needs evidence, not only flags
If the AI layer only sees:
- `High failure rate`
- `Exception increase`

it will produce vague summaries.

If it also sees:
- `ResultCode = 500`
- `ExceptionType = NullReferenceException`
- `SampleMessage = interaction_in_progress`

then the summary becomes much more useful.

### 3. Reporting becomes explainable
The deterministic report should show the concrete evidence behind the candidate, not only the fact that the selector kept it.

## Core Idea
Candidate selection works on:
- **14-day grouped telemetry**

Detail enrichment works on:
- **1-day detail telemetry**

So the step is:

- from **historical anomaly-like signal**
- to **current-day representative evidence**

## Step By Step: From Candidate To Detail Row

### Step 1: Start from a candidate
Example failure candidate:

| Name | AppRoleName | ResourceId | LatestCount | FocusReason |
|---|---|---|---:|---|
| GET /api/orders | MainInterface | resource-a | 4 | High failure rate |

At this point we know:
- this issue was selected
- why it was selected

But we still do not know the detailed failure shape.

### Step 2: Build a one-day detail query
The query builder uses:
- the candidate identity
- the correct KQL template for the category

For a failure candidate, the builder injects:
- `resource_id`
- `operation_name`

So the detail query becomes:
- same resource
- same operation
- last 1 day only

### Step 3: Execute the query
The collector runs the built query against Azure Monitor.

For failures and backend exceptions:
- query the Log Analytics workspace

For frontend exceptions:
- query the frontend App Insights resource through cross-resource KQL

### Step 4: Map returned rows into detail models
The raw KQL result is mapped into typed C# models:
- `AppFailureDetail`
- `AppExceptionDetail`
- `FrontendExceptionDetail`

### Step 5: Carry `FocusReason`
The detail row keeps the original `FocusReason` from candidate selection.

Why:
- later steps still need to know why the issue was selected in the first place

## Why Failures Still Use `AppRequests`
This is one of the easiest parts to misunderstand.

The key distinction is:
- `failure` is the monitoring concept
- `AppRequests` is the telemetry source

So:
- candidate selection says: this operation has suspicious failed requests
- detail enrichment then zooms into those same failed request rows for the last day

It is **not** changing to a different concept.
It is moving from:
- aggregated failed-request signal
to:
- detailed failed-request evidence

## Category 1: Failure Enrichment

### Source table
- `AppRequests`

### Current detail template
- `resource_results.kql`

### Candidate input
Example:

| Name | AppRoleName | ResourceId | LatestCount | FocusReason |
|---|---|---|---:|---|
| GET /api/orders | MainInterface | resource-a | 4 | High failure rate |

### What the detail query looks for
- `ResultCode`
- `_count`
- `customDimensions`

### Why these fields matter
- `ResultCode`
  - tells us what type of failure dominates
  - `500`, `401`, `403`, `404`, `499`, etc.
- `_count`
  - tells us how the failures split across result codes
- `customDimensions`
  - may contain request metadata useful for diagnosis

### Example: before and after
Before enrichment:
- `GET /api/orders`
- 4 failures
- high failure rate

After enrichment:

| ResultCode | Occurrences | customDimensions |
|---|---:|---|
| 500 | 4 | `{ ... }` |

Now we know this is not just “some failures”.
It is specifically a group of `500` server-side failures.

## Category 2: Backend Exception Enrichment

### Source table
- `AppExceptions`

### Current detail template
- `resource_exceptions.kql`

### Candidate input
Example:

| OperationName | Title | ResourceId | LatestCount | FocusReason |
|---|---|---|---:|---|
| GenerateInvoice | NullReferenceException | resource-b | 25 | Exception increase |

### What the detail query looks for
- `ExceptionType`
- `_count`
- `details`

### Why these fields matter
- `ExceptionType`
  - confirms the dominant technical error class
- `_count`
  - shows which exception types dominate
- `details`
  - gives representative diagnostic text or payload

### Example: before and after
Before enrichment:
- `GenerateInvoice`
- exception increase

After enrichment:

| ExceptionType | Occurrences | details |
|---|---:|---|
| NullReferenceException | 25 | `sample stack/details` |

Now the issue is no longer just “an exception spike”.
It is a specific, dominant backend exception with a detail sample.

## Category 3: Frontend Exception Enrichment

### Source
- frontend App Insights via cross-resource query

### Current detail template
- `frontend_exception_details.template.kql`

### Candidate input
Example:

| ProblemId | ResourceId | LatestCount | ImpactedUsers | FocusReason |
|---|---|---:|---:|---|
| BrowserAuthError | appi-frontend | 12 | 4 | Moderate volume, Affects 4 users |

### What the detail query looks for
- `_count`
- `UniqueUsers`
- `UniqueSessions`
- `SampleMessage`
- `SampleStack`
- `Browsers`
- `ClientOSes`
- `AffectedPages`

### Why these fields matter
- `UniqueUsers`
  - tells us how broad the user impact is
- `UniqueSessions`
  - tells us recurrence across sessions
- `SampleMessage`
  - gives the most readable frontend error text
- `SampleStack`
  - gives diagnostic context
- `Browsers` / `ClientOSes`
  - shows whether the issue is environment-specific
- `AffectedPages`
  - tells us where users see the problem

### Example: before and after
Before enrichment:
- `BrowserAuthError`
- 12 occurrences
- 4 users

After enrichment:

| ProblemId | Occurrences | UniqueUsers | SampleMessage | AffectedPages |
|---|---:|---:|---|---|
| BrowserAuthError | 12 | 4 | `interaction_in_progress` | `/operator/shopfloor` |

Now the issue becomes report-ready and AI-ready.

## Category 4: Performance

### Current behavior
Performance findings are not enriched in the current C# step.

Reason:
- the KQL output already contains the main evidence:
  - `count_today`
  - `count_before`
  - `durationMs_50_today`
  - `durationMs_50_before`
  - `durationMs_85_before`
  - `percentIncrease`

So for v1, performance is already close to report-ready when it leaves the KQL query.

## What Enrichment Adds By Category

| Category | Candidate tells us | Enrichment adds | Why it matters |
|---|---|---|---|
| Failures | an operation/resource has suspicious failed requests | `ResultCode`, per-code occurrence counts, sample `customDimensions` | tells us whether the issue is mainly `500`, `401`, `403`, `499`, etc. |
| Backend exceptions | an operation has a new or increasing exception pattern | `ExceptionType`, count per type, sample `details` | tells us which backend exception is dominant |
| Frontend exceptions | a browser exception is worth surfacing | users, sessions, message, stack, browser/OS, pages | tells us user impact and frontend context |
| Performance | the operation already looks like a regression | no extra enrichment yet | enough for first deterministic reporting in v1 |

## Strengths Of The Current Enrichment Step

### 1. Scoped queries
Each detail query is tightly scoped to:
- one candidate
- one resource
- one operation or problem id
- one day

That keeps the evidence relevant.

### 2. Explainable
The evidence fields are easy to understand and easy to connect back to the report.

### 3. Good support for AI
This is the step that gives the AI enough context to produce useful summaries.

### 4. Deterministic
The step is fully rule-based and testable.

## Current Limitations

### 1. Sequential execution per candidate
The current C# implementation loops through candidates sequentially per category.

This is simple, but can become slower if candidate volume grows.

### 2. Limited failure context
For failures, we currently only bring:
- result code
- count
- custom dimensions

In some systems, that may be too little.

### 3. No performance enrichment yet
Performance currently relies on the KQL result alone.

### 4. No fallback metadata shaping
Some fields like browser sets or page sets are still treated as strings; later we may want cleaner normalization.

## Suggested Enhancements

### Enhancement 1: Parallel detail execution
Execute detail queries in parallel with a bounded worker count.

Why:
- faster runs when many candidates are selected

### Enhancement 2: Richer failure context
Add more request-level fields where useful.

Possible additions:
- URL or route pattern
- dependency name
- environment marker
- response time on failed requests

### Enhancement 3: Performance enrichment
Add a dedicated performance-detail query if needed.

Possible additions:
- route-level context
- dependency correlations
- tail-latency indicators

### Enhancement 4: Better normalization of array-like fields
Normalize:
- browsers
- client OSes
- affected pages

Why:
- cleaner report rendering
- easier AI input quality

### Enhancement 5: Explicit confidence markers
Some detail rows could carry:
- whether the evidence is complete
- whether enrichment returned only partial data

Why:
- helps later report generation and AI avoid overclaiming

## Recommendation For V1
Keep the current philosophy:
- scoped
- deterministic
- category-aware
- evidence-focused

The first improvements worth discussing later are:
- parallel detail execution
- performance enrichment
- normalization of frontend detail fields

## Short Summary
Candidate selection answers:
- what deserves attention?

Detail enrichment answers:
- what evidence do we need to explain it?

It is the step that converts:
- suspicious signal
into
- concrete issue evidence
