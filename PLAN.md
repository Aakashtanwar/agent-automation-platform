# Blueprint + MVP Roadmap: A World-Class No/Low-Code Agentic Workflow Platform

## Context

The goal is to build a platform companies use to visually assemble workflows in which **multiple dedicated AI agents perform a series of tasks to achieve a goal**, including calling MCP servers and external APIs — aimed at **non-technical business users** (no/low-code).

Research (market + architecture + UX, 2025–2026) surfaced one decisive insight: this segment is **large and fast-growing (~$7–11B, ~45–50% CAGR) yet structurally underserved**. Every powerful incumbent (n8n, LangGraph, CrewAI, Copilot Studio, Agentforce, Workato) is really a developer/IT tool wearing a "no-code" label; every genuinely non-technical tool (Lindy, MindStudio, Cassidy, Dust) is weak on true visual multi-agent modeling, MCP, observability/debugging, or pricing transparency. Gartner predicts **>40% of agentic AI projects are canceled by end of 2027** — and the four failure modes driving that (reliability, opaque debugging, surprise cost, trust) are *exactly* the non-technical-user problems. The tooling that solves them (durable execution, tracing, evals, guardrails) exists today **only for developers.** That is the wedge.

**Locked decisions:** (1) Deliverable = full blueprint + MVP roadmap. (2) GTM = **horizontal, reliability-first** — general-purpose builder differentiated on reliability, plain-English observability, capped/transparent pricing, and first-class human-in-the-loop. (3) **Architect for enterprise governance from day one, but launch self-serve** (SMB/mid-market cloud SaaS first; enterprise controls designed-in, enabled progressively).

---

## 1. Positioning & Differentiation

**One-line positioning:** *"The agent workflow platform your business team can actually trust — build multi-agent workflows in plain English, see exactly what every agent did, and never get a surprise bill."*

The five differentiators are the **core product**, not add-ons (every competitor treats them as afterthoughts):

1. **Reliability as the product.** Deterministic graph backbone with autonomous agents as *bounded* nodes; durable execution underneath so runs survive crashes and resume. Counter to the "capability over reliability" arms race users are tired of.
2. **Plain-English observability & debugging.** "Show your work" trace UX in business language — what each agent decided and *why*, which tool it called, what it cost — not developer logs. (Best fragments today: Vellum traces, Gumloop cost UX; nobody has assembled it for non-technical users.)
3. **Transparent, capped pricing.** Per-run cost *estimate before you run*, live cost badge, hard per-workflow/tenant budget caps that pause-and-ask. Directly counters the near-universal credit-opacity complaint (Lindy/Make/Relevance) and pricing cliffs (Beam $50→$3,990, Vellum $0→$500).
4. **Human-in-the-loop as a first-class, easy step.** Draft modes, one-click approvals via Slack/email, "start safe → grow autonomy" default. (38% of teams cite HITL as their #1 agent-management approach; only 20% run fully autonomous.)
5. **Zero-setup MCP + vertical templates.** "Connect [App]" buttons (no terminal/JSON), plus job-to-be-done templates so first value lands in minutes, not hours.

**Reference verdicts to beat:** activation → Lindy/Gumloop; guardrails → Zapier Agents; cost transparency → Gumloop/Vellum; "what did the agent decide" → Vellum; enterprise governance → Copilot Studio.

---

## 2. Product / UX Blueprint (non-technical-first)

Core UX principles the build must nail (each maps to a research-validated winning pattern):

- **Lead with conversation, land on structure.** User describes the outcome in plain English → system generates an *editable, inspectable* visual workflow. Conversation drives activation; the visible graph builds trust. (Lindy 3.0, Gumloop "Gummie", Zapier Copilot are references.)
- **Model agents the way businesses hire people** — **role / goal / instructions** (CrewAI mental model) as fill-in-the-blank fields with intent expansion, never a raw prompt box.
- **Tools/MCP as a guided toggle**, not config. "If you can set up a Zap, you can connect a tool."
- **"Show your work" trust engine** — visible tool calls, sources/citations, step traces; surface *all* evaluated options, not just the chosen path.
- **Free, zero-anxiety test/preview** — sandbox runs on sample data, single-node re-run, replay-from-prior-run.
- **Safe-by-default guardrails, invisible until needed** — per-run step caps, spend caps with 75/90% alerts, loop termination, bounded retries — all on by default.
- **JTBD template gallery + onboarding credits + form-based config** so a colleague can run an agent without ever opening the builder.

**Visual builder = a graph of nodes.** Node palette exposes Anthropic's five workflow patterns as first-class primitives plus agent/tool/control nodes:
`Prompt-chain · Router · Parallelize · Orchestrator-workers · Evaluator-optimizer · Agent(ReAct) · Tool/MCP · Human-in-the-loop · Context-compression · Knowledge(RAG)`. Advanced mode gates Swarm/Network patterns.

---

## 3. Reference Architecture

**Cross-cutting thesis:** *Deterministic graph backbone (reliability, auditability, learnable UX) + autonomous agents as bounded, guardrailed nodes + a durable-execution substrate underneath. Buy the integration/sandbox layers; own the graph engine and model gateway; stay standards-based (MCP spec, A2A, OTel GenAI, OWASP LLM Top 10) throughout.*

```
┌──────────────────────────────────────────────────────────────────────────┐
│ L1 EXPERIENCE — Visual builder (canvas = graph). NL→workflow generation.   │
│    Node palette (5 patterns + Agent/Tool/HITL/Context/RAG). "Connect App"  │
│    OAuth buttons. Per-run cost estimate + live cost badge. Plain-English    │
│    trace viewer. JTBD template gallery. Form-based end-user config.         │
├──────────────────────────────────────────────────────────────────────────┤
│ L2 CONTROL PLANE — compile canvas → data-driven graph spec. Versioning ·   │
│    eval-gate before publish · admin publish approval · ReBAC (users+agents) │
├──────────────────────────────────────────────────────────────────────────┤
│ L3 DURABLE EXECUTION SUBSTRATE — Temporal (or Restate). Graph interpreter   │
│    (graph=data). One durable session per run. Journaled LLM/MCP/API steps · │
│    idempotency keys · retries/backoff · HITL waits (days) · budget ceilings │
├──────────────────────────────────────────────────────────────────────────┤
│ L4 AGENT RUNTIME & MODEL LAYER — LiteLLM gateway (own the data path) ·      │
│    auto prompt-caching · context engine (compaction/notes/sub-agents) ·     │
│    Mem0 over pgvector · Pydantic+Instructor typed node I/O · RAG-as-tool     │
├──────────────────────────────────────────────────────────────────────────┤
│ L5 TOOL / INTEGRATION LAYER — native MCP client (Streamable HTTP, OAuth 2.1,│
│    SSRF-hardened) + Composio/Pipedream broker for instant catalog · Tool    │
│    Registry + RAG-over-tools (meta-tools) · namespacing · MCP security scan  │
├──────────────────────────────────────────────────────────────────────────┤
│ L6 EXECUTION SANDBOX POOL — E2B/Daytona (Firecracker), egress allow-lists.  │
│    Secrets injected at creation, NEVER into LLM context (reference tokens).  │
├──────────────────────────────────────────────────────────────────────────┤
│ L7 CROSS-CUTTING — Observability: OTel GenAI → Langfuse | Evals: DeepEval/  │
│    Promptfoo + online | Guardrails: NeMo (+Prompt Guard 2/Llama Guard 4/    │
│    Presidio) | Secrets: KMS envelope + Vault | Tenancy: Postgres RLS bridge │
│    + silo tier | Cost: per-tenant token accounting + hard budget/rate limits │
└──────────────────────────────────────────────────────────────────────────┘
```

**Key architectural decisions & rationale:**

| Area | Decision | Why |
|---|---|---|
| Execution model | **Cyclic graph / state-machine** (LangGraph-style), agent-in-a-node hybrid | DAGs can't loop; pure agent loops compound errors & cost ~10×. Canvas *is* a graph → per-node test, isolated errors, audit trail. Default to determinism; opt into autonomy node-by-node. |
| Durable substrate | **Temporal** for v1 (Restate as the strong alternative) | Checkpointing (LangGraph/CrewAI) ≠ durability. Never build from scratch. Temporal = most battle-tested (accept determinism-versioning tax by compiling graph→**data-driven interpreter** so edits change *data*, not replayed code). Restate avoids the versioning tax + Virtual-Object-per-session — revisit if versioning ops hurt. |
| Multi-agent | Supervisor + Sequential as defaults; Swarm/Network gated "advanced" | #1 failure mode is **context fragmentation** (Cognition + Anthropic agree). Auto-propagate upstream traces + offer a context-compression node. |
| Protocols | **MCP** for tools (down), **A2A** for external agents (across) | Two clearly separated lanes; both now industry standards. MCP spec 2025-06-18 (Streamable HTTP, OAuth 2.1 + RFC 8707 audience binding). |
| Model layer | **LiteLLM** self-hosted gateway; Anthropic/OpenAI/Bedrock/Vertex backends | Own the data path for multi-tenant budgets/guardrails; layered fallback; **gateway must forward `cache_control` untouched** (prompt caching is the biggest cost lever, ~10× on stable prefixes). Default to latest Claude models (Opus 4.8 / Sonnet 4.6 / Haiku 4.5) with routing by cost/latency. |
| Integrations | **Buy** OAuth broker (Composio/Pipedream) for instant breadth; **build** native MCP client for custom servers | Instant catalog + safe per-tenant OAuth now; own the MCP path long-term. Tool Registry + RAG-over-tools (meta-tools) to scale past thousands of tools. |
| Secrets | KMS envelope encryption + Vault; **secrets reach the sandbox, never the LLM context** | OWASP LLM02 — reference-token placeholders (`{{secret:x}}`) resolved by a trusted proxy after the LLM emits the call. |
| Tenancy | **Bridge**: Postgres RLS pool for standard tenants + **silo tier** (dedicated DB/VPC/keys) for regulated | Enterprise-ready from day one without siloing everyone. Per-transaction tenant context under PgBouncer. |
| Authz | **ReBAC** (OpenFGA self-host / Permit.io managed) | Nested/shared multi-tenant workflow resources; agent effective permission = **intersection(user rights, agent capabilities)** (confused-deputy defense). |
| Sandbox | Managed **E2B/Daytona** (Firecracker) + deny-by-default egress | Untrusted tool/code execution isolation without self-hosting Firecracker early. |
| Observability | **OTel GenAI** wire format → **Langfuse** (self-host) | Backend-agnostic; agent/tool span trees map to the graph; native token/cost. |
| Guardrails | **NeMo Guardrails** + Prompt Guard 2 / Llama Guard 4 / Presidio, non-bypassable orchestrator step | Anchor to OWASP LLM Top 10 2025 (injection, excessive agency, unbounded consumption). |
| Evals | **DeepEval/Promptfoo** in CI + Langfuse online evals; **eval-gate before publish** | "Test your agent before you trust it" — unbuilt no-code whitespace. |

---

## 4. Recommended Tech Stack (concrete)

- **Frontend / builder:** TypeScript + React; **React Flow (@xyflow/react)** for the node canvas; Next.js app shell; Tailwind + shadcn/ui.
- **Backend / control plane:** Python (FastAPI) for agent/LLM services; **Temporal** (Python/TS SDK) for the durable substrate; Postgres (+ RLS) as system-of-record; **pgvector** for memory/RAG v1; Redis for queues/cache.
- **Agent/model:** **LiteLLM** gateway; **Pydantic + Instructor** for typed node I/O; **Mem0** for memory; LangGraph *patterns* as the mental model (engine compiled to data-driven interpreter over Temporal).
- **Integrations:** **Composio** (or Pipedream Connect) broker + native MCP client (Streamable HTTP + OAuth 2.1).
- **Sandbox:** **E2B** (or Daytona) managed Firecracker sandboxes.
- **Cross-cutting:** OpenTelemetry GenAI → **Langfuse**; **NeMo Guardrails**; **OpenFGA** (ReBAC); **HashiCorp Vault** + cloud KMS.
- **Infra:** Kubernetes with **KEDA/HPA on queue depth**; tier work by duration (fast inline → queue+worker → durable workflow).

*(Stack is a strong default under "no constraint"; each choice has a named alternative in §3 if a team preference emerges.)*

---

## 5. Phased Build Roadmap

**Phase 0 — Foundations & spikes (validate the risky core).**
De-risk the three hardest bets before UI polish: (a) compile a JSON graph spec → Temporal-executed **data-driven interpreter** with journaled LLM/MCP/API steps and resumable HITL waits; (b) native **MCP client** connecting to 2–3 real servers with OAuth; (c) **per-run cost accounting + hard budget cap** wired to LiteLLM span tokens. Stand up Postgres+RLS tenancy skeleton and OTel→Langfuse tracing.

**Phase 1 — MVP (self-serve, reliability-first).** See §6.

**Phase 2 — Trust & observability depth.** Plain-English trace viewer with "show all evaluated options," replay-from-prior-run, eval-gate before publish (DeepEval/Promptfoo), online evals on sampled prod traces, guardrail rails (NeMo + classifiers) as a non-bypassable step.

**Phase 3 — Scale & catalog.** Composio broker for broad app catalog + Tool Registry with RAG-over-tools; multi-agent Supervisor/Sequential templates; memory (Mem0) and RAG-as-tool node; sandbox pool for tool/code execution.

**Phase 4 — Enterprise controls (progressively enabled).** SSO/SAML/SCIM, ReBAC roles + isolated workspaces, admin publish-approval gate, DLP defaults, audit logs (Purview-style), silo-tier deployment + data residency, A2A for external-agent interop.

---

## 6. MVP Definition (Phase 1 — the smallest thing that proves the wedge)

**Thesis to prove:** a non-technical user can build, test, trust, and run a *multi-agent* workflow — and never fear a surprise bill.

**In scope:**
- Visual canvas (React Flow) with a **minimal node set**: Trigger, Agent (role/goal/instructions), Tool/MCP call, Router, Human-in-the-loop approval, Output.
- **NL → workflow** generation ("describe your workflow") producing an editable graph.
- **Agent node** = role/goal/instructions fields + toggle-attached tools; ReAct loop with max-iterations guardrail.
- **Sequential + one Supervisor** multi-agent pattern (enough to demonstrate "multiple dedicated agents achieving a goal").
- **MCP + API calling** via native MCP client for 3–5 marquee integrations (e.g. Gmail/Slack/HTTP/Google Sheets) with "Connect [App]" OAuth.
- **Durable execution** on Temporal: runs survive restarts, HITL pauses resume.
- **Cost estimate before run + live cost badge + hard per-workflow budget cap** that pauses-and-asks.
- **"Show your work" run trace** in plain language (per-agent decision, tool call, cost) + free test runs on sample data.
- **JTBD template gallery** (5–8 starter workflows) + onboarding credits.
- **Tenancy skeleton**: Postgres RLS, per-tenant secrets via KMS/Vault (never into LLM context), basic roles.

**Out of scope for MVP (deferred):** Swarm/Network patterns, full eval-gate, sandboxed code execution, SSO/SCIM, silo deployment, A2A, RAG-over-thousands-of-tools, long-term memory.

**MVP success criteria:** a non-technical tester builds a 2-agent workflow that calls a real MCP/API, sees the cost before running, runs it with an approval step, and understands from the trace what each agent did — end to end, without help.

---

## 7. Key Risks & Mitigations

- **Reliability of autonomous nodes** → default to deterministic graph; bound every agent (max-iterations, budget, guardrails); auto-propagate context to fight fragmentation.
- **Temporal determinism-versioning tax** (users constantly edit workflows) → compile graph to a *data-driven interpreter* (edits = data changes, not replayed code); keep Restate as a migration option.
- **Cost blowups / runaway loops** → cost accounting from LLM span tokens, hard caps that pause-and-ask, loop detection, bounded retries — on by default.
- **MCP/tool security** (tool poisoning, confused deputy, token passthrough) → pin+hash tool descriptions, per-client consent, RFC 8707 audience validation, SSRF-hardened OAuth discovery, secrets never in LLM context.
- **"No-code that isn't"** (the n8n/Wordware trap) → relentless non-technical usability testing as a gate; NL-first authoring; forms over prompt boxes.
- **Broad-then-shallow** → reliability-first horizontal wedge, but ship 5–8 deep JTBD templates so first value is fast despite horizontal scope.

---

## 8. Verification

Because this is a greenfield build, verification is defined per phase:

- **Phase 0 spikes:** a scripted end-to-end test that (1) submits a JSON graph, kills the worker mid-run, and confirms Temporal resumes to completion; (2) connects the MCP client to a live server and executes a tool with OAuth; (3) asserts a run halts when the budget cap is hit. Traces appear in Langfuse.
- **MVP:** the §6 success-criteria walkthrough run by a genuinely non-technical tester (usability session, recorded), plus an automated integration test covering build → test-run → approval → completion → trace, and a cost-cap enforcement test.
- **Ongoing:** every published workflow runs an eval suite before promotion (Phase 2+); guardrail rails are exercised by red-team prompts (Promptfoo) in CI.

---

## Appendix — Full research sources

Detailed reference architecture with ~150 inline citations (Anthropic, OpenAI, LangGraph, Temporal/Restate/Inngest, MCP spec 2025-06-18, Cognition, OWASP LLM Top 10, OTel GenAI, AWS SaaS Lens, and vendor docs): `/Users/surajtanwar/.claude/plans/go-in-a-complete-peaceful-sky-agent-a666323656f0d344c.md`. Competitive/market and UX findings are synthesized inline in §1–§2 above.
