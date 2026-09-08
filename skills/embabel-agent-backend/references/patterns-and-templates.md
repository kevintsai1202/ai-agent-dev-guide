# Patterns And Templates

## GOAP Table Prompt

```text
Role: You are an Embabel + Spring AI architect.
Task: Break down the following business workflow into an Embabel GOAP action table. For each action, list the Java method name, preconditions, postconditions, executor (Java service / LLM / rule / RAG / MCP), and the failure or replanning path.
Format: Markdown table + terminal goal + risk list.
Constraint: Do not let the LLM freely decide the entire workflow; amounts, permissions, review, and outbound sending must be controlled by Java rules or a review gate.

Business workflow:
```

## Action Injectable Parameters Cheat Sheet

`@Action` method parameters are injected automatically by the framework — no manual wiring required:

| Parameter type | Description |
|----------|------|
| Domain object (e.g. `CustomerQuery`) | A type-matched object on the Blackboard (also serves as a precondition) |
| `Ai` | LLM operation entry point, providing `withDefaultLlm()` / `withAutoLlm()`, etc. |
| `ActionContext` | Access the Blackboard, send messages, start sub-processes |
| `OperationContext` | Lower-level context (includes Blackboard access) |
| `Blackboard` | Direct access to the shared state area |
| `PromptRunner` | When you need finer-grained control over LLM operations |
| Spring bean (e.g. `TravelActivityReportingService`) | Auto-injected via Spring DI |
| `@ConfigurationProperties` record | Business threshold settings |

Example:

```java
@Action
ActivitySummary summarize(
    TravellerActivity activity,  // domain object on the Blackboard
    Ai ai,                       // LLM operation entry point
    ActivitySummarizerProperties props  // Spring settings
) { ... }
```

## Agent Skeleton

```java
/**
 * Customer care Agent: turns a query request into a reviewed, personalized offer.
 */
@Agent(description = "Customer activity summary and personalized offer")
public class CustomerCareAgent {

    /**
     * Reads travel activity data from the existing reporting service.
     */
    @Action
    TravellerActivity fetchActivity(CustomerQuery query, TravelActivityReportingService service) {
        return service.report(query.customerId());
    }

    /**
     * Summarizes travel activity with the LLM, but the statistics are provided by
     * the @Tool methods of TravellerActivity.
     */
    @Action
    ActivitySummary summarize(TravellerActivity activity, Ai ai, ActivitySummarizerProperties props) {
        return ai.withDefaultLlm()
            .withToolObject(activity)
            .createObject("""
                Summarize the customer activity for internal service staff.
                Max words: %d
                High spender threshold: %.2f
                Frequent traveler threshold: %.2f trips per year
                Customer activity: %s
                """.formatted(
                    props.maxWords(),
                    props.highSpenderThreshold(),
                    props.highTripsPerYearThreshold(),
                    activity
                ), ActivitySummary.class);
    }

    /**
     * Generates an offer draft from the summary; policy data can come from RAG or explicit tools.
     */
    @Action
    OfferDraft proposeOffer(ActivitySummary summary, Ai ai) {
        return ai.withDefaultLlm()
            .createObject("Create an offer draft for this summary: " + summary, OfferDraft.class);
    }

    /**
     * Checks the offer against company rules; the goal is achieved only if it passes.
     */
    @AchievesGoal(description = "Produce a sendable personalized offer")
    @Action
    ReviewedOffer reviewOffer(OfferDraft draft, OfferPolicy policy) {
        policy.assertAllowed(draft);
        return new ReviewedOffer(draft.offer(), "policy");
    }
}
```

Adjust method names and imports to match the actual Embabel version in the project.

## Official Template Verification Points (tested 2026-06, Embabel 0.3.5 / 0.4.0)

> These API observations were field-tested on the 0.x line. The annotation/`Ai` programming model is unchanged through 1.5.x, but re-verify exact builder signatures against the version in your build file — Spring AI 2.0 (Embabel 1.5.x) renamed several provider-facing builders.

Source: [embabel/java-agent-template](https://github.com/embabel/java-agent-template).
The following marks what is **"required" vs. "not needed"**, to avoid common redundant annotations and STUCK traps.

### ✅ Required (omitting these breaks things)

| Item | How to write it | Notes |
|------|------|------|
| App entry class | `@SpringBootApplication` | Alone is enough; Embabel auto-configuration scans agents automatically |
| Agent class | `@Agent(description = "...")` | `@Agent` is itself meta-annotated with `@Component`, so Spring scans it as a bean |
| Terminal Action | `@AchievesGoal(description="...")` + `@Action` | The GOAP goal endpoint; omitting it causes STUCK (no producer for the goal type) |
| Action type chain | parameter type = precondition, return type = postcondition | Every type a downstream needs must be produced by an upstream action, or the planner is STUCK |
| LLM entry point | `Ai ai` as an `@Action` parameter (auto-injected by the framework) | Use Embabel's own `Ai`, **not** Spring AI's `ChatClient` |
| Programmatic controller invocation | `AgentInvocation.builder(platform).build(T.class).invoke(input)` | Runs the whole GOAP synchronously; pass a domain object as input (type-based routing) |

### ❌ Not needed (redundant or misleading)

| Anti-pattern | Why it is not needed |
|--------|-----------|
| Adding `@EnableAgents` on the App | Auto-configuration (0.4.x through 1.5.x) enables it automatically. `@EnableAgents` still exists but is only for advanced settings (`loggingTheme`, `mcpServers`), **not required to enable agents** |
| Adding `@Component` on the Agent class | `@Agent` is already a Spring stereotype; adding `@Component` is redundant |
| Adding a Spring AI BOM / starter yourself | Spring AI is transitively provided by the Embabel starter (2.0.x with Embabel 1.5.x, 1.1.x with 1.0.x); adding your own causes version conflicts |
| Calling the LLM with Spring AI `ChatClient` in a controller | The project has no `ChatModel` bean (Embabel uses its own LLM abstraction), causing `UnsatisfiedDependencyException`. Always call the LLM via the `Ai` component |
| `agentPlatform.runAgentFrom(agent, opts, Map.of("k", v))` | A Map does not create a typed fact on the Blackboard, so the planner finds no starting precondition → STUCK. Use `AgentInvocation.invoke(domainObject)` or `createAgentProcessFrom(agent, opts, obj)` to pass a typed object |
| `process.start(p).join()` to wait for completion | The future from `start()` never completes when the process is stuck, causing an infinite wait. Use `AgentInvocation` (synchronous) or add an `EarlyTerminationPolicy` |

### Two idiomatic Ai output styles (proven by the official template)

```java
// Structured output (recommended when you need a record)
Story s = ai.withLlm(LlmOptions.withAutoLlm().withTemperature(.7))
            .withPromptContributor(Personas.WRITER)
            .creating(Story.class)        // 0.4.x idiom: creating(Class)
            .fromPrompt("...");           //             .fromPrompt(String)

// Plain text output
String review = ai.withAutoLlm()
                  .withPromptContributor(Personas.REVIEWER)
                  .generateText("...");
```

Both `creating(Class).fromPrompt(String)` and `createObject(String, Class)` exist in 0.4.0 (the latter comes from `PromptRunnerOperations`); the official template uses the former. LLM selection: `withDefaultLlm()` / `withAutoLlm()` / `withLlm(LlmOptions...)` / `withLlmByRole("...")`.

### Programmatic controller invocation (replaces the process.run trap)

```java
/**
 * Runs the agent synchronously from a REST controller; GOAP plans the whole action path automatically.
 */
@PostMapping("/generate")
ResponseEntity<DashboardSpec> generate(@RequestBody UserQuery query) {
    AgentInvocation<DashboardSpec> invocation =
        AgentInvocation.builder(agentPlatform).build(DashboardSpec.class);
    DashboardSpec spec = invocation.invoke(query);  // pass a domain object, type-based routing
    return ResponseEntity.ok(spec);
}
```

### STUCK troubleshooting order (battle-tested)

1. Confirm the App class has `@SpringBootApplication` and the Agent class has `@Agent` (debug: `agentPlatform.agents()` should be non-empty).
2. Use a debug endpoint to print each action's `getPreconditions()` / `getEffects()` and confirm the type chain has no breaks.
3. Confirm the call style is `AgentInvocation.invoke(domainObject)`, not `Map.of(...)` or `process.start().join()`.
4. Confirm the terminal action has `@AchievesGoal` and its return type is exactly the T you pass to `build(T.class)`.

## Compute-then-Compose Decomposition Pattern (separating computation from generation)

When a dashboard/report needs **numeric statistics + LLM narrative**, split "computing the numbers" and "assembly/narrative" into two actions:
the Java action computes a typed fact (chart data, statistical metrics), and the LLM action only consumes and assembles it, never computing.
This enforces the Hard Rule "do numeric computation in Java, not in the prompt," avoiding LLM arithmetic errors or fabricated numbers.

```java
/** Pure Java statistics, producing a typed Blackboard fact (funnel stages, monthly revenue series, etc.). */
@Action
ChartData computeChartData(CrmContext ctx, ChartDataService svc) {
    return svc.computeAll();   // all numbers computed in Java
}

/** The LLM only assembles: put the already-computed ChartData into the prompt as JSON, requiring "use directly, do not modify the numbers". */
@AchievesGoal(description = "Generate the dashboard spec")
@Action
DashboardSpec generateDashboard(CrmContext ctx, ChartData chartData, Ai ai) {
    String chartJson = objectMapper.writeValueAsString(chartData);
    return ai.withDefaultLlm().createObject(
        catalogPrompt + "\n=== Pre-computed chart data (use directly, do not change the numbers) ===\n" + chartJson +
        "\nAssemble using FunnelChart/LineChart/...; props.data must come directly from the data above.",
        DashboardSpec.class);
}
```

Verification point: compare the numbers in the final output against the Java computation results (did the LLM copy them faithfully rather than fabricate?).

## Multiple Action Design Patterns (fallback / different performance / safe failure)

The value of GOAP is that **a single goal can have multiple paths**; the planner picks one by cost (A*'s g+h) and automatically replans to a fallback when a path fails. The key is to design these paths with **types**, not with if-else inside a single action.

### 1. Fallback Path — main path fails, replan to the fallback

Make "success" and "failure" produce **different types** so the planner can distinguish them and reroute:

```java
/** Main path: call the live API to fetch data. On success, produces LiveData. */
@Action(cost = 0.2)
LiveData fetchLive(CustomerQuery q, PricingApi api) {
    return api.fetch(q);   // on failure throws → this action produces no fact → planner replans
}

/** Fallback path: when the API is unavailable, read from cache/DB instead. On success, produces CachedData. */
@Action(cost = 0.8)   // higher cost, so the planner tries the main path first
CachedData fetchCached(CustomerQuery q, CacheRepository repo) {
    return repo.lastKnown(q);
}

/** The terminal goal can be reached via either LiveData or CachedData (two @Action methods with the same return type). */
@AchievesGoal(description = "Produce a quote")
@Action Quote quoteFromLive(LiveData d) { return Quote.of(d, "live"); }
@AchievesGoal(description = "Produce a quote")
@Action Quote quoteFromCached(CachedData d) { return Quote.of(d, "cached"); }
```

Key point: use `@Action(cost=...)` so the planner **prefers the low-cost main path** and only takes the high-cost fallback on failure. Do not try/catch inside one action and return the same type — the planner cannot see the branch that way.

### 2. Different Performance / Cost Variants (choose a model or algorithm by situation)

Same output type, different implementations; use a static `cost` or a dynamic `@Cost` (reading Blackboard state) to let the planner choose:

```java
/** Small dataset: use a cheap, fast model. */
@Action(cost = 0.2)
Summary summarizeFast(Dataset d, Ai ai) {
    return ai.withLlm("gpt-5-nano").createObject("Brief summary: " + d, Summary.class);
}

/** Large dataset / high-accuracy need: use a strong model. Dynamic cost: the larger the data, the more "worthwhile" this path. */
@Action
@Cost double bigCost(Dataset d) { return d.rows() > 1000 ? 0.1 : 0.9; }
Summary summarizeAccurate(Dataset d, Ai ai) {
    return ai.withLlm("gpt-5.4").createObject("In-depth analysis: " + d, Summary.class);
}
```

You can also use `ai.withLlmByRole("best" / "cheap")` and map roles to actual models under `embabel.models.roles` in `application.yml`, decoupling model selection from code.

### 3. Safe-Failure Goal — preventing the Agent from being unable to finish

**Design at least two `@AchievesGoal`s for every business flow: one for normal completion and one for safe failure.**
Otherwise, when data is missing or all paths fail, the planner finds no reachable goal → STUCK or infinite replanning.

```java
/** Normal-completion goal. */
@AchievesGoal(description = "Produce a reviewed quote")
@Action ReviewedQuote approve(Quote q, Policy p) {
    p.assertAllowed(q);          // throws if not allowed, producing no ReviewedQuote
    return ReviewedQuote.of(q);
}

/** Safe-failure goal: when review fails / data is missing, produce an EscalationTicket that can end the flow. */
@AchievesGoal(description = "Cannot complete automatically; escalate to a human")
@Action EscalationTicket escalate(Quote q, FailureReason reason) {
    return EscalationTicket.of(q, reason);   // give the planner a guaranteed convergence exit
}
```

Three layers of protection (in order):

| Layer | Mechanism | Responsibility |
|---|---|---|
| 1 | Multi-goal design: a normal-completion + a safe-failure `@AchievesGoal` | Developer (preferred; should be solved at this layer) |
| 2 | `StuckHandler`: when the planner is stuck, supply data and request REPLAN, or report failure | Agent |
| 3 | `EarlyTerminationPolicy` (`ProcessOptions`): cap max action count / cost, the last-resort fuse | Platform |

Interpretation: if layers 2 and 3 fire frequently, the GOAP model design is flawed — go back to layer 1 and redesign the goals and types.

### Design Checklist (multiple actions / multiple goals)

- [ ] Success and failure produce **different types** so the planner can distinguish them and replan.
- [ ] The main path costs less than the fallback path (`@Action(cost=...)` or `@Cost`).
- [ ] At least one normal-completion + one safe-failure `@AchievesGoal`.
- [ ] Failure paths do not require the LLM to fabricate missing data.
- [ ] Integration tests cover all three paths: main, fallback, and safe failure all converge.

## Domain Tool Pattern

```java
/**
 * Travel activity data; statistics are computed in Java to avoid LLM arithmetic.
 */
public record TravellerActivity(String name, Instant from, Instant to, List<Trip> trips) {

    /**
     * Returns the total spend over the period.
     */
    @Tool(description = "Total travel spend in the selected period")
    public float totalSpend() {
        return trips.stream().map(Trip::amount).reduce(0f, Float::sum);
    }

    /**
     * Returns the annualized number of trips.
     */
    @Tool(description = "Trips per year in the selected period")
    public float tripsPerYear() {
        long days = Duration.between(from, to).toDays();
        return days == 0 ? trips.size() : (trips.size() * 365f) / days;
    }
}
```

## RunSubagent Delegation Pattern

Use when a sub-workflow has its own GOAP planning needs and may be reused by multiple parent agents. The subagent shares the parent's Blackboard.

```java
/**
 * Subagent: an activity-analysis specialist with its own multi-step planning.
 */
@Agent(description = "Activity analysis specialist")
public class ActivityAnalyzer {

    /** Reads travel activity from the reporting service. */
    @Action
    TravellerActivity fetch(CustomerQuery q, TravelActivityReportingService svc) {
        return svc.report(q.customerId());
    }

    /** Summarizes the activity with the LLM. */
    @AchievesGoal(description = "Complete the activity summary")
    @Action
    ActivitySummary summarize(TravellerActivity a, Ai ai) { /* ... */ }
}

/**
 * Parent Agent: delegates to the subagent inside an @Action.
 */
@Agent(description = "Customer service orchestrator")
public class CustomerCareOrchestrator {

    private final ActivityAnalyzer analyzer;  // Spring-injected

    /** Delegates the activity analysis to the subagent. */
    @Action
    ActivitySummary analyze(CustomerQuery query) {
        return RunSubagent.fromAnnotatedInstance(analyzer, ActivitySummary.class);
    }

    /** Reviews the offer. */
    @AchievesGoal(description = "Produce a sendable offer")
    @Action
    ReviewedOffer review(OfferDraft draft) { /* ... */ }
}
```

Three invocation styles:

| Method | Use when |
|---|---|
| `RunSubagent.fromAnnotatedInstance(bean, Type.class)` | Most common — Spring-injected `@Agent` |
| `RunSubagent.instance(agent)` | Programmatically built Agent |
| `ActionContext.asSubProcess()` | Need `ActionContext` access |

## Subagent Tool Pattern

Use when the LLM (not the developer) should decide whether to delegate to a subagent at runtime. Differs from `RunSubagent` which is developer-determined.

```java
/**
 * Exposes the subagent to the LLM as a Subagent Tool, letting the LLM decide whether to delegate.
 */
// Inside the parent action's prompt runner:
var tool = Subagent.ofClass(ActivityAnalyzer.class)
    .consuming(CustomerQuery.class);

ai.withDefaultLlm()
    .withTool(tool)  // the LLM can decide on its own whether to call it
    .generateText("...");
```

Three styles: `Subagent.ofClass(Agent.class).consuming(Input.class)`, `Subagent.byName("name").consuming(Input.class)`, `Subagent.ofAnnotatedInstance(bean).consuming(Input.class)`.

Rule of thumb: `RunSubagent` = developer writes the delegation; `Subagent Tool` = LLM decides. Use simple `@LlmTool` for deterministic logic that does not need GOAP planning.

## Spring Boot Wiring Checklist

```java
/**
 * Application entry point; enables Embabel agent scanning and configuration binding.
 */
@SpringBootApplication
@EnableConfigurationProperties(ActivitySummarizerProperties.class)
@EnableAgents
public class AntechinusApplication {
    public static void main(String[] args) {
        SpringApplication.run(AntechinusApplication.class, args);
    }
}
```

```java
/**
 * Business threshold settings for summarization and offer generation.
 */
@ConfigurationProperties(prefix = "example.activity-summarizer")
public record ActivitySummarizerProperties(
    int maxWords,
    float highSpenderThreshold,
    float highTripsPerYearThreshold
) {}
```

```yaml
spring:
  application:
    name: customer-care-agent

example:
  activity-summarizer:
    max-words: 80
    high-spender-threshold: 2000.0
    high-trips-per-year-threshold: 10

logging:
  level:
    com.embabel: INFO
```

## AgentInvocation Programmatic Calls

Invoke an agent programmatically from a REST controller or service (rather than via the Shell):

```java
/**
 * Invokes the agent programmatically from a REST controller.
 */
@PostMapping("/analyze")
ResponseEntity<ReviewedOffer> analyze(@RequestBody CustomerQuery query) {
    var invocation = AgentInvocation.builder(agentPlatform)
        .build(ReviewedOffer.class);  // specify the goal type

    ReviewedOffer result = invocation.invoke(query);  // synchronous execution
    return ResponseEntity.ok(result);
}
```

Kotlin equivalent:

```kotlin
val invocation = AgentInvocation.builder(agentPlatform)
    .build<ReviewedOffer>()

val result = invocation.invoke(query)
```

You can also use `Autonomy` to let the LLM dynamically choose an agent:

- **Closed mode**: `autonomy.chooseAndRunAgent(userIntent, ProcessOptions.DEFAULT)` — the LLM picks the most suitable agent
- **Open mode**: `autonomy.chooseAndAccomplishGoal(...)` — the LLM picks the most suitable goal and can compose actions across agents

## Kotlin DSL Comparison

Kotlin can declare an agent via a DSL as an alternative to the annotation model:

```kotlin
/**
 * Kotlin DSL-style agent declaration.
 */
val customerCareAgent = agent {
    name = "CustomerCareAgent"
    description = "Customer activity summary and personalized offer"

    action<CustomerQuery, TravellerActivity>("fetchActivity") {
        // Spring service call
    }

    action<TravellerActivity, ActivitySummary>("summarize") {
        // LLM action
    }

    achievesGoal<OfferDraft, ReviewedOffer>("reviewOffer") {
        description = "Produce a sendable personalized offer"
        // review logic
    }
}
```

The two models can be mixed: the DSL suits rapid prototyping, while the annotation model suits production-grade projects.

## Maven Dependency Shape

Use this as a shape, not as a version authority. Verified combos (2026-08-31): Spring Boot **4.1.x** parent + Embabel **1.5.1**, or Spring Boot **3.5.14** parent + Embabel **1.0.0**. Never pair Boot 4 with Embabel 1.0.x/0.x (startup fails; see troubleshooting). Spring AI comes in transitively from the Embabel starters — do not add a Spring AI BOM or starter yourself.

```xml
<!-- parent: spring-boot-starter-parent 4.1.x (Boot 3 line: 3.5.x + Embabel 1.0.x) -->
<properties>
  <java.version>21</java.version>
  <embabel.version>CHECK-OFFICIAL-DOCS</embabel.version><!-- was 1.5.1 as of 2026-08 -->
</properties>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webmvc</artifactId><!-- Boot 4 name; spring-boot-starter-web on Boot 3.5 -->
  </dependency>
  <dependency>
    <groupId>com.embabel.agent</groupId>
    <artifactId>embabel-agent-starter</artifactId>
    <version>${embabel.version}</version>
  </dependency>
  <dependency>
    <groupId>com.embabel.agent</groupId>
    <artifactId>embabel-agent-starter-openai</artifactId>
    <version>${embabel.version}</version>
  </dependency>
</dependencies>
```

## Review Checklist

- [ ] Each action has typed inputs and a typed return.
- [ ] Terminal action has `@AchievesGoal`.
- [ ] Business thresholds are in properties.
- [ ] Every LLM action has a prompt construction test.
- [ ] Domain calculations are Java methods, not prompt instructions.
- [ ] Tool exposure is minimal per action.
- [ ] A missing input fact cannot cause fabricated data.
- [ ] Logs/audit can reconstruct the action sequence.
