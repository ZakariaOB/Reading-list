# AiMonitoring Coffee Break Recap

Prepared: 2026-05-31

This note is a practical handoff for returning to the AiMonitoring project without needing the full chat context. It explains what the project is, how the implementation is shaped, what has already been built, where we are now, and what we should do next.

## 1. The Project In One Sentence

AiMonitoring is a C#/.NET reproduction of the existing Bobst Python production and pipeline monitoring workflow. The goal is to run daily monitoring in Azure DevOps, collect production telemetry and CI/CD pipeline health, classify issues deterministically first, optionally enrich the results with AI, write markdown artifacts, and send Teams notifications.

The target is not just a script. It is a reusable, Azure-first proof of concept with:

- Bicep infrastructure
- Azure DevOps automation
- production monitoring
- pipeline monitoring
- markdown reports
- metrics JSON files
- Teams adaptive-card notifications
- a sandbox environment that can generate demo telemetry

## 2. The Main Mental Model

Think of the system as a pipeline with strict stages:

1. Load configuration.
2. Collect raw data.
3. Select relevant candidates with deterministic rules.
4. Enrich selected candidates with details.
5. Build a deterministic report shape.
6. Classify known issues before AI runs.
7. Optionally ask AI to improve wording, prioritization, and interpretation.
8. Preserve deterministic evidence even when AI responds.
9. Write artifacts.
10. Send Teams notifications.

The important design rule is:

> AI must not own the truth. Deterministic code owns the evidence and layout. AI only improves interpretation.

This is why we keep adding deterministic tests around rule-heavy behavior.

## 3. Why We Are Doing This

The existing Bobst implementation is Python-based and operationally useful. The C# project is trying to reproduce the same value in a cleaner, reusable, Azure-first package.

The proof of concept should eventually be reproducible from a clean Azure subscription. It should avoid hidden manual setup where possible and make the variable/secret contract explicit.

## 4. Important Source References

Python source of truth:

- `C:\BobstSource\AdminTools.ProductionMonitoring\src\main.py`
- `C:\BobstSource\AdminTools.ProductionMonitoring\src\llm_helper.py`
- `C:\BobstSource\AdminTools.ProductionMonitoring\src\log_helper.py`
- `C:\BobstSource\AdminTools.ProductionMonitoring\src\teams_notify.py`
- `C:\BobstSource\AdminTools.ProductionMonitoring\src\pipeline_main.py`
- `C:\BobstSource\AdminTools.ProductionMonitoring\src\pipeline_helper.py`
- `C:\BobstSource\AdminTools.ProductionMonitoring\src\pipeline_teams_notify.py`
- `C:\BobstSource\AdminTools.ProductionMonitoring\src\config\team_ownership.py`
- `C:\BobstSource\AdminTools.ProductionMonitoring\known_issues.md`

Current C# implementation:

- `C:\Users\zboukhris\Desktop\Work\DT\AI_Bobst\AiMonitoring\Implementation\project\AiMonitoring`
- `C:\Users\zboukhris\Desktop\Work\DT\AI_Bobst\AiMonitoring\Implementation\project\AiMonitoring.Tests`

Core docs:

- `Docs\Implementation\Plan.md`
- `Docs\Implementation\Progress.md`
- `Docs\Implementation\Architecture.md`
- `Docs\Implementation\Decisions.md`
- `Docs\Implementation\production\Progress.md`
- `Docs\Implementation\pipelines\Progress.md`
- `Docs\General\ManualSetupGuide.md`

## 5. Repository Architecture

The C# project is intentionally simple for v1. It mirrors the Python workflow instead of introducing a large architecture too early.

Main folders inside `Implementation\project\AiMonitoring`:

- `Production`: production telemetry flow
- `Pipelines`: Azure DevOps pipeline monitoring flow
- `AI`: Azure OpenAI / Copilot CLI AI abstraction and prompts
- `Notifications`: Teams card and notification rendering
- `Config`: settings, known-issue loading, ownership/dashboard config
- `Common`: shared utilities and contracts

Tests live in:

- `Implementation\project\AiMonitoring.Tests`

Config lives mostly in:

- `Implementation\config\rules\known-issues.json`
- `Implementation\config\rules\known-issues.md`
- `Implementation\config\ownership\pipeline-dashboards.json`

Prompts live in:

- `Implementation\prompts`

## 6. Design Decisions That Matter

### Deterministic Core Before AI

Raw telemetry should not go directly to the LLM. The app first groups, filters, thresholds, and classifies data using C# rules.

Reason:

- easier to test
- less noisy AI input
- safer reporting
- no hallucinated counts or services

### Markdown Artifacts Stay Important

The system writes markdown reports:

- `summary.md` for production
- `pipeline_summary.md` for pipelines

Teams cards are useful, but markdown remains the auditable artifact.

Reason:

- easy to inspect in Azure DevOps artifacts
- easy to test
- easier to compare AI-on vs AI-off output

### AI Uses Structured Overview, Not Free-Form Full Report

Earlier versions let AI produce larger report text. The design moved to this model:

- deterministic renderer owns the final section layout
- AI returns structured JSON overview fields
- deterministic code renders `Summary`, `Action Required`, `Observations`, and `Known Issues`

Reason:

- stable report shape
- easier regression testing
- no AI-owned formatting drift

### AI Grounding Is Explicit

When AI returns an overview, code grounds it against deterministic findings.

The deterministic version preserves:

- severity
- title
- counts
- services
- users
- focus reasons

AI may add an `AI assessment`, but it should not erase evidence.

### Known Issues Are Catalog-Driven

Known issue rules live in JSON rather than being hardcoded in the overview builder.

Current file:

- `Implementation\config\rules\known-issues.json`

Reason:

- easier to extend
- closer to Python FAQ behavior
- deterministic classification can run before AI

## 7. Production Flow Status

Production monitoring is already broadly implemented.

Implemented:

- settings/config loading
- KQL query loading
- real Azure Monitor / Log Analytics collection
- production candidate selection
- detail enrichment
- deterministic markdown generation
- AI enrichment
- report persistence to `summary.md`
- metrics output to `logs\metrics.json`
- Teams notification delivery
- production-specific adaptive card shape
- combined production-plus-pipeline overview notification
- deterministic known-issue catalog matching
- visible AI mode in reports/cards
- AI-off and AI-fallback reports keep the same structured report shape as AI-on

Production categories:

- HTTP failures
- backend exceptions
- frontend exceptions
- performance regressions

## 8. Pipeline Flow Status

Pipeline monitoring is also broadly implemented.

Implemented:

- Azure DevOps settings and typed models
- raw Azure DevOps collection
- candidate prioritization
- deterministic markdown generation
- AI analysis with fallback
- report persistence to `pipeline_summary.md`
- metrics output to `logs\pipeline_metrics.json`
- Teams notification delivery
- environment-split Teams notifications
- dashboard actions
- adaptive cards with mentions
- multi-card handling for large failure groups
- card-count cap and overflow summary
- optional full-report action
- combined production-plus-pipeline overview notification
- structured AI overview baseline
- deterministic pipeline known-issue catalog matching

Still important for pipeline parity:

- ownership mapping from Python `team_ownership.py`
- last merger / commit author parity
- last executor / triggerer parity
- stale-failure parity
- dashboard/environment mapping parity
- API query limit parity with Python fetching up to 3000 runs

## 9. Sandbox And Deployment Status

A POC sandbox exists to generate representative telemetry.

The sandbox shape:

- sandbox resource group
- Log Analytics workspace
- backend Application Insights
- frontend Application Insights
- App Service plan
- backend Web App
- frontend Web App

Useful sandbox scenarios:

- backend 500 failures
- backend exceptions
- slow/performance endpoint
- frontend auth/browser exception
- one-click frontend trigger that dispatches several signals

Important deployment fixes already made:

- nested zip deployment issue fixed
- sandbox app deployment decoupled from infrastructure deployment
- direct `dotnet publish` used for sandbox web apps
- nested zip guard added
- explicit Azure CLI `config-zip` deployment used
- stale `WEBSITE_RUN_FROM_PACKAGE` cleared before redeploy
- frontend can infer backend host if app setting is missing
- app-only redeploy can refresh frontend scenario settings

## 10. AI Provider Status

Target architecture is Azure OpenAI.

Temporary fallback exists:

- `AI_PROVIDER = copilot-cli`

Why:

- Azure OpenAI quota/deployment availability can block validation
- Copilot CLI lets the AI path be tested without changing the final Azure OpenAI target

AI mode is visible in output:

- enabled
- disabled
- fallback to deterministic

Runtime toggle:

- `ENABLE_AI_ENRICHMENT = true`
- `ENABLE_AI_ENRICHMENT = false`

Azure OpenAI guardrails already exist:

- diagnostic logging includes endpoint, deployment, API version, request URL, request id, and response body snippet
- API version validation rejects values that look like deployment model versions, for example `2024-07-18`
- supported runtime API versions currently documented:
  - `2024-10-21`
  - `2024-12-01-preview`

## 11. Where We Are Right Now

We are in the phase:

> Production known-issue parity.

This means we are comparing Python FAQ behavior against the C# deterministic catalog, then adding one small known-issue rule group at a time with tests.

The Python FAQ says these production known/expected cases matter:

- HTTP `429` rate limiting
- HTTP `499` client disconnects
- result code `0` incomplete client requests
- low-volume `401` / `403` token/auth cases
- low-count `404` cases
- JobPreparation decommissioning
- bot/probe traffic like `/.env`, `/wp-admin`, `/robots.txt`
- frontend vs backend ownership classification
- performance threshold behavior

Already covered in the C# deterministic known-issue catalog:

- HTTP `499`
- HTTP `429`
- result code `0`
- frontend `interaction_in_progress`
- one initial pipeline known-flaky example

The latest implemented slice was:

- add `429` as known rate-limiting
- add result code `0` as known incomplete/client request
- add tests proving AI-off production overview moves those into `Known Issues`

Latest test result:

- focused production overview tests: 6 passed
- full test suite: 119 passed

## 12. Current Uncommitted Changes

At the last check, these files were modified:

- `Docs\Implementation\Plan.md`
- `Docs\Implementation\pipelines\Progress.md`
- `Docs\Implementation\production\Progress.md`
- `Implementation\config\rules\known-issues.json`
- `Implementation\project\AiMonitoring.Tests\Production\ProductionRuleBasedOverviewBuilderTests.cs`

Important note:

- `Docs\Implementation\pipelines\Progress.md` was already modified before the latest production known-issue slice.
- It was not touched by the latest implementation work.

Suggested commit message for the latest clean checkpoint:

```text
Add production known-issue parity for HTTP 429 and result code 0
```

Before committing, inspect whether you want to include the pre-existing pipeline progress doc change in the same commit or separate it.

## 13. How Known-Issue Matching Works Today

The catalog object is simple:

- `KnownIssueCatalog`
- `KnownIssueRule`
- `KnownIssueCandidate`

A rule can match on:

- flow, for example `production`
- category, for example `httpFailure`
- exact result code
- text contained in combined searchable fields
- pipeline name text
- environment

For production HTTP failures, the rule-based overview builder creates a known-issue candidate from:

- operation name
- app role/service
- result code
- focus reason

If the catalog matches, the item goes to `Known Issues`.
If the catalog does not match, it goes to `Action Required` with deterministic severity and next step.

This simple design is why `429`, `499`, and result code `0` were easy to implement: exact result-code matching was already enough.

## 14. What Bot/Probe Traffic Means

Bot/probe traffic is automated scanning against common paths that do not belong to the app.

Examples from the Python FAQ:

- `/.env`
- `/wp-admin`
- `/robots.txt`

Other common examples that might appear in telemetry:

- `/wp-login.php`
- `/phpmyadmin`
- `/.git/config`
- `/config.json`
- `/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php`

A `404` for one of these paths is usually good behavior. The app rejected a route that should not exist.

But not every `404` should be downgraded.

Examples:

- `GET /.env` returning `404`: likely known bot/probe traffic
- `GET /wp-admin` returning `404`: likely known bot/probe traffic
- `GET /api/orders` returning `404`: not automatically known, might be a real broken API route

This is why the next bot/probe rule should match specific probe-like paths, not all `404`s.

## 15. Recommended Next Implementation Slice

Next slice:

> JobPreparation + bot/probe known-issue parity.

Why this is the best next slice:

- both rules are explicit in the Python FAQ
- both can probably use the current catalog matcher
- no new telemetry fields should be required
- tests can be focused and deterministic

Expected implementation shape:

1. Add catalog rule for JobPreparation.
2. Add catalog rule or rules for bot/probe paths.
3. Add production overview tests:
   - JobPreparation backend exception or HTTP failure becomes `Known Issues`
   - `GET /.env` or `GET /wp-admin` returning `404` becomes `Known Issues`
   - normal `GET /api/orders` returning `404` remains actionable for now
4. Run focused production overview tests.
5. Run full test suite.
6. Update `Docs\Implementation\production\Progress.md` with the mental model review.

Concrete examples for the next slice:

```text
GET /.env returned HTTP 404
=> Known Issues
=> Bot/probe traffic; no app fix unless load/security alerts increase.
```

```text
JobPreparationException in StartJobPreparation
=> Known Issues
=> Service is being decommissioned; continue cleanup tracking.
```

```text
GET /api/orders returned HTTP 404
=> Action Required for now
=> Could be a real route/client mismatch.
```

## 16. Slices After That

### Low-count 404

Python says low-count 404 cases can be known/expected.

This is trickier than bot/probe traffic because not all 404s are harmless.

Before implementing, inspect whether C# detail rows have enough information:

- endpoint/path
- occurrence count
- maybe request volume or failure rate

Safe goal:

- low-count harmless 404s can become known/info
- concentrated or important API 404s remain actionable/warning

### 401/403 Auth Rules

Python says:

- if auth failures are less than 5 percent of endpoint requests: known token expiration
- if greater than 5 percent: warning, possible auth config issue

This likely needs more telemetry than current `AppFailureDetail` carries.

Before implementing, inspect:

- KQL query output
- `AppFailureFinding`
- `AppFailureCandidate`
- `AppFailureDetail`

Possible outcome:

- add failure-rate or total request count to detail rows
- or document that the rule cannot be faithful yet

### Ownership And Team Routing

Python includes team ownership hints:

- Casa: frontend
- Lausanne: auth/remote assistance/user management
- Mex: device/job/edge related
- Rabat: performance/reports/translations

C# does not fully reproduce this yet.

This should probably come after known-issue classification is more complete.

## 17. Pipeline Work After Production Known Issues

Once production known-issue parity is good enough, switch to pipeline parity.

Highest-value pipeline items:

- compare C# collector behavior against Python `pipeline_helper.py`
- confirm fetch limits and lookback behavior
- implement or verify last success
- implement or verify last merger / commit author
- implement or verify last executor / triggerer
- implement stale failing pipeline detection
- align ownership mapping with Python `team_ownership.py`
- align dashboard/environment mapping

The reason pipeline ownership matters:

A pipeline report is only operationally useful if it says who should act, which environment is affected, and whether this is stale or newly broken.

## 18. Testing Discipline To Keep

For every deterministic rule:

- write the test in the same step
- make the test explain the scenario
- prove expected input and output
- prove why the item is known/actionable
- run focused tests first
- run full tests after

Current useful test command:

```powershell
dotnet test Implementation\project\AiMonitoring.Tests\AiMonitoring.Tests.csproj --no-restore -v minimal
```

Focused production overview command:

```powershell
dotnet test Implementation\project\AiMonitoring.Tests\AiMonitoring.Tests.csproj --no-restore --filter FullyQualifiedName~ProductionRuleBasedOverviewBuilderTests -v minimal
```

## 19. How To Continue Tomorrow

Suggested sequence for tomorrow:

1. Read this note once.
2. Open `Docs\Implementation\Plan.md` and review the parity map.
3. Open `Implementation\config\rules\known-issues.json`.
4. Open `ProductionRuleBasedOverviewBuilderTests.cs`.
5. Check git status.
6. Decide whether to commit current changes.
7. If continuing implementation, start Step 42: JobPreparation + bot/probe known-issue parity.

Suggested first prompt to continue:

```text
Continue from the coffee-break recap. Before implementing, inspect the current known-issue catalog and ProductionRuleBasedOverviewBuilder tests. Implement Step 42: JobPreparation + bot/probe known-issue parity as one small deterministic slice, add tests, run focused and full tests, and update production progress with the mental model review.
```

## 20. The Key Thing To Remember

The project is not stuck in random feature work. It is moving through parity slices.

Current path:

1. Make production deterministic classification match the Python FAQ.
2. Keep AI grounded and secondary.
3. Then align pipeline ownership/stale-failure behavior.
4. Then validate the whole chain through Azure DevOps and the sandbox.

The immediate next step is small and concrete:

> Add deterministic known-issue rules for JobPreparation and bot/probe traffic, with tests.
