# AI Harnesses & Harness Engineering

How you **wrap** an LLM so it can act safely and measurably: the outer loop, tools, policy, eval, and ops — not “pick a bigger model.”

← [README](../README.md) · [Docs index](./README.md) · Related: [AI systems (RAG / LLM)](./ai-systems.md) · [Tool calling & prompt injection](./ai-systems.md#tool--function-calling-who-validates-args) · [Services vs workers](./service-architecture.md#services-vs-workers)

---

## Index

- [What is an AI harness?](#what-is-an-ai-harness)
- [Harness engineering (discipline)](#harness-engineering-discipline)
- [Harness vs model vs RAG vs “agent”](#harness-vs-model-vs-rag-vs-agent)
- [The agent loop (what you draw)](#the-agent-loop-what-you-draw)
- [Harness building blocks](#harness-building-blocks)
- [Policy & safety (non-negotiables)](#policy--safety-non-negotiables)
- [Eval & graders (make it measurable)](#eval--graders-make-it-measurable)
- [Observability & cost](#observability--cost)
- [Where it sits in a product HLD](#where-it-sits-in-a-product-hld)
- [Interview pitfalls](#interview-pitfalls)
- [Agentic interview rounds (human as tech lead)](#agentic-interview-rounds-human-as-tech-lead)
- [See also](#see-also)

---

## What is an AI harness?

**Interview snapshot**
- **What:** The **runtime + control plane** around a model: tools, permissions, state, retries, eval, and stop conditions.
- **Why:** Raw chat completions don’t ship products; harnesses turn “suggest text” into **bounded side effects**.
- **Trade-off:** More harness = safer/measurable, more engineering; thinner harness = demos fast, prod incidents.
- **Example:** Cursor / coding agents: model proposes edits/commands; harness runs them in a workspace with diffs, tests, and user approval.

```text
User goal
   ↓
Harness (policy, tools, memory, loop, graders)
   ↓
LLM (propose plan / tool calls / text)
   ↓
Tools execute (your code validates) → observations back into harness
   ↓
Done / fail / ask human
```

The **model is not the system**. The harness is.

---

## Harness engineering (discipline)

**Harness engineering** = designing and operating that outer system as a first-class product:

| Concern | Harness job |
|---------|-------------|
| **Correctness** | Tools + validators enforce invariants the model can’t be trusted for |
| **Safety** | Allowlists, sandboxes, human gates for irreversible actions |
| **Reliability** | Timeouts, retries, idempotent tool effects, circuit breakers |
| **Measurability** | Trajectories, graders, golden tasks, regression suites |
| **Cost / latency** | Step budgets, model routing, cache, early stop |
| **Debuggability** | Replayable traces (prompts, tool I/O, decisions) |

**Staff line:** “We don’t ‘prompt harder’ for money paths — we engineer the harness so bad proposals can’t execute.”

Contrast with **prompt engineering** (wording inside the model) and **RAG** (what context you feed). Harness engineering owns the **loop and side effects**.

---

## Harness vs model vs RAG vs “agent”

| Term | Meaning | Owns |
|------|---------|------|
| **Model** | Weights that generate text / tool calls | Next-token / structured propose |
| **RAG** | Retrieve docs → stuff context | Knowledge freshness & recall |
| **Agent** | Model + tools + multi-step goal | Often marketing word — pin the harness |
| **Harness** | Everything that **runs, constrains, and grades** the agent | Safety, eval, ops |

**Interview default:** say “agent” only after you can point at the harness boxes (tools, policy, loop, grader).

---

## The agent loop (what you draw)

```text
┌─────────────────────────────────────────┐
│  while not done and steps < budget:     │
│    1. Build context (goal, memory, RAG) │
│    2. Model proposes (text / tool calls)│
│    3. Harness validates & executes tools│
│    4. Append observations to trace      │
│    5. Check stop / escalate to human    │
└─────────────────────────────────────────┘
```

| Step | Who | Staff note |
|------|-----|------------|
| Propose | LLM | May hallucinate tool names/args |
| **Validate** | Harness | Schema, authz, allowlist, rate/amount limits — see [Tool calling](./ai-systems.md#tool--function-calling-who-validates-args) |
| Execute | Harness / workers | Idempotent where retries happen; timeouts |
| Observe | Harness | Sanitize secrets before re-entering the model |
| Stop | Harness | Max steps, success grader, human cancel |

Keep the **sync user path** free of unbounded loops when SLOs are tight — run the harness in a **worker** ([LLM off request path](./ai-systems.md#when-an-llm-should-not-be-in-the-request-path)).

---

## Harness building blocks

| Block | Role | Interview example |
|-------|------|-------------------|
| **Tool registry** | Named actions the model may call | `search_code`, `run_tests`, `create_pr` |
| **Sandbox / isolation** | Bound blast radius of tools | Container, gVisor, no prod creds, network egress allowlist |
| **Policy engine** | What is allowed for this user/tenant/task | Read-only vs write; “never `rm -rf`”; payment tools need dual control |
| **Memory / state** | Short-term trace + optional long-term store | Conversation, working files, checkpoint after each step |
| **Planner (optional)** | Separate “plan then act” or human `PLAN.md` | Coding interviews: plan before implement |
| **Grader / eval** | Did the run succeed? | Unit tests green, rubric, LLM-as-judge on a golden set |
| **Human-in-the-loop** | Approve irreversible steps | Merge PR, send email, move money |
| **Orchestrator** | Schedules steps, retries, fan-out subagents | Queue + worker; Temporal for long runs |

**Not required on every board:** subagents, fancy planners. **Always required:** tools + validate + stop condition + some eval.

---

## Policy & safety (non-negotiables)

Tied to [prompt injection](./ai-systems.md#prompt-injection-in-tool-calling-agents):

1. **Allowlist tools** per task — no open shell unless sandboxed and scored.  
2. **Validate args in your code** — model never talks to the bank/PSP/DB directly.  
3. **Separate trust** — system policy ≠ user text ≠ retrieved docs (label untrusted).  
4. **Least privilege credentials** — short-lived tokens; no secrets in prompts.  
5. **Egress controls** — tools can’t hit arbitrary URLs.  
6. **Human gate** on irreversible / high-blast actions.  
7. **Step & cost budgets** — kill runaway loops.

**Interview line:** “Untrusted content is input to a process with tools — the harness is the security boundary, not the model.”

---

## Eval & graders (make it measurable)

Without graders, harness changes are vibes.

| Layer | What you measure | How |
|-------|------------------|-----|
| **Task success** | Did the goal complete? | Golden tasks: pass/fail (tests, expected artifacts) |
| **Trajectory quality** | Wasteful / unsafe steps? | Step count, forbidden tool calls, human overrides |
| **Component eval** | RAG / tools alone | Retrieval Recall@k; tool mock unit tests |
| **Regression** | New prompt/model breaks old tasks | CI suite of harness runs (deterministic seeds where possible) |

**LLM-as-judge** can score open-ended outputs — calibrate to humans; don’t use it as sole gate for money paths ([LLM-as-judge](./ai-systems.md#llm-as-judge-in-a-scoring-pipeline)).

**Staff practice:** fix **tools + policy** failures before blaming the base model.

---

## Observability & cost

| Signal | Why |
|--------|-----|
| **Trace per run** | Prompt versions, tool I/O, latencies, token counts |
| **Outcome + failure reason** | Timeout, policy deny, tool error, grader fail |
| **Cost per task** | Tokens × steps × model price — budget alarms |
| **Human intervention rate** | Harness too weak or task too hard |

Cost levers for the model layer: [LLM cost doubled](./ai-systems.md#llm-cost-doubled--levers). Harness levers: fewer steps, smaller tools, cache identical sub-tasks, cheaper model for planning / expensive for hard steps.

---

## Where it sits in a product HLD

```text
Client → API (accept job, return 202) → Queue
              → Harness worker (loop + tools + policy)
                    → LLM API
                    → Internal APIs / sandbox / RAG
              → Store result + trace → webhook / SSE
```

| Product shape | Harness note |
|---------------|--------------|
| Coding assistant | Workspace FS + tests as grader; diff review |
| Support / ops agent | Read tools broad; write tools gated |
| RAG chatbot (no tools) | Thin harness: retrieve → generate → cite; still need ACL + eval |
| Multi-agent | Separate harnesses or shared policy; don’t share prod creds across agents |

---

## Interview pitfalls

- Drawing “Agent” as one magic box with no tools/policy/eval  
- Letting the model execute unverified tool args  
- Sync HTTP holding an unbounded agent loop  
- No stop condition / step budget  
- Eval only on vibe demos — no golden tasks  
- Treating RAG as a harness (retrieval ≠ side-effect control)  
- Skipping human gates on irreversible actions  

---

## Agentic interview rounds (human as tech lead)

Some companies (e.g. Razorpay agentic coding) evaluate **you driving a harness**, not vibe-accepting diffs.

| Signal | Do | Don’t |
|--------|-----|-------|
| Ownership | Plan first (`PLAN.md`); you state trade-offs | Let the agent invent architecture unread |
| Harness use | Agent as implementer; you review every diff | Ship unreviewed commits |
| Feedback loop | Tests as done; re-plan after 2 failed same fixes | Retry the same broken prompt forever |
| Proof | Hand-edit a small extension without AI when asked | Claim code you can’t explain |

Prep pointer: [Razorpay Senior SDE — Agentic Coding](../company-prep/razorpay-senior-sde.md#round-1--agentic-coding-90m).

**Interview line:** “I’m the tech lead; the harness/agent is the junior. I own plan, review, and stop conditions.”

---

## See also

- [AI systems](./ai-systems.md) — RAG, embeddings, tool validation, prompt injection, cost  
- [Service architecture](./service-architecture.md) — workers, 202 + async  
- [Reliability](./reliability-and-slos.md) — retries, budgets, blast radius  
- [Security](./security-and-compliance.md) — least privilege, threat model  
