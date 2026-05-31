# Pipeline Report Generation

## Purpose
This note explains the deterministic report-generation step on the pipeline side.

Its job is to turn:
- ordered environment summaries
- ordered failing pipeline candidates

into a readable markdown report before any AI is added.

Current implementation reference:
- `Implementation/project/AiMonitoring/Pipelines/PipelineReportWriter.cs`
- `Implementation/project/AiMonitoring/Pipelines/PipelineMonitoringWorkflow.cs`
- `Implementation/project/AiMonitoring.Tests/Pipelines/PipelineReportWriterTests.cs`

## Position In The Flow
The current pipeline flow is:

1. load Azure DevOps settings
2. collect raw pipeline data
3. select deterministic candidates
4. build a deterministic overview
5. build deterministic markdown report
6. optionally replace the overview with an AI-enriched structured overview
7. write artifacts and notifications

This step is the pipeline equivalent of the deterministic production report step.

## Mental Model

### Purpose
Create a stable factual report that humans can read and AI can later enrich.

### Inputs
- `PipelineMonitoringSettings`
  - Azure DevOps project name
  - active pipeline filter
- `PipelineCandidateSnapshot`
  - ordered environment summaries
  - ordered failing pipeline candidates
- `PipelineReportOverview`
  - summary
  - action required
  - observations
  - known issues

### Processing
1. write a fixed summary section
2. render `Action Required`, `Observations`, and `Known Issues`
3. render environment summary table
4. render failing pipelines table
5. render explicit empty-state rows if needed

### Outputs
- one deterministic markdown report string

### What This Step Does Not Do
- it does not explain shared causes
- it does not reprioritize the deterministic order
- it does not classify ownership risk
- it does not write files directly; persistence is handled by the workflow artifact writer

## Why Markdown Again
Markdown is useful here for the same reasons as on the production side:

- simple to generate
- easy to review in pipeline artifacts
- easy to test with exact assertions
- easy to transform later into Teams or other formats

So markdown is the intermediate report format, not the final presentation format.

## What The Report Adds
Before this step, we only have objects in memory:

```text
Environment summaries
Failing pipeline candidates
Focus reasons
```

After this step, we have a report that already tells a human:
- how many environments are affected
- how many failing pipelines exist
- which findings need action
- which findings are known issues
- which environments are unstable
- which pipelines deserve attention first
- why each pipeline is being surfaced

That is already useful even without AI.

## Step By Step: From Candidates To Report

### Step 1: Start from selected candidates
Example candidate snapshot:

```text
Environment summaries:
- master: 2 total, 1 failing
- release: 4 total, 2 failing

Failing pipelines:
1. device-master
   FocusReason: Production-critical environment, Latest run failing
2. job-release
   FocusReason: Release environment, Persistent failure streak
```

### Step 2: Write the report header
The writer starts with a top-line sentence like:

```text
Analyzed 2 failing pipelines across 2 environments for project IoT Lab
```

and includes the active filter:

```text
Filter: -(develop|master|release)$
```

This gives immediate context to the reader:
- which project
- how broad the scope was
- how many failures were found

### Step 3: Render environment summary
The report writes:

```text
## Environment Summary
```

with a table showing:
- environment
- total
- passing
- failing
- in progress
- failure rate

This helps the reader understand the distribution of failures before diving into individual pipelines.

### Step 4: Render failing pipelines
The report writes:

```text
## Failing Pipelines
```

with a table showing:
- pipeline name
- environment
- consecutive failures
- last success
- executor
- owning team
- focus reason

This is the core deterministic triage table.

### Step 5: Handle empty states explicitly
If no runs or no failing pipelines exist, the writer still renders the section and explains the absence of data.

That is important because:
- quiet days should still produce a readable artifact
- empty sections without explanation are hard to interpret

## Worked Example

### Before report generation

```text
Candidates exist in memory, ordered correctly, but are not yet readable as a report.
```

### After report generation

```text
Analyzed 2 failing pipelines across 2 environments for project IoT Lab

Filter: -(develop|master|release)$

## Environment Summary
| Environment | Total | Passing | Failing | In Progress | Failure Rate |
| master      | 2     | 1       | 1       | 0           | 50.0%        |
| release     | 4     | 2       | 2       | 0           | 50.0%        |

## Failing Pipelines
| device-master | master  | 1 | 2026-03-21 | Alice | Team Core | Production-critical environment, Latest run failing |
| job-release   | release | 5 | 2026-03-20 | Bob   | –         | Release environment, Persistent failure streak |
```

This is already enough for:
- human review
- pipeline artifact publication
- future AI enrichment

## Why This Step Matters
Without this step:
- pipeline AI would have to work from raw structured objects only
- there would be no stable non-AI artifact
- debugging would be harder

With this step:
- the pipeline side has the same factual baseline as production
- AI can be added as interpretation, not replacement
- deterministic and AI outputs can be compared easily

## Current Strengths
- stable and easy to test
- preserves deterministic ordering
- explicit empty states
- simple enough for pipeline artifact publishing

## Current Limitations
- owner/merger enrichment is only as rich as the current Azure DevOps collector data
- AI still depends on deterministic evidence and may fall back
- the markdown report is not the only output; Teams cards now render a more structured view

## Possible Enhancements
- highlight stale failures separately
- add per-team grouping when ownership is available
- add links to Azure DevOps build pages in later delivery formats
