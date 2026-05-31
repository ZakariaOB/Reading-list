# Pipeline AI Layer

## Purpose
This note focuses on one thing only:

- what the pipeline situation looks like **before AI**
- what the same situation looks like **after AI**

This is the **target role** of AI in pipeline monitoring.
The current C# code follows this direction by using a structured overview model instead of letting AI own the whole report.

## Main Idea
Before AI, the deterministic pipeline flow should already know:
- which pipelines are failing
- in which environments
- how many times they failed consecutively
- who last merged
- who last triggered
- which team owns the service
- what the last commit message was

After AI, the system adds:
- pattern detection across several failing pipelines
- likely shared causes
- repair priority
- next actions for teams

The current implementation asks AI for structured overview fields:
- summary
- action required
- observations
- known issues

The deterministic report writer then renders those fields into the same stable markdown layout used when AI is disabled.

So the AI layer does **not** discover the failures.
It interprets the deterministic pipeline facts.

## Example 1: Same Dependency Change Across Multiple Pipelines

### Pipeline Situation Before AI

```text
Pipeline: RemoteAssistance-release
Environment: release
Consecutive failures: 1
Owning team: Team Lausanne
Last merger: Marc
Last commit: Update Magick.NET-Q16-AnyCPU to 14.10.4
```

```text
Pipeline: RemoteAssistance-release-china
Environment: china-release
Consecutive failures: 3
Owning team: Team Lausanne
Last merger: Marc
Last commit: Update Magick.NET-Q16-AnyCPU to 14.10.4
```

```text
Pipeline: RemoteAssistance-develop-china
Environment: china-develop
Consecutive failures: 5
Owning team: Team Lausanne
Last merger: Marc
Last commit: Update Magick.NET-Q16-AnyCPU to 14.10.4
```

Before AI, these are still three separate failing pipelines.

### Pipeline Situation After AI

```text
These failures strongly suggest a shared cause rather than three isolated incidents.
The common Magick.NET update appears to have broken builds or tests across several
branches and environments. Highest-leverage action: revert or patch this dependency
change first, then rerun the affected pipelines.
```

### What AI Added
- grouped several failures into one likely pattern
- proposed a shared root cause
- suggested the best first action

## Example 2: Environment-Level Drift

### Pipeline Situation Before AI

```text
UserManagement-release-china failed
DeviceManagement-release-china failed
JobAndRecipe-release-china failed
Maintenance-release-china failed
```

```text
All belong to different services
All are in china-release
Different teams are involved
```

Before AI, this can look like several unrelated service failures.

### Pipeline Situation After AI

```text
The failures are more consistent with a shared China release environment or template
problem than with independent service regressions. Investigate shared environment
configuration first because one fix may restore several pipelines at once.
```

### What AI Added
- pattern detection across teams
- leverage-based prioritization
- a better first investigation path

## Example 3: Stale Pipeline Failure

### Pipeline Situation Before AI

```text
Pipeline: ToolManagement-release-china
Status: failed
Last successful run: unknown or very old
Consecutive failures: 1
Failure pattern: stale
Owning team: Team Casa
```

Before AI, it is just another red pipeline in the list.

### Pipeline Situation After AI

```text
This looks like a stale failing pipeline that may no longer receive active attention.
It should be reviewed for ownership and cleanup because it reduces confidence in the
delivery chain even if it is not tied to today's main incident cluster.
```

### What AI Added
- operational interpretation of a stale failure
- a reason to act even without a fresh spike

## Example 4: Final Analysis Section

### Before AI
The deterministic report may already contain:

```text
china-release: 8 failing
china-develop: 7 failing
release: 4 failing
master: 1 failing
```

and a list of failing pipelines with:
- team
- merger
- executor
- last commit

### After AI
The AI analysis section can turn that into:

```text
Priority Order
1. Investigate shared China environment failures first because they affect the largest
   number of pipelines across teams.
2. Review the Magick.NET-related Remote Assistance failures next because they share
   the same recent dependency change.
3. Then address isolated service-level failures that remain after the shared causes
   are resolved.
```

### What AI Added
- a fix order
- a management-friendly explanation
- a more actionable report

## Short Summary

### Before AI
The system knows:
- which pipelines are failing
- who owns them
- which environment is affected
- what the recent commit context is

### After AI
The system can say:
- which failures are probably related
- what to fix first
- what shared cause is most likely
- how to explain the situation in one analysis section

That is the real role of AI in pipeline monitoring.
