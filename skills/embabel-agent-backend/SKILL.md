---
name: embabel-agent-backend
description: "Use when building, reviewing, or refactoring Embabel + Spring AI JVM agent applications, especially Spring Boot 4.1.x (Embabel 1.5.x) or Spring Boot 3.5.x (Embabel 1.0.x) / Java 21 projects that need GOAP planning, @Agent/@Action type-driven flows, Blackboard state, domain @Tool methods, MCP or Agentic RAG integration, prompt testing, observability, production-ready guardrails, @State loops/human-in-the-loop, @Condition boolean gates, Agentic/Progressive tools, streaming output, LLM cost tracking, budget guardrails, MCP server publishing, concurrent execution, the workflow DSL builders (ScatterGather fork-join, Consensus, RepeatUntil/RepeatUntilAcceptable), Agent Skills (SKILL.md packages, EmbeddingSkillSelector), or model provider / BYOK wiring. Note: Embabel 1.5.x (latest 1.5.1, verified 2026-09-10) runs on Spring Boot 4.1.x + Spring AI 2.0.x + Jackson 3; keep the Embabel 1.0.x line for Spring Boot 3.5.x projects."
---

# Embabel + Spring AI Development

Use this skill to turn a business process into a testable Embabel agent on the JVM. The source model is the `learn-embabel` course: start from engineering boundaries, model the flow with GOAP, implement it as Java records and annotated actions, wire Spring Boot + Spring AI infrastructure, then verify prompts, tools, and operations.

> **Not sure where to start?** The entry point and routing for the entire AI Agent Dev suite is in **`ai-agent-dev-guide`** (which includes the routing table and the full-stack build order). This skill focuses on the backend agent/GOAP/JVM side.
>
> **Pairing with the frontend skill**: When this Embabel backend needs a dynamic frontend built around "natural language → agent-generated UI spec → progressive frontend rendering" (json-render / SSE streaming / progress-and-cost observability panel / click-to-drill-down / component catalog maintenance), use the companion skill **`json-render-ui`**. This skill covers the agent/GOAP/JVM; that one covers the frontend/backend UI contract, streaming rendering, and interaction.

## Source-First Rule

Before writing code:

1. Read the local product/spec files for the target project.
2. If this repo is available, use these source files as the compact reference:
   - `source/course-package/overview.md`
   - `source/course-package/day1/outline.md`
   - `source/course-package/day1/content.md`
   - `teaching-site/course-data.js`
   - `teaching-site/course-package/materials/CustomerCareAgent.java`
3. Confirm the implementation understanding with the developer in concrete terms: target workflow, input/output domain records, terminal goal, model/tool boundaries, and tests.
4. Verify current Embabel, Spring AI, and Spring Boot versions from the project's docs or build files before pinning dependencies. Do not invent versions.

## Version Compatibility (verified 2026-09-10)

Two supported lines. Pick the one that matches your Spring Boot generation — do not mix.

| Embabel | Released | Spring Boot | Spring AI | Jackson | Kotlin |
|---|---|---|---|---|---|
| **1.5.1** (latest) | 2026-08-24 | **4.1.0** | **2.0.x** (2.0.0 GA / 2.0.2) | **3.x** (`tools.jackson.*`) | 2.2.21 |
| 1.5.0 | 2026-08-11 | 4.x | 2.0.0 GA | 3.x | 2.2.21 |
| **1.0.0** (Boot 3 line) | 2026-07-20 | **3.5.14** | **1.1.7** | 2.x (`com.fasterxml.jackson.*`) | 2.x |

- **Spring Boot 4 is now supported.** It landed in **Embabel 1.5.0** (Spring AI 2.0.0 GA migration, Jackson 3, Boot 4 observability packages), not in a 2.0 release — the old "wait for Embabel 2.0 / issue #1052" guidance is obsolete.
- Spring AI is still pulled in transitively by the Embabel starters. **Do not add your own Spring AI BOM or starter** — the Embabel line dictates whether you get Spring AI 1.1.x or 2.0.x.
- Java 21 remains the baseline (Spring Boot 4's own minimum is Java 17).
- Verify before pinning: `https://repo1.maven.org/maven2/com/embabel/agent/embabel-agent-starter/maven-metadata.xml` (`<release>` element), and read the target artifact's POM to confirm the Spring Boot / Spring AI versions it actually drags in.

**Re-verified 2026-09-10: 1.5.1 (2026-08-24) is still the latest release** — no newer tag on GitHub and `<release>` is 1.5.1 for every `com.embabel.agent` artifact. The table above stands.

**Next-release preview (on `main`, NOT released — do not pin against these).** Useful only for planning:

| Landing next | What it changes |
|---|---|
| Boot 4.1.1 + Spring AI 2.0.1 | The compatibility row moves off 4.1.0 / 2.0.0; nothing to do until it ships |
| `AgentProcess` snapshot / restore persistence + `embabel-agent-cache` (`AgentCacheProvider`) | Durable human-in-the-loop: a `WAITING` process survives a node restart. Today `WaitFor` state is in-memory only (see `references/states-and-loops.md`) |
| Agent / Action delay policy | Declarative pacing (rate-limit friendliness) instead of hand-rolled sleeps |
| LLM retry / failure events + shared retry-and-rate-limit policy | Retries become observable through `AgenticEventListener` (§11 pattern) |
| Podman script-execution engine for Agent Skills | Lifts the "skills' `scripts/` are loaded but never executed" limit noted in §17 |

### Migrating a Boot 3.5 + Embabel 1.0.x app to Boot 4 + Embabel 1.5.1

This is a real migration, not a version bump. Budget for it:

| Area | Change |
|---|---|
| Web starter | `spring-boot-starter-web` → `spring-boot-starter-webmvc` (old name deprecated, still resolves) |
| Boot module splits | Explicit deps now needed: `spring-boot-jackson` (`JacksonAutoConfiguration`), `spring-boot-webmvc-test` (`@AutoConfigureMockMvc`), `spring-boot-security`. `spring-boot-starter-classic` / `spring-boot-starter-test-classic` are the "give me everything back" escape hatches |
| Jackson 3 | Package rebrand `com.fasterxml.jackson.*` → `tools.jackson.*`; `ObjectMapper` is immutable — build with `JsonMapper.builder()`; `registerKotlinModule()` → `jacksonObjectMapper()`; drop `JavaTimeModule` (built in) |
| Spring AI 2.0 model factories | Provider facades deleted in favour of vendor SDKs: `.openAiApi()` → `.openAiClient()`, `.anthropicApi()` → `.anthropicClient()`, `.defaultOptions()` → `.options()`; retry templates are no longer passed to model builders |
| Validation | `javax.validation` → `jakarta.validation` |
| Tests | JUnit 6; drop nullable type args such as `assertThrows<X?>` |

Reference: the `Spring Boot 4 / Spring AI 2.0 Migration` page on the embabel/embabel-agent wiki (written against the 2.0.0 dev branch that shipped as the 1.5.x line — treat its version table as historical, its API cheat sheet as current).

## Development Workflow

### Prerequisite Check (shortcut)

If the target project **already contains `embabel-agent-starter`** in its build file (pom.xml or build.gradle):

- **Skip step 1** (Fit Assessment) — the decision to use Embabel has already been made.
- **Skip step 5** (Spring Boot + Spring AI Wiring) — starter and model provider are already configured.
- Start directly from **step 2 (GOAP Modeling)**.
- Only run step 1 if the user explicitly asks to re-evaluate whether Embabel is the right fit.

If the project does **not** have the Embabel starter, verify that the scenario genuinely needs Embabel (step 1) before proceeding.

Follow this order unless the user explicitly asks for a narrower review:

1. **Decide if Embabel is appropriate.**
   Use Embabel when the workflow has multiple steps, typed business objects, conditions, review gates, or auditability needs. Use plain Spring AI when it is a single model call or a fixed pipeline.

2. **Model the workflow with GOAP — design at least two goals.**
   Create an action table with `Action`, `Preconditions`, `Postconditions`, executor type, failure path, and terminal goal. **Always design two `@AchievesGoal`: one normal-completion goal and one safe-failure goal** (e.g. `EscalationTicket`). Without a safe-failure exit, missing data or all-paths-failed leaves the planner with no reachable goal → STUCK or infinite replanning. For fallback / different-performance paths, make success and failure produce **different types** and use `@Action(cost=...)` / `@Cost` so the planner prefers the cheap main path and reroutes on failure. See `references/patterns-and-templates.md` → "Multi-Action design patterns (fallback / different performance / safe exit)".

3. **Define Java data boundaries.**
   Use records or domain classes for all Blackboard facts. Avoid `Map<String,Object>` and magic string keys.

4. **Implement the agent surface.**
   Use `@Agent`, `@Action`, and `@AchievesGoal`. Method input types are preconditions; return types are postconditions.

5. **Wire Spring Boot and Spring AI.**
   Add Embabel starters and a model-provider starter; auto-configuration enables agent scanning (`@EnableAgents` is **not** required — see "Setup Essentials"). Add `@ConfigurationProperties` for business thresholds and model choices.

6. **Attach tools deliberately.**
   Prefer domain-object `@Tool` for calculations on existing business objects. Use MCP for cross-application reusable tools. Use Agentic RAG / `ToolishRag` for document search where the LLM should decide how to search.

7. **Test the flow before declaring done.**
   Test pure actions with normal unit tests. Test LLM actions by checking prompt construction, model role, exposed tools, and guardrails. Add an integration test for `input -> goal output`.

8. **Add observability and failure handling.**
   Record action execution, LLM call metadata, tool invocation, token/cost where available, failure reason, and audit IDs.

For the full step-by-step checklist, read `references/development-workflow.md`.

## What To Read When

- Read `references/development-workflow.md` when designing or implementing a new Embabel agent.
- Read `references/patterns-and-templates.md` when generating Java records, `@Agent` classes, `application.yml`, Maven snippets, or AI assistant prompts.
- Read `references/testing-and-troubleshooting.md` when adding tests, reviewing reliability, or debugging a stuck plan.
- Read `references/conditions-and-guardrails.md` when an action needs boolean preconditions (`@Condition`, SpEL) or when adding input/output validation guardrails to LLM calls.
- Read `references/states-and-loops.md` when the workflow has looping, branching, human-in-the-loop (`WaitFor`), or state-machine patterns (`@State`).
- Read `references/advanced-features.md` when:
  - You need **Embabel Shell** commands for interactive testing and debugging.
  - An action should fire only on a **specific event** (not just parameter availability) → `trigger` / Reactive Triggers.
  - Agents exposed as **MCP tools need security** → `@SecureAgentTool`.
  - A tool itself needs to **orchestrate sub-tools via LLM** → Agentic Tools (`SimpleAgenticTool`, `PlaybookTool`, `StateMachineTool`).
  - Too many tools overwhelm the LLM and you need **progressive disclosure** → `UnfoldingTool`.
  - Complex prompts require **template management** → Jinja Templates (`rendering()`).
  - Independent sub-tasks should run in **parallel** → `ConcurrentAgentProcess`.
  - The UI needs **streaming output** from the LLM → `StreamingPromptRunnerBuilder`.
  - You need to **inspect LLM reasoning** or validate thinking → `ThinkingResponse` / `ThinkingBlock`.
  - You need to **monitor, log, or transform** LLM/tool interactions → Tool Loop Callbacks / Interceptors.
  - You need to **track LLM cost** or enforce a **budget guardrail** → Cost Tracking / `AgenticEventListener`.
  - You need to **publish agents as MCP servers** or integrate with external systems → MCP Publishing / A2A.
  - You need to **stream GOAP step progress to a UI (SSE)** → §13 Real-time Progress Observability (note: plan/action events only flow through the global listener, not per-call listeners).
  - A **compound / cross-domain query** should trigger **multiple agents and fuse** results → §14 Multi-Agent Orchestration (`chooseAndRunAgent` vs `runAgent(input,opts,agent)`; use distinct output types when fusing to avoid ambiguity).
  - Charts/data must **adapt to query intent** (this-vs-last month, filter by tier/priority) → §15 Intent Parameterization (LLM extracts structured parameters + Java applies deterministic filtering).
  - A **fixed set of branches must run in parallel inside one step** and be fused → §16 Workflow DSL (`ScatterGatherBuilder`, default `maxConcurrency` 6); several models must **vote/agree** → `ConsensusBuilder`; a step must **retry until an evaluator accepts it** → `RepeatUntilAcceptableBuilder`.
  - Reusable **skill packages** (`SKILL.md` from GitHub or a local directory) must be handed to an LLM, or reference knowledge must be injected without the model deciding to ask → §17 Agent Skills (`Skills` as `LlmReference`, `EmbeddingSkillSelector`).
  - A **model provider** must be added, or the end user supplies their **own API key** → §18 Providers / BYOK (note: BYOK calls report zero cost, so budget guardrails cannot rely on cost tracking).

## Setup Essentials (verified against the official template — required vs not needed)

Cross-check against [embabel/java-agent-template](https://github.com/embabel/java-agent-template) to avoid redundant annotations and STUCK traps. See `references/patterns-and-templates.md` for the full table and STUCK troubleshooting.

**✅ Required:**

- App class: `@SpringBootApplication` (sufficient on its own; auto-config scans agents automatically)
- Agent class: `@Agent(description=...)` (itself a Spring stereotype)
- LLM calls: `Ai` as an `@Action` parameter, using `ai.withDefaultLlm().creating(T.class).fromPrompt(...)` or `.generateText(...)`
- Controller calls: `AgentInvocation.builder(platform).build(T.class).invoke(domainObject)`

**❌ Not needed / will break:**

- Adding `@EnableAgents` on the app (not required in 0.4.x; only for advanced configuration)
- Adding `@Component` on the agent (`@Agent` is already a stereotype; redundant)
- Using Spring AI `ChatClient` (no `ChatModel` bean exists → `UnsatisfiedDependencyException`; always go through `Ai`)
- `runAgentFrom(agent, opts, Map.of(...))` (a Map does not create typed facts → STUCK)
- `process.start(p).join()` (the future never completes when stuck → infinite wait; use `AgentInvocation` instead)

## Hard Rules

- Keep LLM reasoning local to actions; do not let the LLM own the full business workflow.
- Keep action signatures type-driven; the planner cannot reason over hidden map keys.
- Put numeric calculations, policy thresholds, and permission checks in Java code or domain tools, not in prompt text.
- Expose the minimum tools needed for each LLM call.
- Put business thresholds in configuration, not hard-coded prompt strings.
- Add Chinese function-level comments when generating project code for this user's workspace, unless the target project has a stronger local convention.
- Do not call an implementation complete until build/test commands or equivalent project verification have run.

## Output Shape

When asked to design or implement, produce these artifacts as appropriate:

- `GOAP action table`
- domain records/classes
- `@Agent` class with `@Action` methods
- Spring Boot dependency/config notes
- tool exposure plan
- test checklist and runnable tests
- observability/audit checklist
- version-compatibility notes and unresolved assumptions
