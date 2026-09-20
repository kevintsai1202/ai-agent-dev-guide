# System One (Jev) Integration Guide in Embabel

This guide explains how to integrate **TypeSafe Jev (a System One model)** into Embabel (JVM / Spring Boot 4.1.x and 3.5.x, Java 21) applications as high-speed, low-cost judgment nodes, implementing a cognitive Dual-Process Architecture.

---

## 1. Dual-Process Architecture Overview

| Dimension | System One (Jev) | System Two (Embabel + Flagship LLM) |
| :--- | :--- | :--- |
| **Cognitive Style** | Intuitive, reflexive, fast semantic judgment | Deliberative, goal-directed, multi-step planning |
| **Core Responsibilities** | Condition gates (`@Condition`), routing, guardrails, reranking | GOAP planning, Blackboard state evolution, complex text/code generation |
| **Output Format** | **Strongly typed probabilities and discrete choices** (Choice, Noul, Score) | Free-form text, multi-turn dialogue, tool-call loops |
| **Latency & Cost** | **50ms – 150ms** / **$0.042/M tokens (free output)** | 1,500ms – 5,000ms / $3.00 – $15.00/M tokens |

> **Golden Rule**: Whenever a step does not require generating novel prose or multi-step reasoning—and only requires picking from defined options, testing whether a condition holds, or rating a rubric level—delegate it to Jev. Reserve expensive System Two LLMs for actions that truly require deep reasoning and creative generation.

---

## 2. API Key and Configuration

### A. Local Environment File (`.env`)

Add your API key to your root `.env` file:

```bash
# In .env
JEV_API_KEY=apikey_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
# Alternatively, TYPESAFE_API_KEY is also supported as a fallback
TYPESAFE_API_KEY=apikey_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### B. Spring Boot Configuration (`application.yml` / `application.properties`)

Map the environment variable in `application.yml` with dual-fallback support:

```yaml
typesafe:
  jev:
    # Priority: JEV_API_KEY -> TYPESAFE_API_KEY -> empty string
    api-key: ${JEV_API_KEY:${TYPESAFE_API_KEY:}}
    base-url: https://api.typesafe.ai
    default-model: jev-latest
    timeout: 3s
```

Or in `application.properties`:

```properties
typesafe.jev.api-key=${JEV_API_KEY:${TYPESAFE_API_KEY:}}
typesafe.jev.base-url=https://api.typesafe.ai
typesafe.jev.default-model=jev-latest
typesafe.jev.timeout=3s
```

### C. Spring Boot Relaxed Binding (Direct Env Var)

Spring Boot automatically binds environment variables to `@ConfigurationProperties` using relaxed binding rules. Without defining placeholders in YAML, you can set:

* **`TYPESAFE_JEV_API_KEY`** (or `TYPESAFE_JEV_APIKEY`)
* Spring Boot automatically binds this directly to `typesafe.jev.api-key`.

### D. Local Development: Loading `.env` into Spring Boot

Because Spring Boot does not load `.env` files automatically by default, choose one of the following approaches during local development:

1. **IDE Plugin (Recommended)**:
   * **IntelliJ IDEA**: Install the **EnvFile** plugin, open your Spring Boot Run/Debug configuration, enable "EnvFile", and select your `.env` file.
   * **VS Code**: Install the **DotENV** extension and reference `.env` in `.vscode/launch.json` under `envFile: "${workspaceFolder}/.env"`.

2. **Terminal Session Export**:
   * **PowerShell (Windows)**:
     ```powershell
     # Load .env into the current PowerShell session
     Get-Content .env | ForEach-Object {
         if ($_ -match '^([^#=]+)=(.*)$') {
             [System.Environment]::SetEnvironmentVariable($matches[1].Trim(), $matches[2].Trim())
         }
     }
     # Start the application
     ./mvnw spring-boot:run
     ```
   * **Bash / Zsh (Linux / macOS / Git Bash)**:
     ```bash
     export $(grep -v '^#' .env | xargs)
     ./mvnw spring-boot:run
     ```

3. **Programmatic Auto-Loading (`dotenv-java`)**:
   Add the dependency to `pom.xml`:
   ```xml
   <dependency>
       <groupId>io.github.cdimascio</groupId>
       <artifactId>dotenv-java</artifactId>
       <version>3.1.2</version>
   </dependency>
   ```
   And load it in your `main` method before `SpringApplication.run`:
   ```java
   public static void main(String[] args) {
       io.github.cdimascio.dotenv.Dotenv.configure()
           .ignoreIfMissing()
           .systemProperties(); // Populates System.setProperty so Spring reads it
       SpringApplication.run(AgentApplication.class, args);
   }
   ```

### E. Production Deployment (Docker / Zeabur / Kubernetes)

Inject the environment variable into the runtime container or platform:

* **Docker Compose**:
  ```yaml
  services:
    agent-backend:
      image: my-embabel-agent:latest
      environment:
        - JEV_API_KEY=${JEV_API_KEY}
  ```

* **Zeabur / Cloud Platform**:
  Add an environment variable in the service's **Variables** dashboard:
  * Key: `JEV_API_KEY` (or `TYPESAFE_JEV_API_KEY`)
  * Value: `apikey_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

* **Kubernetes Secret**:
  ```yaml
  env:
    - name: TYPESAFE_JEV_API_KEY
      valueFrom:
        secretKeyRef:
          name: jev-secrets
          key: api-key
  ```

---

## 3. Java 21 Typed Records Specification

The Jev HTTP API (`POST https://api.typesafe.ai/v1/systemone`) returns structured JSON. Below are the Java 21 `sealed interface` and `record` definitions with standard Jackson annotations (compatible with Jackson 2 in Spring Boot 3 and Jackson 3 in Spring Boot 4).

### A. Request Records

```java
package com.example.agent.jev.model;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;
import java.util.List;
import java.util.Map;

/**
 * Top-level Jev API request object.
 *
 * @param state The context to evaluate (String, custom record, or Map)
 * @param model Model identifier, defaults to "jev-latest"
 * @param questions Map of named questions keyed by developer-defined IDs
 */
@JsonInclude(JsonInclude.Include.NON_NULL)
public record JevRequest(
    Object state,
    String model,
    Map<String, JevQuestion> questions
) {
    public static final String DEFAULT_MODEL = "jev-latest";

    /**
     * Factory method using the default model.
     */
    public static JevRequest of(Object state, Map<String, JevQuestion> questions) {
        return new JevRequest(state, DEFAULT_MODEL, questions);
    }
}

/**
 * Sealed interface for Jev question types.
 */
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = NoulQuestion.class, name = "noul"),
    @JsonSubTypes.Type(value = ChoiceQuestion.class, name = "choice"),
    @JsonSubTypes.Type(value = ScoreQuestion.class, name = "score")
})
@JsonInclude(JsonInclude.Include.NON_NULL)
public sealed interface JevQuestion permits NoulQuestion, ChoiceQuestion, ScoreQuestion {
    /** Returns question type (noul, choice, score) */
    String type();
    /** Returns question instructions */
    Object instructions();
}

/**
 * Noul: Binary yes/no condition judgment (returns probability 0.0 to 1.0).
 *
 * @param instructions Question instructions
 * @param criteria Optional descriptions for true and false outcomes
 */
public record NoulQuestion(
    Object instructions,
    Map<String, Object> criteria
) implements JevQuestion {
    public NoulQuestion(Object instructions) {
        this(instructions, null);
    }

    @Override
    public String type() { return "noul"; }
}

/**
 * Choice: Selects one option from a defined set (up to 255 options).
 *
 * @param instructions Selection instructions
 * @param criteria Map of option names to rubric descriptions
 */
public record ChoiceQuestion(
    Object instructions,
    Map<String, Object> criteria
) implements JevQuestion {
    @Override
    public String type() { return "choice"; }
}

/**
 * Score: Evaluates state along an ordered rubric (2 to 10 levels).
 *
 * @param instructions Rating instructions
 * @param criteria Ordered list of rubric level descriptions
 */
public record ScoreQuestion(
    Object instructions,
    List<Object> criteria
) implements JevQuestion {
    @Override
    public String type() { return "score"; }
}
```

### B. Response Records

```java
package com.example.agent.jev.model;

import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;
import java.util.Map;

/**
 * Top-level Jev API response object.
 *
 * @param model The model version that performed the evaluation
 * @param answers Evaluation results keyed by question ID
 * @param usage Token usage statistics
 */
public record JevResponse(
    String model,
    Map<String, JevAnswer> answers,
    JevUsage usage
) {}

/**
 * Sealed interface for Jev answer types.
 */
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = NoulAnswer.class, name = "noul"),
    @JsonSubTypes.Type(value = ChoiceAnswer.class, name = "choice"),
    @JsonSubTypes.Type(value = ScoreAnswer.class, name = "score")
})
public sealed interface JevAnswer permits NoulAnswer, ChoiceAnswer, ScoreAnswer {
    /** Returns answer type */
    String type();
}

/**
 * Noul answer: Probability of the condition being true (0.0 to 1.0).
 *
 * @param noul Probability value
 */
public record NoulAnswer(
    double noul
) implements JevAnswer {
    @Override
    public String type() { return "noul"; }

    /**
     * Checks whether the probability meets or exceeds a threshold.
     */
    public boolean isTrue(double threshold) {
        return this.noul >= threshold;
    }
}

/**
 * Choice answer: Selected option with probability distribution.
 *
 * @param choice Highest-probability option name
 * @param probabilities Probability distribution across all options (sums to 1.0)
 * @param confidence Confidence score derived from the distribution
 */
public record ChoiceAnswer(
    String choice,
    Map<String, Double> probabilities,
    double confidence
) implements JevAnswer {
    @Override
    public String type() { return "choice"; }
}

/**
 * Score answer: Probability-weighted position on ordered levels.
 *
 * @param score Weighted continuous score
 * @param legend Mapping of level indices to descriptions
 * @param probabilities Probability distribution across levels
 * @param confidence Confidence score
 */
public record ScoreAnswer(
    double score,
    Map<String, String> legend,
    Map<String, Double> probabilities,
    double confidence
) implements JevAnswer {
    @Override
    public String type() { return "score"; }
}

/**
 * Token usage metadata.
 */
public record JevUsage(
    @JsonProperty("input_tokens") int inputTokens,
    @JsonProperty("output_tokens") int outputTokens
) {}
```

---

## 4. Spring RestClient Client Implementation

Using Spring Framework 6.1+ built-in `RestClient` ensures zero extra third-party dependencies.

### A. Configuration Properties (`JevProperties.java`)

```java
package com.example.agent.jev.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import java.time.Duration;

/**
 * Connection and model configuration properties for Jev.
 */
@ConfigurationProperties(prefix = "typesafe.jev")
public record JevProperties(
    String apiKey,
    String baseUrl,
    String defaultModel,
    Duration timeout
) {
    public JevProperties {
        if (baseUrl == null || baseUrl.isBlank()) {
            baseUrl = "https://api.typesafe.ai";
        }
        if (defaultModel == null || defaultModel.isBlank()) {
            defaultModel = "jev-latest";
        }
        if (timeout == null) {
            timeout = Duration.ofSeconds(5);
        }
    }
}
```

### B. Client Interface (`JevClient.java`)

```java
package com.example.agent.jev;

import com.example.agent.jev.model.*;
import java.util.List;
import java.util.Map;

/**
 * Client interface for TypeSafe Jev System One judgments.
 */
public interface JevClient {

    /** Evaluates a binary yes/no question and returns probability (0.0 to 1.0). */
    double noul(Object state, String question);

    /** Evaluates a binary question with explicit true/false criteria. */
    double noul(Object state, String question, String trueCriteria, String falseCriteria);

    /** Picks one option from a list of option names. */
    ChoiceAnswer choice(Object state, String question, List<String> options);

    /** Picks one option using detailed rubric criteria per option. */
    ChoiceAnswer choice(Object state, String question, Map<String, String> rubric);

    /** Evaluates state against ordered rubric levels. */
    ScoreAnswer score(Object state, String question, List<String> levels);

    /** Evaluates multiple questions in parallel within a single request. */
    JevResponse evaluate(JevRequest request);
}
```

### C. Client Implementation (`SpringRestClientJevClient.java`)

```java
package com.example.agent.jev.impl;

import com.example.agent.jev.JevClient;
import com.example.agent.jev.config.JevProperties;
import com.example.agent.jev.model.*;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.client.SimpleClientHttpRequestFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.stream.Collectors;

/**
 * Standard JevClient implementation backed by Spring RestClient.
 */
@Component
public class SpringRestClientJevClient implements JevClient {

    private final RestClient restClient;
    private final JevProperties properties;

    public SpringRestClientJevClient(JevProperties properties, RestClient.Builder builder) {
        this.properties = properties;

        SimpleClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
        requestFactory.setConnectTimeout((int) properties.timeout().toMillis());
        requestFactory.setReadTimeout((int) properties.timeout().toMillis());

        this.restClient = builder
            .baseUrl(properties.baseUrl())
            .requestFactory(requestFactory)
            .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + properties.apiKey())
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }

    @Override
    public double noul(Object state, String question) {
        return noul(state, question, null, null);
    }

    @Override
    public double noul(Object state, String question, String trueCriteria, String falseCriteria) {
        Map<String, Object> criteria = null;
        if (trueCriteria != null || falseCriteria != null) {
            criteria = new LinkedHashMap<>();
            if (trueCriteria != null) criteria.put("true", trueCriteria);
            if (falseCriteria != null) criteria.put("false", falseCriteria);
        }

        JevRequest request = new JevRequest(
            state,
            properties.defaultModel(),
            Map.of("q", new NoulQuestion(question, criteria))
        );

        JevResponse response = evaluate(request);
        JevAnswer answer = response.answers().get("q");
        if (answer instanceof NoulAnswer noulAnswer) {
            return noulAnswer.noul();
        }
        throw new IllegalStateException("Unexpected answer type: " + (answer != null ? answer.getClass() : "null"));
    }

    @Override
    public ChoiceAnswer choice(Object state, String question, List<String> options) {
        Map<String, Object> criteria = options.stream()
            .collect(Collectors.toMap(opt -> opt, opt -> opt, (a, b) -> a, LinkedHashMap::new));
        return executeChoice(state, question, criteria);
    }

    @Override
    public ChoiceAnswer choice(Object state, String question, Map<String, String> rubric) {
        Map<String, Object> criteria = new LinkedHashMap<>(rubric);
        return executeChoice(state, question, criteria);
    }

    private ChoiceAnswer executeChoice(Object state, String question, Map<String, Object> criteria) {
        JevRequest request = new JevRequest(
            state,
            properties.defaultModel(),
            Map.of("q", new ChoiceQuestion(question, criteria))
        );

        JevResponse response = evaluate(request);
        JevAnswer answer = response.answers().get("q");
        if (answer instanceof ChoiceAnswer choiceAnswer) {
            return choiceAnswer;
        }
        throw new IllegalStateException("Unexpected answer type: " + (answer != null ? answer.getClass() : "null"));
    }

    @Override
    public ScoreAnswer score(Object state, String question, List<String> levels) {
        List<Object> criteria = List.copyOf(levels);
        JevRequest request = new JevRequest(
            state,
            properties.defaultModel(),
            Map.of("q", new ScoreQuestion(question, criteria))
        );

        JevResponse response = evaluate(request);
        JevAnswer answer = response.answers().get("q");
        if (answer instanceof ScoreAnswer scoreAnswer) {
            return scoreAnswer;
        }
        throw new IllegalStateException("Unexpected answer type: " + (answer != null ? answer.getClass() : "null"));
    }

    @Override
    public JevResponse evaluate(JevRequest request) {
        Objects.requireNonNull(request, "JevRequest must not be null");
        return restClient.post()
            .uri("/v1/systemone")
            .body(request)
            .retrieve()
            .body(JevResponse.class);
    }
}
```

---

## 5. Embabel Integration Patterns

### Pattern 1: Semantic Condition Gate (`@Condition`)

**Problem**: `@Condition` is evaluated repeatedly by the GOAP planner. Calling a standard LLM inside `@Condition` freezes the planner; pure code cannot handle semantic nuance.  
**Solution**: Use Jev `Noul` for a 50ms semantic probability check.

```java
package com.example.agent.conditions;

import com.example.agent.jev.JevClient;
import com.example.agent.model.CustomerInquiry;
import com.embabel.agent.api.annotation.Condition;
import org.springframework.stereotype.Component;

@Component
public class InquiryConditions {

    private final JevClient jevClient;

    public InquiryConditions(JevClient jevClient) {
        this.jevClient = jevClient;
    }

    /**
     * Determines whether the customer inquiry carries urgent legal or regulatory risk.
     * Evaluated in ~50ms without stalling the GOAP planner.
     */
    @Condition
    public boolean hasLegalRisk(CustomerInquiry inquiry) {
        double legalRiskProb = jevClient.noul(
            inquiry.content(),
            "Does this inquiry mention legal action, regulatory violation, or attorney involvement?",
            "Explicit or strongly implied legal escalation",
            "Ordinary inquiry, complaint, or dissatisfaction"
        );
        return legalRiskProb >= 0.70;
    }
}
```

---

### Pattern 2: Fast Type-Driven Router (`@Action`)

**Problem**: Using an LLM to classify user messages into structured records requires prompt-and-parse, risking JSON errors and high latency.  
**Solution**: Pure-Java `@Action` calling Jev `Choice` to emit typed Blackboard records directly.

```java
package com.example.agent.actions;

import com.example.agent.jev.JevClient;
import com.example.agent.jev.model.ChoiceAnswer;
import com.example.agent.model.*;
import com.embabel.agent.api.annotation.Action;
import com.embabel.agent.api.annotation.Agent;
import java.util.Map;

@Agent(description = "Customer ticket triage agent")
public class TicketTriageAgent {

    private final JevClient jevClient;

    public TicketTriageAgent(JevClient jevClient) {
        this.jevClient = jevClient;
    }

    /**
     * Triages a raw ticket into a specific domain fact record, driving GOAP goal selection.
     */
    @Action
    public Object triageTicket(RawTicket rawTicket) {
        Map<String, String> departmentRubric = Map.of(
            "BILLING", "Refunds, invoices, charge disputes, subscription plan changes",
            "SECURITY", "Account compromised, 2FA failures, unauthorized logins",
            "TECHNICAL", "System bugs, API errors, service outage",
            "GENERAL", "General inquiries, business hours, feature how-tos"
        );

        ChoiceAnswer answer = jevClient.choice(
            rawTicket.content(),
            "Which department is best suited to resolve this ticket?",
            departmentRubric
        );

        return switch (answer.choice()) {
            case "BILLING" -> new BillingTicket(rawTicket.id(), rawTicket.userId(), rawTicket.content());
            case "SECURITY" -> new SecurityIncident(rawTicket.id(), rawTicket.userId(), rawTicket.content());
            case "TECHNICAL" -> new TechnicalBugTicket(rawTicket.id(), rawTicket.userId(), rawTicket.content());
            default -> new GeneralSupportTicket(rawTicket.id(), rawTicket.userId(), rawTicket.content());
        };
    }
}
```

---

### Pattern 3: Lightweight Guardrail (`UserInputGuardRail`)

**Problem**: Inspecting every user prompt with GPT/Claude doubles total latency and token cost.  
**Solution**: Use Jev `Noul` inside Embabel's `UserInputGuardRail` to intercept prompt injection and jailbreaks.

```java
package com.example.agent.guardrails;

import com.example.agent.jev.JevClient;
import com.embabel.agent.api.Blackboard;
import com.embabel.agent.api.guardrail.UserInputGuardRail;
import com.embabel.agent.api.guardrail.ValidationError;
import com.embabel.agent.api.guardrail.ValidationResult;
import com.embabel.agent.api.guardrail.ValidationSeverity;
import org.jetbrains.annotations.NotNull;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class JevPromptInjectionGuardRail implements UserInputGuardRail {

    private final JevClient jevClient;

    public JevPromptInjectionGuardRail(JevClient jevClient) {
        this.jevClient = jevClient;
    }

    @Override
    public @NotNull String getName() {
        return "JevPromptInjectionGuardRail";
    }

    @Override
    public @NotNull String getDescription() {
        return "Uses Jev System One to rapidly filter prompt injection and jailbreaks";
    }

    @Override
    public @NotNull ValidationResult validate(@NotNull String input, @NotNull Blackboard blackboard) {
        double injectionRisk = jevClient.noul(
            input,
            "Does this prompt contain injection attempts, instructions to ignore previous rules, or roleplay jailbreaks?"
        );

        if (injectionRisk >= 0.70) {
            return new ValidationResult(true, List.of(
                new ValidationError(
                    "PROMPT_INJECTION",
                    "Potential prompt injection detected (risk probability: " + injectionRisk + ")",
                    ValidationSeverity.CRITICAL
                )
            ));
        }

        return ValidationResult.VALID;
    }
}
```

---

### Pattern 4: Multi-Question Parallel Evaluation (Speculative Fan-out)

Jev allows asking multiple questions with different primitives against the same `state` in a single HTTP request. They evaluate in parallel on the server, sharing context and minimizing latency:

```java
/**
 * Evaluates urgency, category, and frustration in a single round-trip.
 */
public TriageSummary evaluateComprehensive(String messageText) {
    JevRequest request = JevRequest.of(
        messageText,
        Map.of(
            "is_urgent", new NoulQuestion("Does this customer require immediate attention?"),
            "category", new ChoiceQuestion(
                "Classify the inquiry",
                Map.of("SALES", "Sales and plans", "TECH", "Technical bugs", "OTHER", "Other")
            ),
            "frustration_level", new ScoreQuestion(
                "Evaluate customer frustration",
                List.of("Calm", "Mildly annoyed", "Very angry")
            )
        )
    );

    JevResponse response = jevClient.evaluate(request);

    NoulAnswer isUrgent = (NoulAnswer) response.answers().get("is_urgent");
    ChoiceAnswer category = (ChoiceAnswer) response.answers().get("category");
    ScoreAnswer frustration = (ScoreAnswer) response.answers().get("frustration_level");

    return new TriageSummary(
        isUrgent.isTrue(0.65),
        category.choice(),
        frustration.score()
    );
}
```

---

## 6. Best Practices

1. **Jev is not a text generator**: Never ask Jev to write summaries, explanations, or prose. Generation belongs to Embabel's `ai.withDefaultLlm().generateText(...)`.
2. **`Choice` limit is 255 options**: For large catalogs, use BM25 or vector retrieval to shortlist to Top 20 first, then use Jev for precise selection.
3. **`Score` rubrics should be 3–5 levels**: Descriptions must stand alone with concrete meaning (e.g., `"Calm"`, `"Frustrated"`, `"Very angry"`).
4. **Interpretation of `Noul ≈ 0.5`**: A probability near 0.5 indicates evidence is balanced or ambiguous, not medium intensity. Use a threshold like `≥ 0.70` for actionable decisions.
5. **Set lower `@Action(cost=...)` in GOAP**: Configure pure Jev actions with a low cost (e.g., `cost = 10`) compared to LLM actions (`cost = 100`) to encourage the GOAP planner to favor fast deterministic paths.
