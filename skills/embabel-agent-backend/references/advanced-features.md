# Embabel Advanced Features Reference

> This document covers the advanced or specialized features in the official Embabel documentation. These features are highly useful in specific scenarios,
> but are not commonly needed in typical agent development. **Consult them when the corresponding need arises** — there is no need to master them all at once.

---

## 1. Embabel Shell — Interactive Command-Line Operation (§3)

The fastest way to test an agent during development. After adding the `embabel-agent-starter-shell` dependency, start Spring Boot to enter the shell.

### Core Commands

| Command | Description |
|------|------|
| `help` | List all available commands |
| `execute "text"` | Wrap the text as `UserInput` onto the Blackboard; the planner automatically selects an action |
| `execute -p "text"` | Same as above, but prints the prompt sent to the LLM |
| `execute -r "text"` | Same as above, but prints the raw LLM response |
| `execute -p -r "text"` | Prints both the prompt and the response — maximum visibility for debugging |
| `!!` | Repeat the previous command — especially handy when iterating on prompts |

### How It Works

`execute "some text"` wraps the string into a `UserInput` object and places it on the Blackboard.
If the agent's first `@Action` method accepts `UserInput` as a parameter, the planner automatically selects it.
This mechanism is identical to a REST controller or webhook — only the source of the `UserInput` differs (Shell vs HTTP request vs event).

---

## 2. Reactive Triggers — Event-Driven Action Triggering (§4.6.7)

The `trigger` parameter of `@Action` makes an action fire only when **the specified type is the most recently added value on the Blackboard**.

### Core Usage

```java
// Fires only when UserMessage is the most recently added object on the Blackboard
@AchievesGoal(description = "Respond to the user message")
@Action(trigger = UserMessage.class)
public Response handleMessage(
    UserMessage message,       // Must be the trigger type
    Conversation conversation  // Must also be on the Blackboard, but need not be the trigger
) {
    return new Response("Received: " + message.content());
}
```

### Difference from a Regular Action

| Mode | Trigger Condition |
|------|---------|
| No `trigger` | Fires once all parameter types are on the Blackboard |
| With `trigger` | In addition to all parameter types being present, **the specified type must also be the most recently added** |

### Applicable Scenarios

- Multi-event handlers: multiple actions handling different event types (EventA vs EventB)
- Distinguishing "data already exists" from "an event just occurred"
- Event-driven or reactive workflows
- **The foundational mechanism for chatbot patterns**: `@Action(trigger = UserMessage.class)` ensures the action fires only on each new message

---

## 3. @SecureAgentTool — Security Authorization Control (§4.6.11)

Declares a security contract on an `@Action` method or `@Agent` class. It accepts a Spring Security SpEL expression and evaluates `Authentication` **before** the GOAP planner executes the action body.

### Core Usage

```java
// Class level — protects all @Action methods (including intermediate steps)
@Agent(description = "Research a topic and produce a news digest")
@SecureAgentTool("hasAuthority('news:read')")
public class NewsDigestAgent {

    @Action
    public NewsTopic extractTopic(UserInput userInput, OperationContext context) { ... }

    @AchievesGoal(description = "Produce a news digest",
        export = @Export(remote = true, name = "newsDigest",
                         startingInputTypes = {UserInput.class}))
    @Action
    public NewsDigest produceDigest(NewsTopic topic, OperationContext context) { ... }
}

// Method level — fine-grained control (takes precedence over class level)
@SecureAgentTool("hasRole('ADMIN')")
@Action
public SensitiveReport generateReport(ReportRequest req) { ... }
```

### Notes

- Method-level annotations take precedence over class-level ones
- Intermediate actions without `@SecureAgentTool` execute freely when there is no class-level protection
- Primarily used for Agents exposed as remote MCP tools

---

## 4. Agentic Tools — In-Tool Nested LLM Orchestration (§4.9.7)

An **Agentic Tool** is a tool that uses an LLM to orchestrate other tools. Unlike ordinary tools that execute deterministic logic,
an agentic tool delegates to the LLM to decide which sub-tools to call.

### Three Types of Agentic Tool

| Type | Tool Availability | Applicable Scenario | Example |
|------|-----------|---------|------|
| `SimpleAgenticTool` | All tools immediately available | Simple orchestration, exploratory tasks | Math calculator (add/multiply/divide tools) |
| `PlaybookTool` | Conditional progressive unlocking | Structured workflows, guided flows | Research flow: search → analyze → summarize |
| `StateMachineTool` | Availability based on enum state | Formal state machines, multi-phase flows | Order processing: draft → confirmed → shipped → delivered |

### Core Usage (SimpleAgenticTool)

```java
import com.embabel.agent.api.tool.agentic.simple.SimpleAgenticTool;

// Create an agentic tool
SimpleAgenticTool mathOrchestrator = new SimpleAgenticTool(
        "math-orchestrator", "Orchestrate math operations")
    .withTools(addTool, multiplyTool, divideTool)
    .withParameter(Tool.Parameter.string("expression", "The math expression to evaluate"))
    .withLlm(LlmOptions.withModel("gpt-4"));

// Use it like an ordinary tool
context.ai()
    .withDefaultLlm()
    .withTool(mathOrchestrator)
    .generateText("What is 5 + 3 * 2?");
```

### Shared API (AgenticTool Interface)

| Method | Description |
|------|------|
| `withLlm(LlmOptions)` | Set the LLM model |
| `withSystemPrompt(String)` | Set the system prompt |
| `withSystemPrompt(AgenticSystemPromptCreator)` | Dynamic system prompt (can access ExecutingOperationContext) |
| `withMaxIterations(int)` | Maximum number of tool loop iterations (default 20) |
| `withToolObject(Object)` | Add an `@LlmTool` object |

> **Design recommendation**: Complex workflows (with defined outputs, branching, loops, and state management) should use the GOAP planner, Utility AI, or @State,
> rather than LLM-driven agentic tool orchestration.

---

## 5. Progressive Tools / UnfoldingTool — Progressive Tool Disclosure (§4.9.8)

**Progressive Tools** enable dynamic tool disclosure — first presenting a simplified interface, then revealing more fine-grained tools based on context or LLM intent.

### UnfoldingTool — The Most Common Progressive Tool

`UnfoldingTool` presents a high-level description to the LLM, expanding into its inner tools only after it is called.
It is like unfolding a map — see the whole first, then the details when needed.

```java
import com.embabel.agent.api.tool.progressive.UnfoldingTool;

// Create the inner tools
Tool queryTool = Tool.create("query_table", "Execute a SQL query",
    Tool.InputSchema.of(Tool.Parameter.string("sql", "The SQL query statement")),
    input -> Tool.Result.text("{\"rows\": 5}")
);
Tool insertTool = Tool.create("insert_record", "Insert a new record",
    Tool.InputSchema.of(Tool.Parameter.string("table", "The table name")),
    input -> Tool.Result.text("{\"id\": 123}")
);

// Create the UnfoldingTool facade
var databaseTool = UnfoldingTool.of(
    "database_operations",
    "Use this tool to operate on the database. The specific operations are visible after calling it.",
    List.of(queryTool, insertTool)
);
```

### Applicable Scenarios

- A large number of related tools can make the LLM's choice difficult
- Grouping tools by category (e.g. "database operations", "file operations")
- Letting the LLM express intent first before revealing details
- Reducing token consumption from tool descriptions

### Fluent Builder API

```java
// Compose tools from multiple sources
var combined = UnfoldingTool.of("workspace", "Workspace operations", List.of(baseTool))
    .withTools(searchTool, filterTool)             // Add individual tools
    .withToolObject(new DatabaseOperations())      // Add an @LlmTool class
    .withToolObject(new FileOperations());         // Chained addition
```

---

## 6. Templates / Jinja — Prompt Template Engine (§4.11)

Embabel supports Jinja templates for generating prompts, used via the `PromptRunner.rendering(String)` method.

### Core Usage

```java
// Load the template from classpath:/prompts/factchecker/consolidate_assertions.jinja
DistinctFactualAssertions result = context.ai()
    .withLlm(properties.deduplicationLlm())
    .rendering("factchecker/consolidate_assertions")  // The .jinja extension is added automatically
    .createObject(
        DistinctFactualAssertions.class,
        Map.of(
            "assertions", allAssertions,
            "reasoningWordCount", properties.reasoningWordCount()
        )
    );
```

### Custom Template Renderer

```java
// Load templates from a different source (e.g. a per-user directory or database)
TemplateRenderer perUserRenderer = createRendererForUser(userId);
String result = context.ai()
    .rendering("user-greeting")
    .withTemplateRenderer(perUserRenderer)  // Override the default renderer
    .generateText(Map.of("userName", userName));
```

### Design Recommendation

> Don't rush to externalize prompts. The multi-line strings in modern languages are often easier to maintain.
> Externalization may sacrifice type safety and add complexity. — Official documentation recommendation

---

## 7. Execution Modes — Concurrent Execution Modes (§4.15)

Control the agent execution mode via the `embabel.agent.platform.process-type` setting.

### Two Modes

| Mode | Behavior | Applicable Scenario | Trade-offs |
|------|------|---------|------|
| `SIMPLE` (default) | Selects one best action each time and executes sequentially | Most agents; when action order matters | Predictable; easy to debug; no concurrency overhead |
| `CONCURRENT` | Executes **all achievable actions** in parallel each time | Independent parallel sub-tasks; fan-out/fan-in | High throughput; requires actions to be safe against the shared Blackboard |

### Enabling Concurrent Mode

```yaml
# application.yml
embabel:
  agent:
    platform:
      process-type: CONCURRENT
```

### Replanning in Concurrent Mode

- Multiple parallel actions may throw `ReplanRequestedException` simultaneously
- Only the **first** request is accepted — its Blackboard updates are applied, and the triggering action is temporarily blacklisted
- The blacklist is cleared automatically after a successful plan
- Execution goes through the platform `Asyncer` (Spring managed task executor with virtual threads)

> **This is only one of three parallel mechanisms.** `CONCURRENT` parallelises *the plan*. To fan out
> to N branches **inside a single action** and fuse them, use `ScatterGatherBuilder` / `ConsensusBuilder`
> (§16). To run **several whole agents** and fuse their outputs, use `Autonomy` orchestration (§14).
> See the comparison table at the end of §16.

---

## 8. Working with Streams — Streaming Responses (§4.26)

Supports progressively receiving data from the LLM, including raw text streams, reasoning events (thinking), and structured object streams.

### Core Concepts

| Concept | Description |
|------|------|
| `StreamingEvent` | Wraps either a Thinking event or a user Object |
| `StreamingPromptRunnerBuilder` | A runner with streaming capability |
| Spring Reactive | Reactive support based on the Spring AI ChatClient |

### Core Usage

```java
PromptRunner runner = ai.withDefaultLlm()
    .withToolObject(Tooling.class);

// Object stream + reasoning (thinking)
Flux<StreamingEvent<MonthItem>> results = new StreamingPromptRunnerBuilder(runner)
    .streaming()
    .withPrompt("Florida's two hottest months and their respective record high temperatures")
    .createObjectStreamWithThinking(MonthItem.class);

results
    .timeout(Duration.ofSeconds(150))
    .doOnNext(event -> {
        if (event.isThinking()) {
            logger.info("Thinking: {}", event.getThinking());
        } else if (event.isObject()) {
            logger.info("Received object: {}", event.getObject().getName());
        }
    })
    .doOnComplete(() -> logger.info("Stream complete"))
    .blockLast(Duration.ofSeconds(6000));

// Plain text stream
Flux<String> textStream = new StreamingPromptRunnerBuilder(runner)
    .streaming()
    .withPrompt("What is the tallest building in Paris?")
    .generateStream();
```

---

## 9. Working with LLM Reasoning / Thinking — Reasoning Mode (§4.27)

Obtain the LLM's reasoning process (thinking blocks), useful for validating decision logic or understanding the cause of failures.

### Core Concepts

| Concept | Description |
|------|------|
| `ThinkingBlock` | Carries reasoning details (tag type, tag value, reasoning content) |
| `ThinkingResponse<T>` | A wrapper containing the result object and the reasoning blocks |
| `ThinkingException` | Retains the thinking blocks when object creation fails, aiding debugging |
| `runner.thinking()` | The core API for enabling reasoning extraction |

### Core Usage

```java
// Enable thinking block extraction
ThinkingResponse<MonthItem> response = runner
    .thinking()
    .createObject(prompt, MonthItem.class);

MonthItem result = response.getResult();                    // Structured result
List<ThinkingBlock> blocks = response.getThinkingBlocks();  // Reasoning process

// Handle the possible failure case
ThinkingResponse<MonthItem> response = runner
    .thinking()
    .createObjectIfPossible(prompt, MonthItem.class);
if (response.getResult() == null) {
    // Object creation failed — inspect the reasoning to understand why
    response.getThinkingBlocks().forEach(block ->
        logger.info("LLM reasoning: {}", block.getContent()));
}
```

### Thinking Tag Control (1.5.1)

Filter which XML-tagged reasoning blocks the model produces and returns — **without editing the prompt**:

```java
// Keep only the "analysis" block; the model is auto-instructed (system prompt hint) to use that tag
ThinkingResponse<MonthItem> response = runner
    .thinking(Thinking.withIncludedTags("analysis"))
    .createObject(prompt, MonthItem.class);

// Drop "scratchpad" blocks, keep everything else
ThinkingResponse<MonthItem> response2 = runner
    .thinking(Thinking.withExcludedTags("scratchpad"))
    .createObject(prompt, MonthItem.class);
```

Caveat: these apply **only to named TAG blocks**. Untagged reasoning (PREFIX / NO_PREFIX style) always
passes through, so do not rely on `excludedTags` to keep reasoning out of a response.

### Provider Notes

- Embabel provides a provider-neutral API (`PromptRunner.thinking()` / `LlmOptions.thinking`)
- Under the hood it automatically maps to each provider's specific feature (e.g. Google GenAI's `includeThoughts`)
- The presence and format of reasoning blocks may vary slightly by provider and Spring AI version

---

## 10. Callbacks / Interceptors — Tool Loop Interceptors (§4.28)

### Tool Loop Callbacks

LLM invocations happen inside the `ToolLoop`. Embabel provides two categories of extension point:

| Category | Purpose | Description |
|------|------|------|
| **Inspector** (observer) | Logging, metrics, debugging | Read-only; does not modify state |
| **Transformer** | Truncating tool results, sliding windows, masking sensitive content | Changes what the LLM sees |

### Inspector Callback Points

- `beforeLlmCall` — before the LLM call
- `afterLlmCall` — after the LLM response, before processing tool calls
- `afterToolResult` — after each tool produces a result
- `afterIteration` — after each full iteration

### Built-in Callbacks

| Name | Description |
|------|------|
| `ToolLoopLoggingInspector` | Logs LLM call and tool execution details |
| `ToolResultTruncatingTransformer` | Truncates overly long tool results |
| `SlidingWindowTransformer` | Maintains a sliding window to manage context size |

### Core Usage

```java
var result = ai.withDefaultLlm()
    .withTools(tools)
    .withToolLoopInspectors(callbackTracker, loggingInspector)
    .withToolLoopTransformers(truncatingTransformer, slidingWindowTransformer)
    .creating(RestaurantRecommendation.class)
    .fromPrompt("Recommend an Italian restaurant...");
```

### Tool Call Interceptors (Lightweight Version)

Applicable to both streaming and non-streaming modes, concerned only with individual tool calls (no full loop context required):

```java
PromptRunner runner = ai.withDefaultLlm()
    .withToolObject(new Tooling())
    .withToolCallInspectors(new ToolCallLoggingInspector(
        ToolLoopLoggingInspector.LogLevel.INFO, logger));
```

---

## 11. Cost Tracking / Budget Guardrail — LLM Cost Tracking and Budget Control (§4.29)

### Cost Tracking Events

Embabel emits an event for every LLM and embedding call:

| Event | Trigger Timing |
|------|---------|
| `LlmInvocationEvent` | After each LLM call |
| `EmbeddingInvocationEvent` | After each embedding call |

Each event provides: model name/provider, token count, computed cost, and agent process ID.

### Cost Listener Example

```java
public class OrganizationCostTracker implements AgenticEventListener {

    private final ConcurrentMap<String, DoubleAdder> costPerAgent = new ConcurrentHashMap<>();

    @Override
    public void onProcessEvent(AgentProcessEvent event) {
        if (event instanceof LlmInvocationEvent llm) {
            costPerAgent
                .computeIfAbsent(llm.getAgentProcess().getAgent().getName(),
                    k -> new DoubleAdder())
                .add(llm.getInvocation().cost());
        }
    }
}
```

### Budget Guardrail Pattern

Cost events fire only **after** a call completes, so they cannot stop the current call, but they can stop the **next** one:

1. **Listener counting**: subscribe to `LlmInvocationEvent` and accumulate cost per agent/tenant/user
2. **Guardrail blocking**: `UserInputGuardRail` reads the counter before the next LLM call, and returns `CRITICAL` to block execution if over budget

```text
LLM call ──► LlmInvocationEvent ──┐
                                   ▼
                     counter (per agent / tenant / user)
                                   │
next call ──► UserInputGuardRail reads counter ──────┘
                     │
            over budget? ──► CRITICAL ──► call blocked
```

---

## 12. Integrations — MCP Publishing / A2A / Observability (§4.33)

### MCP Server Publishing

Expose an Embabel Agent as an MCP server for use by MCP clients such as Claude Desktop and VS Code.

#### Configuration

```yaml
spring:
  ai:
    mcp:
      server:
        type: SYNC    # or ASYNC
```

| Mode | Description |
|------|------|
| `SYNC` (default) | Blocking operation; simple and easy to debug |
| `ASYNC` | Non-blocking reactive; high-concurrency throughput |

Transport protocol: SSE (`localhost:8080/sse`). Clients requiring Streamable HTTP can bridge via the `mcpo` proxy.

#### Automatic Publishing

| Published Item | Mechanism |
|---------|------|
| **Tools** | Actions with `@AchievesGoal` + `@Export(remote = true)` are automatically published as MCP tools |
| **Prompts** | Prompt templates are automatically generated based on the goal's `startingInputTypes` |

#### Exposing an Agent Goal as an MCP Tool

```java
@Agent(description = "Provide weather information")
public class WeatherAgent {

    @AchievesGoal(description = "Get the weather")
    @Export(remote = true)  // Automatically becomes an MCP tool
    @Action
    public String getWeather(
        @Param("location") String location,
        @Param("units") String units
    ) {
        return "Weather info: " + location + " (" + units + ")";
    }
}
```

#### Exposing Existing Tools/RAG as MCP Tools

```java
@Configuration
public class RagMcpTools {

    @Bean
    McpToolExport ragTools(SearchOperations searchOperations) {
        var toolishRag = new ToolishRag("docs", "Embabel documentation", searchOperations);
        return McpToolExport.fromLlmReference(toolishRag);
    }
}
```

### Naming Strategies

| Strategy | Example |
|------|------|
| Prefix | `name -> "myservice_" + name` |
| Uppercase | `name -> name.toUpperCase()` |
| Identity (default) | Keep the original name |

---

## 13. Real-Time Progress Observability (SSE Streaming of GOAP Steps) — Field-Tested

Push the agent's execution progress to the front end in real time (showing "current step / total steps / path / elapsed time").

**Key field-test conclusions (Embabel 0.4.0; re-verify against 1.5.x before relying on exact signatures):**

| Event Type | per-call listener (`ProcessOptions.withListener`) | Global listener (`@Component implements AgenticEventListener`) |
|---|---|---|
| `LlmRequestEvent` / `ChatModelCallEvent` / `LlmInvocationEvent` | ✅ Received | ✅ |
| `AgentProcessPlanFormulatedEvent` (planned path) | ❌ **Not received** | ✅ |
| `ActionExecutionStartEvent` / `ActionExecutionResultEvent` (step start/end) | ❌ **Not received** | ✅ |

**Pitfall**: GOAP plan / action lifecycle events are **delivered only to platform-level (global) listeners**, not to the per-call listener registered via `ProcessOptions.withListener` (which only receives nested LLM events).

**Correct approach**: Use a global `@Component` listener as a broker to route events to the current SSE connection:

```java
@Component
class ProgressBroker implements AgenticEventListener {
    private volatile MyForwarder active;   // Single-user sequential scenario; multi-user requires mapping by process id
    void setActive(MyForwarder f) { this.active = f; }
    @Override public void onProcessEvent(AgentProcessEvent e) {
        if (active != null) active.handle(e);
    }
}
```

Accessing events:

- `PlanFormulatedEvent.getPlan().getActions()` → each `getName()` is a step in the path; `getPlan().getGoal().getName()` is the goal.
- `ActionExecutionStartEvent.getAction().getName()` → step "start"; `ActionExecutionResultEvent.getAction()/.getRunningTime().toMillis()` → step "complete" + elapsed time.
- Note that the OODA loop replans after each step, so `PlanFormulatedEvent` fires multiple times; use a flag to take only the first (the full path).
- The UI should compute progress from "steps completed" rather than "steps started" (otherwise the last step shows 100% as soon as it begins, appearing stuck); start → show in progress, result → show complete.

## 14. Multi-Agent Orchestration: Composite Query Fusion — Field-Tested

Enable a cross-domain query like "customer churn risk **and** last month's revenue" to trigger multiple agents simultaneously and fuse them into a single result.

**Two programmatic execution methods (both return `AgentProcessExecution`, from which you can get `getOutput()` and `getAgentProcess().cost()`):**

| Method | Purpose |
|---|---|
| `autonomy.chooseAndRunAgent(intentString, ProcessOptions)` | The ranker **picks one** agent based on intent (demonstrates Autonomy selection) |
| `autonomy.runAgent(inputObject, ProcessOptions, agent)` | Run a **specific designated agent** (use `agentPlatform.agents()` to find the `Agent` by name) |

**Orchestration flow (in the service / controller layer, not inside an agent):**

1. `IntentSplitAgent` (producing the dedicated type `SubQuerySet`) determines single/composite and splits the sub-queries.
2. Single → `chooseAndRunAgent`; composite → call `chooseAndRunAgent` sequentially for each sub-query, collecting multiple results.
3. `FusionAgent` performs the fusion (producing `FusedDashboard` — **the same structure as the main output but a distinct type**, to avoid `AgentInvocation` / type-routing ambiguity).

**Key pitfall**: When multiple agents all produce the same type (e.g. `DashboardSpec`), `AgentInvocation.build(DashboardSpec.class)` becomes **ambiguous**. Solution: use **dedicated output types** for splitting/fusion (`SubQuerySet`, `FusedDashboard`), or use `autonomy.runAgent(..., specificAgent)` to designate the agent.

Cost accumulation: sum the `getAgentProcess().cost() / usage().getTotalTokens() / llmInvocationCount()` of each execution.

## 15. Intent Parameterization: LLM Parameter Extraction + Java Deterministic Filtering — Field-Tested

Solves the problem of "the chart shows fixed data regardless of whether you ask about this month or last month": the data layer should not be hard-coded but driven by query intent.

**Pattern**: Add an LLM "parameter extraction" step before the data computation, producing a structured parameter type, then let Java filter and compute based on the parameters.

```text
UserInput → extractParams (LLM) → AnalysisParams{comparison, periods, tiers, statuses, ...}
          → computeCharts(AnalysisParams) (Java: filter by parameters, empty list = no filter)
          → analyze → render
```

**Division-of-labor principle (echoing the Hard Rule "keep numbers in Java"):**

- **LLM does semantic parsing**: turn relative, fuzzy natural language like "this month / last month" → into absolute values (`2026-05`, `2026-06`).
- **The LLM does not touch the clock**: the baseline for relative time (the list of available months, which one is "this month") is injected into the prompt by the back end (e.g. `availableMonths()`), avoiding reliance on `LocalDate.now()` that is inconsistent with the data.
- **Java does deterministic filtering and computation**: filter and sum by the absolute parameters, guaranteeing correct numbers.
- Use a unified `AnalysisParams` (every dimension field treats "empty list = no filter"); each agent's extraction prompt focuses only on its own domain dimensions (sales → periods, churn → tiers/statuses, ticket → priorities/categories).
- The render step adjusts presentation by parameters (comparison intent → BarChart two-period comparison; trend → LineChart full range).

---

## 16. Workflow DSL — Standard Workflow Builders (fork-join / consensus / loops)

Besides annotation-driven `@Agent` + `@Action`, Embabel ships a **type-safe builder DSL** for self-contained workflows. Reach for it when a step is "one atomic action from the outside, several steps inside" — especially **fan out to N generators in parallel and fuse the results**, which is tedious to express as separate `@Action` methods.

Package: `com.embabel.agent.api.common.workflow.{control, multimodel, loop}` (verified present in 1.5.1).

| Builder | Purpose | Parallel? |
|---|---|---|
| `SimpleAgentBuilder` | Simplest agent: one step, no preconditions/postconditions | No |
| `ScatterGatherBuilder` | **Fork-join**: run N generators in parallel, then consolidate | **Yes** |
| `ConsensusBuilder` | Ask several sources (models / temperatures) and reconcile; a specialization of ScatterGather | **Yes** |
| `RepeatUntilBuilder` | Repeat a step until a condition holds | No |
| `RepeatUntilAcceptableBuilder` | Repeat with a **separate evaluator** producing feedback + acceptance criteria | No |

Every builder ends in one of two ways, with the same API shape:

- `.buildAgent(name, description)` → an `Agent`. Register it as a `@Bean` of type `Agent` — the DSL is **not** auto-scanned the way `@Agent` classes are.
- `.asSubProcess(context)` → build **and run it now** as a subprocess of the current process, so it can be called from inside an `@Action` method. (`.build()` + `context.asSubProcess(...)` is the explicit two-step form.)

Workflow agents are **opaque** (`agent.getOpaque() == true`): their internal actions are not exported to the platform, so they never pollute the parent planner's action space.

### ScatterGatherBuilder — the built-in parallel path

```java
@Action
FactChecks runAndConsolidateFactChecks(
        DistinctFactualAssertions assertions,
        ActionContext context) {
    // One generator per model; these will run in parallel
    var generators = properties.models().stream()
            .map(model -> factCheckWithSingleLlm(model, assertions, context))
            .toList();
    return ScatterGatherBuilder
            .returning(FactChecks.class)       // overall result type
            .fromElements(FactCheck.class)     // element type being gathered
            .generatedBy(generators)           // list of functions run IN PARALLEL
            .consolidatedBy(ctx -> reconcile(ctx.getInput().getResults()))
            .asSubProcess(context);            // run now, inside this action
}
```

- The consolidation lambda reads the gathered elements via `ctx.getInput().getResults()` (a `ResultList`) and returns the overall result type.
- `generatedBy(...)` accepts plain suppliers or functions of `SupplierActionContext<ELEMENT>`, so a generator can be an LLM call, a Java service call, or a nested agent invocation.
- **Concurrency cap**: internally `ScatterGather` calls `context.parallelMap(generators, maxConcurrency)`; the builder default is `ScatterGatherBuilder.DEFAULT_MAX_CONCURRENCY = 6`. Execution runs on the platform `Asyncer` (Spring managed task executor, virtual threads).

### ConsensusBuilder — multi-model agreement

```java
var agent = ConsensusBuilder
        .returning(Age.class)
        .withSources(List.of(               // or .sourcedFrom(List.of(() -> ...)) for plain suppliers
                tac -> askModelA(tac),
                tac -> askModelB(tac)))
        .withConsensusBy(ctx -> average(ctx.getInput().getResults()))
        .buildAgent("ageConsensus", "Reconciles age estimates from several models");
```

Typical use: the same prompt against several `LlmOptions` (different models or temperatures), then take a majority/average — a cheap accuracy guardrail for numeric or classification outputs.

### RepeatUntilAcceptableBuilder — generate / evaluate / retry

```java
var agent = RepeatUntilAcceptableBuilder
        .returning(Report.class)
        .consuming(Person.class)            // optional: declare the input type
        .withMaxIterations(3)               // hard cap — always set one
        .repeating(tac -> {
            var history = tac.getAttemptHistory();   // previous attempts and their feedback
            return writeReport(tac.getInput(), history);
        })
        .withEvaluator(ctx -> {
            var candidate = ctx.getAttemptHistory().resultToEvaluate();
            return new TextFeedback(score(candidate), "why this score");
        })
        .withAcceptanceCriteria(f -> f.getFeedback().getScore() > 0.5)
        .buildAgent("reportWriter", "Writes a report until it passes review");
```

- `RepeatUntilBuilder` is the simpler sibling: `.withMaxIterations(n).repeating(tac -> ...).until(f -> ...)`.
- The generator can read `tac.getAttemptHistory()` / `tac.lastAttempt()` to see prior results **and** the evaluator's feedback — that feedback loop is what makes the retry actually improve.
- Compare with `@State` loops (`references/states-and-loops.md`): use these builders when the loop is self-contained inside one step; use `@State` when the loop spans planner-visible stages or needs human-in-the-loop.

### Choosing between the parallel / multi-agent mechanisms

| Need | Mechanism | Where |
|---|---|---|
| Fan out **inside one action** to N generators of the same element type, then fuse | `ScatterGatherBuilder` / `ConsensusBuilder` | §16 |
| Let the planner run **every currently achievable action** at once | `process-type: CONCURRENT` (`ConcurrentAgentProcess`) | §7 |
| Route a compound query to **several separate agents** and fuse their outputs | `Autonomy.chooseAndRunAgent` / `runAgent(input, opts, agent)` | §14 |
| Hand-rolled parallelism over a collection inside an action | `context.parallelMap(items, maxConcurrency, fn)` | — |

Rule of thumb: `ScatterGather` for a **known, fixed set of branches** in one step; `CONCURRENT` when *the plan itself* has independent branches; `Autonomy` orchestration when the branches are whole agents with their own goals.

---

## 17. Agent Skills — Portable Skill Packages (new in 1.5.1)

Embabel implements the [Agent Skills Specification](https://agentskills.io/specification): a skill is a directory with a `SKILL.md` (YAML frontmatter + markdown instructions) plus optional `scripts/`, `references/`, `assets/`. The `Skills` class implements `LlmReference`, so it plugs straight into a `PromptRunner`.

```java
var skills = new Skills("financial-skills", "Financial analysis skills")
    .withGitHubUrl("https://github.com/owner/repo/tree/main/skills");
    // .withGitHubSkills(owner, repo, skillsPath, branch)  — explicit form
    // .withLocalSkill("/path/to/one-skill")               — a single directory containing SKILL.md
    // .withLocalSkills("/path/to/skills-dir")             — depth-1 scan of immediate subdirectories only

var response = context.ai()
    .withLlm(llm)
    .withReference(skills)
    .withSystemPrompt("You are a helpful financial analyst.")
    .respond(conversation.getMessages());
```

### Two activation modes

| Mode | How it fires | Best for |
|---|---|---|
| **Lazy activation** (default) | Only ~50–100 tokens of metadata per skill enter the system prompt; the model calls the `activate(name)` tool when it decides a skill is relevant. Also exposes `listResources(...)` / `readResource(...)` | **Procedural** skills — "how to do X" |
| **Embedding selection** | `EmbeddingSkillSelector` matches the query against each skill's `description` and injects the instructions directly — no tool call, no model decision, one embedding call | **Reference knowledge** — formulas, domain conventions, terminology a confident model would never think to look up |

```java
var selector = new EmbeddingSkillSelector(embeddingService);
var response = context.ai()
    .withLlm(llm)
    .withPromptContributor(selector.contributorFor(question, skills))   // lazy: unused runner costs no embedding call
    .respond(conversation.getMessages());
```

A skill opts into embedding selection through the spec's open `metadata` map, so it stays portable (other runtimes ignore the key):

```markdown
---
name: financial-metric-formulas
description: Derived financial metrics and how to compute them from statement line items — quick ratio, EBITDA, CAGR, free cash flow.
metadata:
  activation: embedding
---
```

Notes and limits (1.5.1):

- Skills **without** `activation: embedding` are never selected this way — existing skill libraries are unaffected.
- Default threshold `0.30`, maximum 2 skills; both are constructor parameters. Similarity scores are not calibrated across embedding models — retune if you change the model.
- Selection is **fail-open**: an embedding failure yields no skills rather than an error.
- `description` is what gets embedded — write it in the vocabulary of the questions the skill serves.
- Similarity discriminates by **subject, not difficulty**: an embedding-selected skill is injected for *every* question in its subject area, so its instructions must be safe to inject unconditionally.
- Load-time validation: required frontmatter fields, file references in the instructions must exist, and the skill name must match its directory name. Disable the file check with `new DefaultDirectorySkillDefinitionLoader(false)`.
- **`scripts/` are loaded but not executed** in 1.5.1 (a warning is logged); `allowed-tools` is parsed but not enforced.
- Skills compose with other `LlmReference`s (e.g. `LocalDirectory`) via repeated `.withReference(...)`.

---

## 18. Model Providers, BYOK, and Starter Modules

Model support is modular: one starter per provider, each dragging in its matching autoconfigure module. Add **only** the provider(s) you use — never your own Spring AI BOM or starter (see the version table in `SKILL.md`).

Provider starters under `com.embabel.agent` (1.5.1): `embabel-agent-starter-openai`, `-openai-custom`, `-anthropic`, `-gemini`, `-google-genai`, `-bedrock`, `-oci-genai`, `-dashscope` (Alibaba Cloud), `-minimax`, `-zai`, `-deepseek`, `-mistral-ai`, `-ollama`, `-lmstudio`, `-dockermodels`, `-onnx` (local embeddings), plus `-byok`.

Non-model starters worth knowing: `embabel-agent-starter-shell` (§1), `-mcpserver` and `-mcpserver-security` (§12), `-a2a`, `-observability`, `-webmvc`, `-platform`.

### BYOK (bring your own key)

`embabel-agent-starter-byok` is the supported path when the **end user** supplies the key at runtime rather than the deployment holding it:

- The starter already knows the endpoint for every provider it covers; you supply the key per call/session.
- LLM roles resolve through an SPI: your `RoleResolver` beans are consulted first, in `Ordered` order, before the endpoints the starter knows — so a gateway, proxy, or self-hosted endpoint needs no patching.
- BYOK calls are billed to the user's key, so they report `PricingModel.ALL_YOU_CAN_EAT` — **cost tracking (§11) reports zero cost for them**; budget guardrails must not depend on it.
- A deployment still waiting for a key resolves `default-llm` to a placeholder and names the role/model it cannot satisfy, instead of failing obscurely at startup.

---

## Quick Index: When to Consult Which Section

| Need | Consult |
|------|------|
| Quickly testing an agent during development | §1 Shell |
| An action needs to respond to a specific event rather than firing when all parameters are ready | §2 Reactive Triggers |
| An agent exposed as an MCP tool needs authorization control | §3 @SecureAgentTool |
| A tool itself needs an LLM to orchestrate sub-tools | §4 Agentic Tools |
| A large number of tools need grouped, progressive disclosure | §5 Progressive Tools |
| Complex prompts need template-based management | §6 Templates |
| Independent sub-tasks need to run in parallel | §7 Execution Modes (whole plan) / §16 ScatterGather (inside one action) |
| LLM output needs to be returned progressively | §8 Streaming |
| The LLM's reasoning process needs to be validated | §9 Thinking |
| LLM/tool interactions need to be monitored or modified | §10 Callbacks |
| LLM cost needs to be tracked or capped | §11 Cost Tracking |
| An agent needs to be exposed as an MCP server, or cross-system integration is needed | §12 Integrations |
| The front end needs to display GOAP step progress in real time (SSE) | §13 Real-Time Progress Observability |
| A composite query needs to trigger multiple agents and fuse the results | §14 Multi-Agent Orchestration |
| A query needs dynamic filtering (this month vs last month, by tier/priority) | §15 Intent Parameterization |
| A fixed set of branches must fan out in parallel and be fused inside one step | §16 Workflow DSL (`ScatterGatherBuilder`) |
| Several models must vote / agree on one answer | §16 Workflow DSL (`ConsensusBuilder`) |
| A step must retry until an evaluator accepts the result | §16 Workflow DSL (`RepeatUntilAcceptableBuilder`) |
| Reusable skill packages (`SKILL.md`) must be given to an LLM, from GitHub or a local directory | §17 Agent Skills |
| Reference knowledge must be injected without the model deciding to ask for it | §17 Agent Skills (`EmbeddingSkillSelector`) |
| Only specific reasoning tag blocks should be kept or dropped | §9 Thinking Tag Control |
| A model provider must be added, or the end user supplies their own API key | §18 Providers / BYOK |
