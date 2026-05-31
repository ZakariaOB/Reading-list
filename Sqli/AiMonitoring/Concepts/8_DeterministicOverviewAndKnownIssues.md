# Deterministic Overview And Known Issues

## Purpose
This note explains one of the most important design decisions in the current project:

Known issue classification should happen deterministically before AI runs.

Current implementation references:
- `Implementation/project/AiMonitoring/Production/ProductionRuleBasedOverviewBuilder.cs`
- `Implementation/project/AiMonitoring/Pipelines/PipelineRuleBasedOverviewBuilder.cs`
- `Implementation/project/AiMonitoring/Config/KnownIssueCatalog.cs`
- `Implementation/config/rules/known-issues.json`
- `Implementation/project/AiMonitoring.Tests/Production/ProductionRuleBasedOverviewBuilderTests.cs`
- `Implementation/project/AiMonitoring.Tests/Config/KnownIssueCatalogTests.cs`

## The Problem This Solves
Raw monitoring data contains real signals and expected noise.

Examples:

- HTTP `500` from a backend endpoint may need investigation.
- HTTP `499` may only mean the browser closed the request.
- HTTP `429` may be expected rate limiting.
- `interaction_in_progress` may be a known frontend auth-flow conflict.
- pipeline failures in develop may match a known flaky non-production pattern.

If all of these go to AI as equal "problems", the report can overreact to known patterns.

So the C# workflow creates a deterministic overview first.

## Mental Model

### Purpose
Separate actionable findings from known/expected findings before AI adds interpretation.

### Inputs
- selected candidates
- enriched detail rows
- known issue catalog rules
- metrics already computed by the deterministic flow

### Processing
1. create overview candidates from production or pipeline findings
2. test each finding against the known-issue catalog
3. place catalog matches into `Known Issues`
4. place unmatched operational findings into `Action Required`
5. create neutral observations for quiet or partial states

### Outputs
- `Summary`
- `Action Required`
- `Observations`
- `Known Issues`

### What This Step Does Not Do
- it does not call AI
- it does not rewrite evidence
- it does not send Teams notifications
- it does not hide raw detail rows

## Overview Sections
Both production and pipeline reports now use the same conceptual sections.

### Summary
One short sentence describing the overall state.

Example:

```text
2 production findings require follow-up; 3 findings match deterministic known-issue rules.
```

### Action Required
Findings that need investigation or follow-up.

Example:

```text
GET /api/orders returned HTTP 500
4 occurrences in orders-api; High failure rate.
```

### Observations
Useful context that is not directly an incident.

Example:

```text
No performance regressions were reported in the provided data set.
```

### Known Issues
Findings that match known/expected patterns.

Example:

```text
GET /api/status returned HTTP 499
Known issue: HTTP 499 from client/browser disconnects.
```

## Known-Issue Catalog
Known issue rules live in:

```text
Implementation/config/rules/known-issues.json
```

The catalog is loaded at runtime from app output by `KnownIssueRuleLoader`.

The catalog currently supports matching on:

- `flow`
- `category`
- exact `resultCode`
- `textContains`
- `pipelineNameContains`
- `environment`

A rule must have at least one specific matcher. This prevents a broad rule from accidentally matching every finding in a flow/category.

## Known-Issue Candidate
The rule builder converts each finding into a generic candidate:

```text
KnownIssueCandidate
```

The candidate contains fields like:

- flow
- category
- name
- service
- result code
- problem id
- message
- exception type
- pipeline name
- environment
- focus reason

For text rules, those fields are combined into one searchable text string.

That lets a rule like:

```json
{
  "textContains": "interaction_in_progress"
}
```

match either the frontend problem id or sample message.

## Production Known-Issue Examples
Current production catalog behavior covers:

- HTTP `499` client/browser disconnects
- HTTP `429` rate limiting
- result code `0` incomplete client request
- frontend `interaction_in_progress`

Example:

```text
Input:
GET /api/external-status returned HTTP 429

Catalog match:
resultCode = 429

Output:
Known Issues
```

Example:

```text
Input:
GET /api/orders returned HTTP 500

Catalog match:
none

Output:
Action Required
```

## Why Not Let AI Decide Known Issues Alone
AI can help explain known issues, but it should not be the only source of classification.

Reasons:

- classification must be stable
- tests should prove expected downgrades
- AI may be disabled
- AI may fail and fall back
- known issue behavior should be visible in code and config

So the project uses this rule:

```text
deterministic known issue classification first
AI enrichment second
```

## Relationship To The Python FAQ
The Python project has a human-readable FAQ:

```text
C:\BobstSource\AdminTools.ProductionMonitoring\known_issues.md
```

The C# project is gradually translating that FAQ into deterministic catalog rules.

This is what "match the Python FAQ" means:

```text
If the Python FAQ says a pattern is known or expected,
the C# deterministic path should classify the same pattern the same way,
unless the C# telemetry model does not yet contain enough evidence.
```

Already translated:

- `499`
- `429`
- result code `0`
- `interaction_in_progress`

Still to translate:

- JobPreparation decommissioning
- bot/probe traffic
- low-count `404`
- threshold-aware `401` / `403`
- ownership/team hints
- performance threshold behavior

## Important Caution
Known-issue classification must not hide real incidents.

For example:

```text
GET /.env returned 404
```

can be known bot/probe traffic.

But:

```text
GET /api/orders returned 404
```

should not automatically be downgraded, because it may be a real broken route.

This is why bot/probe rules should match specific probe-like paths, not all `404` values.

## How AI Uses The Overview
The workflow builds the deterministic overview first.

If AI is enabled:

1. AI receives deterministic evidence.
2. AI returns a structured overview.
3. Production AI grounding preserves deterministic evidence.
4. The report is rendered again with the grounded overview.

If AI is disabled or fails:

1. the deterministic overview remains
2. the report still has `Action Required`, `Observations`, and `Known Issues`

## Short Summary
The known-issue catalog is the deterministic guardrail that keeps the report operationally sane.

It makes this possible:

```text
same evidence
same classification
same report shape
with or without AI
```

That is why new rule-heavy behavior should normally be added to the catalog and covered by tests before touching AI prompts.
