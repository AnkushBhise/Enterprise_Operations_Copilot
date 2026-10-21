# Enterprise Operations Copilot — Requirements

## 1. Purpose

Build an internal multi-agent platform that helps operations, support, and compliance teams investigate cases, retrieve trusted internal information, coordinate specialized workflows, and route sensitive decisions through human approval.

The platform must be runnable, observable, testable, and deployable as one coherent system.

Example requests:

- Investigate why premium customer refund approvals are delayed.
- Summarize the reimbursement policy and flag possible violations.
- Check yesterday's P1 cases and prepare an escalation note.
- Draft a customer response, but send it for manager approval first.

## 2. Objectives

The system must:

- Understand enterprise requests and route them to the correct workflow.
- Retrieve grounded evidence from internal documents.
- Use approved tools and APIs safely.
- Persist session, case, and workflow state.
- Pause for approval on high-risk actions.
- Run long-running work asynchronously.
- Stream progress updates to the user.
- Capture traces, cost, failures, and evaluation signals.
- Start cleanly from containers with a single command.

## 3. Technology Stack

| Layer | Choice | Role |
|---|---|---|
| Orchestration | LangGraph | Stateful graph, routing, pause/resume, durable workflows (Postgres checkpointer) |
| API | FastAPI | Request, streaming, approval, job status, and health endpoints |
| LLM and embeddings | OpenAI API | Reasoning, generation, embeddings (`text-embedding-3-small` by default) |
| Database and vector store | PostgreSQL + PGVector | Embeddings, metadata, memory, cases, approvals, jobs, checkpoints |
| Observability | Langfuse Cloud (Hobby tier) | Traces, latency, tool calls, failures, token usage, cost |
| Frontend | Chainlit | Simple chat box with streamed progress and approve/reject buttons; no custom frontend |
| Infrastructure as code | AWS CDK (Python) | Every AWS resource created through code, never the console |
| Identity | AWS Cognito (Keycloak as documented backup) | OAuth/OIDC login, roles as groups, machine identities |
| Tool gateway | Own thin FastAPI service, tools as an MCP server | Single point for authorization, validation, resilience, and audit of every tool call and retrieval |
| Packaging | Docker + Docker Compose | Full stack from one entrypoint |
| Redis | Redis | Rate-limit and budget counters, shared circuit-breaker state |
| Optional | Hugging Face datasets | Loading BANKING77 |
| Production option | Microsoft Presidio | Rules-first PII detection in front of the LLM guard classifier; evaluated before production, not used in the prototype |

PostgreSQL and PGVector run in a container (`pgvector/pgvector` image) so the stack stays portable.

## 4. Agents

All agents are coordinated through a single LangGraph graph. Each boundary must be explicit and observable (own trace span).

| Agent | Responsibility |
|---|---|
| Supervisor | Classifies the request, selects the path, decides whether retrieval, tools, approval, or background execution is needed |
| Retrieval | Searches internal knowledge, applies metadata filters, returns evidence with sources |
| Memory | Maintains conversation summary, case memory, extracted entities, workflow state |
| Tool Execution | Calls approved tools with schema validation, retries, structured outputs |
| Compliance | Detects unsafe content, prompt injection, PII, policy violations, restricted actions; blocks or redacts before action proceeds |
| Approval | Pauses high-risk actions, records approval state, resumes after a decision |
| Response Composer | Builds the grounded answer, summary, recommended action, or draft; separates evidence from inference |

## 5. Workflows

1. **Knowledge-grounded investigation.** Retrieve documents, combine with the request, return an answer with cited sources.
2. **Tool-assisted investigation.** Combine retrieved knowledge with tool output; structured inputs and outputs; retries and failure reporting.
3. **Approval-gated action.** Detect a high-risk action, pause, persist the pending action, resume only on approval; handle rejection and expiry.
4. **Background job.** Long-running work (batch ticket triage, document review, compliance scan) runs outside the request cycle, with progress and failure details.
5. **Guardrailed path.** A blocked, redacted, or restricted scenario handled end to end.

## 6. Data

### 6.1 Routing dataset — BANKING77

- Load with `load_dataset("PolyAI/banking77")`; convert train and test splits to structured records.
- Normalize to: `query_id, user_query, intent_label, priority, department, risk_level`, plus `expected_path`.
- Each query is wrapped as an employee request (for example "Customer on ticket T-1042 says: '...'. Help me handle this."), because the copilot's users are employees and the dataset sentences are in a customer's voice.
- Priority, department, and risk level are not in the source data. Define an explicit mapping from the 77 intents and record it as a versioned file.
- Used for routing tests, intent classification, and supervisor benchmarking.

### 6.2 Enterprise email corpus — Enron

- Parse each email: subject, sender, recipients, date, folder path, body.
- Remove duplicates and low-value records; build a combined content field.
- Add metadata: `document_type`, `department`, `sensitivity`, `source_path`.
- Used for communication search, summarization, incident reconstruction, and retrieval experiments.

### 6.3 Synthetic enterprise documents

- At least 25 documents (Markdown, text, or JSON): refund policy, reimbursement policy, escalation SOP, approval matrix, PII guidelines, incident runbook, org directory, and others.
- Include text, tables, and varied structure so the ingestion pipeline is exercised on realistic formats.
- Maintain 2–3 versions of selected documents to verify how updates are handled.
- Include at least one document containing an injected instruction, for guardrail testing.

### 6.4 Ingestion pipeline

Repeatable (script or CLI), idempotent, and runnable inside the stack:

1. Load and clean.
2. Normalize and extract metadata.
3. Chunk (about 400–800 tokens / 1,000–2,000 characters) with overlap.
4. Embed with OpenAI.
5. Store in PostgreSQL / PGVector.

Document updates: detect changes by content hash; new version supersedes the old one (`version`, `is_current`); superseded chunks are excluded from retrieval but retained for audit.

Every chunk keeps metadata for filtering, explanation, and audit (source, version, type, department, sensitivity).

## 7. Storage

Minimum schema:

- `documents` — source-level records and version info.
- `document_chunks` — `chunk_text`, metadata, embedding vector.
- `user_profiles` — team and assignment data keyed by Cognito `sub` (identities and roles live in Cognito).
- `sessions` — session identity, owner, conversation summary.
- `cases` — case state and extracted entities.
- `approvals` — pending action, status (pending / approved / rejected / expired / executed / execution_failed), decider, timestamps.
- `jobs` and `job_events` — background job state, queue, append-only progress events, errors.
- `tool_calls` — full tool inputs and outputs, referenced from graph state.
- `drafts` — drafted customer and internal messages.
- `audit_log` — append-only record of actions and decisions.
- LangGraph checkpoint tables.
- Mock enterprise data (tickets, ledger, org directory) and Chainlit history in separate schemas.

Short-term memory is the conversation and checkpointed graph state per `thread_id`. Long-term memory is case history, summaries, and extracted entities per user and case.

## 8. Tools

Six approved tools, with schema-validated inputs and structured, recoverable outputs:

1. Ticket lookup.
2. Transaction lookup (orders, payments, refunds).
3. Policy lookup.
4. Organization lookup.
5. Approval submission (performed by the Approval agent, not chosen by the LLM).
6. Notification / draft creation.

Approved actions (refund, send communication, policy exception, account flag) run through internal executors that the LLM cannot call.

Requirements: retries with backoff, explicit failure reporting, graceful handling of unavailable services, no unrestricted tool access (allow-list per workflow).

## 9. Async, Approval, and Streaming

- A background worker executes long-running jobs; the API returns a `job_id` immediately.
- Job state and progress persist in the database and survive restarts.
- High-risk actions pause the graph; the pending action and approval state are persisted.
- Approval decisions (approve, reject, expire) are recorded with decider and time, and resume the workflow accordingly.
- Streaming (SSE) covers: request classification, retrieval, tool execution, composing, waiting for approval, background progress, and completion. The answer is sent after the output guard passes (sentence by sentence), not as raw tokens, so unchecked content never reaches the user.

## 10. Guardrails and Security

- Input validation.
- Prompt-injection detection and masking.
- PII detection and masking.
- Output validation.
- Restricted-action handling.
- Policy and compliance checks before any action proceeds.
- Retrieved documents and tool responses treated as untrusted input.
- Audit-friendly logs and evidence traces.
- Identity: users log in through AWS Cognito; the API verifies the access token and takes identity and roles only from it. The copilot acts as the user: every tool call and retrieval is authorized as that user by the tool gateway.
- A deployment design for AWS (managed guardrails, PII services, secrets management, network controls) is documented in the architecture write-up.

## 11. Observability

Captured and inspectable: model calls, retrieval steps, tool calls and failures, latency, retries, token usage, estimated cost, approval and workflow state, evaluation outcomes. A reviewer must be able to see what the system did, why, and where it failed.

## 12. Testing and Evaluation

- **Unit tests:** parsers, tools, validation logic, helpers.
- **Integration tests:** end-to-end retrieval, tool usage, approval pause/resume, background job.
- **Evaluation suite:** at least 20 scenarios covering routing correctness, retrieval usefulness, groundedness, action safety, approval behavior, tool integration, pause/resume, and failure recovery.
- Routing accuracy is measured against the BANKING77-derived dataset.
- Results are tracked per run and failures are analyzed.

## 13. Deployment

- `Dockerfile` and `docker-compose.yml`; `docker compose up --build` brings up the stack.
- Services: app (API), UI (Chainlit), worker, tool gateway, MCP tool server, mock backend, PostgreSQL with PGVector, Redis. Langfuse and Cognito are cloud services; their settings are supplied in `.env`.
- Health and readiness endpoints; documented verification steps.
- `.env.example` with all configuration; no manual wiring beyond supplying secrets.

## 14. Deliverables

- Repository with source, Docker configuration, `.env.example`, tests, and README (setup, run, readiness checks).
- Architecture diagram: graph, agents, tools, database, vector store, observability.
- Data preparation pipeline.
- Working application implementing all workflows.
- Technical write-up: use cases, trade-offs, risks, improvement plan.
- Evaluation report: scenarios, outcomes, failure analysis, next steps.
- Recorded demo of the core flows, including a blocked or redacted scenario.

## 15. Non-Goals

- Replacing production identity, access control, or audit systems.
- Supporting every enterprise tool or workflow.
- Fully autonomous decision-making; high-risk actions always require human review.
- A general-purpose enterprise search platform.

## 16. Open Questions

1. Frontend tool: decided, Chainlit (see design.md 2.1).
2. Background execution: decided, Postgres job table plus worker (see design.md 4.9).
3. Langfuse: decided, Langfuse Cloud (see design.md 4.11).
4. Authentication placement: decided. The API verifies the user's token (inbound); the tool gateway authorizes every tool call and retrieval as the user (see design.md section 5).
5. Mapping of BANKING77 intents to department, priority, and risk level.

## 17. Work Plan

Design (keep each design doc short; revise after the thin slice runs)

- [x] List the work in priority order.
- [x] Define the use case and high- and low-level objectives.
- [x] Choose a ready-made chat UI and define the minimal user experience (see section 18).
- [ ] Decide identity handling (`user_id` scoping, where auth lives).
- [ ] Define workflows, agents, tool contracts (schemas, failure modes, allow-lists), and the full graph on paper.
- [ ] Define memory storage (short-term versus long-term, schema) and knowledge base storage (schema).
- [ ] Define the ingestion pipeline and document-update handling.
- [ ] Define the HITL flow and which scenarios run asynchronously.
- [ ] Define guardrails and their AWS deployment design.
- [ ] Define the observability strategy and trial Langfuse.
- [ ] Define the evaluation scenarios (20+) with expected routes and outcomes.
- [ ] Produce high- and low-level architecture diagrams.
- [ ] Review the architecture for pitfalls.

Data (start early, in parallel with design)

- [ ] Write the 25+ synthetic documents, including versions, tables, and an injected-instruction document.
- [ ] Define the BANKING77 intent-to-department/priority/risk mapping.
- [ ] Download and parse the Enron corpus.

Build

- [ ] Compose baseline: app, UI, worker, mock backend, PostgreSQL / PGVector, Redis, health endpoints, `.env.example` (Langfuse Cloud keys).
- [ ] Thin end-to-end slice: stubbed graph, one request through API, graph, database, and traces.
- [ ] Ingestion pipeline; test update handling.
- [ ] Retrieval with metadata filters and evidence tracing.
- [ ] Memory layer and checkpointer.
- [ ] The five approved tools with validation, retries, and failure reporting.
- [ ] Supervisor routing and the remaining agents.
- [ ] Async execution with job progress.
- [ ] Approval gate (approve, reject, expire; pause/resume).
- [ ] Guardrails (input, PII, injection, output, restricted actions).
- [ ] Streaming endpoints covering all stages.

Verify and deliver

- [ ] Unit and integration tests.
- [ ] Evaluation harness, runs, and failure analysis.
- [ ] Full `docker compose up --build` verification from a clean checkout.
- [ ] README, technical write-up, evaluation report, demo.

## 18. User Interface

The UI is a means to exercise the system, not a deliverable in itself. Use an off-the-shelf chat UI that talks to the FastAPI backend; do not build a custom frontend.

- A chat box with streamed responses.
- Progress messages shown as they arrive (classification, retrieval, tool calls, waiting for approval, background job status).
- Managers open pending approvals with an `/approvals` command and decide with Approve and Reject buttons.
- A way to see background job status.
- Runs as one more container in Compose.
