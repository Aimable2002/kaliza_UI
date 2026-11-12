# Kaliza Web — Frontend Developer Specification

**Repository:** `Aimable2002/kaliza_UI`  
**Document status:** Greenfield interface specification  
**Source of truth:** [Master System Architecture](MASTER_SYSTEM_ARCHITECTURE.md)

No existing Claude-built frontend is assumed. This repository is to be initialized from this specification. The web app is deliberately a small supporting workspace because WhatsApp is Kaliza's primary control interface.

## 1. Product role and non-goals

The web app exists for experiences that are visual, information-dense, document-heavy or operationally awkward in WhatsApp. It is not intended to replace WhatsApp with a large chat-first product.

**It owns:** navigation, visual presentation, accessible interaction, client-side form state, API consumption, realtime updates, file previews/downloads, approval review UI and user-facing settings.

**It does not own:** agent reasoning/orchestration, task state authority, service business rules, permissions, approvals policy, connector credentials, billing calculations, model routing, tool execution or Daytona management. Those belong to the backend.

Primary user interaction remains WhatsApp for instructions, clarifications, progress, approvals and notifications. The web app must still allow a user to inspect and manage work when a visual interface is more useful.

## 2. Users, roles and information architecture

Customer roles are determined by backend membership and permissions. At minimum support workspace owner/admin and member; exact role names are backend-defined. Hide or disable actions the user cannot perform, but always rely on backend authorization.

Recommended customer navigation:
1. **Overview**
2. **Conversations**
3. **Tasks**
4. **Approvals**
5. **Documents**
6. **Integrations**
7. **Business profile**
8. **Usage & plan**
9. **Settings**

Workspace switcher should appear only when the authenticated user belongs to multiple workspaces. Service navigation should initially emphasize Business Growth. Career Growth should not appear as an active service unless enabled for a future release.

Admin console is a separate protected surface or separately deployed application. Do not expose operational/admin navigation to ordinary customers.

## 3. Core screens and requirements

### 3.1 Authentication and workspace selection
- Sign-in/session handling uses the chosen auth provider/backend contract.
- Show loading, expired-session, unauthorized and service-unavailable states.
- Resolve active workspace from backend membership.
- Support switching workspaces only among authorized memberships.
- Never trust a client-supplied workspace ID without backend verification.

### 3.2 Overview
Show concise operational summary:
- Active tasks and current status.
- Items awaiting user input or approval.
- Recent completed work and deliverables.
- Integration connection warnings.
- Usage/plan summary where entitled.
- Relevant Business Growth progress indicators only when supported by actual backend data.

Avoid fabricated KPI values or vanity metrics. Every metric should have a definition, source and time window. Empty state should explain how to start through WhatsApp and show the linked control number/instructions configured by the backend.

### 3.3 Conversations
- List conversations with channel, workspace, last activity and unread/attention state.
- Open message history with timestamps, attachments and delivery status.
- Distinguish messages sent by user, agent and system.
- Link related tasks and generated documents.
- Do not build a complex independent web chat composer as the primary workflow. If a small message/comment input is added for context or approval feedback, it must call backend APIs and preserve WhatsApp-first positioning.

### 3.4 Tasks
Task list filters: active, queued, awaiting input, awaiting approval, completed, failed, cancelled. Search/filter by service, date and status.

Task detail should display:
- User's original objective.
- Service and task status.
- Plan/step progress at a useful level (not hidden chain-of-thought).
- Timeline of meaningful events.
- Clarifications requested and user responses.
- Approval items.
- Result summary and evidence/source links where applicable.
- Generated files.
- Error explanation and safe retry/resume/cancel controls when backend permits.

Do not expose private model reasoning or internal prompts. Present concise action summaries and verifiable outcomes.

### 3.5 Approvals
Approval inbox must make consequential actions understandable before the user decides. Display:
- Action type and target account/recipient.
- Exact content, amount, budget or change where relevant.
- Agent's short rationale and supporting evidence.
- Scope and expected consequence.
- Expiry and task context.
- Approve, reject or request changes (only if backend supports them).

For multi-action batches, make the number and exact scope explicit. Do not provide a blanket approval that silently authorizes future unrelated actions. After a payload changes, show the updated approval request as a new item. The backend is authoritative and may reject stale approvals.

### 3.6 Documents
- List uploaded and generated documents by workspace.
- Show filename, type, source task, creation date and access state.
- Preview supported types safely.
- Download via backend-authorized short-lived URL.
- Show unavailable, expired, deleted and processing states.
- Never expose raw object-storage keys or Daytona filesystem paths.

### 3.7 Integrations
- List provider connections and health (connected, needs attention, expired, disconnected).
- Start OAuth through backend endpoints; handle redirect/callback state.
- Show granted scopes and selected provider assets/accounts.
- Support selecting among multiple eligible Pages, Instagram accounts, ad accounts or Google accounts when backend supports them.
- Explain why a scope is requested.
- Allow disconnect/revoke where supported.
- Never store access/refresh tokens in local storage, browser state, analytics or logs.

Separate the Kaliza WhatsApp control channel from a customer's own connected business WhatsApp identity. Do not confuse their status or settings.

### 3.8 Business profile
Provide editable business context such as business name, website, location/market, products/services, target customers, goals, brand voice, brand constraints and preferred channels. Use backend schemas and validation. Show save/error states and last-updated metadata. Do not put sensitive credentials in profile fields.

### 3.9 Usage and plan
Show plan, relevant limits, current usage, reset period and explanatory units. Distinguish actual consumption from estimates. If a task is blocked by a quota, show the reason and next available action. Billing/payment state comes from backend; the frontend must not calculate authoritative charges.

### 3.10 Settings
Account/profile, workspace members (if permitted), notification preferences, security/session settings, data export/deletion entry points and accessibility preferences. Sensitive account changes require backend reauthentication/approval as applicable.

## 4. WhatsApp-first interaction design

The web app should consistently explain that WhatsApp is the primary command channel. Provide clear instructions to connect or message the configured Kaliza WhatsApp number, but do not hard-code a phone number in source. Retrieve channel configuration from backend.

Use the web app for:
- Reviewing complex research results.
- Inspecting proposals and documents.
- Comparing data and performance.
- Managing connected accounts/assets.
- Reviewing detailed approval payloads.
- Inspecting task history and errors.

Use WhatsApp for:
- Starting and steering tasks conversationally.
- Quick questions and clarifications.
- Routine progress updates.
- Simple approve/reject interactions where safe and supported.
- Notifications and concise reports.

The two channels must share the same conversation/task state. Avoid duplicate task creation when a user switches channels.

## 5. API integration and client architecture

- Generate or maintain typed API clients from backend OpenAPI/JSON schemas.
- Centralize authentication, base URL, request IDs, error normalization and retries.
- Use backend-provided workspace context; do not treat local state as authorization.
- Use server pagination for large lists.
- Use optimistic updates only for reversible UI state; authoritative task/approval status comes from server.
- Support SSE, WebSocket or polling according to the backend's published event contract.
- Handle reconnect, missed events and stale cache by refetching authoritative state.
- Version API contracts and coordinate changes with backend repository.

Expected API domains: identity/workspaces, services, conversations/messages, tasks/task events, approvals, files, integrations/provider assets, business profile, usage/billing, notifications and protected admin endpoints if admin UI is colocated.

## 6. Interaction states and error handling

Every data-backed screen must implement:
- Initial loading and skeleton state.
- Empty state with a useful next step.
- Success state.
- Inline validation.
- Recoverable error with retry.
- Permission denied.
- Expired session.
- Offline/network interruption.
- Stale data/reconnection.
- Long-running task state.
- Not found/deleted/expired resource.

Never display a success state before the backend confirms the operation. For external actions, distinguish “approval submitted,” “queued,” “provider accepted,” and “verified complete.”

## 7. Design system, accessibility and responsiveness

Create reusable tokens and components for typography, spacing, colors, borders, elevation, buttons, inputs, tables, badges, dialogs, toasts, timelines, status indicators and empty states. Avoid one-off styling for every screen.

Requirements:
- Responsive desktop/tablet/mobile layouts.
- Keyboard navigation and visible focus.
- Semantic landmarks and accessible labels.
- Sufficient contrast and non-color status cues.
- Clear confirmation for destructive actions.
- Respect reduced-motion preferences.
- Internationalization-ready text and date/number formatting.
- Avoid overly dense dashboards; prioritize the next action.

Visual direction should communicate a reliable execution workspace, not a generic AI chat clone. Keep the UI calm, clear and operational.

## 8. Security and privacy

- Treat all client input and URL parameters as untrusted.
- Backend validates all workspace/resource access.
- Never embed secrets or provider tokens in frontend bundles.
- Avoid storing sensitive content in local storage unless specifically justified.
- Use secure cookies/session strategy according to backend auth design.
- Protect against XSS, CSRF, clickjacking and unsafe file previews.
- Sanitize/render user and agent content safely.
- Avoid logging message/document contents in analytics.
- Do not expose chain-of-thought, internal prompts, tool credentials or raw infrastructure details.
- Gate admin routes and operations by backend authorization, not merely hidden navigation.

## 9. Admin interface (if included)

Admin functionality is not part of the ordinary customer workspace. If implemented in this repository, isolate it behind a separate route group and role guard, preferably a separate deployment. Potential views: tenants, subscriptions, usage/cost, task queue, failed jobs, sandbox health, connector health, audit/security events and operational configuration.

Require least privilege, step-up authentication for sensitive operations, explicit confirmation, reason capture and audit logging. Customer data access must be minimized and purpose-limited.

## 10. Suggested application structure

Adapt to the selected frontend framework, preserving feature boundaries:

```text
src/
  app/                 # routing, providers, layouts
  features/
    auth/
    overview/
    conversations/
    tasks/
    approvals/
    documents/
    integrations/
    business-profile/
    usage-billing/
    settings/
    admin/             # isolated/protected if included
  components/
    ui/
    layout/
    data-display/
    feedback/
  lib/
    api/               # generated/typed client
    auth/
    events/
    validation/
    formatting/
  styles/
  types/               # generated shared contracts where possible
  tests/
```

Do not duplicate API types manually if they can be generated from backend schemas.

## 11. Frontend delivery phases

1. App foundation: framework, routing, design tokens, auth integration, API client, error boundaries.
2. Workspace shell: navigation, workspace context, responsive layout and role-aware access.
3. Core visibility: overview, tasks, task detail, conversations.
4. High-value workflows: approvals, documents and notifications/realtime.
5. Business setup: integrations, asset selection and business profile.
6. Commercial/account: usage, plan, billing and settings.
7. Admin surface only when operational needs and access model are defined.
8. Accessibility, security, performance, responsive and end-to-end hardening.

Build against mocked contract fixtures only while backend APIs are being defined; replace fixtures with real API integration and contract tests before release.

## 12. Frontend acceptance criteria

- [ ] WhatsApp is clearly the primary control channel; web remains a supporting workspace.
- [ ] No duplicate business logic, task authority, approval policy or agent orchestration exists in the frontend.
- [ ] All data is scoped to the active authorized workspace by backend.
- [ ] Task state is consistent with WhatsApp and backend.
- [ ] Approval screens show the exact action payload and stale approvals cannot be submitted successfully.
- [ ] Documents are accessed through authorized backend links, not sandbox paths.
- [ ] Integration tokens never enter browser storage or logs.
- [ ] All major screens implement loading, empty, error and permission states.
- [ ] Admin functionality is isolated from customer navigation and access.
- [ ] API contracts are typed/versioned and covered by tests.
- [ ] Mobile and desktop layouts are usable and accessible.
- [ ] No UI indicates an external action succeeded until backend/provider confirmation.

## 13. Implementation agent instructions

Read this specification and the shared master architecture before coding. Propose the framework, route map, component structure and API contract dependencies. Build the smallest useful workspace, not a large chat clone. Prioritize task visibility, approvals, documents and integration management. Coordinate all API/event contracts with the backend repository. Do not invent endpoints, sample live provider credentials, hard-code the WhatsApp number, or claim integration functionality before backend support exists.
