# Reports, Metrics, And Notifications

## Purpose
This note explains how the project turns monitoring results into outputs people can consume.

There are three output layers:

- markdown reports
- metrics JSON artifacts
- Teams notifications

Current implementation references:
- `Implementation/project/AiMonitoring/Production/ProductionReportWriter.cs`
- `Implementation/project/AiMonitoring/Pipelines/PipelineReportWriter.cs`
- `Implementation/project/AiMonitoring/Production/ProductionMetricsBuilder.cs`
- `Implementation/project/AiMonitoring/Pipelines/PipelineMetricsBuilder.cs`
- `Implementation/project/AiMonitoring/Notifications/TeamsNotificationFactory.cs`
- `Implementation/project/AiMonitoring/Notifications/TeamsNotifier.cs`
- `Implementation/project/AiMonitoring/Notifications/CombinedMonitoringOverviewService.cs`

## Core Idea
The report artifact is the durable source of truth.

Teams notifications are a delivery view.

Metrics JSON is the machine-readable sidecar.

So the relationship is:

```text
deterministic evidence
-> markdown report
-> metrics JSON
-> Teams notification
```

Teams should not be the only place where the result exists.

## Mental Model

### Purpose
Produce human-readable, machine-readable, and notification-ready outputs from the same workflow result.

### Inputs
- candidates
- detail snapshots
- structured overview
- metrics
- AI mode text
- optional Teams webhook URL

### Processing
1. render markdown report
2. prepend visible AI mode text
3. write report artifact
4. compute and write metrics JSON
5. build Teams notification payload
6. send notification only when configured

### Outputs
- `summary.md`
- `pipeline_summary.md`
- `logs/metrics.json`
- `logs/pipeline_metrics.json`
- production Teams adaptive card
- pipeline Teams adaptive cards
- optional combined overview card

### What This Layer Does Not Do
- it does not select candidates
- it does not enrich telemetry
- it does not classify known issues
- it does not call AI

## Markdown Reports
Production writes:

```text
summary.md
```

Pipeline monitoring writes:

```text
pipeline_summary.md
```

Markdown is kept because it is:

- easy to read in Azure DevOps artifacts
- easy to diff
- easy to test
- independent from Teams rendering quirks
- useful when notifications fail

## Why AI Mode Is Printed
Reports include an AI mode line so demos and troubleshooting are clear.

Examples:

```text
AI Enrichment: Enabled (Azure OpenAI)
```

```text
AI Enrichment: Disabled (Azure OpenAI not configured)
```

```text
AI Enrichment: Fallback to deterministic (Copilot CLI)
```

This matters because the same telemetry can be shown in deterministic-only mode and AI-enriched mode.

## Metrics Artifacts
Metrics are JSON files written next to the markdown report.

Production metrics include:

- timestamp
- execution time
- issue counts
- placeholder LLM usage data

Pipeline metrics include:

- timestamp
- execution time
- total failing pipelines
- failing count by environment
- stats by environment

Metrics are used by:

- Teams summary cards
- combined overview notification
- future dashboarding or automation

## Production Teams Card
The production Teams card is an adaptive-card view built from:

- production metrics
- production details
- structured overview
- AI mode text

It renders:

- header/status
- counters
- `Action Required`
- `Observations`
- `Known issues`
- grouped detail sections:
  - errors
  - backend exceptions
  - frontend exceptions
  - performance

Important design point:

The card is deterministic. AI may enrich the overview wording, but the card layout is not AI-written.

## Pipeline Teams Cards
Pipeline notifications are grouped by environment family.

Current groups:

- Production
- Staging Global
- Staging China
- Develop Global
- Develop China

The card factory builds one or more cards per group.

Why:

- production/master failures must be visible first
- develop-only failures should not hide release/master problems
- large failure sets should not produce one unreadable card

Pipeline cards include:

- counters
- action-required lines
- known-issue lines
- failing pipeline rows
- dashboard actions where configured
- optional `Full Report` action on the last visible card

## Card Count Cap
Pipeline notifications can become noisy when many pipelines fail.

So the factory limits visible cards with:

```text
MaxPipelineNotificationCards = 4
MaxFailuresPerPipelineGroupCard = 8
```

If more cards would be produced, the last visible card gets an overflow summary.

This is a deliberate operational choice:

```text
show enough detail to act
avoid spamming Teams
keep the full report available as artifact/action
```

## Combined Overview
When the app runs in `both` mode:

1. production workflow runs
2. pipeline workflow runs
3. combined overview service reads the already-written metrics files
4. one combined Teams card can be sent

The combined card is a management-level summary, not a replacement for the detailed cards.

It answers:

- are production systems unhealthy?
- are critical pipelines failing?
- should someone open the monitoring run?

## Why Notifications Are Last
Notifications happen after artifacts are written.

Reason:

- if Teams fails, the report still exists
- artifacts are easier to inspect and debug
- Teams is a delivery channel, not the source of truth

This ordering is intentional.

## Short Summary
The output model is:

```text
markdown for humans and audit
JSON metrics for machines and summaries
Teams cards for fast operational consumption
```

When changing reporting or notification code, preserve this separation.
