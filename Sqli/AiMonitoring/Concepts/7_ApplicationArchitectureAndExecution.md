# Application Architecture And Execution

## Purpose
This note explains how the application is shaped at runtime.

The earlier concept notes explain individual algorithms. This note explains how those pieces are orchestrated by the C# console application.

Current implementation references:
- `Implementation/project/AiMonitoring/Program.cs`
- `Implementation/project/AiMonitoring/MonitoringApplication.cs`
- `Implementation/project/AiMonitoring/Production/ProductionMonitoringWorkflow.cs`
- `Implementation/project/AiMonitoring/Pipelines/PipelineMonitoringWorkflow.cs`

## Core Idea
The application has one executable entry point and two main monitoring workflows:

- production monitoring
- pipeline monitoring

The app can run:

- production only
- pipelines only
- both

When it runs both, production runs first, pipeline monitoring runs second, then a combined overview notification can be sent.

## Mental Model

### Purpose
Coordinate the monitoring workflows without hiding their internal logic.

### Inputs
- command-line run mode
- environment variables
- config files copied to the app output
- credentials/secrets supplied by Azure DevOps or the local environment

### Processing
1. parse the requested run mode
2. create workflow objects
3. run the selected workflow
4. return the final report text
5. send combined notification only when both workflows run

### Outputs
- production report text
- pipeline report text
- markdown artifacts
- metrics artifacts
- Teams notifications when webhook settings are present

### What This Layer Does Not Do
- it does not contain monitoring rules
- it does not query Azure directly
- it does not build reports directly
- it does not classify known issues directly

It is orchestration only.

## Runtime Shape
At a high level:

```text
Program
-> MonitoringApplication
   -> ProductionMonitoringWorkflow
   -> PipelineMonitoringWorkflow
   -> CombinedMonitoringOverviewService
```

`Program` is the console entry point.

`MonitoringApplication` chooses which workflow to run.

`ProductionMonitoringWorkflow` owns the production flow.

`PipelineMonitoringWorkflow` owns the pipeline flow.

`CombinedMonitoringOverviewService` reads already-written metrics artifacts and sends one shared management-level notification.

## Production Workflow Shape
Production runs like this:

```text
MonitoringSettings.FromEnvironment()
-> LogAnalyticsCollector.CollectAsync(...)
-> ProductionCandidateSelector.SelectCandidates(...)
-> ProductionDetailCollector.CollectAsync(...)
-> ProductionRuleBasedOverviewBuilder.Build(...)
-> ProductionReportWriter.Build(...)
-> optional AiAnalysisService.GenerateProductionOverviewAsync(...)
-> ProductionAiOverviewGrounder.Ground(...)
-> ProductionReportWriter.Build(...)
-> ProductionReportArtifactWriter.WriteAsync(...)
-> ProductionMetricsArtifactWriter.WriteAsync(...)
-> TeamsNotificationFactory.BuildProductionReport(...)
-> TeamsNotifier.SendReportAsync(...)
```

Important point:

The deterministic report is built before AI is attempted. If AI fails or is disabled, the deterministic report still works.

## Pipeline Workflow Shape
Pipeline monitoring runs like this:

```text
PipelineMonitoringSettings.FromEnvironment()
-> AzureDevOpsPipelineCollector.CollectAsync(...)
-> PipelineCandidateSelector.SelectCandidates(...)
-> PipelineRuleBasedOverviewBuilder.Build(...)
-> PipelineReportWriter.Build(...)
-> optional AiAnalysisService.GeneratePipelineOverviewAsync(...)
-> PipelineReportWriter.Build(...)
-> PipelineReportArtifactWriter.WriteAsync(...)
-> PipelineMetricsArtifactWriter.WriteAsync(...)
-> TeamsNotificationFactory.BuildPipelineReportsByEnvironmentGroup(...)
-> TeamsNotifier.SendReportAsync(...)
```

Important point:

The pipeline flow mirrors production: deterministic first, AI second, artifacts and notifications last.

## Why The Workflows Are Separate
Production and pipeline monitoring share some ideas:

- deterministic candidate selection
- structured overview
- optional AI enrichment
- markdown report
- metrics artifact
- Teams notification

But their inputs and algorithms are different.

Production reads operational telemetry from Azure Monitor.

Pipelines read Azure DevOps build/run data.

Keeping the workflows separate avoids mixing two different domains too early.

## Dependency Style
The workflow classes accept dependencies through constructors, but also have default constructors for normal execution.

This supports both:

- simple production execution
- focused unit tests with fake collectors/notifiers

Example idea:

```text
ProductionMonitoringWorkflow(
    fakeCollector,
    realSelector,
    fakeDetailCollector,
    fakeAiService,
    fakeArtifactWriter,
    fakeMetricsWriter,
    fakeTeamsNotifier)
```

This makes the workflow testable without calling Azure or Teams.

## Run Modes
The app supports run modes through `MonitoringRunMode`.

Conceptually:

```text
production -> run only production monitoring
pipelines  -> run only pipeline monitoring
both       -> run production, then pipelines, then combined overview
```

The combined mode matters because a manager may want one top-level health card after the detailed production and pipeline cards.

## Why This Architecture Is Deliberately Simple
The project is in a v1 reproduction phase.

The chosen style is:

- one executable project
- domain folders
- simple workflow classes
- explicit config loaders
- explicit artifact writers
- direct Azure/Teams adapters

This keeps the C# implementation close to the Python workflow.

The goal is not to introduce a large framework. The goal is to reproduce the operational behavior in a clean, testable shape.

## Current Architecture Strengths
- production and pipeline flows are isolated
- deterministic logic is easy to test
- AI is optional and has fallback behavior
- artifacts are written before notifications
- Teams notifications are adapters, not the source of truth
- run mode orchestration is small and readable

## Current Architecture Limitations
- thresholds are still mostly hardcoded
- ownership mapping is not fully Python-parity yet
- some pipeline collector parity still needs review
- runtime hosting is still a console-style model
- notification adapter strategy may evolve later

## Short Summary
The application is built around small workflow orchestrators.

The main rule is:

```text
workflow owns order
selectors own triage rules
overview builders own deterministic classification
AI owns interpretation only
artifact writers own files
notification factory owns Teams payloads
```

If you keep that separation in mind, the source code becomes much easier to navigate.
