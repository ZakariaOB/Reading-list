# Concepts Reading Order

This folder contains the high-signal concepts worth understanding outside the code.

The files are ordered so they can be read as a guided path.

## Production Flow
1. [1_ProductionCandidateSelection.md](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Docs/Concepts/1_ProductionCandidateSelection.md)
   - how raw telemetry becomes selected issues
2. [2_ProductionDetailEnrichment.md](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Docs/Concepts/2_ProductionDetailEnrichment.md)
   - what evidence is added before reporting and AI
3. [3_ProductionAiLayer.md](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Docs/Concepts/3_ProductionAiLayer.md)
   - what the AI layer changes before and after

## Pipeline Flow
4. [4_PipelineCandidateSelection.md](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Docs/Concepts/4_PipelineCandidateSelection.md)
   - how raw Azure DevOps failures become prioritized pipeline candidates
5. [5_PipelineReportGeneration.md](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Docs/Concepts/5_PipelineReportGeneration.md)
   - why the pipeline flow builds a deterministic report and overview before AI
6. [6_PipelineAiLayer.md](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Docs/Concepts/6_PipelineAiLayer.md)
   - what AI adds to the deterministic pipeline report

## Cross-Cutting Concepts
7. [7_ApplicationArchitectureAndExecution.md](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Docs/Concepts/7_ApplicationArchitectureAndExecution.md)
   - how the console app, workflows, and run modes fit together
8. [8_DeterministicOverviewAndKnownIssues.md](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Docs/Concepts/8_DeterministicOverviewAndKnownIssues.md)
   - why the project classifies known issues before AI runs
9. [9_ReportsMetricsAndNotifications.md](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Docs/Concepts/9_ReportsMetricsAndNotifications.md)
   - how markdown artifacts, metrics JSON, and Teams cards relate
10. [10_ConfigurationSandboxAndDeployment.md](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Docs/Concepts/10_ConfigurationSandboxAndDeployment.md)
   - how configuration, Bicep, Azure DevOps, and the sandbox boundary work
11. [11_ImplementationWorkflowAndTesting.md](C:/Users/zboukhris/Desktop/Work/DT/AI_Bobst/AiMonitoring/Docs/Concepts/11_ImplementationWorkflowAndTesting.md)
   - how to continue the project safely with parity slices and tests

## How To Use This Folder
- start with `0_Intro.md`
- read the production notes in order
- read the pipeline notes in order
- read the cross-cutting notes before making architecture or rollout changes
- use the code references inside each note when you want to connect the concept back to the implementation
