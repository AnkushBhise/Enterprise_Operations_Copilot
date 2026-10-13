# Enterprise Operations Copilot — Design

Design decisions are recorded here one topic at a time, after being refined in discussion.

## 1. Use Case

### 1.1 Company

**Company X** is a mid-size online marketplace (about 2,000 employees). Customers buy from sellers, and the company handles payments, refunds, and disputes itself. It has a paid premium membership with priority service levels.

### 1.2 Purpose

A single internal copilot that lets support, compliance, operations, and manager roles:

- investigate cases and answer questions with evidence from internal documents,
- combine that evidence with data from internal systems,
- draft responses, notes, and actions,
- route high-risk actions to a manager for approval,
- run long batch work in the background.

The copilot is an **advisor and drafter**. It acts on its own only through approved low-risk tools. Anything high-risk waits for a human decision.

### 1.3 Roles

| Role | Typical work | Approver? |
|---|---|---|
| Support agent | Case investigation, customer replies, escalation notes | No |
| Compliance analyst | Policy checks, PII review, scans, fraud review | No |
| Operations specialist | Approval delays, triage, incidents, ownership lookup | No |
| Manager | Reviews and decides on high-risk actions | Yes |

Roles determine which tools and data a user can reach. Only managers (within the limits of the approval matrix) can approve.

### 1.4 Scenarios

**Support**

- Refund status investigation.
- Duplicate or wrong charge dispute.
- Draft a customer reply.
- Ticket summary and escalation note.

**Compliance**

- Policy violation check.
- PII and data-privacy review.
- Bulk compliance scan (background).
- Suspicious activity / fraud flag.

**Operations**

- Delayed approval investigation.
- Batch ticket triage (background).
- Incident runbook lookup.
- Organization and owner lookup.

**Cross-team core flows**

1. **Delayed premium refund.** Support investigates, operations locates the stuck approval, compliance checks policy, a manager approves a refund or exception.
2. **Dispute to fraud flag.** A duplicate-charge dispute reveals a suspicious pattern; compliance reviews; flagging the account needs approval.
3. **Daily P1 triage to escalation.** A background batch triages cases and prepares escalation notes; high-risk items go to managers.

### 1.5 High-risk actions (manager approval)

- Issue a refund above the threshold.
- Send customer communication.
- Grant a policy exception.
- Escalate or flag an account.

### 1.6 Business rules

**Refund approval ladder (USD)**

| Amount | Handling |
|---|---|
| Under 50 | Within standard policy; no approval |
| 50 to 500 | Team lead approval |
| Over 500 | Manager approval |

**Premium membership.** Paid membership. Refund SLA of 24 hours versus 5 business days for standard customers, plus priority support routing.

**Ticket priority and SLA**

| Priority | Meaning | Response SLA |
|---|---|---|
| P1 | Critical (payment outage, fraud) | 1 hour |
| P2 | High | 4 hours |
| P3 | Normal | 1 business day |
| P4 | Low | 3 business days |

**Fraud / suspicious-activity signals (mock data)**

- Repeated duplicate-charge disputes from the same customer in a short window.
- Refund velocity (unusually many or large refunds on one account or seller).
- Identity details in a ticket that do not match the customer record.

### 1.7 Systems and tools

The copilot reaches company data through approved tools. The systems behind them are mock services in one container, seeded reproducibly, and exposed over HTTP so tool calls, retries, and failures are real.

| Tool | Backing system | Data |
|---|---|---|
| Ticket lookup | Ticketing system | customers, tickets, ticket_events |
| Transaction lookup | Orders / payments / refunds ledger | orders, payments, refunds |
| Policy lookup | Knowledge base | documents, document_chunks, structured thresholds |
| Org lookup | Org directory and approval matrix | employees, org_units, approval_matrix |
| Approval submission | Copilot's own approvals store | approvals |
| Notification / draft | Notification service | drafts, outbox (send only after approval) |

The handout's five tools plus transaction lookup, which covers the refund, dispute, and fraud flows.

The mock data and the policy documents must state the same facts (premium definition, thresholds, SLAs), and the data must contain the situations the core flows need: stuck premium refunds, a duplicate-charge pattern, a P1 batch, an approval matrix with real thresholds, and a draft that contains PII.

### 1.8 Data sources

- **BANKING77:** routing and supervisor evaluation. Keep about 30 intents that fit an e-commerce company (payments, refunds, charges, disputes, fraud, identity). Intents that don't fit become out-of-scope routing tests.
- **Enron emails:** role to be decided during data pipeline design (candidate: historical communications archive for search, incident reconstruction, and compliance scans).
- **Synthetic enterprise documents:** at least 25 Company X documents (policies, SOPs, runbooks, approval matrix, privacy guidance, org directory), with versions.

### 1.9 High-level objectives

1. Route each request to the correct workflow.
2. Give answers grounded in cited sources, separating evidence from inference.
3. Use tools safely, within the user's role.
4. Never run a high-risk action without a recorded approval.
5. Run long work in the background with visible progress.
6. Stream progress and results to the user.
7. Make every decision traceable: what was done, why, and where it failed.
8. Save investigation time for support, compliance, and operations staff.

### 1.10 Low-level objectives

Targets are proposals to be confirmed during evaluation design.

| Area | Objective |
|---|---|
| Routing | Correct workflow selected on the BANKING77-derived set and the scenario suite; out-of-scope requests detected |
| Grounding | Every factual claim in an answer maps to a retrieved source; unsupported claims are labeled as inference |
| Tools | Inputs and outputs schema-validated; transient failures retried; permanent failures reported in a structured way |
| Approvals | 100% of high-risk actions gated; approve, reject, and expire all handled; workflow resumes only after a valid decision |
| Async | Background jobs survive restarts, report progress, and return failure details |
| Guardrails | Prompt injection, PII, and restricted actions detected; blocked or redacted before any action |
| Observability | Every request has a trace with model calls, retrieval, tools, latency, tokens, and cost |
| Time saved | Investigation steps that normally involve several systems complete in one request |

### 1.11 Out of scope

- Replacing production identity, access control, or audit systems.
- Fully autonomous decisions on high-risk actions.
- A general-purpose enterprise search product.
- Real integrations with external systems (all backing systems are mock).

### 1.12 Open items

- Confirm that refunds under 50 execute without approval, within the advisor-and-drafter model (which tool performs the refund, and does the copilot call it directly).
- Decide the role of the Enron corpus (data pipeline design).
- Confirm the intent subset and the mapping of BANKING77 intents to department, priority, and risk.
- Define how roles are assigned to users (identity decision).

## 2. User Interface and Authentication

### 2.1 Approach

A ready-made chat UI, with no custom frontend. The UI exists to exercise the platform; the FastAPI backend is the product.

**Tool: Chainlit.** It provides a chat box with streamed output and visible intermediate steps, a sidebar of past conversations, a login screen, and action buttons, all without frontend code.

### 2.2 Screen

- **Chat area:** text input and streamed responses. Progress steps appear as they happen (classification, retrieval, tool calls, waiting for approval, completion).
- **Left sidebar:** past conversations, each tied to a session or case.
- **Pending approvals:** a manager sees pending items with approve and reject buttons, inline in the chat.
- **Background jobs:** running and finished jobs appear as progress messages in the conversation.
- **Identity:** the logged-in user's name and role are shown.

### 2.3 Authentication

**Password login against a seeded `users` table.**

- Four demo users, one per role (support, compliance, operations, manager), with hashed passwords and a stored role.
- Chainlit's password callback validates credentials against Postgres.
- The role decides which tools and data the user can reach, and whether they can approve.
- Seeded users make role switching easy, which is needed to demonstrate permissions and approvals.

**Why not OAuth now:** it needs an external OAuth app, client secrets, internet access, and a real account per demo user, which conflicts with a one-command start. The API only receives `user_id` and `role`, so OAuth or SSO can replace the login callback later without touching the backend.

### 2.4 UI to API trust

- The UI runs as its own container and calls FastAPI over HTTP and SSE.
- After login, the UI calls the API with a shared secret plus `user_id` and `role` headers. The API accepts only calls that carry the secret.
- This is not production identity; the production design (signed tokens issued by an identity provider) is described in the write-up.

### 2.5 Conversation history

- Chainlit stores conversation history in the same PostgreSQL instance, in a separate schema.
- A Chainlit thread maps one-to-one to a `sessions` row and the LangGraph checkpoint `thread_id`, so history is not kept twice.

### 2.6 Pending approvals

The approver is a different person from the requester, so approvals cannot depend on a button inside the requester's conversation. Approvals live in the database and are exposed by the API (`GET /approvals?status=pending`, `POST /approvals/{id}/decision`).

- **Manager view:** a `/approvals` command lists pending items, each with Approve and Reject buttons and the evidence needed to decide. At login (and on opening a chat), a message such as "You have N pending approvals" links to the same list.
- **Requester view:** the conversation shows "waiting for approval" and is updated with the decision when it arrives.
- **Fallback:** Chainlit's left sidebar is not extensible to my knowledge. A short spike confirms what is possible; if `/approvals` or its buttons hit a limit, approvals appear inline in the manager's open chat.

### 2.7 Conversation management

Past conversations are listed and reopened only. Delete and rename are not supported in this phase.

### 2.8 Job progress and returning users

- Each job writes an append-only `job_events` stream (sequence number, stage, message, timestamp). The job's final result and any failure details are stored on the job record.
- The stream endpoint accepts "events after N": it replays missed events, then continues live.
- When a past conversation is opened, the UI lists the jobs linked to its thread, re-attaches to running ones, and renders results of finished ones.
- A `/jobs` command lists the user's recent jobs and their status.
- The worker cannot push into a conversation the user is not viewing, so results are delivered on the next open.

### 2.9 Decision notification

When a manager decides, the workflow resumes immediately. The requester sees the decision live if the conversation is open, otherwise when it is next opened. Out-of-band notification (email, push) is out of scope for this phase.
