# Proactis / Victor Garcia - 45 Minute Interview Cheat Sheet

Use this during the live interview. Keep the detailed plan open only if you need deeper answer guidance.

## Interview Objective

Validate whether Victor can act as a senior .NET/Azure engineer who can also help Proactis adopt AI-assisted development practices.

Do not spend time proving that he knows basic .NET. His CV is broad. Test depth, judgement, ownership, and practicality.

## 0-5 min - Opening Context

Say:

"Proactis is working on its Gen 3 modernization program. There are two immediate priorities: first, an AI-assisted SDLC transformation using tools such as Claude; second, an initial Supplier Portal delivery with login, registration, and document upload."

"The goal of this interview is to understand how you would operate as a senior .NET/Azure engineer: how you design, how you handle security and quality, how you use AI responsibly, and how you help a distributed team work consistently."

Transition:

"I will use your CV as context, but I will focus mainly on decision-making and practical delivery."

## 5-9 min - Q1: Relevant Experience

Ask:

"Which recent project is closest to this Proactis context, and what did you personally own?"

Listen for:
- Specific recent project, ideally UST or MyneHub.
- Personal ownership, not just team exposure.
- Architecture, implementation, CI/CD, security, observability, reviews, or mentoring.

Follow-up:

"Where were you hands-on, and where were you acting as architect or technical lead?"

Red flag:
- Lists technologies without explaining his role.

## 9-17 min - Q2: Supplier Portal .NET Architecture

Ask:

"For the first Supplier Portal milestone, we need login, registration, and document upload. How would you structure the .NET solution?"

Listen for:
- Modular monolith first, unless there is a reason for microservices.
- API layer, application/use-case layer, domain layer, infrastructure layer.
- Domain concepts: Supplier, Registration, Document, DocumentType, DocumentStatus.
- Infrastructure: identity provider, database, Blob Storage, notifications, logging.
- Validation, tests, error handling, observability.
- Clean Architecture used pragmatically.

Follow-up:

"Which parts are real domain logic, and which parts are infrastructure?"

Good signal:
- "Blob upload is infrastructure; document ownership and supplier eligibility are business/application rules."

Red flag:
- Says Clean Architecture, CQRS, DDD, MediatR without explaining where they add value.

## 17-25 min - Q3: Azure Document Upload

Ask:

"Design the document upload flow on Azure. How do you handle storage, metadata, security, and failure cases?"

Listen for:
- Azure Blob Storage, private containers.
- Backend uses managed identity to access storage.
- Metadata in DB: SupplierId, DocumentId, BlobName, type, status, timestamps, audit fields.
- File size/type validation.
- Malware scanning or quarantine.
- Supplier-level authorization.
- Failure handling and cleanup.

Follow-up:

"What happens if the blob upload succeeds but metadata persistence fails?"

Good signal:
- Pending DB record, finalization step, retry/cleanup, idempotency.

Red flags:
- Stores files on the web server.
- No supplier ownership check.
- No partial failure strategy.

## 25-29 min - Q4: Azure Hosting Choice

Ask:

"For the first production milestone, would you deploy this on App Service, Container Apps, or AKS? Defend your choice."

Listen for:
- App Service is probably enough for first milestone.
- Container Apps if containers/event-driven workloads are already standard.
- AKS only if Proactis has Kubernetes maturity or a strong platform reason.
- Awareness of operational complexity.

Follow-up:

"What would make you change that choice later?"

Good signal:
- "I would choose the simplest production-ready option that fits the operating model."

Red flag:
- Defaults to AKS because it is more scalable.

## 29-35 min - Q5: Concrete AI Experience

Ask:

"Your CV mentions AI-assisted development, Kiro, MCP, and RAG integration. Give me a concrete example of how you used AI to improve development work."

Listen for:
- Specific tool and use case.
- Prompt/context he provided.
- Output produced.
- How he validated it.
- What he rejected or corrected.

Follow-up:

"What did the AI get wrong, and how did you catch it?"

Good signal:
- Mentions tests, review, security, architecture fit, and human accountability.

Red flags:
- "AI writes code faster."
- No concrete example.
- No validation process.

## 35-40 min - Q6: AI Governance

Ask:

"If Proactis asks you to define AI-assisted development practices for a distributed .NET team, what rules would you introduce first?"

Listen for:
- No secrets or confidential data in prompts unless approved.
- Approved tools and use cases.
- Developers own generated code.
- Mandatory review and tests.
- Prompting standards and reusable examples.
- PR checklist for AI-assisted code.
- Dependency/security scanning.
- Metrics: cycle time, review rework, escaped defects, test quality, onboarding time.

Follow-up:

"How would you prevent AI from increasing review debt?"

Good signal:
- Smaller PRs, explicit acceptance criteria, tests, review discipline, no blind generated code.

Red flag:
- "Everyone can use AI however they want."

## 40-43 min - Q7: Team Standardization

Ask:

"The engagement includes a 6-7 person T&M team plus Proactis and offshore teams. How would you standardize engineering practices across these groups?"

Listen for:
- Definition of done.
- Coding standards.
- PR templates and review rules.
- CI/CD quality gates.
- Architecture examples.
- Written decisions for distributed teams.
- Pairing, office hours, or review calibration.

Follow-up:

"How would you convert your instructor experience into practical engineering enablement?"

Good signal:
- Uses real repository examples, real stories, example PRs, and review feedback.

Red flag:
- Only talks about Scrum ceremonies.

## 43-45 min - Q8: Final Risk Question

Ask:

"Based on the Proactis context, what are the top three technical or delivery risks you would raise immediately?"

Listen for:
- Identity and supplier authorization.
- Document upload security/compliance.
- AI adoption without governance.
- Environment and pipeline readiness.
- Scope/design ambiguity.
- Distributed team consistency.

Follow-up if time:

"Which risk would you tackle first in the first two weeks?"

Good signal:
- Starts with risks expensive to retrofit: identity, document security, environments, standards.

## Quick Scoring

Score each from 1 to 4:

| Area | Score |
| --- | ---: |
| .NET architecture depth |  |
| Azure/security judgement |  |
| AI-assisted development maturity |  |
| Senior ownership and communication |  |
| Team enablement |  |

Decision guide:

- 17-20: Strong fit.
- 13-16: Possible fit, validate weak areas.
- 10-12: Solid engineer, probably weak for advisory + delivery role.
- Below 10: Not recommended.

## Protect These If Time Slips

Must ask:
- Q2 Supplier Portal architecture.
- Q3 Azure document upload.
- Q5 Concrete AI experience.
- Q6 AI governance.

Skip or shorten:
- Q4 hosting choice.
- Q7 team standardization.
