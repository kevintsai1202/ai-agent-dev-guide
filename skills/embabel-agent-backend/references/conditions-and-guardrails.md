# Conditions and Guardrails

## @Condition — explicit boolean preconditions

A method annotated with `@Condition` lets you add a boolean gate at planning time that affects action availability. It complements type-driven preconditions.

### Basic usage

```java
/**
 * Condition method: determines whether this is a high-spending customer.
 * Condition methods should have no side effects, as they may be called multiple times.
 */
@Condition
boolean isHighSpender(TravellerActivity activity) {
    return activity.totalSpend() > 5000;
}
```

- If a parameter is a domain object, the condition automatically returns `false` when no instance of that type exists on the Blackboard.
- It can accept an `OperationContext` to access the Blackboard and infrastructure.

### SpEL dynamic conditions

Besides `@Condition` methods, you can use SpEL expressions directly in the `pre` array of an `@Action`:

```java
/**
 * Run only when urgency > 0.5.
 */
@Action(
    pre = {"spel:assessment.urgency > 0.5"}
)
public void handleUrgentIssue(Issue issue, IssueAssessment assessment) {
    // ...
}
```

A SpEL expression starts with the `spel:` prefix, references Blackboard objects by their camelCase class name, and must return a boolean. Supported forms:

- Simple property comparison: `spel:issueAssessment.urgency > 0.0`
- Type checks: `spel:ghIssue instanceof T(org.kohsuke.github.GHPullRequest)`
- Collection filtering: `spel:newEntity.newEntities.?[...].size() > 0`

---

## Guardrails — input/output validation framework

Guardrails inject validation logic before and after LLM calls, and are Embabel's standard framework for safety guardrails.

### Core concepts

| Interface | When it validates | Description |
|------|---------|------|
| `UserInputGuardRail` | Before the LLM call | Validates user input or the prompt |
| `AssistantMessageGuardRail` | After the LLM response | Validates LLM output (including thinking blocks) |

### ValidationResult and severity

```java
// A validation result contains a set of ValidationErrors
new ValidationResult(true, List.of(
    new ValidationError("policy-violation", "Safety policy violated", ValidationSeverity.CRITICAL)
));
```

| Severity | Behavior |
|---------|------|
| `INFO` | Logged; does not block execution |
| `WARN` | Logged as a warning |
| `CRITICAL` | Throws `GuardRailViolationException`, blocking LLM execution |

### Implementation example — block execution

```java
/**
 * Blocks LLM execution when prohibited content is detected.
 */
class SafetyGuardRail implements UserInputGuardRail {

    @Override
    public @NotNull String getName() { return "SafetyGuardRail"; }

    @Override
    public @NotNull String getDescription() { return "Safety policy check"; }

    @Override
    public @NotNull ValidationResult validate(
            @NotNull String input, @NotNull Blackboard blackboard) {
        if (containsProhibitedContent(input)) {
            return new ValidationResult(true, List.of(
                new ValidationError("safety", "Contains prohibited content",
                    ValidationSeverity.CRITICAL)
            ));
        }
        return ValidationResult.VALID;
    }
}
```

### Attaching guardrails

```java
// Attach on the PromptRunner (multiple can be chained)
PromptRunner runner = ai.withDefaultLlm()
    .withGuardRails(new SafetyGuardRail(), new AuditGuardRail());
```

### Global guardrail configuration

Declare global guardrails in `application.properties` to apply them to all LLM operations:

```properties
# All user input passes through these guardrails
embabel.agent.guardrails.user-input=com.example.ProfanityFilter,com.example.LengthValidator

# All LLM responses pass through these guardrails
embabel.agent.guardrails.assistant-message=com.example.OutputValidator

# Whether to fail-fast if construction fails at startup (default false)
embabel.agent.guardrails.fail-on-error=false
```

Note: global guardrails are POJOs constructed via `BeanUtils.instantiateClass()`, not Spring beans. If you need to access Spring dependencies, perform a lazy lookup through `SpringContextHolder` inside the `validate()` method.

### Design decisions

- **A guardrail's `validate()` can access the `Blackboard`**, so it can validate against other entities in the workflow.
- Global guardrails and per-call `withGuardRails()` are merged automatically.
- Budget Guardrail pattern: combine an `AgenticEventListener` (to count LLM cost) with a `UserInputGuardRail` (to block with CRITICAL when over budget); see official docs §4.29.3.
