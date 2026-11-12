# KALIZA — Master System Architecture

**Document type:** Product and technical master blueprint  
**Status:** Proposed architecture baseline; implementation has not yet been established by this document  
**Product:** Kaliza  
**Repositories:** `Aimable2002/kaliza` (backend), `Aimable2002/kaliza_UI` (web frontend)  
**Audience:** Product owner, backend engineers, frontend engineers, agent/AI engineers, infrastructure engineers, Claude, Lovable and future contributors

## How to read this blueprint

Decisions are deliberately classified:
- **Established direction** — a product requirement already agreed: WhatsApp-first control, a small supporting frontend, agentic execution, Daytona Linux VM execution, modular services, and Business Growth as the initial service.
- **Recommended design** — an architecture proposed to implement those requirements.
- **Open decision** — a choice that must be tested or confirmed before it becomes a binding implementation decision.

This is a greenfield blueprint for the two repositories. It does not assume an existing Claude-built application, codebase, schema, integrations, or deployed service. Implementation choices below are specifications and recommendations, not claims that code already exists.

## 1. Product architecture

Kaliza is an AI-powered opportunity discovery and execution platform: one platform, multiple specialized services, one shared agent execution infrastructure.

The product should not be positioned as a chatbot with integrations. It should understand a user's objective, discover opportunities, plan work, use authorized tools, execute multi-step tasks, verify results, and report measurable outcomes.

### Initial and future services

| Service | Purpose | Release position |
|---|---|---|
| Business Growth | Help businesses improve visibility, attract customers, increase sales, develop prospects, and automate existing business workflows | Initial service |
| Career Growth | Help individuals discover jobs, prepare CVs, apply for roles, and prepare for interviews | Future service |
| Future opportunity services | Additional services that directly support users' economic opportunities | Add only after product validation |

Business Growth and Career Growth are service modules on one shared platform, not separate AI products. They share identity, workspace, task, agent, execution, integration, approval, billing, and notification infrastructure.

### Product principles
1. Outcome and execution first; conversation is the control surface.
2. WhatsApp is the primary user interface.
3. The web frontend exists for visual, administrative, and document-heavy experiences that are unsuitable for chat.
4. Service capabilities are modular; shared platform capabilities are not duplicated.
5. Agents can plan and execute autonomously within explicit policies.
6. Sensitive or consequential actions require human approval.
7. Tasks and agent state are durable across restarts and execution-environment interruptions.
8. External integrations are reusable connectors, not service-specific one-off code.
9. Usage, cost, permissions, and audit are enforced by platform services, not trusted to an LLM.
10. New services should be added by implementing a service contract, not rebuilding the platform.

## 2. System architecture at a glance

The platform is separated into five layers:

1. **Channels:** WhatsApp, supporting web application, internal admin console.
2. **Kaliza Core:** identity, tenant/workspace, conversation, service registry, task/workflow, permissions, approvals, integrations, usage, billing, files, notifications and audit.
3. **Agent intelligence:** orchestration, planning, specialist agents, memory, model gateway and tool selection.
4. **Execution infrastructure:** Daytona-managed isolated Linux environments, browser/terminal/files/scripts, media and document processing, plus specialized coding tools where authorized.
5. **External systems:** Meta, Google, email, commerce, analytics, payment and other provider APIs.

Logical flow:

`User → Channel Gateway → Identity/Workspace Resolution → Conversation Context → Service Router → Task Manager → Policy Check → Agent Orchestrator → Tool/Connector or Daytona → Verification → Result/Notification`

Persistent data supports every layer. The agent is not the system of record.

## 3. User interaction and channel architecture

### 3.1 WhatsApp — primary control interface (established direction)

Users should be able to:
- Start a task and explain an outcome in natural language.
- Ask for research, recommendations, and workflow automation.
- Supply missing information and files.
- Receive progress updates, clarifying questions, and completion summaries.
- Review summaries and links to generated files.
- Approve, reject, or request changes to proposed consequential actions.
- Pause, resume, cancel, or check the status of work.
- Request reports and follow-up actions.

WhatsApp is a channel, not the place where business logic runs. Messages arrive through the Meta WhatsApp Cloud API webhook and are normalized into Kaliza's internal message model. The backend resolves the user, tenant, workspace, conversation, permissions, and service before creating or resuming a task.

The user should not need to know which model, agent, tool, or connector is being used.

**Important distinction:** Kaliza's own WhatsApp control channel is separate from a customer's connected WhatsApp Business account used for that customer's business operations. These have separate credentials, scopes, consent, routing, and audit trails.

### 3.2 Web frontend — deliberately small

The frontend is a visual workspace, not a second full conversational product. It should support:
- Overview and activity.
- Conversation history and task detail.
- Active, queued, paused, completed, and failed workflows.
- Approval review.
- Documents and generated deliverables.
- Connected integrations and account health.
- Business profile and service configuration.
- Usage, package, billing, and account settings.

WhatsApp and web must read and update the same backend records. Do not implement task state, agent orchestration, permission rules, billing calculations, or connector secrets in the frontend.

### 3.3 Admin console

Provide a separately authorized administrative experience for tenant/customer support, subscriptions, usage, execution health, failed jobs, sandbox lifecycle, connector health, operational configuration, audit events, and security incidents. Admin access must be role-scoped, logged, and designed to avoid unnecessary access to customer content. It may live in a separate app or protected deployment; do not expose internal admin capabilities in ordinary customer navigation.

## 4. Kaliza Core — shared platform backend

Kaliza Core owns durable business rules and remains independent of the selected agent framework.

| Module | Responsibility |
|---|---|
| Identity and access | Authentication, sessions, account lifecycle, roles |
| Tenant/workspace | Individual and business workspaces, memberships, isolation |
| Conversation | Channel-neutral conversations, messages, attachments, delivery status |
| Channel gateway | WhatsApp webhooks, outbound messages, web event delivery |
| Service registry | Service definitions, capabilities, versions, availability |
| Task manager | Objectives, task records, state transitions, ownership |
| Workflow engine | Durable steps, retries, checkpoints, dependencies, schedules |
| Policy/permission engine | Action authorization, scopes, risk classification |
| Approval manager | Approval requests, decisions, expiry, evidence |
| Integration manager | OAuth, provider accounts, scopes, token lifecycle |
| Agent gateway | Start, resume, pause, cancel and inspect orchestration runs |
| Model gateway | Provider abstraction, routing, fallback, cost and policy controls |
| Tool registry | Tool schemas, permissions, availability, execution adapters |
| Memory/context | User, business, task, conversation and knowledge context |
| File/document service | Uploads, generated outputs, metadata, access control |
| Notification service | WhatsApp and web notifications, delivery/retry |
| Usage metering | Model, tool, provider, storage and sandbox consumption |
| Billing/entitlements | Plans, quotas, limits, payment state and entitlements |
| Audit service | Immutable or append-only record of consequential activity |
| Observability | Logs, traces, metrics, alerts and correlation IDs |

**Non-negotiable boundary:** an LLM/agent must never be the authority for its own permissions, approval requirements, tenant access, billing limits, or entitlement status. Core services enforce these decisions before execution and again at the action boundary.

### 4.1 Tenancy and workspace model

Support:
- Individual accounts.
- Business organizations.
- Multiple members and roles per business.
- A user who belongs to multiple workspaces.
- Service-specific entitlements within a workspace.
- Multiple connected provider accounts where supported.

Every tenant-owned record must carry an explicit tenant/workspace ownership relationship. Enforce isolation in API queries, background workers, connector callbacks, file access, agent context, and sandbox access. PostgreSQL row-level security is a recommended defense-in-depth mechanism, not a substitute for application authorization. Test cross-tenant access paths.

### 4.2 Durable task lifecycle

A task should be persisted before an agent starts doing work.

Recommended states:

`RECEIVED → CLASSIFIED → PLANNED → (AWAITING_APPROVAL) → QUEUED → EXECUTING → (AWAITING_INPUT) → VERIFYING → COMPLETED`

Terminal/exception states: `FAILED`, `CANCELLED`, `EXPIRED`. A task may return from `AWAITING_INPUT` or a recoverable failure to a resumable state.

Each transition records actor, timestamp, reason, task version, and relevant execution/run reference. Use idempotency keys for inbound webhooks and external mutations. A crash must not silently duplicate a publication, message, payment, or other side effect.

Persist task intent, plan, step status, checkpoints, tool calls, approval state, outputs, errors, retry counts, and timestamps. Agent process memory is never the only copy of task state.

## 5. Service architecture

Services define customer-facing capabilities and domain workflows. They call shared Core interfaces and do not own duplicated identity, billing, task, approval, or connector infrastructure.

### 5.1 Business Growth — initial service

Business Growth automates existing manual business activities and adds proactive business-development capabilities. It is broader than social-media scheduling.

Six capability groups:

**A. Online presence and brand visibility**
- Social content strategy, calendar, drafts, adaptation and scheduling.
- Facebook and Instagram publishing where API permissions allow.
- Comment and authorized message monitoring/management.
- Brand monitoring and engagement workflows.
- Google Business Profile updates and review/visibility workflows where supported.
- Website/search visibility analysis and improvement recommendations.
- Competitor and market presence research.

**B. Advertising management**
- Campaign research, planning and draft creation.
- Audience, creative, placement and budget recommendations.
- Meta Ads campaign/ad-set/ad management through authorized APIs.
- Performance reporting, anomaly and budget monitoring.
- Conversion/event measurement support.
- Human approval before activating campaigns, materially changing spend, or making other consequential changes.

**C. Business development and prospect discovery**
- Define ideal customer profiles and target markets.
- Research prospects, organizations, decision-makers and opportunities from permitted sources.
- Qualify, score, deduplicate and organize leads.
- Prepare prospect briefs and tailored proposals/offers.
- Draft outreach and follow-up sequences.
- Execute approved outreach through connected channels.
- Track responses, next steps and pipeline outcomes.
- Do not scrape prohibited sources or send unsolicited bulk outreach; respect provider terms, privacy and applicable law.

**D. Sales and customer operations**
- Lead intake, qualification and routing.
- Customer inquiry triage and response drafts.
- Quote/proposal preparation.
- Follow-up reminders and status tracking.
- CRM synchronization where integrated.
- Escalation of complaints, sensitive issues, negotiations, refunds or unusual requests to a human.

**E. Business workflow automation**
- Discover repetitive workflows with the business.
- Map existing manual processes, inputs, rules, exceptions and outputs.
- Automate approved steps across connected services.
- Provide human checkpoints for exceptions and consequential operations.
- Maintain run history, failure recovery and measurable time/cost savings.

**F. Analytics and performance**
- Business and channel performance summaries.
- Campaign, content, engagement, lead and sales reporting.
- Goal tracking and trend analysis.
- Recommendations tied to evidence and measurable outcomes.
- Distinguish observed metrics from estimates and model-generated interpretations.

The exact release scope of each capability is a product decision; the six groups define the service architecture, not a promise that every provider integration is available at launch.

### 5.2 Career Growth — future service

Future service for job and career opportunity workflows: profile and CV preparation, job discovery, role matching, application preparation/submission, application tracking, interview preparation and follow-up. It should reuse the same task, agent, file, approval, connector, notification and billing systems. It is not a first-release dependency.

### 5.3 Future service contract

Every service module should declare: service ID/version, description, required entitlements, capabilities, supported channels, task-intent schemas, required context, permitted tools, risk/approval policy, workflow definitions, output types, usage categories and observability metadata. Core routes tasks to a service; services cannot bypass Core authorization.

## 6. Agent intelligence and orchestration

### 6.1 Design principles

The agent system must plan, research, use multiple tools, execute complex tasks, verify outcomes, recover from errors, and continue work over time. It should ask humans for approval only for sensitive or consequential actions, and ask clarifying questions when required information is missing or ambiguity materially affects the result.

Separate:
- **Orchestration:** run lifecycle, graph/state, retries, checkpoints and handoffs.
- **Reasoning/planning:** objective decomposition, prioritization and decisions.
- **Specialist capabilities:** research, business growth, content, operations, coding.
- **Tool execution:** controlled API and sandbox actions.
- **Policy enforcement:** external, deterministic Core decision.
- **Verification:** evidence that intended effects occurred.

### 6.2 Framework evaluation

| Option | Potential role | Evaluation |
|---|---|---|
| LangGraph | Primary durable multi-step orchestration candidate | Strong fit for stateful graphs, checkpoints, human-in-loop and provider flexibility; validate operational complexity |
| OpenAI Agents SDK | Alternative or component for agent/tool handoffs and tracing | Evaluate against real multi-step, resumable Kaliza workflows and provider strategy |
| OpenCode | Specialized coding agent/tool for software tasks inside authorized environments | Not the platform-wide orchestrator; sandbox and permission boundaries still apply |
| LangChain | Optional model/tool/document integration utilities | Use selectively; avoid adopting a broad abstraction layer without a concrete need |

**Recommended approach:** define a Kaliza-owned `AgentOrchestrator` interface first. Candidate methods: `createRun`, `resumeRun`, `pauseRun`, `cancelRun`, `getRun`, and event streaming. Implement one orchestration adapter after a proof-of-concept compares at least two real workflows. Do not install every framework or bind Core task state to one vendor's internal schema.

### 6.3 Logical agent roles

Roles may be nodes, prompts, worker processes or separate agents; they do not all need separate deployed services:
- Supervisor/planner: classify objective, create plan, delegate, monitor and synthesize.
- Research: gather, cross-check and cite information.
- Business Growth: execute service-specific business workflows.
- Content: draft/adapt content and maintain brand constraints.
- Operations: coordinate connected workflows and exceptions.
- Career Growth: future service-specific role.
- Coding: specialized software development tasks through OpenCode or another approved runtime.

Define input/output schemas and allowed tools per role. Avoid unrestricted agent-to-agent delegation.

### 6.4 Memory and context

Separate memory domains:
- Account/user preferences and profile.
- Business profile, goals, brand voice, products and constraints.
- Career profile (future).
- Conversation history.
- Task/run memory and checkpoints.
- Curated knowledge and source documents.
- Operational memory (known errors, connector status, successful workflow patterns).

Memory retrieval must be scoped by tenant/workspace and purpose. Record provenance, freshness, permissions and retention. Do not place secrets or unrestricted cross-tenant data into prompts. Provide mechanisms to correct, export and delete user data according to policy.

## 7. Google ecosystem integration architecture

Build a reusable Google connector family behind a common integration framework. Candidate products include Google Drive, Docs, Sheets, Calendar, Gmail and Business Profile, subject to the actual product scope and Google API approval.

Connector requirements:
- OAuth 2.0 authorization-code flow with state/PKCE where appropriate.
- Per-workspace connection records and encrypted refresh/access tokens.
- Least-privilege scopes; request scopes when a feature needs them.
- Token refresh, revocation, re-consent and connection-health handling.
- Provider quota/rate-limit handling, retries and idempotency.
- Webhook/push notification support where available.
- Normalized connector errors and audit events.
- Explicit account selection and multi-account support where provider APIs allow it.

Do not assume a single first account, first page or first drive is the final architecture. Keep provider-specific API details inside connector adapters. Confirm exact Google API availability, verification requirements and scopes before implementation.

## 8. Meta and communication integrations

Use a Meta connector family for the Kaliza-owned WhatsApp channel and customers' authorized Facebook/Instagram/WhatsApp Business assets.

Potential capabilities, subject to permissions and API availability:
- WhatsApp inbound/outbound messages, templates and delivery status.
- Facebook/Instagram publishing and media workflows.
- Comment and messaging management.
- Page/account discovery and selection.
- Meta Ads campaign, ad-set, ad and reporting operations.
- Webhooks for messages, comments and account events.

OAuth and asset selection must support the appropriate business, portfolio, page, Instagram account, ad account and permission combinations. Do not hard-code “first page” or “first ad account.” Store provider account identifiers, granted scopes, token status, ownership and selected workspace association.

Separate channel identities:
1. Kaliza's WhatsApp number used by the user to control Kaliza.
2. A customer's connected WhatsApp Business identity used by the agent for that customer's business.

These must never share message routing, tokens or tenant boundaries.

**Approval policy:** drafting is generally lower risk; publishing externally, sending outreach/messages, activating ads, changing budgets, deleting assets or other consequential operations require policy evaluation and, where configured, explicit human approval. Enforce Meta platform policies, user consent, messaging windows/templates, rate limits and applicable laws.

## 9. Daytona execution infrastructure

**Established direction: Daytona is required for the intended agentic execution model.** It provides isolated Linux workspaces in which agents can perform authorized real work rather than merely describe it.

Use Daytona for:
- Files and directory operations.
- Shell commands and scripts.
- Installing approved dependencies.
- Browser-based research and web workflows where permitted.
- Document, spreadsheet and data processing.
- Media preparation and conversion.
- Running tests and validation.
- Building deliverables and packaging outputs.
- Controlled coding-agent execution.

Daytona is an execution substrate, not the source of truth for users, tasks, billing, approvals or long-term files.

### 9.1 Sandbox lifecycle

Model lifecycle explicitly: `REQUESTED → PROVISIONING → INITIALIZING → READY/ACTIVE → IDLE → PAUSED → RESUMING → ACTIVE`; also `FAILED`, `SUSPENDED`, `TERMINATING`, `PURGED`.

Persist sandbox ID, tenant/workspace, purpose, owner task, image/runtime version, resource limits, status, timestamps, last activity, lease/expiry, and recovery metadata. Define policies for idle pause, resume, maximum lifetime, cleanup, storage persistence and failed provisioning.

### 9.2 Isolation and safety

- Use tenant/workspace isolation and least-privilege credentials.
- Never expose provider master secrets to the sandbox.
- Provide short-lived scoped credentials or brokered connector calls.
- Restrict network egress where feasible.
- Set CPU, memory, disk, runtime and concurrency quotas.
- Separate user uploads, generated artifacts and temporary workspace data.
- Scan/validate uploads and generated files as appropriate.
- Log commands/actions with task correlation, while avoiding secret leakage.
- Do not treat sandbox persistence as durable application storage.
- Validate pause/resume behavior and recovery before relying on it in production.

Define whether workspaces are per task, per user/workspace, or pooled based on isolation, latency, cost and persistence tests. A workspace-per-business may improve continuity but requires strict isolation and lifecycle controls.

## 10. MCP and tool architecture

MCP may provide a standard interface to tools and resources, but it is not itself the security model.

Create a Kaliza Tool Registry containing:
- Tool ID, version, description and input/output schema.
- Owning connector/service.
- Required scopes and entitlements.
- Risk classification and approval requirement.
- Tenant/workspace context requirements.
- Timeout, retry and idempotency policy.
- Execution location (Core, connector worker, Daytona).
- Audit and usage category.

Tools should be invoked through a policy-aware Tool Gateway. Validate schemas, authorize every invocation, inject scoped context, enforce timeouts/quotas, normalize results/errors, and record audit/usage. Support native API tools and MCP adapters through the same policy boundary. Treat tool output and external content as untrusted data, never as system instructions.

## 11. Model gateway and AI provider architecture

Create a provider-independent Model Gateway. Responsibilities:
- Provider/model catalogue and availability.
- Task-based model routing.
- Structured output and tool-call normalization.
- Token/latency/cost metering.
- Retry, timeout and fallback.
- Provider policy and data-handling controls.
- Per-plan model entitlements and budget enforcement.
- Tracing and evaluation metadata.

Choose models by task needs (reasoning, research, classification, extraction, content, coding), not brand loyalty. Keep model selection configurable. Do not let agent prompts override cost limits or data-handling rules. Evaluate quality, latency, cost, tool reliability and structured-output compliance on representative Kaliza tasks.

## 12. Data and storage architecture

Recommended baseline:
- PostgreSQL for transactional records and durable workflow/task state.
- Object storage (S3-compatible, Cloudflare R2 or equivalent) for uploads, generated files and media.
- Redis or equivalent only where useful for cache, ephemeral coordination and queue acceleration; never the sole durable task store.
- Durable job queue/event mechanism for asynchronous work.
- Search/vector capability only when justified by retrieval requirements; maintain source provenance.

Core domains/tables should include users, organizations, workspaces, memberships, roles, service entitlements, conversations, messages, attachments, tasks, task_steps, workflow_runs, checkpoints, approvals, integration_connections, provider_assets, agent_runs, tool_invocations, model_usage, sandbox_instances, files, notifications, subscriptions, invoices/usage ledger, audit_events and webhook_events.

Use migrations, constraints, indexes, tenant ownership, retention policy, encryption, backup/restore tests and schema versioning. Avoid storing large blobs in relational rows. Keep secrets in a dedicated secret-management/encryption system.

## 13. Security, approvals and governance

Security must be built into every boundary:
- Authentication and secure session management.
- Tenant/workspace authorization.
- Role-based access and scoped service permissions.
- Encryption in transit and at rest.
- Secret storage, rotation and revocation.
- Webhook signature verification and replay protection.
- Input validation and output handling.
- Rate limiting and abuse controls.
- Audit logging and incident response.
- Data retention, export and deletion.
- Dependency, container and sandbox security.

### 13.1 Action risk and human approval

Use deterministic risk classes, for example:
- **Low:** internal research, draft generation, analysis, read-only retrieval within granted scopes.
- **Moderate:** edits to drafts, internal record updates, reversible changes within configured limits.
- **High/consequential:** external publication, outbound customer/prospect messages, campaign activation or spend changes, purchases/payments, deletion, account/security changes, legal/financial commitments, or irreversible actions.

Exact classifications must be configurable by action and service. The policy engine decides whether approval is required. An approval record must show the proposed action, target, content/amount, rationale, evidence, scope, expiry and consequences. Approval must be bound to the exact action payload/version; if the payload changes, obtain a new approval. Support approve, reject, edit/request changes, expiry and revocation. Record who approved and when.

## 14. Billing, packages and usage architecture

Keep packages understandable and few in number. Differentiate mainly by execution capacity, usage volume, research depth, concurrency and number of active workflows—not by removing the core product idea from lower tiers.

Meter at least:
- Model input/output tokens and model class.
- Daytona active/allocated time and resource use.
- Tool and provider API usage where billable.
- Storage and file volume.
- Workflow/task execution volume.
- Premium capabilities and concurrency.

A usage ledger should record tenant, event, unit, quantity, cost basis, plan/entitlement, task/run and timestamp. Enforce quotas before expensive execution and at runtime. Provide user-facing usage explanations and admin cost visibility. Keep estimated cost distinct from provider-reported actual cost. Billing and entitlements must be enforced by Core.

## 15. Infrastructure and deployment architecture

Recommended logical deployables:
- Public API/channel gateway.
- Web application.
- Background worker(s) and durable queue.
- Agent orchestration service/worker.
- Connector workers/webhook handlers.
- Daytona control adapter.
- PostgreSQL and object storage.
- Redis/cache if needed.
- Observability stack and admin surface.

Use separate development, staging and production environments. Manage configuration and secrets outside source control. Add CI checks, migrations, automated tests, dependency scanning, deployment health checks, rollback strategy, backups and restore drills.

Observability should connect a single correlation chain across inbound message, task, workflow run, agent run, model call, tool call, sandbox, connector request and final notification. Track latency, failures, retries, cost, sandbox capacity, queue depth, provider quota and task completion.

## 16. Recommended software project structure

One possible backend structure (adapt to selected language/framework):

```text
kaliza/
  apps/
    api/
    workers/
    agent-runtime/
    webhook-gateway/
    admin-api/
  packages/
    core/
    identity/
    tenancy/
    conversations/
    channels/
    services/
    tasks/
    workflows/
    policies/
    approvals/
    integrations/
    tools/
    models/
    memory/
    files/
    billing/
    usage/
    notifications/
    audit/
    observability/
    shared-contracts/
  services/
    business-growth/
    career-growth/
  adapters/
    orchestration/
    daytona/
    mcp/
    meta/
    google/
  migrations/
  tests/
  infra/
  docs/
```

This is a logical target, not a mandate to create every package on day one. Choose a modular monolith initially if it reduces operational overhead; preserve module boundaries so high-load components can be separated later.

## 17. End-to-end workflow examples

### A. Prospect discovery and approved outreach
1. User asks in WhatsApp to find prospects in a defined market.
2. Channel gateway authenticates and resolves workspace.
3. Core creates task and checks Business Growth entitlement.
4. Planner asks clarifying questions if ICP/region/offer is unclear.
5. Research agent finds and cross-checks permitted sources.
6. Results are deduplicated, qualified and saved with source evidence.
7. Agent drafts tailored proposals and outreach.
8. Policy engine classifies outbound contact as consequential and creates approval request.
9. User reviews/approves exact recipients and message payload.
10. Connector sends only approved messages; responses are tracked.
11. Verification checks provider status; task/report is updated and user notified.

### B. Social content workflow
1. User asks for a weekly content plan.
2. Agent retrieves brand profile, goals, past content and channel connections.
3. Agent researches topics and creates a calendar and drafts.
4. User reviews in WhatsApp or dashboard.
5. Publishing actions pass policy and approval requirements.
6. Meta connector publishes and records provider IDs/status.
7. Agent monitors permitted engagement and prepares response drafts.
8. Report summarizes published content and available performance metrics.

### C. Existing business workflow automation
1. User describes a repetitive manual workflow.
2. Agent maps trigger, inputs, steps, rules, exceptions and desired output.
3. Agent proposes a workflow and identifies required integrations/scopes.
4. User confirms setup and permissions.
5. Workflow runs with checkpoints, retries and exception handling.
6. Consequential actions pause for approval.
7. Run history records outcome, exceptions and time/cost metrics.

### D. Career Growth (future)
Reuse the same task and file infrastructure for profile preparation, job research, role matching, CV tailoring and application tracking. External submission requires explicit authorization and may require per-application review. This is future scope, not a dependency for Business Growth launch.

## 18. Development roadmap

1. **Architecture and product contracts:** finalize service boundaries, data ownership, risk policy, API/event contracts and technology decision records.
2. **Core foundation:** identity, tenancy, workspace, conversation, task lifecycle, durable queue, audit and file metadata.
3. **Agent infrastructure:** orchestration adapter, durable runs/checkpoints, model gateway, tool registry, memory and evaluation harness.
4. **Execution infrastructure:** Daytona provisioning/lifecycle, sandbox policy, scoped tool execution, recovery and metering.
5. **Initial connectors:** WhatsApp control channel, Google/Meta connector foundations and OAuth/account selection.
6. **Business Growth workflows:** deliver a narrow, testable vertical slice first; expand capability groups based on provider access and customer validation.
7. **Web frontend and operational readiness:** task/approval/files/integrations/usage UI, admin controls, monitoring, security and production deployment.
8. **Expansion:** improve Business Growth, then add Career Growth and other validated service modules.

Each phase needs acceptance criteria, automated tests, security review, cost review and operational runbooks before production release.

## 19. Architectural decision register

Maintain an ADR for every material choice. Initial decisions:

| Topic | Current position | Status |
|---|---|---|
| Product identity | One shared platform with modular services | Established direction |
| Primary control | WhatsApp | Established direction |
| Web frontend | Small supporting workspace | Established direction |
| Agent model | Multi-agent capable, durable and tool-using | Established direction |
| Execution | Daytona Linux VM/sandbox | Required direction; lifecycle design to validate |
| Orchestration | Kaliza-owned abstraction; compare LangGraph and OpenAI Agents SDK | Open |
| Coding agent | Evaluate OpenCode as specialized coding capability | Open |
| Backend deployment | Modular monolith initially is a viable recommendation | Recommended |
| Database | PostgreSQL | Recommended baseline |
| Object storage | S3-compatible/R2/S3 | Open provider choice |
| Queue | Durable queue/event system | Open technology choice |
| Connectors | Reusable OAuth/provider adapter framework | Recommended |
| Service order | Business Growth first; Career Growth later | Established direction |

Do not silently convert recommendations into established decisions. Record rationale, alternatives, consequences, owner and date in each ADR.

## 20. Quality attributes and acceptance criteria

The platform is not ready for production until:
- Tenant isolation tests cover API, worker, files, connectors, agent context and sandbox.
- Tasks survive process restarts and resume from persisted checkpoints.
- Retries and webhook replays do not duplicate side effects.
- Approval is bound to an exact action payload and revalidated at execution.
- Tool permissions, entitlements and quotas are enforced outside the model.
- Sandbox provisioning, timeout, pause, resume, cleanup and failure recovery are tested.
- OAuth tokens are encrypted, scoped, revocable and never exposed to prompts or sandbox environments.
- Every consequential action is auditable.
- Usage and cost are attributed to workspace/task/run.
- Users can see task status, failures, approvals and generated outputs.
- WhatsApp and web show consistent task state.
- A new service can be registered without duplicating Core infrastructure.

## 21. Implementation guidance for the two repositories

### Backend repository: `Aimable2002/kaliza`
Owns Core domain logic, APIs, workers, channel/webhook gateway, task/workflow persistence, policy and approval services, agent orchestration adapters, model/tool gateways, Daytona control adapter, integrations, metering, billing, files, notifications, audit and observability.

### Frontend repository: `Aimable2002/kaliza_UI`
Owns the customer-facing supporting web workspace and (if included) a separately protected admin UI. It consumes backend APIs and events. It does not own business rules, agent execution, secrets, connector logic or authoritative task state.

Both repositories must share versioned API schemas and event contracts. Backend changes that affect UI must update the contract and tests. The frontend must not invent undocumented endpoints.

## 22. Instruction to implementation agents and developers

Treat this document as the product/architecture source of truth, while treating all proposed technologies as subject to the decision register.

Before coding:
1. Read this master blueprint and the repository-specific developer specification.
2. Propose a concrete implementation plan with dependencies, milestones and acceptance criteria.
3. Identify any contradiction or missing decision and ask only where a decision materially blocks implementation.
4. Compare orchestration candidates using a real durable workflow proof of concept.
5. Validate Daytona lifecycle, isolation, credential brokering, recovery and metering.
6. Define API schemas, events, database migrations and service contracts before coupling frontend to backend.
7. Build a thin end-to-end vertical slice: WhatsApp message → authenticated workspace → durable task → agent plan → approved tool action → verified result → WhatsApp notification and dashboard record.
8. Add automated tests, observability and security controls with each increment.
9. Do not claim a feature is implemented until code, tests and runtime evidence demonstrate it.
10. Keep Business Growth first and Career Growth future; do not overbuild future service screens or integrations prematurely.

**Final architectural test:** Can Kaliza add a new opportunity-focused service by registering a new service module and its workflows while reusing the same identity, tasks, agent runtime, Daytona execution, connectors, approvals, billing, files and WhatsApp control layer? If not, the platform boundary is not yet being respected.
