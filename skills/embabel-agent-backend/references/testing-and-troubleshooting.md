# Testing And Troubleshooting

## Unit Testing Targets

Pure Java actions:

- service lookup success
- service lookup missing data
- policy approval and rejection
- guardrail boundary values
- domain `@Tool` calculation methods

LLM actions:

- prompt includes all required domain facts
- prompt includes configured thresholds
- prompt uses intended model/profile
- prompt exposes only required tools
- structured output target matches record/class fields

## Prompt Runner Test Strategy

Prefer fake or mock prompt runners over live LLM calls for unit tests.

When using framework test utilities, capture:

- rendered prompt
- selected model/role
- tool objects
- tool groups
- structured output class

With Mockito-style tests, verify the fluent chain:

```java
/**
 * Verify that the summarize action exposes only the TravellerActivity tool and sends the prompt.
 */
@Test
void summarizeExposesOnlyActivityTools() {
    Ai ai = Mockito.mock(Ai.class);
    PromptRunner runner = Mockito.mock(PromptRunner.class);
    TravellerActivity activity = sampleActivity();

    when(ai.withDefaultLlm()).thenReturn(runner);
    when(runner.withToolObject(activity)).thenReturn(runner);
    when(runner.createObject(anyString(), eq(ActivitySummary.class)))
        .thenReturn(new ActivitySummary("ok", true, false));

    agent.summarize(activity, ai, props);

    verify(runner).withToolObject(activity);
    verify(runner).createObject(contains("High spender threshold"), eq(ActivitySummary.class));
}
```

Adapt exact API names to the installed Embabel version.

## Integration Testing Targets

- `CustomerQuery` produces `ReviewedOffer` for happy path.
- Missing report data does not fabricate `TravellerActivity`.
- Rejected `OfferDraft` does not produce terminal goal.
- Planner logs show expected action sequence.
- Audit events include input/output type names and correlation ID.

## Common Problems

### Agent flow does not complete

Likely cause: action type flow is broken.

Check:

- no terminal `@AchievesGoal`
- terminal action requires a type nobody produces
- action returns `void` or an unexpected wrapper type
- circular dependency between actions
- custom condition is not declared consistently

Fix: draw the action table again and verify every precondition has a producer.

### Planner selects an unexpected path

Likely cause: ambiguous action outputs or missing cost/condition guidance.

Check:

- multiple actions produce the same fact
- a fallback action is too broadly applicable
- failed action still produces a fact that looks successful

Fix: split success/failure output types or tighten pre/postconditions.

### LLM cannot use a tool

Likely cause: the tool was not exposed on that prompt runner.

Check:

- `withToolObject(...)`, `withToolObjects(...)`, `withTool(...)`, or `withToolGroup(...)` is called on the same runner used for generation.
- domain object methods have Spring AI `@Tool`.
- MCP client/server is actually configured and reachable.
- tool descriptions are narrow and accurate.

### Structured output fields are null

Likely cause: field names or output schema do not match the Java record/class.

Check:

- record component names match expected JSON field names
- nullable lists are normalized to empty lists
- examples in prompt match Java field names
- output validation rejects partial records

### Java/Spring Boot startup fails on Windows

Check:

```powershell
java -version
mvn -v
$env:JAVA_HOME
```

For course projects, ensure Java 21+ is used by both `java` and Maven. Do not trust only system environment UI; verify the shell that runs Maven.

### Startup fails with NoSuchMethodError: HttpHeaders.addAll

Symptom: compile passes, but context startup throws
`java.lang.NoSuchMethodError: 'void org.springframework.http.HttpHeaders.addAll(org.springframework.util.MultiValueMap)'`.

Cause: the project uses Spring Boot 4 (Spring Framework 7) with released Embabel (<= 0.4.0), which is built on Spring Boot 3.5.x + Spring AI 1.1.x. Overriding Spring AI to 2.0 milestones/RCs fails on other removed APIs (`OpenAiApi`, `OpenAiChatModel$Builder.retryTemplate`).

Fix: pin Spring Boot parent to 3.5.x until Embabel 2.0 ships Boot 4 / Spring AI 2.0 support (tracking: embabel/embabel-agent#1052).

### @SpringBootTest class never runs

Maven Surefire excludes `*IT`-suffixed classes by default (those belong to Failsafe). Name context tests `*Test` and confirm the `Tests run:` count actually includes them.

## Done Criteria

Do not mark implementation complete until:

- compile/test command has passed, or the blocker is explicitly reported
- GOAP action table matches code
- terminal goal is covered by a test
- prompt/tool exposure is tested
- version assumptions are documented
- operational audit path is described
