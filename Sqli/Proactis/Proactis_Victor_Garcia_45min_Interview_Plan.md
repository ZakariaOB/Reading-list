# Proactis Interview Plan - Victor Garcia - 45 Minutes

Source material reviewed:
- `Proposal_Proactis.pptx`
- `Contexte.md`
- `Victor Garcia CV SQLI.pdf`

## Candidate-Specific Reading

Victor Garcia presents as a senior backend/software engineer with 15+ years of experience. His CV shows strong .NET background, Clean/Hexagonal Architecture, CQRS with MediatR, DDD, REST APIs, SQL Server, Azure-hosted systems, AKS, Service Bus, Cosmos DB, Redis, CI/CD, SonarQube, Veracode, OpenTelemetry, Grafana, and regulated/financial environments.

For the Proactis context, do not spend too much time validating basic .NET familiarity. The interview should instead test whether his experience is current, deep, and transferable to:

- Supplier Portal delivery: login, registration, document upload.
- .NET + Azure production architecture.
- AI-assisted development practices and governance.
- Coaching and standardization across distributed teams.
- Senior-level ownership in an agile T&M delivery model.

## Main Hypotheses To Validate

### Likely Strengths

- Long experience across .NET versions and backend systems.
- Clean/Hexagonal Architecture, DDD, CQRS, MediatR, AutoMapper, EF, Dapper, FluentValidation.
- Azure experience: AKS, Azure AD/JWT, Service Bus, Cosmos DB, Redis, Azure DevOps/GitHub pipelines.
- Observability and quality tooling: OpenTelemetry, Grafana, SonarQube, Veracode.
- Regulated environments and complex legacy modernization.

### Main Risks Or Gaps

- Recent SQLI/JTI work appears to be WPF desktop/regulatory maintenance, not supplier-facing portal delivery.
- AI-assisted development is mentioned in the summary but not strongly evidenced in project details.
- Azure experience exists but should be tested for practical design depth, not service-name familiarity.
- Leadership/coaching/offshore enablement is not strongly demonstrated in the CV.
- He may be very broad technically; the interview must check depth, prioritization, and pragmatic decision-making.

## Interviewer Opening Presentation

Use this phase to align the candidate before asking technical questions. Keep it concise; the goal is to give enough context for practical answers without consuming the interview.

### 0-5 min - Proactis Context And Interview Goal

Suggested opening:

"Before we go into your experience, I want to give you the context of the discussion. Proactis is working on its Gen 3 transformation program. The objective is to modernize the platform, improve delivery capacity, and strengthen engineering practices."

"There are two immediate priorities. The first one is an AI-assisted SDLC transformation, including the definition of practical development practices around tools such as Claude, coding assistants, review discipline, prompt usage, and governance. The second one is a concrete delivery stream: a Supplier Portal with login, registration, and document upload."

"The expected role is not only to write code. We are looking for senior engineers who can contribute to delivery on a .NET and Azure platform, make pragmatic architecture decisions, maintain quality, and help distributed teams adopt AI-assisted development practices in a safe and productive way."

"The goal of this interview is therefore to understand how you think and operate in realistic situations: how you structure a .NET solution, how you design secure Azure components, how you use AI without lowering quality, and how you help a team work consistently."

Transition:

"I will use examples from your CV, but I will focus mainly on decision-making, depth, and practical delivery."

Do not over-explain:
- Avoid turning this into a sales pitch.
- Do not describe every slide from the proposal.
- Keep the tone collaborative: this is a technical alignment discussion, not a quiz.

## 45-Minute Structure

### 0-5 min - Proactis Context And Interview Goal

Use the opening presentation above.

### 5-9 min - CV Calibration

Question 1:
"Your CV covers many technologies and roles. Which recent project is closest to the Proactis context, and what did you personally own?"

What to listen for:
- Clear ownership, not just team exposure.
- A recent .NET/Azure/API project, ideally UST or MyneHub.
- Concrete decisions around architecture, pipelines, security, observability, or delivery.

Follow-up:
"Where were you acting as an implementer, and where were you acting as an architect or technical lead?"

Interviewer alignment answer:
- A strong answer should probably reference either the UST Azure-hosted .NET 6 project, the MyneHub backend/architecture project, or a similar recent API/cloud delivery engagement.
- He should clearly separate team context from personal ownership: architecture decisions, implementation, pipelines, reviews, observability, security, or mentoring.
- The best answer is specific: "I designed X, implemented Y, reviewed Z, and was accountable for these outcomes."

Red flags:
- Long list of technologies without specific responsibility.
- Cannot explain recent hands-on work.

### 9-17 min - .NET Architecture Scenario

Question 2:
"Imagine we start the Supplier Portal with login, registration, and document upload. How would you structure the .NET solution?"

Expected senior answer:
- API layer, application/use-case layer, domain model, infrastructure layer.
- Clear dependency direction.
- Validation with FluentValidation or equivalent.
- Persistence with EF/Dapper where appropriate.
- DTO mapping discipline.
- Tests at unit, integration, and API levels.
- Avoids overengineering the first milestone.

CV-specific follow-ups:
- "You mention Clean/Hexagonal Architecture and DDD. What would be a real domain concept here, and what would simply remain application logic?"
- "When would you use MediatR here, and when would you avoid it?"
- "How would you prevent Clean Architecture from becoming ceremony?"

Interviewer alignment answer:
- A good practical architecture would likely be a modular .NET API or web application, not necessarily microservices.
- Expected structure: API/controllers or minimal APIs; application use cases for registration, login support, document upload, and document metadata; domain model for supplier, registration status, document, document type, and ownership; infrastructure for database, identity integration, blob storage, email/notification, and external services.
- Clean Architecture should be used to protect business logic and testability, not to create excessive layers for simple CRUD.
- MediatR can be useful for commands/queries and pipeline behaviors such as validation/logging, but should not hide simple flows or make debugging painful.
- A senior candidate should distinguish real domain rules from application workflow. For example, document ownership, supplier status, allowed document types, and approval state may be domain concepts; raw file streaming to Blob Storage is infrastructure/application logic.

Strong signals:
- Can distinguish domain complexity from CRUD.
- Can explain tradeoffs, not only patterns.
- Keeps the design simple but testable.

### 17-25 min - Azure, Security, And Document Upload

Question 3:
"Design the document upload flow for suppliers on Azure. Include identity, storage, metadata, security, and failure handling."

Expected senior answer:
- Azure Blob Storage or equivalent.
- Private containers, managed identity, Key Vault.
- Virus/malware scanning or quarantine process.
- File type and size validation.
- Metadata persisted in database.
- Correlation ID and audit trail.
- Idempotency and retry behavior.
- Failure handling when storage succeeds but database persistence fails.

CV-specific follow-ups:
- "You have used Azure AD/JWT and database-backed authorization. How would you model supplier access to documents?"
- "Would you expose SAS tokens directly to the frontend? Why or why not?"
- "Where would you use async processing, for example Service Bus or Functions?"

Interviewer alignment answer:
- A strong answer should use Azure Blob Storage for documents, with private containers and managed identity from the application.
- The application should persist document metadata in a database: supplier ID, document type, blob URI/key, status, size, content type, checksum if useful, upload timestamp, and audit metadata.
- The candidate should discuss validation before and after upload: size limits, allowed extensions/MIME types, malware scanning, quarantine or pending status, and business validation.
- Supplier authorization matters: one supplier must not access another supplier's documents. Access should be checked through claims, roles, supplier account mapping, and database-backed authorization if needed.
- For failure handling, a senior answer should mention idempotency and compensation. If the blob upload succeeds but database persistence fails, the system should either delete the orphaned blob, mark it for cleanup, or complete metadata persistence through a reliable retry process.
- Direct browser-to-Blob upload with short-lived SAS can be acceptable in some designs, but it must be tightly scoped, short-lived, audited, and paired with server-side metadata validation. A simpler first milestone can proxy uploads through the backend if file sizes and load are modest.

Red flags:
- Stores files on the web server.
- Ignores authorization boundaries between suppliers.
- Cannot reason about partial failure.

Question 4:
"For the first production milestone, would you deploy this on App Service, Container Apps, or AKS? Defend your choice."

Expected senior answer:
- App Service or Container Apps may be enough for early portal delivery.
- AKS only if operational complexity is justified.
- Shows awareness that his AKS experience does not mean AKS is always the right choice.

Interviewer alignment answer:
- For an early Supplier Portal milestone, App Service is often the pragmatic default for a .NET web/API workload: simpler operations, easy deployment slots, managed scaling, identity integration, diagnostics, and lower platform overhead.
- Container Apps is reasonable if the team already packages services as containers and wants event-driven scaling or sidecar-friendly deployment without managing Kubernetes.
- AKS is defensible only if Proactis already has an AKS platform, needs Kubernetes-native operations, has multiple services with mature platform support, or has non-standard networking/runtime requirements.
- Strong answer: "I would not choose AKS just because I know it. I would choose the simplest production-capable platform that fits Proactis operations."

### 25-34 min - AI-Assisted SDLC

Question 5:
"Your CV mentions AI-assisted development, Kiro, MCP, and RAG integration. Give me a concrete example of how you used AI to improve development work."

What to listen for:
- Specific workflow, not buzzwords.
- What AI generated or helped analyze.
- How he verified the output.
- Whether this changed speed, quality, onboarding, or maintainability.

Follow-up:
"What did you reject or correct from the AI output?"

Interviewer alignment answer:
- A strong answer should be concrete: generating tests, refactoring a legacy module, explaining unfamiliar code, producing migration options, creating documentation, building a small integration, or accelerating investigation.
- He should describe the input context he gave the tool, the output he received, and how he verified it.
- Good verification examples: compiling, unit tests, integration tests, manual review, security review, checking generated dependencies, comparing behavior before/after, and asking AI to critique its own assumptions.
- The key senior signal is ownership: AI may assist, but the engineer remains responsible for correctness, maintainability, and security.
- If he mentions RAG or MCP, ask for the practical use case: what data/tools were connected, how access was controlled, and what value was produced.

Question 6:
"If Proactis asks you to help define AI-assisted development practices for a distributed .NET team, what rules would you introduce first?"

Expected senior answer:
- No secrets or client-sensitive data in prompts unless approved.
- Human accountability for generated code.
- Mandatory tests and review for AI-assisted code.
- Prompt/context standards.
- PR checklist updated for AI-assisted work.
- Approved tools and dependency policies.
- Security and IP guardrails.
- Reusable examples and playbooks.

CV-specific follow-ups:
- "How would you adapt these rules for junior or offshore developers?"
- "How would you measure whether AI is helping rather than creating review debt?"
- "Where can AI be useful in a legacy modernization context?"

Interviewer alignment answer:
- First rule: protect client data, credentials, source code, and confidential architecture information according to approved tooling and policy.
- AI-generated code must be treated like code written by a developer: reviewed, tested, owned, and traceable.
- Define approved use cases: test generation, code explanation, refactoring support, documentation, API examples, threat-model prompts, PR review assistance, and migration analysis.
- Define prohibited or controlled use cases: pasting secrets, unapproved production data, blind dependency adoption, bypassing code review, and committing generated code without understanding it.
- Add team practices: reusable prompts, examples in the repository, PR checklist items, review calibration, coding standards, and training sessions.
- Measure impact through cycle time, PR rework, defect leakage, test quality, onboarding time, and developer feedback. Do not measure success by lines of code generated.

Red flags:
- Treats AI as autocomplete only.
- Trusts AI output because it compiles.
- Cannot define governance or measurement.

### 34-40 min - Senior Delivery And Team Enablement

Question 7:
"The engagement includes a 6-7 person T&M team plus collaboration with Proactis and Manila teams. How would you help standardize engineering practices across these groups?"

Expected senior answer:
- Shared definition of done.
- Coding standards and architecture examples.
- PR review calibration.
- Pairing or office hours.
- CI quality gates.
- Documentation and reusable prompts.
- Sprint demos and technical reviews.
- Clear escalation for architecture decisions.

CV-specific follow-up:
"You have instructor experience. How would you convert that into practical enablement for an engineering team rather than classroom training?"

Interviewer alignment answer:
- A strong answer should convert training into delivery habits: shared repository examples, architecture decision records, pull request templates, definition of done, coding guidelines, test patterns, and onboarding walkthroughs.
- Good enablement is embedded in work: pairing, mob reviews, office hours, example PRs, lightweight workshops based on real backlog items, and recurring architecture reviews.
- For Manila or distributed teams, he should mention time-zone-aware communication, written decisions, async documentation, clear escalation paths, and review consistency.
- He should avoid creating a theoretical training program disconnected from sprint delivery.

Question 8:
"You discover during sprint 1 that the existing Supplier Portal design does not align with the target architecture. What do you do?"

Expected senior answer:
- Identify the conflict precisely.
- Propose options with impact on scope, cost, and risk.
- Escalate through product/architecture governance.
- Protect delivery by slicing scope.
- Document decisions.

Interviewer alignment answer:
- The right response is not to block delivery or silently deviate from architecture.
- A senior candidate should identify the exact mismatch: identity flow, data model, UX assumption, file handling, integration dependency, security constraint, or deployment constraint.
- He should bring options with consequences: adapt the design, adjust the architecture, split the feature, defer a non-critical part, or create a temporary implementation with a planned refactor.
- The decision should involve Product Owner, architecture leadership, and delivery lead. The outcome should be documented as an architecture decision or backlog refinement.

### 40-45 min - Closing Risk Question And Score

Final question:
"Based on the Proactis context and your experience, what are the top three technical or delivery risks you would raise immediately, and how would you reduce them?"

Strong answer should mention several of:
- Ambiguous scope or incomplete functional design.
- Identity and supplier authorization model.
- Document upload security/compliance.
- Environment and pipeline readiness.
- Architecture alignment with existing platform.
- AI adoption without governance.
- Distributed team consistency.

Interviewer alignment answer:
- Strong risk answer 1: AI adoption risk. Mitigation: define approved tools, data policy, PR checklist, training, and measurable adoption indicators.
- Strong risk answer 2: Supplier identity and authorization risk. Mitigation: clarify identity provider, supplier account model, claims/roles, document ownership, and audit requirements early.
- Strong risk answer 3: Document upload security/compliance risk. Mitigation: private storage, scanning/quarantine, metadata, retention policy, audit trail, and failure handling.
- Strong risk answer 4: Environment and pipeline readiness risk. Mitigation: confirm repos, branching, CI/CD, test environments, secrets, infrastructure-as-code, and release process during ramp-up.
- Strong risk answer 5: Distributed delivery consistency risk. Mitigation: coding standards, review calibration, examples, shared ceremonies, and explicit technical governance.

Use the last minute to score silently.

## Scoring Sheet

Score each area from 1 to 4.

| Area | Score | Evidence |
| --- | ---: | --- |
| .NET architecture depth |  |  |
| Azure production judgement |  |  |
| Security and document handling |  |  |
| AI-assisted SDLC maturity |  |  |
| Senior ownership and communication |  |  |
| Distributed team enablement |  |  |

Interpretation:

- 20-24: Strong fit for senior/lead role.
- 16-19: Potential fit, validate weak areas with references or a technical exercise.
- 12-15: Solid engineer but likely not enough for this advisory + delivery role.
- Below 12: Not recommended for this context.

## Decision Guidance

### Strong Hire If

- He can explain recent hands-on ownership in .NET/Azure delivery.
- He chooses pragmatic Azure services, not over-complex architecture.
- He handles document upload with security, auditability, and failure modes.
- He demonstrates real AI-assisted coding discipline with verification.
- He can turn his instructor background into engineering enablement and coaching.

### Concern If

- He mostly lists technologies from the CV without depth.
- He defaults to AKS or microservices without justification.
- He cannot give a concrete AI-assisted development example.
- He lacks a clear approach to team standards, reviews, and adoption.
- He struggles to explain current .NET 8/9 practices despite listing them.

## Optional Deep-Dive Questions If Time Allows

- "How would you implement observability for this portal using OpenTelemetry?"
- "How would you structure integration tests for Blob Storage and database persistence?"
- "How do you handle API versioning and backward compatibility?"
- "How would you approach migration from a legacy .NET application to a modern modular architecture?"
- "How would you decide whether a supplier registration workflow needs synchronous or asynchronous processing?"

## Recommended Interviewer Stance

This candidate should not be interviewed as a generic senior developer. Treat him as someone with a broad and credible CV, then challenge the sharp edges:

- Current depth versus historical breadth.
- Practical AI governance versus AI buzzwords.
- Azure service judgement versus service familiarity.
- Technical leadership versus individual contribution.
- Production-readiness of the Supplier Portal design.
