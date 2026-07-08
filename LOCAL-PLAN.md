# Learn-by-Building Roadmap: Your Agentic Workflow Platform

## Context — read this first

This plan is deliberately **different** from the grand blueprint. The full vision — a world-class, multi-tenant, India-first platform competing with n8n and Lindy — lives in the repo as [`PLAN.md`](./PLAN.md) and stays your **north star**. That is a funded-team, multi-year product.

This document is what *you and I actually build together*. Throughout this doc, **"you" means Suraj** — the non-technical builder — and **"I" means Claude**, writing the code. It's shaped by the real constraints you told me:

- **You** are non-technical but willing to follow clear step-by-step instructions and learn as you go.
- **I** write 100% of the code. You run the steps I give you. I cannot touch your machine — you are my hands.
- **Budget:** a small amount for API usage / cheap hosting is OK. Your **$200 Claude subscription does NOT power the app** — an app that calls Claude needs a separate **Anthropic API key** (pay-per-use, billed separately). We'll start it with ~$5–10 of credit.
- **Approach:** build the *real* platform, but in tiny achievable milestones. Each milestone produces something you can *see working* before we go further. We start with one agent doing one task on your own computer, and grow toward the vision.

**Honest expectation-setting:** we are building a *learning prototype that becomes real over time*, not a launch-ready SaaS. Heavy production infrastructure from `PLAN.md` (Temporal, LiteLLM, Langfuse, Firecracker sandboxes, multi-tenancy, Kubernetes) is **deliberately deferred** — we add each piece only when a milestone actually needs it and you understand why.

---

## Tech choices (and why they fit a solo non-technical builder)

| Choice | What it is | Why |
|---|---|---|
| **TypeScript** (one language) | The language for both the engine and, later, the visual web builder | One language for everything = less to learn. The visual canvas (React Flow) is JS-native anyway. |
| **Anthropic TypeScript SDK** (`@anthropic-ai/sdk`) | Official library to call Claude | First-party, well-documented; we use its **tool runner** so the agent loop is handled for us. |
| **Claude model: `claude-opus-4-8`** (default) | The model the agents think with | Most capable. **Cost note:** Opus is $5 in / $25 out per million tokens. For cheap learning/testing we can switch to **`claude-haiku-4-5`** ($1 / $5) or **`claude-sonnet-5`** ($3 / $15 — currently discounted to $2 / $10 through 2026-08-31). Your call — I'll default to Opus but flag when Haiku would save money. |
| **Tool runner + Zod** (`client.beta.messages.toolRunner`) | Auto-runs the "agent calls a tool → gets result → continues" loop | You don't have to hand-write the loop. Tools are plain typed functions defined with `betaZodTool` (from `@anthropic-ai/sdk/helpers/beta/zod`). *Note: the tool runner is a **beta** feature of the SDK — solid and documented, but its exact shape can shift between SDK versions. I'll pin the version so your code keeps working.* |
| **MCP via the SDK** | How agents call external MCP servers | The SDK supports MCP two ways: a direct `mcp_servers` connector (beta), and helpers that convert an MCP server's tools into tools the runner can use. This is the "calling MCP" from your original goal. *Also a **beta** surface — same version-pinning caveat.* |
| **Node + a simple local web app**, later **React Flow** | Runs on your computer first; visual builder comes later | See it in a browser without hosting costs; add the drag-and-drop canvas once the engine works. |
| **A `.env` file for your API key** | Keeps your secret out of the code | Standard, safe, simple. |

Everything here is a stepping stone *toward* the `PLAN.md` architecture — not a throwaway.

---

## Phase A — Setup (one-time, guided; ~30–45 min)

Goal: get your computer ready and prove Claude answers from code. I write everything; you run each step and paste me any errors.

1. **Install the basics** (I'll give exact commands/links): **Node.js** (the runtime) and **VS Code** (the editor). On your Mac we'll check `node --version` works.
2. **Get an Anthropic API key** — you sign up at the Anthropic Console, add **~$5–10 of credit**, and create an API key. *You* do this (it's a credential — I never handle it). We store it in a `.env` file.
3. **Create the project** — I generate a tiny TypeScript project (`package.json`, `tsconfig.json`, one folder). You run `npm install`.
4. **"Hello, Claude" test** — I write a ~15-line script that sends one message to Claude and prints the reply. You run it. **Cost: a fraction of a cent.**

✅ **You'll see:** Claude replying in your terminal, powered by *your* key. That proves the whole foundation works.

---

## Phase B — The milestone ladder (build the real thing, small step by small step)

Each milestone = I write the code, you run it, you see a result, you learn one concept. We only move on when the current one works.

**Milestone 1 — One agent, one tool.**
A single agent (role/goal/instructions) that can call **one local tool** (e.g. "read today's date" or "do a calculation") using the tool runner. Concretely, the first real code has the shape `client.beta.messages.toolRunner({ model, tools: [betaZodTool({ ... })], messages })` — that one call *is* the agent loop. Teaches: what an "agent" and a "tool" actually are. *Cost: cents.*

**Milestone 2 — Agent calls a real API.**
Give the agent a tool that hits a real external API (e.g. fetch weather, or read a Google Sheet). Teaches: how agents reach the outside world (the foundation of "calling APIs" from your goal). *Cost: cents.*

**Milestone 3 — Two agents achieving a goal (your original example).**
A **supervisor** agent that delegates to **worker** agents in sequence to complete a multi-step goal (e.g. "research a topic → draft a summary"). This is the literal thing you described: *multiple dedicated agents performing a series of tasks to achieve a goal.* Teaches: multi-agent orchestration. *Cost: a few cents per run.*

**Milestone 4 — Human-in-the-loop.**
Before the agent does something real (e.g. "send this"), it **pauses and asks you to approve** in the terminal. Teaches: the trust/safety pattern that's central to the vision. *Cost: cents.*

**Milestone 5 — Connect an MCP server.**
Wire the agent to a real **MCP server** so it gains a whole toolset with no custom code. Teaches: the "calling MCP" half of your goal, and why MCP matters. *Cost: cents.*

**Milestone 6 — A cost guardrail.**
Add a hard **per-run spend cap** that stops a workflow if it exceeds, say, $0.50 — plus printing the cost of every run. Teaches: the "no surprise bill" differentiator, made real and simple. *Cost: negligible.*

**Milestone 7 — See it in a browser (first UI).**
A minimal local web page where you type a goal, click "Run," and watch the agents work + see the cost. Teaches: turning the engine into something visual. *Cost: cents per run.*

**Milestone 8 — The visual builder (first real taste of the product).**
Introduce **React Flow**: drag nodes (Agent, Tool, Approval) onto a canvas, connect them, and run that workflow. This is the first version of the actual `PLAN.md` product — tiny, but real and yours. *Cost: cents per run.*

After Milestone 8 we reassess: you'll understand enough to decide whether to keep growing it (memory, more integrations, saving workflows, cheap hosting so others can try it) toward the north-star vision — and by then you'll know if this is worth pursuing further.

---

## What we are deliberately NOT building yet (and when we would)

These are in `PLAN.md` but are wrong to tackle now — I'll flag the milestone that would justify each:

- **Durable execution (Temporal), model gateway (LiteLLM), tracing (Langfuse), sandboxes, multi-tenancy, Kubernetes, WhatsApp Business, DPDP residency, SSO/RBAC.** All real, all later. We add any one of these *only* when a concrete milestone can't work without it — and I'll explain the tradeoff first.

---

## Cost — the honest picture

- **Development/testing:** cents to a few dollars total across all milestones, if we use Haiku/Sonnet for routine testing. Opus runs cost more; I'll tell you before anything pricey.
- **Your $200 Claude subscription:** pays for *us working here in Claude Code*. It does **not** pay for the app's Claude calls — those come from your Anthropic API credit.
- **Hosting:** $0 while everything runs on your computer. Only when you want *other people* to use it do we discuss cheap hosting (a few $/month).
- **Guardrail:** Milestone 6 builds a hard spend cap so a runaway workflow can't surprise you.

---

## Verification (how we know each step actually works)

- **Phase A:** `node --version` prints a version; the "Hello, Claude" script prints a real reply using your key.
- **Every milestone:** you run the milestone's command and *see the described result* (an agent answering, two agents completing a goal, an approval prompt, a cost printout, a workflow running in the browser). If it errors, you paste me the error and I fix it — that's the loop.
- **No milestone is "done" on my say-so** — it's done when *you* see it work on your screen.

---

## How we work together (the ground rules)

1. I write all code and give you exact, copy-paste steps — one small step at a time.
2. You run the step and tell me what you see (success or the exact error text).
3. When something breaks, we debug it together; I never assume it worked.
4. We keep the grand `PLAN.md` as the destination but judge every step by "can you see this working?"

---

## Appendix — the north-star vision

The full market + architecture blueprint (India-first, reliability-first, enterprise-grade) is preserved in the repo: [`PLAN.md`](./PLAN.md) and [`ARCHITECTURE-RESEARCH.md`](./ARCHITECTURE-RESEARCH.md) (with ~150 citations). Nothing here throws that away — this roadmap is the realistic path that *starts* walking toward it.
