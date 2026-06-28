# Embabel + Spring AI Development Workflow

## 1. Fit Assessment

Use Embabel when the business workflow needs planner-controlled execution:

- multiple typed steps
- conditional routing or replanning
- review/approval gates
- access to existing Spring services and domain model
- audit trail for each step
- testable prompt/tool boundaries

Use plain Spring AI when the task is a single LLM call, a fixed pipeline, or a simple `ChatClient` use case. Use Spring AI agent-utils style autonomous tool loops for exploration-heavy tasks where the path does not need strict review.

Decision axis: who decides the next step?

| Path | Next step decided by | Best fit |
|---|---|---|
| Spring AI | developer-written sequence | single call or fixed pipeline |
| agent-utils style loop | LLM | exploratory assistant tasks |
| Embabel | GOAP planner from pre/postconditions | enterprise workflows with review, tests, and audit |

### Choosing an Embabel Built-in Planner

After choosing Embabel, select a planner based on the scenario (set via `@Agent(planner = PlannerType.XXX)`):

| Planner | Best fit | Notes |
|---|---|---|
| **GOAP** (default) | Business workflows with a clear output | Plans a path from the current state to the goal; highest determinism |
| **Utility** | Exploratory / event-driven / Chatbot | Picks the action with the highest net value (value − cost) each step, without knowing the endpoint in advance |
| **Hybrid** | Reducer pipelines (collect → synthesize → stop) | Selects actions like Utility, but stops as soon as the goal is satisfied |
| **Supervisor** | Flexible multi-step | The LLM selects actions based on type schemas and the artifacts collected so far |

Most business scenarios use GOAP. Chatbots / event-reactive systems use Utility AI (paired with `@EmbabelComponent`).

### Chatbot Mode

To build a conversational application (multi-turn dialogue, event reactions), Embabel provides a complete Chatbot mechanism:

- Define conversation actions with `@EmbabelComponent` + `@Action(trigger = UserMessage.class, canRerun = true)`
- Create the chatbot bean with `AgentProcessChatbot.utilityFromPlatform()` (automatically uses the Utility AI planner)
- Supports Jinja prompt templates, Conversation persistence, and `@State` cross-message state management
- See the official rag-demo project for a complete example

See official docs §4.13 (Chatbot Architecture) for details.

## 2. GOAP Modeling

Create a table before code:

| Action | Preconditions | Postconditions | Executor | Failure / replanning path |
|---|---|---|---|---|
| fetchActivity | CustomerQuery | TravellerActivity | Spring service | no data -> escalate (safe-failure goal) |
| fetchCached (fallback) | CustomerQuery | CachedActivity | Spring service | higher cost; only when fetchActivity fails |
| summarize | TravellerActivity | ActivitySummary | LLM action | malformed summary -> retry or stronger model |
| proposeOffer | ActivitySummary | OfferDraft | LLM action | policy gap -> RAG or human |
| reviewOffer (normal goal) | OfferDraft | ReviewedOffer @AchievesGoal | Java rule | rejected -> escalate |
| escalate (safe-failure goal) | FailureReason | EscalationTicket @AchievesGoal | Java | terminal — guarantees the planner can always converge |

Goal: state the terminal Blackboard fact, usually the return type of an `@AchievesGoal` action.
**Design two goals**: a normal-completion goal (`ReviewedOffer`) and a safe-failure goal (`EscalationTicket`). The safe-failure goal is what stops a STUCK plan or an infinite replanning loop — see `patterns-and-templates.md` → "Multiple Action Design Patterns" for fallback / different-performance / safe-failure code.

A*search and cost: The planner uses A* to find the lowest-total-cost path. `g-score` is the cumulative action cost from start (controlled by the developer via `@Action(cost=...)` and `@Cost` dynamic cost). `h-score` is the framework's heuristic estimate of remaining distance. The planner sorts candidates by `f = g + h` and expands the cheapest first.

- `@Action(cost = 0.3)` — static cost, set at declaration time.
- `@Cost` on a method — dynamic cost evaluated at planning time with access to the current Blackboard state (e.g., charge more for large datasets). Return a `double` between 0.0 and 1.0.

Checklist:

- Every action has at least one clear input fact unless it is a trigger/input action.
- Every needed downstream fact is produced by exactly one clear upstream action, or the ambiguity is intentional.
- Failure paths do not require the LLM to invent missing data.
- Replanning is described as a state transition, not as a free-form instruction.
- Design at least two goals per business flow: a normal completion goal and a safe-failure goal (e.g., `EscalationTicket`), to prevent infinite replanning loops.

## 3. Domain Types

Represent Blackboard facts as Java records or domain classes:

- input request record, such as `CustomerQuery`
- retrieved domain state, such as `TravellerActivity`
- LLM output records, such as `ActivitySummary` or `OfferDraft`
- reviewed/approved output, such as `ReviewedOffer`
- audit/event objects, such as `ActionAudit`

Avoid `Map<String,Object>`, untyped JSON blobs, and prompt-only state. Embabel planning depends on typed input/output flow.

## 4. Agent Implementation

Translate the GOAP table into an `@Agent` class:

- agent class = capability group
- `@Action` method = one planner-visible step
- method parameters = required facts / infrastructure
- return type = produced fact
- `@AchievesGoal` = terminal success condition

Keep actions focused. If one action retrieves data, summarizes, reviews, and persists output, split it.

## 4.5 Blackboard, Replanning, And Safety

**Blackboard** is the shared state area between actions. It is not a free-form string map — Embabel binds objects by type and name to method parameters. After each action, the planner re-evaluates the Blackboard (OODA loop: Observe → Orient → Decide → Act) and decides whether the goal is met, whether to continue the current plan, or whether to replan.

**Replanning** happens automatically when an action's result does not match expectations (e.g., `reviewOffer` rejects a draft and does not produce `ReviewedOffer`). The planner searches for an alternative path from the current state — such as `escalateToHuman` producing `EscalationTicket` as a safe-failure goal.

**Three-layer safety against infinite loops:**

| Layer | Mechanism | Responsibility |
|---|---|---|
| 1. Multi-Goal design | Define both a normal-completion `@AchievesGoal` and a safe-failure `@AchievesGoal` so the planner always has an exit | Developer |
| 2. `StuckHandler` interface | Called when the planner finds no executable action. The handler can add data to the Blackboard and request `REPLAN`, or report failure | Agent |
| 3. `EarlyTerminationPolicy` | Set via `ProcessOptions` to cap max actions or max cost. Forces termination as a last-resort circuit breaker | Platform |

Good agents should resolve at layer 1. If `StuckHandler` or `EarlyTerminationPolicy` fires frequently, redesign the GOAP model.

**Subagent delegation (RunSubagent):**

When a sub-workflow needs its own GOAP planning and may be reused across agents, extract it as a separate `@Agent` and invoke it from the parent:

| Method | Use when |
|---|---|
| `RunSubagent.fromAnnotatedInstance(bean, ReturnType.class)` | Most common — pass a Spring-injected `@Agent` instance |
| `RunSubagent.instance(agent)` | Programmatically built Agent (e.g., via `AgentMetadataReader`) |
| `ActionContext.asSubProcess()` | Need `ActionContext` access for extra operations |

Do not split into subagents unless the sub-workflow is reused or has its own GOAP planning needs. A single linear flow in one `@Agent` is fine.

## 5. Spring Boot + Spring AI Wiring

Minimum implementation concerns:

- Java 21+ for the course target; verify the actual project requirement.
- Use **Spring Boot 3.5.x** with `spring-boot-starter-web`. Released Embabel (<= 0.4.0) does not run on Spring Boot 4 (binary-incompatible with Spring Framework 7; startup fails with `NoSuchMethodError: HttpHeaders.addAll`). Boot 4 support arrives with Embabel 2.0 (issue embabel/embabel-agent#1052). `spring-boot-starter-webmvc` is the Boot 4-era name — only relevant after that upgrade.
- Add Embabel agent starter and model-provider starter. Spring AI comes in transitively from Embabel; do not add your own Spring AI BOM (version-conflict risk).
- Add `@EnableAgents` on the application entry point.
- Put API keys in environment variables.
- Put thresholds and model choices in `@ConfigurationProperties`.

Never hard-code current dependency versions from memory. Read the project's `pom.xml`, Gradle files, official Embabel docs, or existing lockfiles.

## 6. Tool Strategy

Choose tools by ownership and scope:

| Mode | Use when | Implementation guidance |
|---|---|---|
| Domain `@Tool` | logic lives on a Java object and needs object state | annotate methods on the domain object; expose with `withToolObject(...)` |
| MCP | tool must be reused across apps or runtimes | configure Spring AI MCP client/server and verify auth/runtime |
| Agentic RAG / ToolishRag | LLM needs to search a document corpus flexibly | provide `SearchOperations`; expose RAG tools only to actions that need them |

Rules:

- Calculations, totals, thresholds, and permissions stay in Java.
- LLM actions receive only the tools needed for that action.
- Tool names/descriptions must be narrow enough that the LLM cannot misuse them.
- For large tool sets, use hierarchical or deferred discovery patterns where supported.

### LLM Configuration Cheat Sheet

Control LLM selection within an action via `Ai` or `PromptRunner`:

| Method | Description |
|------|------|
| `ai.withDefaultLlm()` | Use the model configured in `embabel.models.default-llm` |
| `ai.withAutoLlm()` | Let the framework auto-select the best-fit model |
| `ai.withLlm(LlmOptions.withAutoLlm().withTemperature(0.7))` | Customize the temperature |
| `ai.withLlm("gpt-4o")` | Specify a model name |
| `ai.withLlmByRole("best")` | Map by role (defined under `embabel.models.roles` in application.yml) |

Common `LlmOptions` settings: `withTemperature()`, `withModel()`, `withMaxTokens()`.
Use the `@LlmCall` annotation to specify the model declaratively.

## 7. Testing Workflow

Test in layers:

1. Pure action tests: service calls, null cases, rule checks, review gates.
2. Prompt construction tests: required facts included, thresholds included, model role selected, tools exposed.
3. Tool tests: domain `@Tool` methods return correct values independently of LLM.
4. Planner/integration tests: input fact reaches terminal goal fact.
5. Failure/replanning tests: missing fact or rejected draft takes expected path.

Do not assert exact LLM prose unless the action is deterministic through a fake runner. Assert structure, prompt inputs, tool exposure, guardrails, and resulting record shape.

## 8. Observability And Production Readiness

Capture:

- agent process ID / correlation ID
- selected goal and plan
- each action input/output type
- LLM model, prompt ID, token/cost if available
- tool name, arguments, result summary
- failure reason and retry/replanning result
- reviewer/approval identity for external-facing actions

For support workflows, define the customer complaint drill-down: business ID -> AgentProcess -> action timeline -> LLM/tool call -> final reviewed output.
