# Live LLM Agent Execution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #160 — Epic: Live LLM agent execution via AgentProvider
**Issue group:** #160

**Goal:** Activate real LLM agents making clinical decisions governed by the CaseHub platform — trust routing, oversight gates, and Merkle audit govern real agent outputs.

**Architecture:** Each agent follows the per-agent SPI pattern: interface + `@DefaultBean` stub + `@ApplicationScoped` LLM implementation. A shared `ClinicalAgentSupport` utility handles prompt building, Jackson JSON parsing, `InvocationComplete` metrics capture, and per-agent fallback. Dependencies `casehub-platform-agent-router` + `casehub-platform-agent-claude` activate the routing agent provider.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-agent-api, Jackson ObjectMapper, Mutiny Multi

## Global Constraints

- Model and backend are configurable per capability via `application.properties` — never hardcoded
- Each agent defines a domain-appropriate conservative fallback on LLM failure (D7)
- `InvocationComplete` metrics serialized as JSON into existing `ComplianceSupplement.detail` field — no platform schema change
- Grade 3 SUSAR scope remains deferred (clinical#76) — LLM maintains Grade 4/5 boundary
- No real LLM calls in tests — `@InjectMock AgentProvider` controls all responses
- `DemoDataSeeder` bypasses LLM agents — startup remains fast and deterministic

---

## Batch 1: Foundation — ClinicalAgentSupport + Bootstrap

### Task 1: ClinicalAgentSupport shared utility + config

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/agent/InvocationMetrics.java`
- Create: `runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentRequest.java`
- Create: `runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentResult.java`
- Create: `runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentConfig.java`
- Create: `runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentSupport.java`
- Test: `runtime/src/test/java/io/casehub/clinical/agent/ClinicalAgentSupportTest.java`

**Interfaces:**
- Consumes: `io.casehub.platform.agent.AgentProvider`, `AgentSessionConfig`, `AgentEvent.TextDelta`, `AgentEvent.InvocationComplete`
- Produces: `ClinicalAgentSupport.invoke(ClinicalAgentRequest<T>)` → `ClinicalAgentResult<T>` — used by all subsequent agent tasks

- [ ] **Step 1: Write InvocationMetrics record**

```java
// runtime/src/main/java/io/casehub/clinical/agent/InvocationMetrics.java
package io.casehub.clinical.agent;

public record InvocationMetrics(
        String model,
        int inputTokens,
        int outputTokens,
        int thinkingTokens,
        int cacheReadTokens,
        int cacheWriteTokens,
        Double totalCostUsd,
        long durationMs,
        long apiDurationMs,
        String sessionId,
        int numTurns,
        boolean isError) {

    public String toJson() {
        return "{\"model\":\"%s\",\"inputTokens\":%d,\"outputTokens\":%d,\"thinkingTokens\":%d,\"totalCostUsd\":%s,\"durationMs\":%d}"
                .formatted(model, inputTokens, outputTokens, thinkingTokens,
                        totalCostUsd != null ? totalCostUsd.toString() : "null", durationMs);
    }
}
```

- [ ] **Step 2: Write ClinicalAgentRequest record**

```java
// runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentRequest.java
package io.casehub.clinical.agent;

public record ClinicalAgentRequest<T>(
        String systemPrompt,
        String userPrompt,
        Class<T> responseClass,
        T fallbackValue,
        String configKey,
        String correlationId) {}
```

- [ ] **Step 3: Write ClinicalAgentResult record**

```java
// runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentResult.java
package io.casehub.clinical.agent;

public record ClinicalAgentResult<T>(
        T response,
        String rawText,
        InvocationMetrics metrics,
        boolean fallbackUsed,
        String failureReason) {

    public static <T> ClinicalAgentResult<T> success(T response, String rawText, InvocationMetrics metrics) {
        return new ClinicalAgentResult<>(response, rawText, metrics, false, null);
    }

    public static <T> ClinicalAgentResult<T> fallback(T fallbackValue, String reason, InvocationMetrics metrics) {
        return new ClinicalAgentResult<>(fallbackValue, null, metrics, true, reason);
    }
}
```

- [ ] **Step 4: Write ClinicalAgentConfig interface**

```java
// runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentConfig.java
package io.casehub.clinical.agent;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import java.time.Duration;
import java.util.Map;
import java.util.Optional;

@ConfigMapping(prefix = "casehub.clinical.agent")
public interface ClinicalAgentConfig {
    Map<String, AgentCapabilityConfig> capabilities();

    interface AgentCapabilityConfig {
        @WithDefault("sonnet")
        String model();

        @WithDefault("PT30S")
        Duration timeout();
    }
}
```

- [ ] **Step 5: Write failing tests for ClinicalAgentSupport**

```java
// runtime/src/test/java/io/casehub/clinical/agent/ClinicalAgentSupportTest.java
package io.casehub.clinical.agent;

import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import io.smallrye.mutiny.Multi;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class ClinicalAgentSupportTest {

    private AgentProvider agentProvider;
    private ClinicalAgentSupport support;

    record TestResponse(String value, int count) {}

    @BeforeEach
    void setup() {
        agentProvider = mock(AgentProvider.class);
        support = new ClinicalAgentSupport(agentProvider, new ObjectMapper());
    }

    @Test
    void validJsonResponse_parsedCorrectly() {
        when(agentProvider.invoke(any())).thenReturn(Multi.createFrom().items(
                new AgentEvent.TextDelta("{\"value\":\"hello\",\"count\":42}"),
                new AgentEvent.InvocationComplete(10, 20, 0, 0, 0, 0.001, 500L, 400L, "sess-1", 1, false)));
        var request = new ClinicalAgentRequest<>("system", "user", TestResponse.class,
                new TestResponse("fallback", 0), "test", "corr-1");
        var result = support.invoke(request);
        assertFalse(result.fallbackUsed());
        assertEquals("hello", result.response().value());
        assertEquals(42, result.response().count());
        assertNotNull(result.metrics());
        assertEquals(10, result.metrics().inputTokens());
    }

    @Test
    void markdownFencedJson_extractedCorrectly() {
        when(agentProvider.invoke(any())).thenReturn(Multi.createFrom().items(
                new AgentEvent.TextDelta("Here is the result:\n```json\n{\"value\":\"hi\",\"count\":1}\n```\n"),
                new AgentEvent.InvocationComplete(5, 10, 0, 0, 0, null, 200L, 150L, "sess-2", 1, false)));
        var request = new ClinicalAgentRequest<>("system", "user", TestResponse.class,
                new TestResponse("fallback", 0), "test", "corr-2");
        var result = support.invoke(request);
        assertFalse(result.fallbackUsed());
        assertEquals("hi", result.response().value());
    }

    @Test
    void malformedJson_returnsFallback() {
        when(agentProvider.invoke(any())).thenReturn(Multi.createFrom().items(
                new AgentEvent.TextDelta("not valid json"),
                new AgentEvent.InvocationComplete(5, 10, 0, 0, 0, null, 200L, 150L, "sess-3", 1, false)));
        var request = new ClinicalAgentRequest<>("system", "user", TestResponse.class,
                new TestResponse("fallback", 0), "test", "corr-3");
        var result = support.invoke(request);
        assertTrue(result.fallbackUsed());
        assertEquals("fallback", result.response().value());
    }

    @Test
    void emptyResponse_returnsFallback() {
        when(agentProvider.invoke(any())).thenReturn(Multi.createFrom().empty());
        var request = new ClinicalAgentRequest<>("system", "user", TestResponse.class,
                new TestResponse("fallback", 0), "test", "corr-4");
        var result = support.invoke(request);
        assertTrue(result.fallbackUsed());
    }

    @Test
    void invocationCompleteIsError_returnsFallback() {
        when(agentProvider.invoke(any())).thenReturn(Multi.createFrom().items(
                new AgentEvent.TextDelta("{\"value\":\"ok\",\"count\":1}"),
                new AgentEvent.InvocationComplete(5, 10, 0, 0, 0, null, 200L, 150L, "sess-5", 1, true)));
        var request = new ClinicalAgentRequest<>("system", "user", TestResponse.class,
                new TestResponse("fallback", 0), "test", "corr-5");
        var result = support.invoke(request);
        assertTrue(result.fallbackUsed());
    }

    @Test
    void exceptionDuringInvocation_returnsFallback() {
        when(agentProvider.invoke(any())).thenReturn(
                Multi.createFrom().failure(new RuntimeException("connection failed")));
        var request = new ClinicalAgentRequest<>("system", "user", TestResponse.class,
                new TestResponse("fallback", 0), "test", "corr-6");
        var result = support.invoke(request);
        assertTrue(result.fallbackUsed());
        assertNotNull(result.failureReason());
    }

    @Test
    void modelFromConfigKey_passedToAgentSessionConfig() {
        when(agentProvider.invoke(any())).thenReturn(Multi.createFrom().items(
                new AgentEvent.TextDelta("{\"value\":\"ok\",\"count\":1}"),
                new AgentEvent.InvocationComplete(5, 10, 0, 0, 0, null, 200L, 150L, "sess-7", 1, false)));
        var request = new ClinicalAgentRequest<>("system", "user", TestResponse.class,
                new TestResponse("fallback", 0), "safety", "corr-7");
        support.invoke(request);
        ArgumentCaptor<AgentSessionConfig> captor = ArgumentCaptor.forClass(AgentSessionConfig.class);
        verify(agentProvider).invoke(captor.capture());
        assertNotNull(captor.getValue());
    }
}
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=ClinicalAgentSupportTest --batch-mode`
Expected: FAIL — `ClinicalAgentSupport` does not exist yet

- [ ] **Step 7: Write ClinicalAgentSupport implementation**

```java
// runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentSupport.java
package io.casehub.clinical.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
import java.util.stream.Collectors;

@ApplicationScoped
public class ClinicalAgentSupport {

    private static final Logger LOG = Logger.getLogger(ClinicalAgentSupport.class);
    private static final Pattern JSON_FENCE = Pattern.compile("```(?:json)?\\s*\\n(\\{.*?})\\s*\\n```", Pattern.DOTALL);
    private static final Duration DEFAULT_TIMEOUT = Duration.ofSeconds(30);
    private static final String DEFAULT_MODEL = "sonnet";

    private final AgentProvider agentProvider;
    private final ObjectMapper objectMapper;

    @Inject
    public ClinicalAgentSupport(AgentProvider agentProvider, ObjectMapper objectMapper) {
        this.agentProvider = agentProvider;
        this.objectMapper = objectMapper;
    }

    public <T> ClinicalAgentResult<T> invoke(ClinicalAgentRequest<T> request) {
        InvocationMetrics metrics = null;
        try {
            String model = resolveModel(request.configKey());
            Duration timeout = resolveTimeout(request.configKey());
            AgentSessionConfig config = new AgentSessionConfig(
                    request.systemPrompt(), request.userPrompt(),
                    List.of(), timeout, request.correlationId(), model);

            List<AgentEvent> events = agentProvider.invoke(config)
                    .collect().asList()
                    .await().atMost(timeout.plusSeconds(5));

            String rawText = events.stream()
                    .filter(e -> e instanceof AgentEvent.TextDelta)
                    .map(e -> ((AgentEvent.TextDelta) e).text())
                    .collect(Collectors.joining());

            metrics = events.stream()
                    .filter(e -> e instanceof AgentEvent.InvocationComplete)
                    .map(e -> mapMetrics(model, (AgentEvent.InvocationComplete) e))
                    .findFirst().orElse(null);

            if (metrics != null && metrics.isError()) {
                return ClinicalAgentResult.fallback(request.fallbackValue(),
                        "InvocationComplete.isError=true", metrics);
            }

            if (rawText == null || rawText.isBlank()) {
                LOG.warnf("ClinicalAgentSupport[%s]: empty response — using fallback", request.configKey());
                return ClinicalAgentResult.fallback(request.fallbackValue(), "empty response", metrics);
            }

            String json = extractJson(rawText);
            T parsed = objectMapper.readValue(json, request.responseClass());
            return ClinicalAgentResult.success(parsed, rawText, metrics);

        } catch (Exception e) {
            LOG.errorf(e, "ClinicalAgentSupport[%s]: invocation failed — using fallback", request.configKey());
            return ClinicalAgentResult.fallback(request.fallbackValue(), e.getMessage(), metrics);
        }
    }

    String extractJson(String text) {
        Matcher m = JSON_FENCE.matcher(text);
        if (m.find()) return m.group(1);
        int start = text.indexOf('{');
        int end = text.lastIndexOf('}');
        if (start >= 0 && end > start) return text.substring(start, end + 1);
        return text;
    }

    private String resolveModel(String configKey) {
        try {
            String value = org.eclipse.microprofile.config.ConfigProvider.getConfig()
                    .getOptionalValue("casehub.clinical.agent." + configKey + ".model", String.class)
                    .orElse(DEFAULT_MODEL);
            return value;
        } catch (Exception e) {
            return DEFAULT_MODEL;
        }
    }

    private Duration resolveTimeout(String configKey) {
        try {
            String value = org.eclipse.microprofile.config.ConfigProvider.getConfig()
                    .getOptionalValue("casehub.clinical.agent." + configKey + ".timeout", String.class)
                    .orElse(null);
            return value != null ? Duration.parse(value) : DEFAULT_TIMEOUT;
        } catch (Exception e) {
            return DEFAULT_TIMEOUT;
        }
    }

    private InvocationMetrics mapMetrics(String model, AgentEvent.InvocationComplete ic) {
        return new InvocationMetrics(model, ic.inputTokens(), ic.outputTokens(),
                ic.thinkingTokens(), ic.cacheReadTokens(), ic.cacheWriteTokens(),
                ic.totalCostUsd(), ic.durationMs(), ic.apiDurationMs(),
                ic.sessionId(), ic.numTurns(), ic.isError());
    }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalAgentSupportTest --batch-mode`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/agent/ runtime/src/test/java/io/casehub/clinical/agent/
git commit -m "feat(#160): add ClinicalAgentSupport shared utility with Jackson parsing and InvocationMetrics

Refs #160"
```

### Task 2: Bootstrap dependencies + refactor LlmProtocolAmendmentAdvisor

**Files:**
- Modify: `runtime/pom.xml`
- Modify: `runtime/src/main/resources/application.properties`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/LlmProtocolAmendmentAdvisor.java`
- Modify: `runtime/src/test/java/io/casehub/clinical/service/LlmProtocolAmendmentAdvisorTest.java` (existing)
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java`

**Interfaces:**
- Consumes: `ClinicalAgentSupport.invoke()` from Task 1
- Produces: Updated `LlmProtocolAmendmentAdvisor` using shared utility; `ClinicalComplianceSupplement` with metrics overloads

- [ ] **Step 1: Add agent-router + agent-claude dependencies to runtime/pom.xml**

Add inside `<dependencies>`:
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-agent-router</artifactId>
  <scope>runtime</scope>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-agent-claude</artifactId>
  <scope>runtime</scope>
</dependency>
```

- [ ] **Step 2: Add agent config to production application.properties**

```properties
# Agent provider
casehub.platform.agent.default-backend=claude
casehub.clinical.agent.amendment.model=sonnet
casehub.clinical.agent.amendment.timeout=PT30S
casehub.clinical.agent.eligibility.model=sonnet
casehub.clinical.agent.eligibility.timeout=PT30S
casehub.clinical.agent.safety.model=sonnet
casehub.clinical.agent.safety.timeout=PT30S
casehub.clinical.agent.dsmb.model=sonnet
casehub.clinical.agent.dsmb.timeout=PT60S
casehub.clinical.agent.supervision.model=sonnet
casehub.clinical.agent.supervision.timeout=PT60S

# Jandex indexing for agent modules
quarkus.index-dependency.agent-router.group-id=io.casehub
quarkus.index-dependency.agent-router.artifact-id=casehub-platform-agent-router
quarkus.index-dependency.agent-claude.group-id=io.casehub
quarkus.index-dependency.agent-claude.artifact-id=casehub-platform-agent-claude
```

- [ ] **Step 3: Add metrics overload to ClinicalComplianceSupplement**

Add a static helper method that enriches an existing `ComplianceSupplement` with `InvocationMetrics` JSON in the `detail` field. Pattern: `withMetrics(ComplianceSupplement base, InvocationMetrics metrics)` returns a new `ComplianceSupplement` with `detail` set to `metrics.toJson()`.

- [ ] **Step 4: Refactor LlmProtocolAmendmentAdvisor to use ClinicalAgentSupport**

Replace manual `extractJsonValue()` string parsing with `ClinicalAgentSupport.invoke()`. Inject `ClinicalAgentSupport` instead of `AgentProvider` directly. Use Jackson-friendly response record:

```java
record AmendmentResponse(String recommendation, String reasoning) {}
```

Map `AmendmentResponse.recommendation` → `AmendmentRecommendation.valueOf()` with fallback to PROCEED.

- [ ] **Step 5: Update existing LlmProtocolAmendmentAdvisorTest**

Same test cases, adapted for the new internal structure. Verify `ClinicalAgentSupport` is called with correct system prompt and model config key `"amendment"`.

- [ ] **Step 6: Run all tests**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: PASS — all existing tests still green, refactored advisor works identically

- [ ] **Step 7: Commit**

```bash
git add runtime/pom.xml runtime/src/main/resources/application.properties \
  runtime/src/main/java/io/casehub/clinical/service/LlmProtocolAmendmentAdvisor.java \
  runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java \
  runtime/src/test/java/io/casehub/clinical/service/LlmProtocolAmendmentAdvisorTest.java
git commit -m "feat(#160): bootstrap agent dependencies and refactor LlmProtocolAmendmentAdvisor to ClinicalAgentSupport

Refs #160"
```

---

## Batch 2: Eligibility Screening Agent

### Task 3: EligibilityCriteriaEvaluator SPI + LLM implementation

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/spi/EligibilityCriteriaEvaluator.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/DefaultEligibilityCriteriaEvaluator.java`
- Create: `runtime/src/main/java/io/casehub/clinical/agent/EligibilityResponse.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/LlmEligibilityCriteriaEvaluator.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/EvaluateScreenRequest.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/EligibilityScreeningService.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/LlmEligibilityCriteriaEvaluatorTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/EligibilityAgentIntegrationTest.java`

**Interfaces:**
- Consumes: `ClinicalAgentSupport.invoke()`, `CriterionResult(String id, boolean met, boolean marginal)`, `EligibilityScreeningService.screen()`
- Produces: `EligibilityCriteriaEvaluator.evaluate(PatientEnrollment, List<String>)` → `List<CriterionResult>`; `EligibilityScreeningService.evaluateAndScreen(PatientEnrollment, List<String>)`; `POST /api/patients/{id}/evaluate-and-screen`

- [ ] **Step 1: Write SPI interface**

```java
// api/src/main/java/io/casehub/clinical/api/spi/EligibilityCriteriaEvaluator.java
package io.casehub.clinical.api.spi;

import io.casehub.clinical.api.model.CriterionResult;
import io.casehub.clinical.entity.PatientEnrollment;
import java.util.List;

public interface EligibilityCriteriaEvaluator {
    List<CriterionResult> evaluate(PatientEnrollment enrollment, List<String> protocolCriteria);
}
```

- [ ] **Step 2: Write DefaultEligibilityCriteriaEvaluator stub**

Returns all criteria as MET. `@DefaultBean @ApplicationScoped`.

- [ ] **Step 3: Write EligibilityResponse record**

```java
// runtime/src/main/java/io/casehub/clinical/agent/EligibilityResponse.java
package io.casehub.clinical.agent;

import java.util.List;

public record EligibilityResponse(List<CriterionEvaluation> criteria) {
    public record CriterionEvaluation(String criterionId, boolean met, boolean marginal, String reasoning) {}
}
```

- [ ] **Step 4: Write failing unit tests for LlmEligibilityCriteriaEvaluator**

Test valid JSON → correct `CriterionResult` mapping, malformed JSON → all MARGINAL fallback, empty response → all MARGINAL, prompt contains patient data and criteria text.

- [ ] **Step 5: Run tests to verify they fail**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=LlmEligibilityCriteriaEvaluatorTest --batch-mode`

- [ ] **Step 6: Write LlmEligibilityCriteriaEvaluator**

`@ApplicationScoped`, injects `ClinicalAgentSupport`. System prompt: clinical trial eligibility expert. Maps `EligibilityResponse` → `List<CriterionResult>`. Config key `"eligibility"`. Fallback: all criteria as MARGINAL.

- [ ] **Step 7: Run unit tests**

Expected: PASS

- [ ] **Step 8: Write EvaluateScreenRequest + add evaluateAndScreen to service + endpoint**

`EvaluateScreenRequest(List<String> protocolCriteria)` in api module. `EligibilityScreeningService.evaluateAndScreen()` calls SPI then existing `screen()`. `PatientResource` gains `POST /api/patients/{id}/evaluate-and-screen`.

- [ ] **Step 9: Write integration test**

`@QuarkusTest` with `@InjectMock AgentProvider`. Verify CDI displacement (injected `EligibilityCriteriaEvaluator` is LLM impl), end-to-end via REST endpoint.

- [ ] **Step 10: Run all tests**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`

- [ ] **Step 11: Commit**

```bash
git add api/src/main/java/io/casehub/clinical/api/spi/EligibilityCriteriaEvaluator.java \
  api/src/main/java/io/casehub/clinical/api/model/EvaluateScreenRequest.java \
  runtime/src/main/java/io/casehub/clinical/service/DefaultEligibilityCriteriaEvaluator.java \
  runtime/src/main/java/io/casehub/clinical/agent/EligibilityResponse.java \
  runtime/src/main/java/io/casehub/clinical/service/LlmEligibilityCriteriaEvaluator.java \
  runtime/src/main/java/io/casehub/clinical/service/EligibilityScreeningService.java \
  runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java \
  runtime/src/test/java/io/casehub/clinical/service/LlmEligibilityCriteriaEvaluatorTest.java \
  runtime/src/test/java/io/casehub/clinical/service/EligibilityAgentIntegrationTest.java
git commit -m "feat(#160): add eligibility screening LLM agent with EligibilityCriteriaEvaluator SPI

Refs #160"
```

---

## Batch 3: Safety Monitoring Agent (SUSAR Evaluation)

### Task 4: LlmSusarCriteriaEvaluator

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/agent/SusarAssessmentResponse.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/LlmSusarCriteriaEvaluator.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/LlmSusarCriteriaEvaluatorTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/SusarAgentIntegrationTest.java`

**Interfaces:**
- Consumes: `SusarEvaluatorFunction` (existing interface), `ClinicalAgentSupport.invoke()`, `AdverseEvent` entity, `PlannedAction.of()`, `ClinicalActionType.SUSAR_CRITERIA_DECISION`
- Produces: `LlmSusarCriteriaEvaluator` displacing `SusarCriteriaEvaluator @DefaultBean`; same `WorkerResult<Map<String,Object>>` output contract

- [ ] **Step 1: Write SusarAssessmentResponse record**

```java
// runtime/src/main/java/io/casehub/clinical/agent/SusarAssessmentResponse.java
package io.casehub.clinical.agent;

public record SusarAssessmentResponse(
        boolean susarRequired,
        String causalityAssessment,
        String expectednessAssessment,
        String reasoning,
        float confidence) {}
```

- [ ] **Step 2: Write failing unit tests**

Mock `AgentProvider`. Test: valid SUSAR-required response → `WorkerResult` with `susarRequired=true` + `PlannedAction`; valid not-required → `WorkerResult` with `susarRequired=false` + no `PlannedAction`; LLM failure → fallback `susarRequired=true` (conservative); prompt includes AE grade, eventType, patient context.

- [ ] **Step 3: Run tests to verify they fail**

- [ ] **Step 4: Write LlmSusarCriteriaEvaluator**

`@ApplicationScoped`, implements `SusarEvaluatorFunction`. `@Transactional`. Loads `AdverseEvent` from DB by `aeId`. Builds user prompt from AE + patient context. Calls `ClinicalAgentSupport.invoke()`. Maps `SusarAssessmentResponse` → `WorkerResult<Map>` with `PlannedAction` when `susarRequired=true`. Config key `"safety"`. Fallback: `susarRequired=true`.

Maintains Grade 4/5 scope — system prompt explicitly states "Evaluate only Grade 4 and Grade 5 adverse events."

- [ ] **Step 5: Run unit tests**

Expected: PASS

- [ ] **Step 6: Write integration test**

`@QuarkusTest` with `@InjectMock AgentProvider`. Verify CDI displacement, full worker flow through `ClinicalSusarOversightCaseHub`.

- [ ] **Step 7: Run all tests**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/agent/SusarAssessmentResponse.java \
  runtime/src/main/java/io/casehub/clinical/service/LlmSusarCriteriaEvaluator.java \
  runtime/src/test/java/io/casehub/clinical/service/LlmSusarCriteriaEvaluatorTest.java \
  runtime/src/test/java/io/casehub/clinical/service/SusarAgentIntegrationTest.java
git commit -m "feat(#160): add LLM SUSAR criteria evaluator displacing rule-based SusarCriteriaEvaluator

Refs #160"
```

---

## Batch 4: DSMB Safety Signal Analysis Agent

### Task 5: SafetySignalAnalyzer SPI + LLM implementation

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/spi/SafetySignalAnalyzer.java`
- Create: `api/src/main/java/io/casehub/clinical/api/spi/TrialSafetyContext.java`
- Create: `runtime/src/main/java/io/casehub/clinical/agent/SignalAnalysis.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/DefaultSafetySignalAnalyzer.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/LlmSafetySignalAnalyzer.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/cbr/TrialSafetyAggregationJob.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/LlmSafetySignalAnalyzerTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/DsmbAgentIntegrationTest.java`

**Interfaces:**
- Consumes: `ClinicalAgentSupport.invoke()`, `TrialSafetyAggregationJob.DetectedSignal`, `TrialSafetyAggregationJob.SiteAeSummary`
- Produces: `SafetySignalAnalyzer.analyze(TrialSafetyContext)` → `SignalAnalysis`; `TrialSafetyAggregationJob.aggregateTrial()` calls analyzer after rule-based detection

- [ ] **Step 1: Write TrialSafetyContext record**

```java
// api/src/main/java/io/casehub/clinical/api/spi/TrialSafetyContext.java
package io.casehub.clinical.api.spi;

import java.util.List;
import java.util.Map;
import java.util.UUID;

public record TrialSafetyContext(
        UUID trialId,
        String trialPhase,
        Map<UUID, List<SiteAeSummary>> siteData,
        List<DetectedSignalSummary> detectedSignals) {

    public record SiteAeSummary(String grade, String eventType, int count) {}
    public record DetectedSignalSummary(String signalType, List<String> affectedSiteIds,
                                         String summary, String dominantGrade, String dominantEventType) {}
}
```

- [ ] **Step 2: Write SafetySignalAnalyzer SPI + DefaultSafetySignalAnalyzer stub**

SPI in api module. Default stub returns pass-through (no LLM enrichment). `@DefaultBean @ApplicationScoped`.

- [ ] **Step 3: Write SignalAnalysis response records**

```java
// runtime/src/main/java/io/casehub/clinical/agent/SignalAnalysis.java
package io.casehub.clinical.agent;

import java.util.List;

public record SignalAnalysis(
        List<AnalyzedSignal> signals,
        String overallAssessment,
        List<String> additionalPatterns) {

    public record AnalyzedSignal(String signalType, String severity,
                                  List<String> affectedSites, String narrative,
                                  String recommendedAction) {}
}
```

- [ ] **Step 4: Write failing unit tests for LlmSafetySignalAnalyzer**

- [ ] **Step 5: Write LlmSafetySignalAnalyzer**

`@ApplicationScoped`, injects `ClinicalAgentSupport`. Config key `"dsmb"`. Fallback: `FLAG_FOR_REVIEW` — returns analysis with all signals flagged for human review.

- [ ] **Step 6: Run unit tests**

- [ ] **Step 7: Integrate into TrialSafetyAggregationJob**

Inject `SafetySignalAnalyzer`. After `detectSignals()` completes, call `analyzer.analyze(context)`. Use `SignalAnalysis.overallAssessment` to enrich `TrialSafetySignal.summary`. The LLM analysis augments — rule-based signals always fire.

- [ ] **Step 8: Write integration test**

- [ ] **Step 9: Run all tests**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/clinical/api/spi/SafetySignalAnalyzer.java \
  api/src/main/java/io/casehub/clinical/api/spi/TrialSafetyContext.java \
  runtime/src/main/java/io/casehub/clinical/agent/SignalAnalysis.java \
  runtime/src/main/java/io/casehub/clinical/service/DefaultSafetySignalAnalyzer.java \
  runtime/src/main/java/io/casehub/clinical/service/LlmSafetySignalAnalyzer.java \
  runtime/src/main/java/io/casehub/clinical/cbr/TrialSafetyAggregationJob.java \
  runtime/src/test/java/io/casehub/clinical/service/LlmSafetySignalAnalyzerTest.java \
  runtime/src/test/java/io/casehub/clinical/service/DsmbAgentIntegrationTest.java
git commit -m "feat(#160): add DSMB safety signal analysis LLM agent augmenting rule-based detection

Refs #160"
```

---

## Batch 5: Trial Supervision Agent

### Task 6: TrialSupervisionAdvisor SPI + LLM implementation + YAML binding

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/spi/TrialSupervisionAdvisor.java`
- Create: `api/src/main/java/io/casehub/clinical/api/spi/TrialSupervisionContext.java`
- Create: `runtime/src/main/java/io/casehub/clinical/agent/SupervisionAssessment.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/DefaultTrialSupervisionAdvisor.java`
- Create: `runtime/src/main/java/io/casehub/clinical/service/LlmTrialSupervisionAdvisor.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/ClinicalTrialCaseHub.java`
- Modify: `runtime/src/main/resources/clinical/trial-coordination.yaml`
- Test: `runtime/src/test/java/io/casehub/clinical/service/LlmTrialSupervisionAdvisorTest.java`
- Test: `runtime/src/test/java/io/casehub/clinical/service/TrialSupervisionIntegrationTest.java`

**Interfaces:**
- Consumes: `ClinicalAgentSupport.invoke()`, `ClinicalTrialCaseHub`, `Worker.builder().function()`
- Produces: `TrialSupervisionAdvisor.assess(TrialSupervisionContext)` → `SupervisionAssessment`; `trial-coordination.yaml` gains `trial-supervision` capability binding

- [ ] **Step 1: Write TrialSupervisionContext + TrialSupervisionAdvisor SPI + default stub**

- [ ] **Step 2: Write SupervisionAssessment response records**

```java
// runtime/src/main/java/io/casehub/clinical/agent/SupervisionAssessment.java
package io.casehub.clinical.agent;

import java.util.List;

public record SupervisionAssessment(
        String overallHealth,
        List<SupervisionFinding> findings,
        String summary) {

    public record SupervisionFinding(String findingType, List<String> affectedSites,
                                      String severity, String narrative,
                                      String recommendedAction) {}
}
```

- [ ] **Step 3: Write failing unit tests for LlmTrialSupervisionAdvisor**

- [ ] **Step 4: Write LlmTrialSupervisionAdvisor**

`@ApplicationScoped`, injects `ClinicalAgentSupport`. Config key `"supervision"`. Fallback: `REVIEW_REQUIRED` assessment.

- [ ] **Step 5: Run unit tests**

- [ ] **Step 6: Add capability binding to trial-coordination.yaml**

Add a `trial-supervision` capability binding alongside the existing DSMB humanTask. Fires on `contextChange` when safety metrics update.

- [ ] **Step 7: Register worker in ClinicalTrialCaseHub.augment()**

Override `augment()` in `ClinicalTrialCaseHub`. Register `trial-supervision` worker via `Worker.builder().function()` calling `TrialSupervisionAdvisor.assess()`.

- [ ] **Step 8: Write integration test**

- [ ] **Step 9: Run all tests**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/clinical/api/spi/TrialSupervisionAdvisor.java \
  api/src/main/java/io/casehub/clinical/api/spi/TrialSupervisionContext.java \
  runtime/src/main/java/io/casehub/clinical/agent/SupervisionAssessment.java \
  runtime/src/main/java/io/casehub/clinical/service/DefaultTrialSupervisionAdvisor.java \
  runtime/src/main/java/io/casehub/clinical/service/LlmTrialSupervisionAdvisor.java \
  runtime/src/main/java/io/casehub/clinical/service/ClinicalTrialCaseHub.java \
  runtime/src/main/resources/clinical/trial-coordination.yaml \
  runtime/src/test/java/io/casehub/clinical/service/LlmTrialSupervisionAdvisorTest.java \
  runtime/src/test/java/io/casehub/clinical/service/TrialSupervisionIntegrationTest.java
git commit -m "feat(#160): add trial supervision LLM agent with ClinicalTrialCaseHub capability binding

Refs #160"
```

---

## References

- [2026-09-11-live-llm-agent-execution-design.md](/Users/mdproctor/claude/public/casehub/clinical/specs/issue-160-live-llm-agent-execution/2026-09-11-live-llm-agent-execution-design.md) — design spec this plan implements
- [LlmProtocolAmendmentAdvisor.java:17](runtime/src/main/java/io/casehub/clinical/service/LlmProtocolAmendmentAdvisor.java) — existing AgentProvider integration pattern
- [SusarCriteriaEvaluator.java:30](runtime/src/main/java/io/casehub/clinical/service/SusarCriteriaEvaluator.java) — rule-based evaluator to displace
- [SusarEvaluatorFunction.java:15](runtime/src/main/java/io/casehub/clinical/service/SusarEvaluatorFunction.java) — worker function interface
- [EligibilityScreeningService.java:17](runtime/src/main/java/io/casehub/clinical/service/EligibilityScreeningService.java) — eligibility integration point
- [TrialSafetyAggregationJob.java:42](runtime/src/main/java/io/casehub/clinical/cbr/TrialSafetyAggregationJob.java) — DSMB integration point
- [ClinicalTrialCaseHub.java:8](runtime/src/main/java/io/casehub/clinical/service/ClinicalTrialCaseHub.java) — trial supervision binding target
- [ClinicalComplianceSupplement.java:19](runtime/src/main/java/io/casehub/clinical/service/ClinicalComplianceSupplement.java) — compliance supplement factory
- [CriterionResult.java:3](api/src/main/java/io/casehub/clinical/api/model/CriterionResult.java) — eligibility result record
- [issue-86 spec](docs/specs/issue-86-protocol-amendment-llm/2026-07-30-protocol-amendment-llm-advisor-design.md) — reference pattern
- GitHub #160 — focal epic
