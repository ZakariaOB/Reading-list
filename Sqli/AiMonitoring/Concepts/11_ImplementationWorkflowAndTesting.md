# Implementation Workflow And Testing

## Purpose
This note explains how to continue the project without losing direction.

The project is easiest to advance when treated as a sequence of small parity slices against the Python baseline.

Current references:
- `AGENTS.md`
- `Docs/Implementation/Plan.md`
- `Docs/Implementation/Progress.md`
- `Docs/Implementation/production/Progress.md`
- `Docs/Implementation/pipelines/Progress.md`
- `Implementation/project/AiMonitoring.Tests`

## Core Idea
Do not implement from memory or screenshots.

For v1, use the Python workflow as the acceptance contract.

The implementation loop should be:

```text
compare Python behavior
-> identify one small C# gap
-> implement deterministic behavior
-> add tests
-> run focused tests
-> run full tests when practical
-> update progress log with mental model
```

## Mental Model

### Purpose
Keep the project understandable and safe while moving toward parity.

### Inputs
- Python source behavior
- current C# implementation
- existing tests
- implementation plan
- current progress logs

### Processing
1. choose one small slice
2. inspect both Python and C# surfaces
3. implement the smallest behavior change
4. add or update deterministic tests
5. verify locally
6. document the mental model and result

### Outputs
- scoped code change
- focused tests
- updated progress log
- clear next step

### What This Process Avoids
- large unreviewable rewrites
- AI prompt changes before deterministic rules
- undocumented behavior drift
- hidden assumptions about Python behavior

## What A Good Slice Looks Like
A good slice has one main concept.

Examples:

- add HTTP `429` known-issue classification
- name the high-failure-rate threshold
- add JobPreparation known-issue matching
- add bot/probe path classification
- add stale pipeline failure detection
- add ownership mapping for one pipeline family

A bad slice mixes too many concerns.

Example:

```text
change collector KQL, add known issues, redesign Teams card, and update deployment variables
```

That is too much for one step.

## Mental Model Review
After each meaningful implementation step, write a short review.

Use this shape:

```text
Purpose
Inputs
Processing
Outputs
What is not included yet
Concrete example
Verification
```

Production steps go here:

```text
Docs/Implementation/production/Progress.md
```

Pipeline steps go here:

```text
Docs/Implementation/pipelines/Progress.md
```

The top-level progress file should stay a small index.

## Testing Rules
Deterministic logic should be tested in the same step.

The tests should explain:

- scenario
- input
- expected output
- why the case matters
- why the assertion protects behavior

Useful focused test commands:

```powershell
dotnet test Implementation\project\AiMonitoring.Tests\AiMonitoring.Tests.csproj --no-restore --filter FullyQualifiedName~ProductionCandidateSelectorTests -v minimal
```

```powershell
dotnet test Implementation\project\AiMonitoring.Tests\AiMonitoring.Tests.csproj --no-restore --filter FullyQualifiedName~ProductionRuleBasedOverviewBuilderTests -v minimal
```

Full suite:

```powershell
dotnet test Implementation\project\AiMonitoring.Tests\AiMonitoring.Tests.csproj --no-restore -v minimal
```

## How To Read Tests
The tests are part of the documentation.

When learning a concept, read tests before editing code.

Examples:

- production selection behavior:
  - `ProductionCandidateSelectorTests`
- production known-issue classification:
  - `ProductionRuleBasedOverviewBuilderTests`
- AI evidence preservation:
  - `ProductionAiOverviewGrounderTests`
- pipeline candidate ordering:
  - `PipelineCandidateSelectorTests`
- pipeline known issues:
  - `PipelineRuleBasedOverviewBuilderTests`
- notification shape:
  - `TeamsNotificationFactoryTests`
  - `TeamsNotifierTests`

## Current Safe Next Slice
The current recommended next slice is:

```text
JobPreparation + bot/probe known-issue parity
```

Why it is safe:

- the Python FAQ explicitly names both
- current catalog matching can likely express both
- no new telemetry model fields should be required
- tests can prove action-vs-known behavior

Expected tests:

- JobPreparation finding becomes `Known Issues`
- `GET /.env` or `GET /wp-admin` returning `404` becomes `Known Issues`
- normal `GET /api/orders` returning `404` remains actionable for now

## What Should Wait
Some rules should wait until we inspect telemetry fields.

### 401 / 403 Auth Rules
Python behavior depends on percentage:

```text
< 5% endpoint requests -> known token expiration
> 5% endpoint requests -> warning
```

Before implementing, verify whether C# detail rows have:

- total request count
- failure percentage
- enough endpoint context

If not, update the model or document the gap.

### General Low-Count 404
This is riskier than bot/probe traffic.

Reason:

- some 404s are harmless
- some 404s mean a real broken route or client mismatch

Start with specific bot/probe path patterns first.

## Working With Dirty Git State
The worktree may contain changes from another step.

Before editing:

```powershell
git status --short
```

If unrelated files are modified:

- do not revert them
- do not include them accidentally in the mental model
- mention them in the final summary if relevant

## Documentation Discipline
Keep documentation useful, not endless.

Core docs should remain maintained:

- `Docs/Implementation/Plan.md`
- `Docs/Implementation/Architecture.md`
- `Docs/Implementation/Decisions.md`
- `Docs/Implementation/Progress.md`
- flow-specific progress files
- manual setup guide
- concepts folder for learning-oriented explanations

Avoid creating new tracking docs unless they directly help implementation.

## Short Summary
The safe way to continue is:

```text
small parity slice
deterministic first
tests in same step
progress log after step
AI only after evidence and classification are stable
```

This process is what keeps the C# implementation close to the Python workflow while still becoming cleaner and more reusable.
