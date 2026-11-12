# Kaliza Frontend Blueprint

**Status:** Starting product/interface specification  
**Product:** Kaliza  
**Repository:** `Aimable2002/kaliza_UI`

## 1. Frontend's role

Kaliza is WhatsApp-first. WhatsApp is the main interface for instructions, conversation, task updates, clarifications, and approvals. The web frontend must therefore stay focused and should only own experiences that are difficult, inefficient, or unsuitable to complete in WhatsApp.

Do not rebuild WhatsApp as a large web chat product. A lightweight task/activity view may provide context, but the web app is not a second primary control plane.

## 2. What belongs in the web app

### Essential areas
1. **Authentication and workspace onboarding** — account access, workspace creation, profile, and initial setup.
2. **Connections and permissions** — connect external services, configure credentials through secure backend flows, inspect connection health and granted permissions.
3. **Business/service configuration** — configure business profile, goals, preferences, and enabled services.
4. **Work visibility** — see active, queued, completed, failed, and paused tasks; inspect task steps, status, timestamps, and relevant logs/summaries.
5. **Reports and analytics** — detailed visual reports, trends, downloadable summaries, and historical performance that are cumbersome in chat.
6. **Files and artifacts** — browse, preview, organize, and download user-provided files and agent-generated outputs.
7. **Approvals when visual review is useful** — provide a clear review surface for complex or high-impact actions, while keeping WhatsApp approval supported where appropriate.
8. **Account administration** — team access, security, billing, usage, and account settings as they enter scope.

### Not a frontend priority
- A full duplicate agent chat interface.
- Recreating every WhatsApp interaction as a dashboard workflow.
- Large social-media-style navigation or pages unless validated by the actual first-release service scope.
- Business logic, orchestration, integrations, or secret handling in the browser.

## 3. Initial information architecture

Keep navigation small and task-oriented. A reasonable starting structure is:

- **Home** — workspace status, key outcomes, current work, attention needed.
- **Work** — task/workflow list and detailed execution history.
- **Reports** — detailed reports and analytics.
- **Files** — source documents and generated artifacts.
- **Connections** — connected services, permissions, and health.
- **Settings** — workspace, profile, team, security, usage, and billing.

Onboarding can be a guided flow rather than a permanent top-level section. The final navigation should follow validated first-release use cases.

## 4. Core interaction principles

- WhatsApp remains the default place to tell Kaliza what to do.
- Every task shown on the web should correspond to the same backend task record used by WhatsApp.
- Provide clear task states: queued, planning, running, waiting for input/approval, completed, failed, and cancelled.
- Show what Kaliza is doing, what it needs, what changed, and what the user can do next.
- Keep approval language specific: action, target, consequences, and approve/reject choices.
- Make errors understandable and show whether retry or recovery is underway.
- Design responsive layouts, accessible controls, and useful loading, empty, and error states.

## 5. Technical direction

The earlier frontend blueprint proposed Next.js App Router, TypeScript, and Tailwind CSS. These remain candidate choices, not an implementation constraint until confirmed against the new project setup.

Frontend responsibilities:
- Render UI and collect user input.
- Call backend APIs through a typed client/service layer.
- Subscribe to or poll task/event updates using the backend's chosen mechanism.
- Never contain provider secrets or directly execute privileged integrations.
- Keep mock/demo data isolated and replaceable; do not let mock behavior leak into production paths.

## 6. Shared backend contract

The frontend should consume backend-owned records for:
- User and workspace profile.
- Connected integrations and permission health.
- Tasks, task events, progress, and results.
- Approval requests and decisions.
- Reports, metrics, and artifacts.
- Usage, billing, and team access.

The web app and WhatsApp must show consistent state because they use the same backend records.

## 7. Suggested implementation sequence

1. Establish app shell, routing, design system, linting, type checks, and tests.
2. Implement authentication and workspace onboarding.
3. Implement Connections and secure integration setup flows.
4. Implement Work list and task detail views against the shared task API.
5. Implement approval review surfaces.
6. Implement Files/artifact browsing and previews.
7. Implement Reports and analytics.
8. Add team, usage, billing, and advanced settings when required by release scope.
9. Verify responsive behavior, accessibility, empty/error/loading states, and end-to-end API integration.

## 8. Acceptance criteria for the first useful frontend milestone

A user can sign in, complete initial workspace setup, connect required services, see current and historical work created through WhatsApp, inspect task outcomes and generated files, and review detailed reports. The web app does not require users to switch away from WhatsApp for routine instructions or ordinary task control.
