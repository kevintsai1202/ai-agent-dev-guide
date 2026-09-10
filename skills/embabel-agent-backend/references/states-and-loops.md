# States and Loops

## Overview

Looping scenarios are hard to express in plain GOAP planning. Embabel's `@State` mechanism lets an agent define state-machine behavior within a GOAP plan, supporting linear stages, branching workflows, looping patterns, and human-in-the-loop.

## How @State coexists with GOAP

Inside each state, GOAP planning works normally. When an action returns a `@State`-annotated class:

1. **Hides the previous state object** (non-state objects such as user data are retained)
2. **Binds the new state** to the Blackboard
3. **Replans**, considering only the actions within the new state
4. Continues executing until the goal is reached

## Basic declaration

```java
/**
 * Annotate a parent interface with @State, and all implementing classes inherit state behavior automatically.
 */
@State
interface Stage {}

/** Assessment stage. */
record AssessStory(String content) implements Stage {
    @Action
    Stage assess() {
        if (isAcceptable()) {
            return new Done(content);
        } else {
            return new ReviseStory(content);
        }
    }
}

/** Revision stage. */
record ReviseStory(String content) implements Stage {
    @Action
    AssessStory revise() {
        return new AssessStory(improvedContent());
    }
}

/** Completion stage. */
record Done(String content) implements Stage {
    @AchievesGoal(description = "Processing complete")
    @Action
    Output complete() {
        return new Output(content);
    }
}
```

Key points:

- `@State` can be placed on an interface, abstract class, or concrete class, and subclasses inherit it automatically.
- A Java record declared inside a class is implicitly static, which makes it well suited as a state class.
- Kotlin data classes are inner classes by default, so they **must be declared top-level**.

## Looping States

If an action needs to return to a state type that has existed before, use `clearBlackboard = true`:

```java
/**
 * Looping processing: clears the Blackboard on each iteration before re-entering the same state type.
 */
@State
record ProcessingState(String data, int iteration) implements LoopOutcome {
    @Action(clearBlackboard = true)  // Enable looping
    LoopOutcome process() {
        if (iteration >= 3) {
            return new DoneState(data);       // Termination condition
        }
        return new ProcessingState(data + "+", iteration + 1);  // Loop
    }
}
```

Without `clearBlackboard = true`, the planner sees the output type already exists and skips the action.

Note: avoid using `clearBlackboard = true` on an `@AchievesGoal` action, because it removes the `hasRun` tracking.

## Staying in the current State

Returning `this` lets an action stay in the same state without transitioning, which suits a chatbot's conversational responses:

```java
@State
record ChitchatState(String context) {
    @Action(canRerun = true)  // Must be true to allow repeated execution
    ChitchatState respond(UserMessage message, Ai ai) {
        var response = ai.generateText("Respond to: " + message.content());
        // Send the response...
        return this;  // Stay in the same state
    }
}
```

## Human-in-the-Loop — WaitFor

`WaitFor.formSubmission()` pauses the agent and waits for human input:

```java
record HumanFeedback(String comments) {}

@State
record AssessStory(UserInput userInput, Story story) implements Stage {

    /** Pause and wait for human feedback. */
    @Action
    HumanFeedback getFeedback() {
        return WaitFor.formSubmission("""
                Please provide feedback on the story
                %s
                """.formatted(story.text()),
                HumanFeedback.class);
    }

    /** Decide the next step based on the feedback. */
    @Action(clearBlackboard = true)
    Stage assess(HumanFeedback feedback, Ai ai) {
        var assessment = ai.withDefaultLlm().createObject(
            "Is this story acceptable? " + story.text() + " Feedback: " + feedback.comments(),
            AssessmentOfHumanFeedback.class);
        if (assessment.acceptable()) {
            return new Done(userInput, story);
        } else {
            return new ReviseStory(userInput, story, feedback);
        }
    }
}
```

Execution flow:

1. The action calls `WaitFor.formSubmission()` → the agent enters the `WAITING` state
2. The framework generates a form based on the record structure
3. The user fills it in and submits → `HumanFeedback` is added to the Blackboard
4. The agent resumes execution

## Passing data across States

When using `clearBlackboard = true`, all required data must be carried through state record fields:

```java
@State
record ReviseStory(
    UserInput userInput,       // Original request
    Story story,               // Current draft
    HumanFeedback feedback,    // Revision instructions
    Properties properties      // Configuration values
) implements Stage { ... }
```

Consider packing configuration values into a single `Properties` record to avoid repeating individual fields across states.

## State class constraints

- Java: must be a **static nested class** (records are implicitly static, so they work directly)
- Kotlin: must be a **top-level class** (inner classes cause serialization problems due to the outer reference)
- Violating this rule throws `IllegalStateException`

## When to use

| Scenario | Recommendation |
|------|------|
| Linear multi-step, each step flowing naturally into the next | ✅ Good fit |
| Branching workflow (decision points leading to different processing paths) | ✅ Good fit |
| Looping pattern (revise-review cycle) | ✅ Good fit (with `clearBlackboard`) |
| Human-in-the-Loop | ✅ Good fit (with `WaitFor`) |
| Pure linear GOAP with no looping need | Not needed; standard `@Action` is enough |
| Self-contained retry loop inside **one** step (generate → evaluate → retry, with an iteration cap) | Prefer the workflow DSL: `RepeatUntilBuilder` / `RepeatUntilAcceptableBuilder` — see `references/advanced-features.md` §16 |

**`@State` vs the `RepeatUntil*` builders**: use `@State` when the loop spans planner-visible stages, needs to persist across turns, or waits on a human (`WaitFor`). Use the builders when the whole loop is one atomic step from the planner's point of view — they give you `withMaxIterations`, an attempt history, and a separate evaluator for free, with no `clearBlackboard` bookkeeping.
