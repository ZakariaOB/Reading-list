# Pipeline Candidate Selection

## Purpose
Pipeline candidate selection is the deterministic step that decides which failing pipelines should appear first in the pipeline monitoring flow.

This step exists to add operational judgment before:
- markdown report generation
- AI analysis
- notification

Current implementation reference:
- `Implementation/project/AiMonitoring/Pipelines/PipelineCandidateSelector.cs`
- `Implementation/project/AiMonitoring/Pipelines/PipelineEnvironmentPriority.cs`
- `Implementation/project/AiMonitoring.Tests/Pipelines/PipelineCandidateSelectorTests.cs`

## Position In The Flow
The current pipeline flow is:

1. load Azure DevOps settings
2. collect raw build/run data
3. select deterministic pipeline candidates
4. build deterministic markdown report
5. optionally enrich the structured overview with AI
6. write artifacts and send notifications

This step is the first place where the pipeline side applies deterministic triage.

## Mental Model

### Purpose
Turn raw failing pipeline records into an ordered list of candidates with an explicit reason for each one.

### Inputs
- `PipelineTelemetrySnapshot`
  - `SummaryByEnvironment`
  - `FailingPipelines`
- environment priority rules
- failure streak rules

### Processing
1. sort environment summaries by criticality
2. convert each raw failure record into a candidate
3. assign a `FocusReason`
4. sort the final candidate list

### Outputs
- `PipelineCandidateSnapshot`
  - ordered environment summaries
  - ordered failing pipeline candidates

### What This Step Does Not Do
- it does not fetch ownership from another source yet
- it does not detect shared root causes yet
- it does not generate AI explanations
- it does not write the final artifact yet

## Core Idea
The selector is intentionally simple:

- critical environments first
- then longer failure streaks
- then stable alphabetical order

So the algorithm answers:

**which pipeline failures should a human read first?**

It does not try to predict root cause.
It tries to produce a deterministic and defensible triage order.

## Algorithm Family
This is not anomaly detection in the same sense as the production candidate selector.

The best name for it is:

**deterministic priority-based triage with domain rules**

It combines:
- environment criticality
- failure persistence
- recency of last success
- optional carried-forward failure pattern labels

So this is not statistical detection.
It is a ranking heuristic based on operational risk.

## What Is Policy Vs What Is Observed Data

### Observed data
These come directly from Azure DevOps collection:
- environment
- consecutive failures
- last success date
- pipeline name
- failure pattern if already known

### Policy rules
These are explicit monitoring choices:
- `master` / `production` should outrank `release`
- `release` should outrank `develop`
- `5+` failures means persistent
- `3+` failures means repeated
- no recent success should be called out

These are not learned from data.
They encode current operational priorities.

## Step By Step: From Raw Failure Record To Candidate

### Step 1: Start from raw failing pipeline records
Example raw failing pipeline rows:

| Pipeline | Environment | Consecutive Failures | Last Success |
|---|---|---:|---|
| device-master | master | 1 | 2026-03-21 |
| job-release | release | 5 | 2026-03-20 |
| frontend-develop | develop | 6 | – |

At this point, these are only raw failing pipelines.
They are not yet ordered candidates.

### Step 2: Assign environment priority
The current priority mapping is:

- `master`, `production` -> `100`
- `china-master` -> `95`
- `release`, `staging` -> `80`
- `china-release`, `china-staging` -> `75`
- `develop` -> `50`
- `china-develop` -> `45`

So the example becomes:

| Pipeline | Environment | Priority |
|---|---|---:|
| device-master | master | 100 |
| job-release | release | 80 |
| frontend-develop | develop | 50 |

### Step 3: Build the focus reason
The selector builds a textual reason from deterministic rules.

Environment part:
- priority `>= 95` -> `Production-critical environment`
- priority `>= 75` -> `Release environment`
- otherwise -> `Development environment`

Failure streak part:
- `>= 5` -> `Persistent failure streak`
- `>= 3` -> `Repeated failure streak`
- otherwise -> `Latest run failing`

Additional context:
- if `LastSuccessDate is null` -> `No recent success recorded`
- if `FailurePattern` exists -> append it

So the example candidates become:

| Pipeline | Focus Reason |
|---|---|
| device-master | `Production-critical environment, Latest run failing` |
| job-release | `Release environment, Persistent failure streak` |
| frontend-develop | `Development environment, Persistent failure streak, No recent success recorded` |

### Step 4: Order candidates
Final sort order is:

1. environment priority descending
2. consecutive failures descending
3. pipeline name ascending

That produces:

1. `device-master`
2. `job-release`
3. `frontend-develop`

This is the most important behavior to understand:

**environment criticality wins before streak length**

That is why a single failure on `master` still outranks a long failure streak on `develop`.

## Worked Example

### Raw input

```text
device-master: master, 1 failure, last success yesterday
job-release: release, 5 failures, last success two days ago
frontend-develop: develop, 6 failures, no recent success
```

### After candidate selection

```text
1. device-master
   FocusReason: Production-critical environment, Latest run failing

2. job-release
   FocusReason: Release environment, Persistent failure streak

3. frontend-develop
   FocusReason: Development environment, Persistent failure streak, No recent success recorded
```

### Why the order looks like this
- `device-master` is first because `master` is the most critical environment
- `job-release` is second because `release` outranks `develop`
- `frontend-develop` stays third even with the longest streak because it is still a development pipeline

## Why This Step Matters
Without this step:
- the report would be a flat list of failing pipelines
- a long-running `develop` failure could distract from a fresh `master` break
- AI would receive less structured operational context

With this step:
- the deterministic report already has a clear reading order
- AI can focus on interpretation instead of basic sorting

## Current Strengths
- simple and explainable
- deterministic
- easy to test
- aligned with operational severity

## Current Limitations
- does not yet use ownership
- does not yet detect stale failures explicitly
- does not yet detect cross-pipeline shared causes
- environment priority is fixed in code

## Possible Enhancements
- move environment priority rules to config
- add an explicit stale-failure rule
- boost pipelines with the same recent commit or same owning team
- distinguish `release` from `staging` if the business needs it
- let AI consume this deterministic ranking instead of replacing it
