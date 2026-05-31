# Configuration, Sandbox, And Deployment Boundaries

## Purpose
This note explains the boundaries around configuration, Azure resources, sandbox resources, and deployment automation.

The project is not only a C# app. It is also an Azure-first proof of concept that should be reproducible and operable through Azure DevOps.

Current implementation references:
- `Docs/Implementation/Architecture.md`
- `Docs/Implementation/Decisions.md`
- `Docs/General/ManualSetupGuide.md`
- `Implementation/azure-pipelines*.yml`
- `Implementation/infra`
- `Implementation/project/AiMonitoring/Config`

## Core Idea
The monitoring solution and the monitored systems are separate concerns.

The monitoring solution owns:

- its application code
- monitoring-owned Azure resources
- known issue rules
- report generation
- notifications

The monitored systems provide:

- telemetry
- Log Analytics workspace id
- Application Insights resource ids
- Azure DevOps project/pipeline history

The monitoring app consumes monitored identifiers. It should not assume it owns the monitored application.

## Mental Model

### Purpose
Keep the POC reproducible without mixing monitoring ownership and monitored-system ownership.

### Inputs
- Bicep parameters
- pipeline variables
- environment variables
- secret values
- sandbox outputs
- external platform setup such as Teams webhook and Azure DevOps PAT

### Processing
1. provision monitoring-owned resources
2. optionally provision sandbox monitored resources
3. output identifiers needed by the app
4. pass identifiers into the monitoring run
5. run the C# workflows

### Outputs
- deployed monitoring infrastructure
- optional sandbox telemetry sources
- application artifacts
- monitoring reports and notifications

### What This Layer Does Not Do
- it does not make Bicep own every external platform
- it does not make the monitoring app own business applications
- it does not remove the need for secrets or external access rights

## Monitoring-Owned Resources
The monitoring stack can own resources such as:

- resource group
- Azure OpenAI resource
- Azure OpenAI deployment
- Key Vault
- optional notification-side resources

These are resources needed to run the monitoring solution.

## Monitored Resources
The monitored systems are external inputs.

Examples:

- Log Analytics workspace being queried
- backend Application Insights
- frontend Application Insights
- applications generating telemetry
- Azure DevOps project and pipelines

The app reads their identifiers from configuration.

Important rule:

```text
monitoring stack consumes monitored identifiers
it does not hardcode or own them
```

## Sandbox Exception
For the POC, the same repository can include a sandbox monitored stack.

That sandbox exists only to generate representative telemetry.

It should remain conceptually separate from the monitoring stack.

The sandbox currently represents:

- backend failures
- backend exceptions
- performance slowdown
- frontend browser/auth exceptions
- app-only redeploy scenarios

The sandbox helps prove the chain before connecting to richer real-world systems.

## Subscription Bootstrap
The desired POC shape is a clean subscription entry point.

Conceptually:

```text
subscription bootstrap
-> monitoring stack
-> sandbox monitored stack
-> outputs consumed by monitoring pipeline
```

The bootstrap should output values like:

- `LOG_ANALYTICS_WORKSPACE_ID`
- `FRONTEND_APP_INSIGHTS_ID`
- backend URL
- frontend URL
- sandbox resource group name

Those outputs become the variable contract for the monitoring runner.

## Azure DevOps Boundary
Bicep is for Azure resources.

Azure DevOps project, pipeline, and history setup are external platform concerns.

That means:

- Azure resources should be deployed with Bicep
- Azure DevOps bootstrap may need separate scripts or manual setup
- Teams webhook setup is treated as an external secret/setup concern for v1

This boundary avoids pretending that subscription-scope Bicep can create every external platform dependency.

## Configuration Surfaces
The application uses several configuration layers.

### Environment Variables
Used for runtime values such as:

- workspace ids
- Azure DevOps org/project/PAT
- AI provider settings
- output directory
- Teams webhook URL
- AI toggle

### JSON Config
Used for editable rule/config data such as:

- known issues
- dashboard links
- ownership-style mappings later

### Prompt Files
Used for AI prompt text.

### Bicep Parameters
Used for infrastructure values.

### Pipeline Variables
Used to glue deployed outputs and runtime execution together.

## AI Provider Configuration
The intended provider is Azure OpenAI.

There is also a temporary Copilot CLI provider:

```text
AI_PROVIDER = copilot-cli
```

This exists to keep AI-path validation possible while Azure OpenAI quota or deployment setup is pending.

Important toggle:

```text
ENABLE_AI_ENRICHMENT = true|false
```

This lets the same telemetry be shown with AI enabled or disabled.

## Manual Setup Guide
Manual or external setup changes should be reflected in:

```text
Docs/General/ManualSetupGuide.md
```

This is part of the rollout contract.

Update it when:

- a manual prerequisite changes
- Teams setup changes
- Azure DevOps setup changes
- a workaround is discovered
- an external constraint appears

## Cost Control
A separate destroy pipeline exists for cleanup.

Important design:

- dry-run by default
- confirmation string required
- deletes selected monitoring/sandbox resource groups
- preserves identity/service-connection setup

This prevents normal monitoring runs from mixing with destructive cleanup.

## Short Summary
The deployment model is:

```text
Bicep owns Azure resources
Azure DevOps runs the app
environment variables connect runtime to deployed resources
sandbox generates test telemetry
manual guide documents external setup
```

When continuing the project, keep monitoring ownership and monitored-system ownership separate.
