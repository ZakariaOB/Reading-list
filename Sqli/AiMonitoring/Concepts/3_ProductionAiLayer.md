# AI Layer

## Purpose
This note focuses on one thing only:

- what an issue looks like **before AI**
- what the same issue looks like **after AI**

The examples below are production-like and faithful to the current workflow.
They are meant to make the AI contribution visible.

## Main Idea
Before AI, the system already has:
- counts
- result codes
- exception types
- affected users
- pages
- focus reasons

After AI, the system adds:
- probable cause
- clearer explanation
- suggested next step
- final operational wording

So the AI layer does **not** create the factual signal.
It improves the interpretation of that signal.

## Example 1: Frontend Auth Issue

### Issue Before AI
This is the kind of enriched detail row the deterministic workflow can already produce:

```text
Category: Frontend exception
ProblemId: BrowserAuthError
Occurrences: 12
UniqueUsers: 4
UniqueSessions: 4
SampleMessage: interaction_in_progress
AffectedPages: /operator/shopfloor
Browsers: Chrome
FocusReason: Moderate volume, Affects 4 users, Exception increase
```

This tells us:
- the issue exists
- it affects users
- it is growing

But it does not yet clearly explain what is probably wrong.

### Issue After AI
The AI summary can turn that into something like:

```text
MSAL BrowserAuthError: interaction_in_progress on /operator/shopfloor.
Multiple interactive authentication calls are likely being triggered while a prior
interaction is still in progress. Check the auth flow around redirect/token
acquisition and ensure only one interactive call is triggered per user action.
```

### What AI Added
- probable cause: overlapping auth interactions
- clearer operational explanation
- next investigation step

## Example 2: Backend Exception Spike

### Issue Before AI

```text
Category: Backend exception
OperationName: GenerateInvoice
ExceptionType: NullReferenceException
Occurrences: 25
Details: sample stack/details payload
FocusReason: Exception increase
```

This tells us:
- a backend exception is spiking
- the dominant exception type is `NullReferenceException`

But it still reads like a technical event, not like an actionable daily report item.

### Issue After AI

```text
Invoice generation is now dominated by NullReferenceException failures.
This likely indicates a recent code or data regression in the GenerateInvoice path.
Review recent changes and verify null handling and input assumptions in the invoice flow.
```

### What AI Added
- converts the exception into a likely business/technical problem
- suggests where to investigate first
- makes the issue easier to read in a daily report

## Example 3: Failure With Result Codes

### Issue Before AI

```text
Category: Failure
Operation: GET /api/orders
ResultCode: 500
Occurrences: 4
FocusReason: High failure rate
```

This already tells us the issue is serious enough to report.

### Issue After AI

```text
GET /api/orders is currently failing with server-side 500 responses and a high
failure rate. This points to a backend error rather than a client-side or
authorization issue. Review recent backend changes and inspect the failing request context.
```

### What AI Added
- interpretation of `500` as a backend/server problem
- clearer wording for operators
- a natural next action

## Example 4: Performance Regression

### Issue Before AI

```text
Category: Performance
OperationName: GetAllTools
CountToday: 131
DurationTodayP50Ms: 666.49
DurationBeforeP50Ms: 315.37
PercentIncrease: 111.3
```

This tells us there is a measurable regression.

### Issue After AI

```text
GetAllTools shows a clear performance regression: median latency more than doubled
compared with the recent baseline. Investigate recent deployments, dependency changes,
or query-path regressions before this becomes a broader user-facing slowdown.
```

### What AI Added
- a more natural severity interpretation
- a proactive recommendation
- a sentence that can be used directly in the report

## Example 5: Final Daily Report Narrative

This is where AI becomes especially visible.

### Before AI
The deterministic report can already say:

```text
Analyzed 0 failures, 1 backend exception, 1 frontend exception, 1 performance issue

Backend exception:
- GenerateInvoice / NullReferenceException / 25 / Exception increase

Frontend exception:
- BrowserAuthError / 12 / 4 users / Moderate volume, Affects 4 users

Performance:
- GetAllTools / 666.49 ms / 315.37 ms / 111.3%
```

This is correct, but still mostly a structured artifact.

### After AI
The AI-enhanced report can turn that into:

```text
Critical:
- Browser auth flow issue affecting /operator/shopfloor users
- Invoice-generation backend exception spike likely linked to a regression

Warning:
- GetAllTools performance regression should be investigated before it spreads

Known issues:
- None matched
```

### What AI Added
- prioritization language
- compact operational narrative
- easier reading for humans who do not want to parse all raw tables

## Example 6: Same Issue, But Interpreted With A Known-Issue Rule

This example shows an important point:

- AI does not only explain issues
- AI can also change how an issue is **interpreted** when a known rule exists

### Before AI
The deterministic layer may produce something like:

```text
Category: Failure
Operation: GET /api/userauthorization
ResultCode: 499
Occurrences: 420
FocusReason: Failure increase
```

If you read only this row, it looks alarming:
- many failures
- increasing volume

### Known Rule Available To AI
The AI layer also receives a known-rule context such as:

```text
HTTP 499 from browser/client disconnects can be downgraded unless concentrated abnormally on one endpoint.
```

### After AI
With that context, the AI can produce something like:

```text
Most failures on GET /api/userauthorization are HTTP 499 responses, which usually
mean the client closed the request before completion. Based on the known issue rules,
this is more likely a client-disconnect pattern than a backend service failure.
Monitor concentration on this endpoint, but do not treat it as a critical backend incident by default.
```

### What AI Added
- it used prior knowledge, not only the raw issue row
- it changed the interpretation from:
  - "rising failures"
  to:
  - "likely known client-disconnect behavior"
- it reduced the risk of overreacting to noisy telemetry

### Why This Matters
This is one of the strongest parts of the AI layer:
- the deterministic pipeline surfaces the issue honestly
- the AI layer adds context from known patterns
- the final report becomes more operationally useful

So the AI contribution is not only:
- better wording

It is also:
- better contextual interpretation
- better triage quality

## Short Summary

### Before AI
The system knows:
- what happened
- how often
- where
- why it was selected

### After AI
The system can say:
- what it probably means
- what to investigate
- how to present it in a daily operational report

That is the real contribution of the AI layer in this project.
