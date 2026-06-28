---
name: ai-agent-dev-guide
description: "Use when starting, scoping, or reviewing any AI agent application — backend agent logic and/or an LLM-generates-UI frontend — and you are not yet sure which skill to open. This is the entry-point / router for the AI Agent Dev suite. It dispatches to backend framework skills (currently Embabel via embabel-agent-backend — GOAP / @Agent / @Action / Spring AI / JVM) and to the backend-agnostic generative-UI frontend (json-render-ui — LLM-produced JSON UI spec, SSE streaming, progressive rendering), and defines the recommended full-stack build order. Triggers: building an AI agent backend, a dynamic LLM-generated dashboard, SSE streaming of agent progress, or work spanning both backend and frontend."
---

# AI Agent Dev Guide (suite overview / router)

This is the **entry point for the AI Agent Dev suite**. When you want to build an AI agent application but aren't yet sure which skill to open, start here to decide the angle, then move on to the matching skill. This skill **only points the way and defines the full-stack build order**; all implementation details live in the domain skills.

The suite splits into two domains:

- **Backend** — agentic backend logic. Current framework: **Embabel** (`embabel-agent-backend`). Designed so additional frameworks can be added later as sibling `*-agent-backend` skills.
- **Frontend** — **`json-render-ui`**: a backend-agnostic, LLM-generates-UI generative rendering frontend. Usable on its own with any backend that meets its Backend Contract.

Evidence sources: the `learn-embabel` course (backend) plus the `embabel-json-render` POC (frontend). Version baseline (Embabel backend): Spring Boot 3.5.x + Embabel 0.4.0 + Spring AI 1.1.x (**Boot 4 is not yet supported — wait for Embabel 2.0**; see the backend skill for details).

## Routing table

| Your need | Which skill |
| ---- | ---------- |
| `@Agent`/`@Action`/`@AchievesGoal`, GOAP planning, Blackboard, type-driven routing (Embabel) | **embabel-agent-backend** |
| Domain `@Tool`, MCP, Agentic RAG, `@Condition`/SpEL gating (Embabel) | **embabel-agent-backend** |
| `@State` loops / human-in-the-loop (`WaitFor`), streaming output, cost tracking and budget guardrails (Embabel) | **embabel-agent-backend** |
| Autonomy `chooseAndRunAgent` / `AgentInvocation`, multi-agent orchestration and fusion, intent parameterization (Embabel) | **embabel-agent-backend** |
| Spring Boot / Spring AI wiring, version compatibility, testing, observability and failure handling (Embabel) | **embabel-agent-backend** |
| A dynamic dashboard for "natural language → backend generates a UI spec → frontend progressive rendering" | **json-render-ui** |
| json-render flat element-tree spec contract, component catalog ↔ frontend registry reconciliation | **json-render-ui** |
| SSE streaming (`fetch` + `ReadableStream`), `lenientParse`/`sanitize` fault-tolerant progressive rendering | **json-render-ui** |
| Backend step-progress / cost observability panel, click-to-drill-down, catalog maintenance page | **json-render-ui** |

## Backend frameworks

The backend domain is framework-pluggable. Today there is one:

| Framework | Skill | Status |
| ---- | ---- | ---- |
| Embabel (Spring AI / JVM, GOAP) | **embabel-agent-backend** | Available |
| _(others, e.g. LangChain, plain Spring AI, Node)_ | _(future sibling `*-agent-backend` skills)_ | Planned |

Adding a framework is additive: drop a sibling backend skill, add a row here, and add it to the plugin manifest. **`json-render-ui` needs no change** — it only requires the backend to satisfy its 4-point Backend Contract.

## Should you use an agent framework like Embabel? (decide before you start)

- **A good fit**: the flow has multiple steps, typed business objects, conditional branching, review gates, or audit requirements.
- **Not needed**: a single model call or a fixed pipeline → plain Spring AI (or your platform's LLM SDK) is enough; don't bolt on GOAP.
- For the detailed decision flow, see step 1 (Fit Assessment) in `embabel-agent-backend`.

## Full-stack build order (backend first, frontend second)

Backend only → use your backend framework skill (currently `embabel-agent-backend`). Frontend only → use `json-render-ui` (it works with any conformant backend). For a complete full-stack dynamic application, proceed in order:

1. **Design the backend flow** (`embabel-agent-backend`): use GOAP to design the action table, define Java record boundaries, implement `@Agent`/`@Action`, wire up Spring Boot/Spring AI, attach tools, and write tests.
2. **Nail down the frontend/backend spec contract** (`json-render-ui`): backend `{root, elements}` spec (▼ Embabel: a `DashboardSpec(root, elements)` record) ↔ frontend `Spec` type — lock the flat element-tree shape first.
3. **Wire SSE streaming + progressive rendering** (`json-render-ui`): backend opens SSE (▼ Embabel: `SseEmitter`); frontend `fetch`+`ReadableStream`, `lenientParse → sanitize → progressive setSpec`.
4. **Observability and interaction** (`json-render-ui`): step-progress + cost panel (▼ Embabel: GOAP plan/step), click-to-drill-down, catalog maintenance page.

> Cross-skill iron rules at a glance (details in the domain skills): **data arrays are always assembled deterministically into the spec by backend code, never produced by the LLM** (which drops items → empty table, breaks the JSON, tampers with numbers); the LLM only touches narrative text. **(Embabel) GOAP plan/action lifecycle events only go through the global listener**, not a per-call listener.

## Domain split

| Topic | Which to read |
| ---- | ---------- |
| Agent / GOAP / tools / backend orchestration and observability (Embabel) | **embabel-agent-backend** |
| Frontend/backend UI contract / streaming rendering / observability panel / drill-down / catalog maintenance | **json-render-ui** |

The skills are cross-linked; this guide is their shared upper-level entry point. When the requirement is vague or spans frontend and backend, dispatching from here is the fastest route.
