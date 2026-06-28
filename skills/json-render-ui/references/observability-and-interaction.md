# Observability panel and drill-down interaction

## 1. GOAP step-progress panel (showing how the agent works)

Goal: a fixed sidebar showing in real time ⓪ intent classification → ① the agent picked by Autonomy → ② the GOAP step list (running/done + elapsed time) → ③ progress bar → ④ cost. This lets viewers understand "what the backend is doing and how much it costs".

### Backend: where events come from (key landmine)

**GOAP plan/action lifecycle events are only sent to the "platform-level global listener", not to the per-call `ProcessOptions.withListener`** (the latter only receives nested LLM events). So to push progress to "a specific SSE connection", you must:

1. Write a global `@Component` `AgenticEventListener` (broker) that holds a reference to the "currently active SSE listener".
2. Create a per-request `SseProgressListener` (holding that request's emitter) for each request; at request start call `broker.setActive(it)`, at end call `broker.clear(it)`.
3. In `onProcessEvent`, the broker forwards events to the active listener.

Events to listen for → SSE mapping:

```java
if (e instanceof AgentProcessPlanFormulatedEvent p && !planSent) {
    // p.getPlan().getActions() → list of step names; p.getPlan().getGoal().getName() → goal
    send("plan", {steps, goal, total});  planSent = true;
} else if (e instanceof ActionExecutionStartEvent s) {
    send("step", {index: ++counter, name: s.getAction().getName(), status: "running"});
} else if (e instanceof ActionExecutionResultEvent r) {
    send("step", {index: ++done, name: ..., status: "done", durationMs: r.getRunningTime().toMillis()});
}
```

For compound queries, call `listener.reset()` before each sub-agent starts (resets the step counter and planSent=false) so progress counts from the start.

### Frontend: ProgressState

```ts
interface ProgressState {
  agent: string | null;          // the agent picked by Autonomy
  steps: string[]; goal: string | null;
  runningStep: number;           // running (1-based)
  completedSteps: number;
  durations: Record<number, number>; // elapsed ms per step
  compound: boolean; subQueries: string[];
  currentSubAgent: { index; total; query } | null;
  intentStatus: "idle" | "running" | "done";
}
```

- `agent` internal name → display label via an `AGENT_LABELS` map; action internal name → display step via `STEP_LABELS` (an action name may carry a category prefix, so match on the last segment).
- Remember to add to these two maps for every new agent / action, otherwise the panel shows the raw English method name.

## 2. Cost display

General concept: the backend carries `cost / totalTokens / llmCalls` in the `complete` event, accumulated across each run (omit the cost panel if none). ▼ Embabel example:

```java
var p = exec.getAgentProcess();
cost   += p.cost();
tokens += p.usage().getTotalTokens();
llmCalls += p.llmInvocationCount();
```

For compound queries, accumulate every segment — "intent decomposition + each sub-agent + fusion" — so the frontend sees the true total cost and LLM call count.

## 3. Frontend presentation of compound queries / fusion (requires backend multi-agent orchestration; ▼ Embabel example)

- The `intent` event first lets the panel show "single / compound" and the 1st LLM call.
- The `compound` + `subagent` events draw the list "split into N sub-analyses → run one by one → 🔗 LLM fusion", highlighting which cell is currently in progress.
- The fusion result type is `DashboardSpec`, same as a single query (the backend converts `FusedDashboard` back to `DashboardSpec`), so the frontend rendering path is identical with no branching needed.

(For why the backend uses separate output types `SubQuerySet`/`FusedDashboard` to avoid Autonomy ambiguity, see embabel-agent-backend §14.)

## 4. Click drill-down interaction

Lets a dynamically rendered component call back into the App to "regenerate" a new dashboard, chaining into a multi-level analysis flow (e.g. customer list → click a row → 360 analysis of that customer).

### Why React Context instead of the json-render action system

A dynamically rendered component can't reach the App-layer `generate()`. The lowest-coupling bridge is a Context injecting `onDrill` that components consume via `useContext` — without touching json-render's action/dispatch pipeline.

```tsx
// DrillContext: onDrill triggers regeneration; components disable clicks while busy
export const DrillProvider = DrillContext.Provider;
export const useDrill = () => useContext(DrillContext);

// App: wrap outside the Renderer
<DrillProvider value={{ onDrill: handleDrill, busy: isLoading }}>
  <JSONUIProvider registry={registry}><Renderer .../></JSONUIProvider>
</DrillProvider>

// handleDrill: switch back to the dashboard view + regenerate with the new query
const handleDrill = (q: string) => { if (isLoading) return; setView("dashboard"); setInput(q); generate(q); };
```

### drillTemplate: which row drills to what, decided by backend data

A component (such as `DataTable`) takes an optional `drillTemplate` containing `{columnName}` placeholders; when present, rows become clickable, and on click the row's column values are substituted into the template to form a query string, which calls `onDrill`:

```tsx
const buildDrillQuery = (row) =>
  drillTemplate.replace(/\{([^}]+)\}/g, (_, k) => String(row[k.trim()] ?? "")).trim();
```

When the backend deterministically assembles a list spec, it sets this directly: `drillTemplate = "Comprehensive analysis of customer {company}"`. Clicking a row → `"Comprehensive analysis of customer Delta Logistics"` → `generate()` → Autonomy routes to the existing "single-customer 360" agent. **Zero hardcoding on the frontend**: the drill-down target is composed entirely from backend data + an existing agent.

Key points:

- Ignore clicks while `busy` (isLoading) to avoid re-entrancy.
- The `{columnName}` used for template substitution must match the keys of the row object (use `LinkedHashMap` when deterministically building rows to keep column names consistent).
- For tables you don't want clickable (e.g. an LLM-freely-generated order history), just omit `drillTemplate` and it naturally degrades to read-only.

## 5. Regression checklist for panels/interactions

- [ ] Added an agent → update `AGENT_LABELS`, `STEP_LABELS`, otherwise the panel shows the English method name.
- [ ] Added a drillable component → confirm the `drillTemplate`'s `{columnName}` matches the row keys.
- [ ] Compound query → is cost accumulating every segment.
- [ ] The global broker must `clear()` at request end, to avoid events leaking to another connection.
