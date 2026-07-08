# Reference Architecture: A No-Code Platform for Visually Building Production-Grade Multi-Agent Workflows

Research synthesis (2025–2026). Target: non-technical users visually compose workflows of dedicated AI agents that call MCP servers and APIs, on a production-grade engine.

The single strongest cross-cutting finding: **build a deterministic graph as the backbone, expose autonomous agents as bounded nodes inside it, and run the whole thing on a durable-execution substrate.** Every area below reinforces that spine.

---

## 1. Workflow Execution Models

Every architecture sits on one axis: how much control flow is fixed by the builder (deterministic) vs. decided at runtime by an LLM (autonomous). Anthropic's framing: **workflows** orchestrate LLMs/tools through predefined code paths; **agents** dynamically direct their own process. Agents trade latency + cost for flexibility. ([Anthropic — Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents))

**Models compared:**
- **Pure DAG (acyclic)** — Airflow/Prefect/Dagster. Mature, auditable, but cannot express loops. Poor fit for agents.
- **Cyclic graph / state machine** — **LangGraph** is the reference. It explicitly rejected DAG engines because "LLM agents really benefit from looping... cannot be handled by DAG algorithms," and rejected generic durable engines because it also needs streaming + low-latency steps. Uses Pregel/BSP with versioned channels; framework non-determinism is isolated from LLM non-determinism. LangGraph 1.0 shipped Oct 2025 with primitives unchanged. ([Building LangGraph](https://www.langchain.com/blog/building-langgraph))
- **Agentic loop / ReAct** — single LLM in reason→act→observe loop. OpenAI says build an agent (not a workflow) only for: complex judgment, unmaintainable rulesets, or unstructured data. Weakness: **error compounding** and ~10× cost. ([OpenAI — Practical Guide to Building Agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf))
- **Event-driven / durable** — the reliability substrate beneath both (see §2).
- **Hybrid (emerging consensus)** — deterministic graph frames the workflow; agents operate inside bounded nodes. "The workflow defines the frame. The agent operates within it."

**When deterministic beats agentic:** knowable solution path, auditability/compliance needs, cost/latency predictability, isolated errors. Why <5% of enterprise apps run true autonomous agents. **When agentic wins:** unpredictable step count, brittle-if-coded rulesets, unstructured input, trusted sandboxed environment. Both Anthropic and OpenAI: **start simple, add complexity only when it measurably helps.**

Anthropic's five workflow patterns become the visual primitives: **prompt chaining, routing, parallelization (sectioning/voting), orchestrator-workers, evaluator-optimizer.**

**Recommendation:** **Graph-first (cyclic state-machine), agent-in-a-node hybrid.** A canvas *is* a graph — matches the audience and gives per-node testing, isolated errors, audit trails. Ship an "Agent node" (ReAct + max-iterations + guardrails) users drop into a deterministic graph. Ship the five patterns as first-class node templates. Default users toward determinism; opt into autonomy node-by-node.

---

## 2. Durable / Reliable Execution

The field converges on one pattern: **deterministic orchestration + journaled non-deterministic (LLM/tool) steps + exactly-once retries + suspendable human-in-the-loop waits.** Wrap every LLM/MCP/API call so its *result is persisted once* and replayed rather than re-executed.

**Critical distinction ([Diagrid](https://www.diagrid.io/blog/checkpoints-are-not-durable-execution-why-langgraph-crewai-google-adk-and-others-fall-short-for-production-agent-workflows)):** *Checkpointing* (LangGraph, CrewAI, Google ADK) is a save-point mechanism — no automatic failure detection, mid-node crashes not durable, no distributed coordination. *Durable execution* (Temporal, Restate, Inngest, DBOS) is a runtime guarantee. **Do not rely on LangGraph's checkpointer as your durability guarantee** — use it as an authoring layer above a real engine.

**Substrates:**
- **Temporal** — event-history + full replay. Most battle-tested. Big drawback for this product: **determinism-versioning tax** — editing in-flight workflow code breaks replay. Heavy cluster ops. ([Temporal](https://temporal.io/blog/durable-execution-meets-ai-why-temporal-is-the-perfect-foundation-for-ai))
- **Restate** — log-journal + snapshots (partial replay), **no determinism-versioning tax** (plain async handlers), first-class **idempotency keys with stale-attempt fencing**, **Virtual Objects** (single-writer, keyed, stateful — maps 1:1 to "one durable agent session per key"), 3–10ms/step, single Rust binary. ([Restate](https://www.restate.dev/blog/building-a-modern-durable-execution-engine-from-first-principles))
- **Inngest + AgentKit** — step-memoization, exactly-once, `waitForEvent` for HITL, MCP-as-tools. TypeScript-first; interactive low-latency is a stated weak spot. ([Inngest](https://www.inngest.com/blog/durable-execution-key-to-harnessing-ai-agents))
- **AWS Step Functions + Bedrock AgentCore** — managed state machines; fits staged pipelines, awkward for free-form loops; AWS lock-in.
- **DBOS** — durable execution as a Postgres library; minimal ops; youngest ecosystem.

**Research lean:** on the technical merits, **Restate** is the closest fit — no versioning tax (users constantly edit workflows), Virtual Objects fit agent sessions natively, first-class idempotency for side-effecting MCP/API calls, low latency, single-binary ops.

**Product decision (see `PLAN.md` §3): Temporal for v1.** Compiling the visual graph to a **data-driven interpreter** (graph = data, interpreter = stable code) means edits change data, not replayed code — which *neutralizes Temporal's determinism-versioning tax*, the one thing that made Restate look decisively better. With the tax removed, we pick on **maturity, ecosystem depth, and battle-tested durable timers for days-long HITL waits → Temporal.** **Restate remains the strong alternative** — revisit if the interpreter proves complex or versioning-ops still bite (its Virtual-Object-per-session maps 1:1 to an agent session). Other alternatives: **Inngest/AgentKit** (TS fast-path), **DBOS** (Postgres-only simplicity). **Do not build durable execution from scratch.**

---

## 3. Multi-Agent Orchestration

**Patterns:** orchestrator-worker (Anthropic's research system: lead + parallel subagents, +90.2% vs single agent, but ~15× tokens), supervisor (central router — best default), hierarchical (supervisors-of-supervisors), handoffs/swarm (decentralized peer transfer — lower latency, harder to debug), sequential/parallel (the primitives), network (any-to-any — usually over-engineering).

**Frameworks:** OpenAI Agents SDK (handoffs, guardrails-as-tripwires, sessions), CrewAI (Crews for autonomy + Flows for deterministic restartable pipelines), LangGraph (`StateGraph` + `Command` handoffs, `langgraph-supervisor`/`langgraph-swarm`), Microsoft Agent Framework (AutoGen+Semantic Kernel merger, 1.0 GA Apr 2026, graph workflows + enterprise/OTel/Entra), Google ADK + A2A.

**MCP vs A2A:** complementary. **MCP** connects an agent *down* to tools/data (hub-and-spoke, stateless). **A2A** connects agents *across* to peers (peer-to-peer, stateful, Agent Cards at `/.well-known/agent-card.json`). ([Google agent protocols guide](https://developers.googleblog.com/developers-guide-to-ai-agent-protocols/))

**When multi-agent helps vs hurts:** Cognition ("[Don't Build Multi-Agents](https://cognition.com/blog/dont-build-multi-agents)") — share full context/traces; conflicting implicit decisions produce broken results; prefer single-threaded agent + context compression. Anthropic ([multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)) — multi-agent wins on breadth-first, parallelizable, context-exceeding tasks. **They agree:** the failure mode is context fragmentation. Multi-agent HELPS on loosely-coupled read-heavy fan-out; HURTS on tightly-coupled sequential artifacts (most coding).

**Recommendation:** LangGraph-style `StateGraph` + `Command` handoffs as the engine (or Microsoft Agent Framework for .NET/enterprise). **Default users to Supervisor + Sequential; gate Swarm/Network behind "advanced."** Auto-propagate full upstream traces into downstream nodes + offer a "context compression" node (defeats the #1 failure mode). Two clearly-separated protocol lanes: MCP for tools, A2A for external agents. Surface per-run token/cost + effort-scaling controls (token multipliers are large). Nudge tightly-coupled workflows toward single-agent templates.

---

## 4. Tool / MCP / API Integration Layer

**MCP (spec 2025-06-18):** hosts/clients/servers over JSON-RPC 2.0; primitives = **tools/resources/prompts** (+ client-side sampling/roots/**elicitation**). Transport = **Streamable HTTP** (single endpoint, POST + optional SSE); **HTTP+SSE deprecated** — build Streamable-HTTP-first with legacy fallback. 2025-06-18 changes: structured tool output, MCP servers as **OAuth Resource Servers**, mandatory **RFC 8707 Resource Indicators** (audience binding), elicitation, `MCP-Protocol-Version` header. **Authorization** = OAuth 2.1 + PKCE + RFC 9728 (Protected Resource Metadata) + RFC 8414 + RFC 7591 (Dynamic Client Registration). ([MCP spec](https://modelcontextprotocol.io/specification/2025-06-18))

**Client architecture:** connection manager (1 client per server), dynamic discovery via `tools/list` + `list_changed` notifications, **central Tool Registry** (Anthropic donated a Registry OpenAPI spec to AAIF Dec 2025), **namespacing** (`{server}__{tool}`). **Scaling to thousands of tools = RAG over tools:** expose ~2 meta-tools (`search_tools`/`load_tools`, Dynamic ReAct pattern), vector-index over LLM-enriched descriptions (Anthropic-enriched descriptions raised Top-5 retrieval ~50% relative). ([Dynamic ReAct arXiv:2509.20386](https://arxiv.org/html/2509.20386v1))

**Third-party OAuth + credentials:** authorization code + PKCE, token refresh/rotation, per-tenant vaulting. **Keep two token planes separate** (client→MCP-server vs MCP-server→upstream API); **never pass tokens through.** Incumbents: n8n (Vault/Infisical external stores), Make (AES-256 + AWS KMS), Zapier (managed). No-code brokers: **Zapier MCP** (zero-config, 9k+ apps), **Composio** (`user_id`-scoped sessions, 1000+ toolkits, MCP mode), **Pipedream Connect** (embeddable, doesn't retain payloads).

**MCP security:** tool poisoning / rug pulls (pin+hash descriptions, re-consent on change), confused deputy (per-client server-side consent), token passthrough (forbidden; RFC 8707 audience validation), SSRF during OAuth discovery (block private IPs), scope minimization.

**Recommendation:** **Buy the third-party OAuth broker initially** (Composio or Pipedream Connect) for instant catalog breadth + safe per-tenant OAuth. Build a **native MCP client** (Streamable HTTP, full OAuth 2.1 stack, SSRF-hardened) for custom servers. Central **Tool Registry + RAG-over-tools** with meta-tools and LLM-enriched descriptions. **Per-tenant envelope encryption (KMS)** floor + Vault/Infisical bring-your-own for enterprise. MCP security gateway scanning descriptions before they reach the model. UX: "Connect [App]" buttons, show `title` not `name`, per-user connected accounts, audit log + revoke.

---

## 5. Agent Runtime & Model Layer

**Model gateway** (highest-leverage decision — decouples engine from any provider): **LiteLLM** (self-host, 100+ providers, richest routing: simple-shuffle/latency/cost/usage, failover tiers, cooldowns), **OpenRouter** (managed, shallow routing), **Portkey** (most feature-complete: guardrails/PII/semantic cache/conditional routing, now Apache-2.0), Cloudflare AI Gateway (edge), Vercel AI Gateway (fallbacks GA, no conditional routing), Bedrock/Vertex (backends, not gateways). Tradeoffs: extra hop/failure domain (mitigate with layered client+gateway fallback); lowest-common-denominator feature lag (esp. **prompt caching** breakpoints).

**Context management** ([Anthropic context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)): **context rot** — accuracy degrades before the nominal limit; don't naively concatenate. Three strategies: **compaction**, **structured note-taking** (external notes file), **sub-agent architectures** (clean windows → condensed summaries — maps 1:1 to the visual graph). **Prompt caching is the biggest cost lever** — cache reads at ~10% of input cost; structure calls as `[tool defs][system][history]` with breakpoints on the stable prefix. ([Anthropic prompt caching](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching))

**Memory:** short-term (buffer + compaction) vs long-term (episodic/semantic). **Mem0** (fast drop-in personalization, cheap), **Zep/Graphiti** (temporal knowledge graph — best when facts evolve, +15pts on LongMemEval), **Letta/MemGPT** (self-curating long-running agents). Vector substrate: **pgvector** (if already on Postgres, competitive to ~1M rows), Qdrant (filtered RAG), Pinecone (zero-ops), Turbopuffer (cost at scale).

**Structured outputs** (the node-to-node wiring): constrained decoding (OpenAI Structured Outputs — guarantees your schema), tool-calling (Anthropic), **Pydantic AI + Instructor** (portable, validate-and-reask retry across providers).

**RAG as a tool, not a fixed step:** default = hybrid search + cross-encoder rerank → top-5/10; "Agentic Search" toggle (multi-hop) for hard queries with adaptive routing.

**Recommendation:** **LiteLLM** self-hosted core (own the data path for multi-tenant guardrails/budgets), Anthropic/OpenAI/Bedrock/Vertex as backends, layered fallback, **gateway must forward `cache_control` untouched**; Portkey as drop-in upgrade if you want managed guardrails. Build **compaction + note-taking + sub-agents** as first-class engine features. **Automatic, invisible prompt caching.** Two-tier memory: **Mem0 default**, Zep optional, over **pgvector→Qdrant/Turbopuffer**. **Pydantic + Instructor** typed I/O (UI output shapes → schema). RAG as a tool node.

---

## 6. Observability, Evals & Guardrails

**Instrument once in OpenTelemetry GenAI semantic conventions** — `invoke_agent`/`chat`/`execute_tool` spans, `gen_ai.usage.*` token attributes; multi-agent trees map naturally (root agent span → chat children → tool children). Decouples you from any backend. (Still "Development" status as of SemConv 1.40.0, April 2026.) ([OTel GenAI spans](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/))

**Observability backends:** **Langfuse** (OSS/MIT core, OTel-native, sessions, agent graph view, native token/cost — best OSS fit; operate Postgres/ClickHouse/Redis/S3), LangSmith (best if on LangGraph, proprietary), Arize Phoenix (OTel-native, framework-agnostic), Braintrust (best eval loop, closed), Helicone (simplest proxy cost capture), W&B Weave.

**Evals:** offline (CI regression) + online (sampled prod traces); LLM-as-judge (+ Agent-as-a-Judge for trajectories); **agent trajectory evaluation** (tool-call correctness, step ordering). **DeepEval** (pytest-native) / **Promptfoo** (YAML, red-teaming; now OpenAI-owned) in CI; **Langfuse online evals** on prod. Eval-driven development: every published workflow runs a suite before promotion.

**Guardrails** (anchor to **[OWASP LLM Top 10 2025](https://genai.owasp.org/llm-top-10/)** — highest risk: LLM01 injection incl. indirect via tool content, LLM06 excessive agency, LLM10 unbounded consumption): **NeMo Guardrails** (input/output/dialog/retrieval/**execution** rails — maps to agent→MCP flow), Guardrails AI (validators). Classifiers: **Prompt Guard 2** (injection, cheap), **Llama Guard 4** / OpenAI Moderation (content), **Presidio** (PII). Avoid Rebuff (archived); Lakera/Protect AI now enterprise-acquired.

**Cost:** source of truth = leaf LLM span tokens × versioned price table; roll up span→agent→run→tenant; wire to hard per-tenant budget/step/rate limits (LLM10 cost-DoS defense).

**Recommendation:** **OTel GenAI wire format + Langfuse (self-hosted) + DeepEval/Promptfoo (CI) + Langfuse online evals + NeMo Guardrails (with Prompt Guard 2 / Llama Guard 4 / Presidio) + custom per-tenant cost/budget service.** Guardrails run as a required, non-bypassable orchestrator step. **India-aware PII:** extend Presidio with custom recognizers for **Aadhaar, PAN, UPI IDs, and Indian phone formats** (the platform targets Indian businesses).

---

## 7. Multi-Tenancy, Security & Deployment

Core tension: running untrusted LLM-generated code/tool calls for mutually-distrusting tenants with access to their secrets. **Decide this area first.**

**Sandboxing (isolation spectrum):** Docker/K8s (insufficient for untrusted) < gVisor (~10-15% overhead, no GPU — Modal uses) < **Firecracker microVMs** (own kernel, ~150ms cold / 5-30ms snapshot — E2B uses) ; Kata/Sysbox (middle, Daytona optional); V8 isolates (ms, JS/WASM only — Cloudflare). Managed: **Daytona** (27-90ms, fastest), **E2B** (Firecracker + Python templates), **Modal** (best GPU), **Cloudflare Sandbox** (edge + durable state). Self-hosting Firecracker breaks even ~11k sandbox-hrs/mo. Always layer seccomp + read-only FS + **deny-by-default egress allow-lists**. ([Spheron](https://www.spheron.network/blog/ai-agent-code-execution-sandbox-e2b-daytona-firecracker/), [ZenML](https://www.zenml.io/blog/e2b-vs-daytona))

**Tenant isolation ([AWS silo/pool/bridge](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/silo-pool-and-bridge-models.html)):** **Bridge** — pool app + Postgres with **RLS** (runtime `app.current_tenant`, `FORCE ROW LEVEL SECURITY`, non-owner app role, **per-transaction context under PgBouncer** to avoid session-var leakage) for standard tenants; **silo** (dedicated DB/VPC/keys) as paid tier for regulated customers.

**Secrets:** **KMS envelope encryption with per-tenant encryption context** (data at rest) + **Vault** runtime issuance. **The agent-specific rule:** secrets reach the sandbox at creation time (env/files, least privilege) but **never enter the LLM context** — use reference-token placeholders (`{{secret:x}}`) resolved by a trusted proxy after the LLM emits the call. OWASP LLM02: never rely on "the model won't reveal it."

**Authz (two planes: platform users + agents-on-behalf-of-users):** **ReBAC (Zanzibar)** — superset of RBAC, ideal for nested/shared/multi-tenant workflow resources. **OpenFGA/SpiceDB** (self-host) or **Permit.io/Oso** (managed, faster velocity).

**Agent security (OWASP LLM06 Excessive Agency):** minimize tools per agent; **effective permission = intersection(user rights, agent capabilities)** (confused-deputy defense); short-lived scoped agent identity (OAuth 2.1 + RFC 8693 token exchange, or SPIFFE/SPIRE SVIDs); **human-approval gates** on destructive/costly/irreversible actions; enforce authz **in downstream systems, never in the LLM.**

**Deployment:** durable execution orchestrator (§2) **decoupled** from an ephemeral managed-sandbox pool; workers on **Kubernetes with KEDA/HPA on queue depth**; tier work by duration (edge isolate → queue+worker → durable workflow). Mistral Workflows validates the pattern (Temporal core + multi-tenancy + AI streaming).

**Recommendation:** managed **E2B or Daytona** sandboxes (Daytona if per-turn cold start dominates; Modal when GPU needed) + egress allow-lists; **Bridge** tenancy (RLS pool + silo tier); **KMS envelope + Vault** with secrets injected to sandbox never LLM; **ReBAC** (Permit.io fast / OpenFGA owned); intersection-permission + scoped agent identity + human-approval gates; durable orchestrator decoupled from sandbox pool on K8s. **India data-residency (DPDP Act 2023):** region-pin standard tenants to an India region; the silo tier provides dedicated in-region DB/VPC/keys for regulated customers.

---

## Proposed Reference Architecture (Layered)

```
┌──────────────────────────────────────────────────────────────────────────┐
│  L1  EXPERIENCE — Visual Builder (canvas = graph of nodes+edges)           │
│      Node palette: Prompt-chain · Router · Parallelize · Orchestrator-     │
│      workers · Evaluator-optimizer · Agent(ReAct) · Tool/MCP · HITL ·      │
│      Context-compression · Knowledge(RAG). "Connect [App]" OAuth buttons.  │
│      Per-run token/cost + effort controls. Advanced-mode: Swarm/Network.   │
├──────────────────────────────────────────────────────────────────────────┤
│  L2  CONTROL PLANE — compile canvas → data-driven graph spec               │
│      Versioning · eval-gate before publish · ReBAC (users + agent scopes)  │
├──────────────────────────────────────────────────────────────────────────┤
│  L3  DURABLE EXECUTION SUBSTRATE — Temporal for v1 (Restate alternative)   │
│      Graph interpreter (graph=data). One durable workflow per agent        │
│      session (Restate: Virtual Object per session). Journaled LLM/MCP/API  │
│      steps · idempotency keys · retries/backoff · durable-timer HITL       │
│      waits (days-long) · global per-run retry/token budget                 │
├──────────────────────────────────────────────────────────────────────────┤
│  L4  AGENT RUNTIME & MODEL LAYER                                           │
│      Model gateway (LiteLLM→Portkey) · auto prompt-caching · context       │
│      engine (compaction/notes/sub-agents) · Mem0/Zep over pgvector·        │
│      Pydantic+Instructor typed I/O · RAG-as-tool (hybrid+rerank)           │
├──────────────────────────────────────────────────────────────────────────┤
│  L5  TOOL / INTEGRATION LAYER                                              │
│      Native MCP client (Streamable HTTP, OAuth 2.1, SSRF-hardened) +       │
│      Composio/Pipedream broker · Tool Registry + RAG-over-tools (meta-     │
│      tools) · namespacing · MCP security gateway (desc hash/scan)          │
├──────────────────────────────────────────────────────────────────────────┤
│  L6  EXECUTION SANDBOX POOL — E2B/Daytona (Firecracker), egress allow-list │
│      Secrets injected at creation (never to LLM); reference-token proxy    │
├──────────────────────────────────────────────────────────────────────────┤
│  L7  CROSS-CUTTING                                                         │
│      Observability: OTel GenAI → Langfuse  |  Evals: DeepEval/Promptfoo +  │
│      online  |  Guardrails: NeMo (+Prompt Guard 2/Llama Guard 4/Presidio)  │
│      Secrets: KMS envelope + Vault  |  Tenancy: Postgres RLS bridge + silo │
│      Cost: per-tenant token accounting + hard budget/rate limits           │
└──────────────────────────────────────────────────────────────────────────┘
```

**One-line thesis:** Deterministic graph backbone (reliability, auditability, learnable UX) + autonomous agents as bounded guardrailed nodes + durable-execution substrate underneath; buy the integration/sandbox layers, own the graph engine and gateway; standards-based OTel/OWASP/MCP-spec throughout.
