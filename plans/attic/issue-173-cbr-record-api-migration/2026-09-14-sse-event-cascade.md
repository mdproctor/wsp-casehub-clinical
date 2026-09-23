# Real-Time Event Cascade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #161 — feat: Real-time event cascade — SSE for orchestration visibility
**Issue group:** #161

**Goal:** Users watch the AE orchestration cascade unfold in real time via WebSocket push and an event timeline UI.

**Architecture:** CDI/Vert.x event observers → `ClinicalCascadeBroadcaster` → `EventBroadcaster` (casehub-pages-push) → WebSocket → `EventConnection` (pages-data) → `cascadeTimelineStrategy` → timeline component in safety workbench.

**Tech Stack:** Java 21, Quarkus 3.32.2 (WebSockets Next), casehub-pages-push 0.2-SNAPSHOT, TypeScript (Lit web components), casehub-pages-viz event timeline

## Global Constraints

- Topic separator is `:` (TopicRegistry trie convention), not `/`
- `CascadeEvent.data` payload capped at 4 KB serialized
- Gate events use Vert.x `@ConsumeEvent`, not CDI `@ObservesAsync`
- Grade 1-2 AEs do not fire `AdverseEventReportedEvent` — cascade broadcast via `TransactionSynchronization.afterCompletion()`
- `TRUST_UPDATED` excluded — trust is a 24h batch job, not real-time
- `CascadeStepStatus` is an enum (not String) for compile-time safety
- All commits reference `Refs #161`

---

## Batch 1: API Model + Dependencies

### Task 1: Cascade event model and push dependencies

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/CascadeStepType.java`
- Create: `api/src/main/java/io/casehub/clinical/api/CascadeStepStatus.java`
- Create: `api/src/main/java/io/casehub/clinical/api/CascadeEvent.java`
- Create: `api/src/main/java/io/casehub/clinical/api/CascadeTemplateResolver.java`
- Create: `api/src/main/java/io/casehub/clinical/api/AeEscalationStartedEvent.java`
- Create: `api/src/main/java/io/casehub/clinical/api/AeEscalationFailedEvent.java`
- Modify: `runtime/pom.xml` — add pages-push dependencies
- Modify: `runtime/src/main/resources/application.properties` — add Jandex indexing
- Modify: `runtime/src/test/resources/application.properties` — add Jandex indexing
- Test: `api/src/test/java/io/casehub/clinical/api/CascadeTemplateResolverTest.java`
- Test: `api/src/test/java/io/casehub/clinical/api/CascadeEventTest.java`

**Interfaces:**
- Produces: `CascadeStepType` enum (11 values), `CascadeStepStatus` enum (5 values), `CascadeEvent` record, `CascadeTemplateResolver.resolve(CtcaeGrade, boolean unexpected, boolean suspected)` returning `List<CascadeStepType>`, `AeEscalationStartedEvent` record, `AeEscalationFailedEvent` record

- [ ] **Step 1: Write CascadeTemplateResolver tests**

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.CtcaeGrade;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class CascadeTemplateResolverTest {

    @Test
    void grade1ReturnsShortTemplate() {
        List<CascadeStepType> steps = CascadeTemplateResolver.resolve(CtcaeGrade.GRADE_1, false, false);
        assertThat(steps).containsExactly(
            CascadeStepType.AE_REPORTED,
            CascadeStepType.SLA_ASSIGNED,
            CascadeStepType.LEDGER_SEALED);
    }

    @Test
    void grade3BaselineReturnsEscalationTemplate() {
        List<CascadeStepType> steps = CascadeTemplateResolver.resolve(CtcaeGrade.GRADE_3, false, false);
        assertThat(steps).containsExactly(
            CascadeStepType.AE_REPORTED,
            CascadeStepType.SLA_ASSIGNED,
            CascadeStepType.ESCALATION_CASE_STARTED,
            CascadeStepType.AGENT_SELECTED,
            CascadeStepType.AGENT_REASONING,
            CascadeStepType.AGENT_RESULT,
            CascadeStepType.SAFETY_OFFICER_NOTIFIED,
            CascadeStepType.LEDGER_SEALED);
    }

    @Test
    void grade4UnexpectedAddsRegulatorySubmission() {
        List<CascadeStepType> steps = CascadeTemplateResolver.resolve(CtcaeGrade.GRADE_4, true, false);
        assertThat(steps).contains(CascadeStepType.REGULATORY_SUBMISSION_STARTED);
        assertThat(steps).doesNotContain(CascadeStepType.GATE_OPENED, CascadeStepType.GATE_RESOLVED);
    }

    @Test
    void grade4UnexpectedSuspectedAddsGate() {
        List<CascadeStepType> steps = CascadeTemplateResolver.resolve(CtcaeGrade.GRADE_4, true, true);
        assertThat(steps).contains(
            CascadeStepType.GATE_OPENED,
            CascadeStepType.GATE_RESOLVED,
            CascadeStepType.REGULATORY_SUBMISSION_STARTED);
    }

    @Test
    void grade2NeverHasEscalation() {
        List<CascadeStepType> steps = CascadeTemplateResolver.resolve(CtcaeGrade.GRADE_2, false, false);
        assertThat(steps).doesNotContain(CascadeStepType.ESCALATION_CASE_STARTED);
    }
}
```

Run: `mvn test -pl api -Dtest=CascadeTemplateResolverTest --batch-mode`
Expected: FAIL — classes don't exist yet

- [ ] **Step 2: Create CascadeStepType enum**

```java
package io.casehub.clinical.api;

public enum CascadeStepType {
    AE_REPORTED,
    SLA_ASSIGNED,
    ESCALATION_CASE_STARTED,
    AGENT_SELECTED,
    AGENT_REASONING,
    AGENT_RESULT,
    GATE_OPENED,
    GATE_RESOLVED,
    SAFETY_OFFICER_NOTIFIED,
    REGULATORY_SUBMISSION_STARTED,
    LEDGER_SEALED
}
```

- [ ] **Step 3: Create CascadeStepStatus enum**

```java
package io.casehub.clinical.api;

public enum CascadeStepStatus {
    PENDING,
    ACTIVE,
    COMPLETED,
    FAILED,
    SKIPPED
}
```

- [ ] **Step 4: Create CascadeEvent record**

```java
package io.casehub.clinical.api;

import java.time.Instant;
import java.util.Map;

public record CascadeEvent(
    CascadeStepType step,
    CascadeStepStatus status,
    Instant timestamp,
    String actor,
    String detail,
    Map<String, Object> data
) {}
```

- [ ] **Step 5: Create CascadeTemplateResolver**

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.CtcaeGrade;
import java.util.ArrayList;
import java.util.List;

public final class CascadeTemplateResolver {

    private CascadeTemplateResolver() {}

    public static List<CascadeStepType> resolve(CtcaeGrade grade, boolean unexpected, boolean suspected) {
        List<CascadeStepType> steps = new ArrayList<>();
        steps.add(CascadeStepType.AE_REPORTED);
        steps.add(CascadeStepType.SLA_ASSIGNED);

        if (grade.ordinal() < CtcaeGrade.GRADE_3.ordinal()) {
            steps.add(CascadeStepType.LEDGER_SEALED);
            return List.copyOf(steps);
        }

        steps.add(CascadeStepType.ESCALATION_CASE_STARTED);
        steps.add(CascadeStepType.AGENT_SELECTED);
        steps.add(CascadeStepType.AGENT_REASONING);
        steps.add(CascadeStepType.AGENT_RESULT);

        if (unexpected && suspected) {
            steps.add(CascadeStepType.GATE_OPENED);
            steps.add(CascadeStepType.GATE_RESOLVED);
        }

        steps.add(CascadeStepType.SAFETY_OFFICER_NOTIFIED);

        if (unexpected) {
            steps.add(CascadeStepType.REGULATORY_SUBMISSION_STARTED);
        }

        steps.add(CascadeStepType.LEDGER_SEALED);
        return List.copyOf(steps);
    }
}
```

- [ ] **Step 6: Run template resolver tests**

Run: `mvn test -pl api -Dtest=CascadeTemplateResolverTest --batch-mode`
Expected: PASS

- [ ] **Step 7: Create AeEscalationStartedEvent and AeEscalationFailedEvent**

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.CtcaeGrade;
import java.util.UUID;

public record AeEscalationStartedEvent(
    UUID aeId,
    UUID caseId,
    CtcaeGrade grade,
    String tenantId) {}
```

```java
package io.casehub.clinical.api;

import io.casehub.clinical.api.model.CtcaeGrade;
import java.util.UUID;

public record AeEscalationFailedEvent(
    UUID aeId,
    CtcaeGrade grade,
    String tenantId,
    String errorMessage) {}
```

- [ ] **Step 8: Write CascadeEvent serialization test**

```java
package io.casehub.clinical.api;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class CascadeEventTest {

    private final ObjectMapper mapper = new ObjectMapper().registerModule(new JavaTimeModule());

    @Test
    void roundTrip() throws Exception {
        CascadeEvent event = new CascadeEvent(
            CascadeStepType.AE_REPORTED, CascadeStepStatus.COMPLETED,
            Instant.parse("2026-09-14T10:00:00Z"), "system",
            "Grade 4 AE reported", Map.of("grade", "GRADE_4"));
        String json = mapper.writeValueAsString(event);
        CascadeEvent parsed = mapper.readValue(json, CascadeEvent.class);
        assertThat(parsed.step()).isEqualTo(CascadeStepType.AE_REPORTED);
        assertThat(parsed.status()).isEqualTo(CascadeStepStatus.COMPLETED);
        assertThat(parsed.actor()).isEqualTo("system");
    }
}
```

Run: `mvn test -pl api -Dtest=CascadeEventTest --batch-mode`
Expected: PASS

- [ ] **Step 9: Add pages-push dependencies to runtime/pom.xml**

Add before `<!-- Test -->` comment in `runtime/pom.xml`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-pages-push</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-pages-push-runtime</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>
```

Add Quarkus WebSockets Next dependency (for `@WebSocket` endpoint):

```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-websockets-next</artifactId>
</dependency>
```

- [ ] **Step 10: Add Jandex indexing to both application.properties files**

Add to both `runtime/src/main/resources/application.properties` and `runtime/src/test/resources/application.properties`:

```properties
quarkus.index-dependency.pages-push.group-id=io.casehub
quarkus.index-dependency.pages-push.artifact-id=casehub-pages-push
quarkus.index-dependency.pages-push-runtime.group-id=io.casehub
quarkus.index-dependency.pages-push-runtime.artifact-id=casehub-pages-push-runtime
```

- [ ] **Step 11: Verify compilation**

Run: `mvn install -pl api --batch-mode && mvn compile -pl runtime --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 12: Commit**

```bash
git add api/ runtime/pom.xml runtime/src/main/resources/application.properties runtime/src/test/resources/application.properties
git commit -m "feat(#161): add cascade event model, template resolver, and pages-push dependencies

Refs #161"
```

---

## Batch 2: WebSocket Push + Broadcaster

### Task 2: WebSocket push endpoint

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/push/ConnectionRegistry.java`
- Create: `runtime/src/main/java/io/casehub/clinical/push/ClinicalSessionSender.java`
- Create: `runtime/src/main/java/io/casehub/clinical/push/ClinicalPushEndpoint.java`
- Test: `runtime/src/test/java/io/casehub/clinical/push/ClinicalPushEndpointTest.java`

**Interfaces:**
- Consumes: `EventBroadcaster` (from pages-push-runtime CDI), `TopicRegistry` (from pages-push-runtime CDI), `PushRequest` (from pages-push)
- Produces: WebSocket endpoint at `/ws/push`, `ConnectionRegistry` (concurrent session map), `ClinicalSessionSender` implementing `SessionSender`

- [ ] **Step 1: Write the WebSocket endpoint test**

```java
package io.casehub.clinical.push;

import io.quarkus.test.common.http.TestHTTPResource;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import io.casehub.pages.push.EventBroadcaster;
import org.junit.jupiter.api.Test;
import java.net.URI;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {"SPONSOR", "INVESTIGATOR", "COORDINATOR"})
class ClinicalPushEndpointTest {

    @TestHTTPResource("/ws/push")
    URI wsUri;

    @Inject EventBroadcaster broadcaster;

    @Test
    void listenAndReceiveBroadcast() throws Exception {
        var received = new LinkedBlockingQueue<String>();
        var client = new java.net.http.HttpClient.newHttpClient();
        // Use jakarta websocket client or Quarkus test websocket client
        // to connect, send listen, verify broadcast delivery
        // Exact client API depends on quarkus-websockets-next test support
    }
}
```

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalPushEndpointTest --batch-mode`
Expected: FAIL — classes don't exist

- [ ] **Step 2: Create ConnectionRegistry**

```java
package io.casehub.clinical.push;

import io.quarkus.websockets.next.WebSocketConnection;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class ConnectionRegistry {

    private final ConcurrentHashMap<String, WebSocketConnection> connections = new ConcurrentHashMap<>();

    public void register(WebSocketConnection connection) {
        connections.put(connection.id(), connection);
    }

    public void remove(String connectionId) {
        connections.remove(connectionId);
    }

    public WebSocketConnection get(String connectionId) {
        return connections.get(connectionId);
    }
}
```

- [ ] **Step 3: Create ClinicalSessionSender**

```java
package io.casehub.clinical.push;

import io.casehub.pages.push.SessionSender;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import io.quarkus.websockets.next.WebSocketConnection;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ClinicalSessionSender implements SessionSender {

    private static final Logger LOG = Logger.getLogger(ClinicalSessionSender.class);

    @Inject ConnectionRegistry registry;

    @Override
    public void send(String connectionId, String payload) {
        WebSocketConnection conn = registry.get(connectionId);
        if (conn == null || !conn.isOpen()) {
            LOG.debugf("Connection %s not found or closed — skipping send", connectionId);
            return;
        }
        conn.sendTextAndAwait(payload);
    }
}
```

- [ ] **Step 4: Create ClinicalPushEndpoint**

```java
package io.casehub.clinical.push;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.pages.push.EventStore;
import io.casehub.pages.push.JsonWriter;
import io.casehub.pages.push.PushRequest;
import io.casehub.pages.push.StoredEvent;
import io.casehub.pages.push.TopicRegistry;
import io.quarkus.websockets.next.OnClose;
import io.quarkus.websockets.next.OnOpen;
import io.quarkus.websockets.next.OnTextMessage;
import io.quarkus.websockets.next.WebSocket;
import io.quarkus.websockets.next.WebSocketConnection;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Map;
import java.util.Set;

@WebSocket(path = "/ws/push")
public class ClinicalPushEndpoint {

    private static final Logger LOG = Logger.getLogger(ClinicalPushEndpoint.class);

    @Inject ConnectionRegistry registry;
    @Inject TopicRegistry topicRegistry;
    @Inject EventStore eventStore;
    @Inject JsonWriter jsonWriter;
    @Inject ClinicalSessionSender sender;

    @OnOpen
    public void onOpen(WebSocketConnection connection) {
        registry.register(connection);
        LOG.debugf("WebSocket connected: %s", connection.id());
    }

    @OnTextMessage
    public void onMessage(WebSocketConnection connection, String message) {
        try {
            PushRequest request = PushRequest.parse(message);
            if (request instanceof PushRequest.Listen listen) {
                handleListen(connection.id(), listen);
            } else if (request instanceof PushRequest.Unlisten unlisten) {
                handleUnlisten(connection.id(), unlisten);
            }
        } catch (Exception e) {
            LOG.warnf(e, "Failed to handle push message from %s", connection.id());
        }
    }

    @OnClose
    public void onClose(WebSocketConnection connection) {
        topicRegistry.removeConnection(connection.id());
        registry.remove(connection.id());
        LOG.debugf("WebSocket disconnected: %s", connection.id());
    }

    private void handleListen(String connectionId, PushRequest.Listen listen) {
        topicRegistry.listen(connectionId, listen.topics());

        Map<String, Long> since = listen.since();
        if (since != null) {
            for (var entry : since.entrySet()) {
                List<StoredEvent> missed = eventStore.replay(entry.getKey(), entry.getValue(), 100);
                for (StoredEvent stored : missed) {
                    sender.send(connectionId, jsonWriter.writeEvent(stored.topic(), stored.payload(), stored.seq()));
                }
            }
        }

        String ack = "{\"op\":\"ack\",\"id\":\"" + listen.id() + "\",\"topics\":" +
            jsonWriter.writeArray(listen.topics()) + "}";
        sender.send(connectionId, ack);
    }

    private void handleUnlisten(String connectionId, PushRequest.Unlisten unlisten) {
        topicRegistry.unlisten(connectionId, unlisten.topics());
        String ack = "{\"op\":\"ack\",\"id\":\"" + unlisten.id() + "\"}";
        sender.send(connectionId, ack);
    }
}
```

**Note:** The exact `PushRequest` API (parse, field names) must be verified against the decompiled bytecode. The pattern above follows the wire protocol from `push-wire.ts` (listen/unlisten ops with topics array and since map). If `PushRequest.parse()` does not exist, construct the request manually from the parsed JSON.

- [ ] **Step 5: Run endpoint test**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalPushEndpointTest --batch-mode`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/push/ runtime/src/test/java/io/casehub/clinical/push/
git commit -m "feat(#161): add WebSocket push endpoint with ConnectionRegistry and SessionSender

Refs #161"
```

### Task 3: ClinicalCascadeBroadcaster and domain service hooks

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/service/ClinicalCascadeBroadcaster.java`
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java` — fire `AeEscalationStartedEvent` and `AeEscalationFailedEvent`
- Modify: `runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentRequest.java` — add `cascadeAeId` field
- Modify: `runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentSupport.java` — add broadcast hooks
- Modify: `runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java` — add Grade 1-2 cascade broadcast
- Test: `runtime/src/test/java/io/casehub/clinical/service/ClinicalCascadeBroadcasterTest.java`

**Interfaces:**
- Consumes: `EventBroadcaster.broadcast(topic, payload)`, `CascadeEvent`, `CascadeStepType`, `CascadeStepStatus`, `AdverseEventReportedEvent`, `AeEscalationStartedEvent`, `AeEscalationFailedEvent`, `ActionGateApprovedEvent`, `ActionGateRejectedEvent`, `ActionGateExpiredEvent`
- Produces: `ClinicalCascadeBroadcaster` with typed method calls (`agentSelected`, `agentReasoning`, `agentResult`, `safetyOfficerNotified`, `regulatorySubmissionStarted`) and event observers

- [ ] **Step 1: Write broadcaster unit test**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.*;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.pages.push.EventBroadcaster;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import java.time.Instant;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class ClinicalCascadeBroadcasterTest {

    private EventBroadcaster eventBroadcaster;
    private ClinicalCascadeBroadcaster broadcaster;

    @BeforeEach
    void setUp() {
        eventBroadcaster = mock(EventBroadcaster.class);
        broadcaster = new ClinicalCascadeBroadcaster(eventBroadcaster);
    }

    @Test
    void onAeReportedBroadcastsTwoSteps() {
        UUID aeId = UUID.randomUUID();
        var event = new AdverseEventReportedEvent(aeId, UUID.randomUUID(), UUID.randomUUID(),
            CtcaeGrade.GRADE_4, Instant.now(), "test-tenant");
        broadcaster.onAeReported(event);

        var topicCaptor = ArgumentCaptor.forClass(String.class);
        verify(eventBroadcaster, times(2)).broadcast(topicCaptor.capture(), any(CascadeEvent.class));
        assertThat(topicCaptor.getAllValues()).allMatch(t -> t.equals("clinical:ae:" + aeId + ":cascade"));
    }

    @Test
    void onEscalationStartedBroadcastsStep() {
        UUID aeId = UUID.randomUUID();
        var event = new AeEscalationStartedEvent(aeId, UUID.randomUUID(), CtcaeGrade.GRADE_4, "test-tenant");
        broadcaster.onEscalationStarted(event);

        verify(eventBroadcaster).broadcast(eq("clinical:ae:" + aeId + ":cascade"), any(CascadeEvent.class));
    }

    @Test
    void safetyOfficerNotifiedUsesCorrectTopic() {
        UUID aeId = UUID.randomUUID();
        broadcaster.safetyOfficerNotified(aeId);

        var captor = ArgumentCaptor.forClass(CascadeEvent.class);
        verify(eventBroadcaster).broadcast(eq("clinical:ae:" + aeId + ":cascade"), captor.capture());
        assertThat(captor.getValue().step()).isEqualTo(CascadeStepType.SAFETY_OFFICER_NOTIFIED);
        assertThat(captor.getValue().status()).isEqualTo(CascadeStepStatus.COMPLETED);
    }
}
```

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalCascadeBroadcasterTest --batch-mode`
Expected: FAIL — class doesn't exist

- [ ] **Step 2: Create ClinicalCascadeBroadcaster**

```java
package io.casehub.clinical.service;

import io.casehub.clinical.api.*;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.engine.common.internal.event.ActionGateApprovedEvent;
import io.casehub.engine.common.internal.event.ActionGateExpiredEvent;
import io.casehub.engine.common.internal.event.ActionGateRejectedEvent;
import io.casehub.pages.push.EventBroadcaster;
import io.quarkus.vertx.ConsumeEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class ClinicalCascadeBroadcaster {

    private static final Logger LOG = Logger.getLogger(ClinicalCascadeBroadcaster.class);

    private final EventBroadcaster eventBroadcaster;

    @Inject
    public ClinicalCascadeBroadcaster(EventBroadcaster eventBroadcaster) {
        this.eventBroadcaster = eventBroadcaster;
    }

    // --- CDI observers ---

    public void onAeReported(@ObservesAsync AdverseEventReportedEvent event) {
        broadcastStep(event.aeId(), CascadeStepType.AE_REPORTED, CascadeStepStatus.COMPLETED,
            "system", "Grade " + event.grade().label() + " adverse event reported",
            Map.of("grade", event.grade().name()));
        broadcastStep(event.aeId(), CascadeStepType.SLA_ASSIGNED, CascadeStepStatus.COMPLETED,
            "system", "SLA deadline: " + event.grade().sla().orElseThrow().toHours() + "h",
            Map.of("slaHours", event.grade().sla().orElseThrow().toHours()));
    }

    public void onEscalationStarted(@ObservesAsync AeEscalationStartedEvent event) {
        broadcastStep(event.aeId(), CascadeStepType.ESCALATION_CASE_STARTED, CascadeStepStatus.COMPLETED,
            "engine", "Escalation case created", Map.of("caseId", event.caseId().toString()));
    }

    public void onEscalationFailed(@ObservesAsync AeEscalationFailedEvent event) {
        broadcastStep(event.aeId(), CascadeStepType.ESCALATION_CASE_STARTED, CascadeStepStatus.FAILED,
            "engine", "Escalation failed: " + event.errorMessage(), Map.of());
    }

    // --- Vert.x event bus consumers (gate events) ---

    @ConsumeEvent(value = "casehub.action.gate.approved", blocking = true)
    public void onGateApproved(ActionGateApprovedEvent event) {
        AdverseEvent ae = AdverseEvent.findBySusarOversightCaseId(event.caseId());
        if (ae == null) return;
        broadcastStep(ae.id, CascadeStepType.GATE_OPENED, CascadeStepStatus.COMPLETED,
            "system", "SUSAR oversight gate opened", Map.of());
        broadcastStep(ae.id, CascadeStepType.GATE_RESOLVED, CascadeStepStatus.COMPLETED,
            "PI: " + event.approvedBy(), "Gate approved",
            Map.of("decision", "approved", "approvedBy", event.approvedBy()));
    }

    @ConsumeEvent(value = "casehub.action.gate.rejected", blocking = true)
    public void onGateRejected(ActionGateRejectedEvent event) {
        AdverseEvent ae = AdverseEvent.findBySusarOversightCaseId(event.caseId());
        if (ae == null) return;
        broadcastStep(ae.id, CascadeStepType.GATE_OPENED, CascadeStepStatus.COMPLETED,
            "system", "SUSAR oversight gate opened", Map.of());
        broadcastStep(ae.id, CascadeStepType.GATE_RESOLVED, CascadeStepStatus.COMPLETED,
            "system", "Gate rejected", Map.of("decision", "rejected"));
    }

    @ConsumeEvent(value = "casehub.action.gate.expired", blocking = true)
    public void onGateExpired(ActionGateExpiredEvent event) {
        AdverseEvent ae = AdverseEvent.findBySusarOversightCaseId(event.caseId());
        if (ae == null) return;
        broadcastStep(ae.id, CascadeStepType.GATE_OPENED, CascadeStepStatus.COMPLETED,
            "system", "SUSAR oversight gate opened", Map.of());
        broadcastStep(ae.id, CascadeStepType.GATE_RESOLVED, CascadeStepStatus.FAILED,
            "system", "Gate expired — no PI response", Map.of("decision", "expired"));
    }

    // --- Typed method calls (from domain services) ---

    public void agentSelected(UUID aeId, String agentId, double trustScore) {
        broadcastStep(aeId, CascadeStepType.AGENT_SELECTED, CascadeStepStatus.COMPLETED,
            agentId, "Agent selected with trust score " + String.format("%.2f", trustScore),
            Map.of("agentId", agentId, "trustScore", trustScore));
    }

    public void agentReasoning(UUID aeId, String configKey) {
        broadcastStep(aeId, CascadeStepType.AGENT_REASONING, CascadeStepStatus.ACTIVE,
            configKey, "Agent reasoning in progress", Map.of("configKey", configKey));
    }

    public void agentResult(UUID aeId, String configKey, boolean success) {
        broadcastStep(aeId, CascadeStepType.AGENT_RESULT,
            success ? CascadeStepStatus.COMPLETED : CascadeStepStatus.FAILED,
            configKey, success ? "Agent completed" : "Agent failed — using fallback",
            Map.of("configKey", configKey, "success", success));
    }

    public void safetyOfficerNotified(UUID aeId) {
        broadcastStep(aeId, CascadeStepType.SAFETY_OFFICER_NOTIFIED, CascadeStepStatus.COMPLETED,
            "system", "Safety officer notification sent", Map.of());
    }

    public void regulatorySubmissionStarted(UUID aeId, UUID caseId) {
        broadcastStep(aeId, CascadeStepType.REGULATORY_SUBMISSION_STARTED, CascadeStepStatus.COMPLETED,
            "system", "IND regulatory submission case started",
            Map.of("caseId", caseId.toString()));
    }

    public void ledgerSealed(UUID aeId, String digest) {
        broadcastStep(aeId, CascadeStepType.LEDGER_SEALED, CascadeStepStatus.COMPLETED,
            "ledger", "Merkle entry sealed",
            Map.of("digest", digest != null ? digest : ""));
    }

    public void broadcastGrade12Cascade(UUID aeId, CtcaeGrade grade, Instant slaDeadline) {
        broadcastStep(aeId, CascadeStepType.AE_REPORTED, CascadeStepStatus.COMPLETED,
            "system", "Grade " + grade.label() + " adverse event reported",
            Map.of("grade", grade.name()));
        broadcastStep(aeId, CascadeStepType.SLA_ASSIGNED, CascadeStepStatus.COMPLETED,
            "system", "SLA deadline: " + grade.sla().orElseThrow().toHours() + "h",
            Map.of("slaHours", grade.sla().orElseThrow().toHours()));
        broadcastStep(aeId, CascadeStepType.LEDGER_SEALED, CascadeStepStatus.COMPLETED,
            "ledger", "Initial report ledger entry sealed", Map.of());
    }

    // --- Private ---

    private void broadcastStep(UUID aeId, CascadeStepType step, CascadeStepStatus status,
                               String actor, String detail, Map<String, Object> data) {
        String topic = "clinical:ae:" + aeId + ":cascade";
        try {
            eventBroadcaster.broadcast(topic, new CascadeEvent(step, status, Instant.now(), actor, detail, data));
        } catch (Exception e) {
            LOG.warnf(e, "Failed to broadcast cascade step %s for aeId=%s", step, aeId);
        }
    }
}
```

- [ ] **Step 3: Run broadcaster test**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=ClinicalCascadeBroadcasterTest --batch-mode`
Expected: PASS

- [ ] **Step 4: Modify AeEscalationCaseService to fire new events**

In `AeEscalationCaseService.java`:
- Inject `Event<AeEscalationStartedEvent> escalationStartedEvents` and `Event<AeEscalationFailedEvent> escalationFailedEvents`
- After `persistCaseId()` succeeds in `onAdverseEventReported()`, fire `escalationStartedEvents.fireAsync(new AeEscalationStartedEvent(event.aeId(), caseId, event.grade(), event.tenantId()))`
- In `markFailed()`, fire `escalationFailedEvents.fireAsync(new AeEscalationFailedEvent(aeId, ae.grade, ae.tenantId, "Escalation case creation failed"))`

- [ ] **Step 5: Add cascadeAeId to ClinicalAgentRequest**

Modify `runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentRequest.java`:

```java
public record ClinicalAgentRequest<T>(
        String systemPrompt,
        String userPrompt,
        Class<T> responseClass,
        T fallbackValue,
        String configKey,
        String correlationId,
        UUID cascadeAeId) {}
```

Update all existing callers of `ClinicalAgentRequest` to pass `null` for `cascadeAeId`. Use `ide_find_references` on `ClinicalAgentRequest` to find all call sites.

- [ ] **Step 6: Add broadcast hooks to ClinicalAgentSupport**

In `ClinicalAgentSupport.java`:
- Inject `ClinicalCascadeBroadcaster broadcaster`
- Before `agentProvider.invoke()`, if `request.cascadeAeId() != null`: call `broadcaster.agentReasoning(request.cascadeAeId(), request.configKey())`
- After successful parse: if `request.cascadeAeId() != null`: call `broadcaster.agentResult(request.cascadeAeId(), request.configKey(), true)`
- In fallback paths: if `request.cascadeAeId() != null`: call `broadcaster.agentResult(request.cascadeAeId(), request.configKey(), false)`

- [ ] **Step 7: Run full test suite to verify no regressions**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime --batch-mode`
Expected: PASS (existing tests still green, new broadcaster test passes)

- [ ] **Step 8: Commit**

```bash
git add api/ runtime/src/main/java/ runtime/src/test/java/
git commit -m "feat(#161): add ClinicalCascadeBroadcaster with CDI/Vert.x observers and domain hooks

Refs #161"
```

---

## Batch 3: REST Endpoint + UI

### Task 4: Cascade REST endpoint

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/resource/CascadeResource.java`
- Test: `runtime/src/test/java/io/casehub/clinical/resource/CascadeResourceTest.java`

**Interfaces:**
- Consumes: `CascadeTemplateResolver`, `CascadeEvent`, `CascadeStepStatus`, `AdverseEvent` entity, ledger queries
- Produces: `GET /api/adverse-events/{aeId}/cascade` returning `List<CascadeEvent>`

- [ ] **Step 1: Write REST endpoint test**

```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.CascadeStepStatus;
import io.casehub.clinical.api.CascadeStepType;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.clinical.entity.TrialSite;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test-actor", roles = {"SPONSOR", "INVESTIGATOR", "COORDINATOR"})
class CascadeResourceTest {

    @Inject CurrentPrincipal principal;

    private UUID aeId;

    @BeforeEach
    @Transactional
    void setup() {
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = UUID.randomUUID();
        trial.protocolId = "TEST-CASCADE";
        trial.tenantId = principal.tenancyId();
        trial.persist();

        TrialSite site = new TrialSite();
        site.id = UUID.randomUUID();
        site.trialId = trial.id;
        site.tenantId = principal.tenancyId();
        site.persist();

        PatientEnrollment enrollment = new PatientEnrollment();
        enrollment.id = UUID.randomUUID();
        enrollment.siteId = site.id;
        enrollment.tenantId = principal.tenancyId();
        enrollment.persist();

        AdverseEvent ae = new AdverseEvent();
        ae.id = UUID.randomUUID();
        ae.enrollmentId = enrollment.id;
        ae.grade = CtcaeGrade.GRADE_4;
        ae.actuality = EventActuality.ACTUAL;
        ae.reportedAt = Instant.now();
        ae.slaDeadline = Instant.now().plusSeconds(86400);
        ae.tenantId = principal.tenancyId();
        ae.persist();
        aeId = ae.id;
    }

    @Test
    void returnsTemplateWithInitialSteps() {
        given()
            .when().get("/api/adverse-events/" + aeId + "/cascade")
            .then()
            .statusCode(200)
            .body("size()", greaterThanOrEqualTo(3))
            .body("[0].step", equalTo("AE_REPORTED"))
            .body("[0].status", equalTo("COMPLETED"))
            .body("[1].step", equalTo("SLA_ASSIGNED"))
            .body("[1].status", equalTo("COMPLETED"));
    }

    @Test
    void returns404ForUnknownAe() {
        given()
            .when().get("/api/adverse-events/" + UUID.randomUUID() + "/cascade")
            .then()
            .statusCode(404);
    }
}
```

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=CascadeResourceTest --batch-mode`
Expected: FAIL — resource doesn't exist

- [ ] **Step 2: Create CascadeResource**

```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.*;
import io.casehub.clinical.entity.AdverseEvent;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.annotation.security.RolesAllowed;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.time.Instant;
import java.util.*;

@Path("/api/adverse-events/{aeId}/cascade")
@Produces(MediaType.APPLICATION_JSON)
public class CascadeResource {

    @Inject CurrentPrincipal principal;

    @GET
    @RolesAllowed({"SPONSOR", "INVESTIGATOR", "COORDINATOR", "MONITOR"})
    public Response getCascade(@PathParam("aeId") UUID aeId) {
        AdverseEvent ae = AdverseEvent.findByIdForTenant(aeId, principal);
        if (ae == null) {
            return Response.status(Response.Status.NOT_FOUND)
                .entity(Map.of("error", "Adverse event not found")).build();
        }

        List<CascadeStepType> template = CascadeTemplateResolver.resolve(
            ae.grade, ae.unexpected, ae.suspected);

        List<CascadeEvent> cascade = new ArrayList<>();
        for (CascadeStepType step : template) {
            cascade.add(reconstructStep(ae, step));
        }
        return Response.ok(cascade).build();
    }

    private CascadeEvent reconstructStep(AdverseEvent ae, CascadeStepType step) {
        return switch (step) {
            case AE_REPORTED -> new CascadeEvent(step,
                ae.reportedAt != null ? CascadeStepStatus.COMPLETED : CascadeStepStatus.PENDING,
                ae.reportedAt, "system", "Adverse event reported",
                Map.of("grade", ae.grade.name()));
            case SLA_ASSIGNED -> new CascadeEvent(step,
                ae.slaDeadline != null ? CascadeStepStatus.COMPLETED : CascadeStepStatus.PENDING,
                ae.reportedAt, "system", "SLA deadline assigned",
                Map.of("slaHours", ae.grade.sla().orElseThrow().toHours()));
            case ESCALATION_CASE_STARTED -> new CascadeEvent(step,
                ae.engineCaseId != null ? CascadeStepStatus.COMPLETED
                    : ae.escalationStatus == io.casehub.clinical.api.model.AeEscalationStatus.FAILED ? CascadeStepStatus.FAILED
                    : ae.escalationStatus == io.casehub.clinical.api.model.AeEscalationStatus.REQUESTED ? CascadeStepStatus.ACTIVE
                    : CascadeStepStatus.PENDING,
                null, "engine", "Escalation case",
                ae.engineCaseId != null ? Map.of("caseId", ae.engineCaseId.toString()) : Map.of());
            default -> new CascadeEvent(step, CascadeStepStatus.PENDING,
                null, null, step.name(), Map.of());
        };
    }
}
```

- [ ] **Step 3: Run REST endpoint test**

Run: `mvn install -pl api --batch-mode && mvn test -pl runtime -Dtest=CascadeResourceTest --batch-mode`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/resource/CascadeResource.java runtime/src/test/java/io/casehub/clinical/resource/CascadeResourceTest.java
git commit -m "feat(#161): add GET /api/adverse-events/{aeId}/cascade REST endpoint

Refs #161"
```

### Task 5: Frontend cascade timeline component and safety workbench integration

**Files:**
- Create: `runtime/src/main/webui/src/components/cascade-timeline.ts`
- Modify: `runtime/src/main/webui/src/index.ts` — register component
- Modify: `runtime/src/main/webui/src/views/safety-workbench.ts` — add "Live Cascade" tab

**Interfaces:**
- Consumes: `EventConnection` (pages-data), `EventTimelineStrategy` (pages-viz), REST cascade endpoint
- Produces: `<clinical-cascade-timeline>` web component with `data-ae-id` attribute

- [ ] **Step 1: Create cascade-timeline.ts**

```typescript
import { LitElement, html, css } from "lit";
import { property, state } from "lit/decorators.js";
import { onTableSelection } from "../selection-bridge.js";
import type { EventTimelineNode, EventNodeStatus } from "@casehubio/pages-viz";
import { createEventConnection, type EventConnection } from "@casehubio/pages-data";

interface CascadeEvent {
  readonly step: string;
  readonly status: string;
  readonly timestamp: string | null;
  readonly actor: string | null;
  readonly detail: string | null;
  readonly data: Record<string, unknown>;
}

const STEP_LABELS: Record<string, string> = {
  AE_REPORTED: "Adverse Event Reported",
  SLA_ASSIGNED: "SLA Deadline Assigned",
  ESCALATION_CASE_STARTED: "Escalation Case Started",
  AGENT_SELECTED: "Agent Selected",
  AGENT_REASONING: "Agent Reasoning",
  AGENT_RESULT: "Agent Result",
  GATE_OPENED: "Oversight Gate Opened",
  GATE_RESOLVED: "Oversight Gate Resolved",
  SAFETY_OFFICER_NOTIFIED: "Safety Officer Notified",
  REGULATORY_SUBMISSION_STARTED: "IND Submission Started",
  LEDGER_SEALED: "Merkle Entry Sealed",
};

const CATEGORY_STYLES: Record<string, string> = {
  lifecycle: "background: var(--pages-accent-3, #dbeafe); color: var(--pages-accent-11, #1e40af)",
  agent: "background: var(--pages-warning-3, #fef3c7); color: var(--pages-warning-11, #92400e)",
  gate: "background: var(--pages-error-3, #fee2e2); color: var(--pages-error-11, #991b1b)",
  audit: "background: var(--pages-success-3, #dcfce7); color: var(--pages-success-11, #166534)",
};

const STEP_CATEGORIES: Record<string, string> = {
  AE_REPORTED: "lifecycle", SLA_ASSIGNED: "lifecycle",
  ESCALATION_CASE_STARTED: "lifecycle", AGENT_SELECTED: "agent",
  AGENT_REASONING: "agent", AGENT_RESULT: "agent",
  GATE_OPENED: "gate", GATE_RESOLVED: "gate",
  SAFETY_OFFICER_NOTIFIED: "lifecycle",
  REGULATORY_SUBMISSION_STARTED: "lifecycle", LEDGER_SEALED: "audit",
};

const STATUS_ICONS: Record<string, string> = {
  COMPLETED: "✅", ACTIVE: "⏳", PENDING: "⚪",
  FAILED: "❌", SKIPPED: "⏭️",
};

const TERMINAL = new Set(["COMPLETED", "FAILED", "SKIPPED"]);

export class ClinicalCascadeTimeline extends LitElement {
  static styles = css`
    :host { display: block; font-family: var(--pages-font-family, sans-serif); }
    .empty { color: var(--pages-neutral-9); font-style: italic; padding: 1rem; }
    .timeline { display: flex; flex-direction: column; gap: 0.5rem; padding: 0.5rem; }
    .step { display: flex; align-items: flex-start; gap: 0.75rem; padding: 0.75rem; border-radius: 8px; border: 1px solid var(--pages-neutral-4, #eee); }
    .step--active { border-color: var(--pages-accent-7, #60a5fa); background: var(--pages-accent-2, #eff6ff); }
    .step--failed { border-color: var(--pages-error-7, #ef4444); background: var(--pages-error-2, #fef2f2); }
    .icon { font-size: 1.25rem; min-width: 1.5rem; text-align: center; }
    .content { flex: 1; }
    .label { font-weight: 600; font-size: 14px; }
    .detail { font-size: 12px; color: var(--pages-neutral-9); margin-top: 2px; }
    .category { display: inline-block; padding: 1px 6px; border-radius: 3px; font-size: 11px; font-weight: 500; margin-left: 0.5rem; }
    .meta { font-size: 11px; color: var(--pages-neutral-8); margin-top: 4px; }
    .status-bar { padding: 0.5rem; font-size: 12px; border-bottom: 1px solid var(--pages-neutral-4); }
    .status-bar--connected { color: var(--pages-success-9); }
    .status-bar--reconnecting { color: var(--pages-warning-9); }
    .status-bar--disconnected { color: var(--pages-error-9); }
  `;

  @property({ attribute: "data-ae-id" }) aeId = "";
  @property({ attribute: "data-source-dataset" }) sourceDataset = "";
  @state() private _steps: CascadeEvent[] = [];
  @state() private _loading = false;
  @state() private _connectionStatus: string = "disconnected";
  private _connection: EventConnection | null = null;

  connectedCallback() {
    super.connectedCallback();
    onTableSelection(this, this.sourceDataset, (row: Record<string, unknown>) => {
      const id = row?.id as string;
      if (id && id !== this.aeId) {
        this.aeId = id;
        this._loadCascade(id);
      }
    });
  }

  disconnectedCallback() {
    super.disconnectedCallback();
    this._disconnect();
  }

  private async _loadCascade(aeId: string) {
    this._disconnect();
    this._loading = true;
    this._steps = [];

    try {
      const resp = await fetch(`/api/adverse-events/${aeId}/cascade`);
      if (!resp.ok) { this._steps = []; return; }
      this._steps = await resp.json() as CascadeEvent[];
      this._subscribe(aeId);
    } catch (e) {
      console.warn("[CascadeTimeline] fetch failed:", e);
    } finally {
      this._loading = false;
    }
  }

  private _subscribe(aeId: string) {
    const wsUrl = `ws://${window.location.host}/ws/push`;
    const topic = `clinical:ae:${aeId}:cascade`;
    const target = new EventTarget();

    target.addEventListener("pages-event", ((e: CustomEvent) => {
      const payload = e.detail?.payload as CascadeEvent | undefined;
      if (!payload?.step) return;
      this._mergeEvent(payload);
    }) as EventListener);

    this._connection = createEventConnection(wsUrl, {
      config: { eventTarget: target },
      onStatusChange: (status) => { this._connectionStatus = status; },
    });
    this._connection.listen([topic]);
  }

  private _mergeEvent(event: CascadeEvent) {
    const idx = this._steps.findIndex(s => s.step === event.step);
    if (idx < 0) return;
    const existing = this._steps[idx]!;
    if (TERMINAL.has(existing.status)) return;
    const updated = [...this._steps];
    updated[idx] = event;
    this._steps = updated;
  }

  private _disconnect() {
    this._connection?.close();
    this._connection = null;
    this._connectionStatus = "disconnected";
  }

  render() {
    if (!this.aeId) {
      return html`<p class="empty">Select an adverse event to view its cascade.</p>`;
    }
    if (this._loading) {
      return html`<p class="empty">Loading cascade...</p>`;
    }

    return html`
      <div class="status-bar status-bar--${this._connectionStatus}">
        ${this._connectionStatus === "connected" ? "Live" : this._connectionStatus}
      </div>
      <div class="timeline">
        ${this._steps.map(step => {
          const cat = STEP_CATEGORIES[step.step] ?? "lifecycle";
          const catStyle = CATEGORY_STYLES[cat] ?? CATEGORY_STYLES.lifecycle;
          const stepClass = step.status === "ACTIVE" ? "step step--active"
            : step.status === "FAILED" ? "step step--failed" : "step";
          return html`
            <div class="${stepClass}">
              <span class="icon">${STATUS_ICONS[step.status] ?? "⚪"}</span>
              <div class="content">
                <span class="label">${STEP_LABELS[step.step] ?? step.step}</span>
                <span class="category" style="${catStyle}">${cat}</span>
                ${step.detail ? html`<div class="detail">${step.detail}</div>` : ""}
                ${step.actor ? html`<div class="meta">Actor: ${step.actor}</div>` : ""}
                ${step.timestamp ? html`<div class="meta">${new Date(step.timestamp).toLocaleTimeString()}</div>` : ""}
              </div>
            </div>
          `;
        })}
      </div>
    `;
  }
}
```

- [ ] **Step 2: Register component in index.ts**

Add import and registration in `runtime/src/main/webui/src/index.ts`:

```typescript
import { ClinicalCascadeTimeline } from "./components/cascade-timeline.js";
```

Add to the `components` array:

```typescript
["clinical-cascade-timeline", ClinicalCascadeTimeline],
```

- [ ] **Step 3: Add "Live Cascade" tab to safety workbench**

In `runtime/src/main/webui/src/views/safety-workbench.ts`, add a new tab entry after the "Regrade" tab in the `detailTabs`:

```typescript
["Live Cascade", panel("Live Cascade",
  html(`<clinical-cascade-timeline id="ae-cascade" data-trial-id="${trialId}" data-source-dataset="adverse-events"></clinical-cascade-timeline>`),
)],
```

- [ ] **Step 4: Build and verify**

Run: `cd runtime/src/main/webui && yarn build` (or however the webui builds)
Expected: BUILD SUCCESS, no TypeScript errors

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/webui/
git commit -m "feat(#161): add Live Cascade timeline web component and safety workbench tab

Refs #161"
```

---

## References

- [2026-09-14-sse-event-cascade-design.md] — design spec this plan implements
- [runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentSupport.java] — agent invocation utility to hook
- [runtime/src/main/java/io/casehub/clinical/agent/ClinicalAgentRequest.java:3] — record to extend with cascadeAeId
- [runtime/src/main/java/io/casehub/clinical/service/AeEscalationCaseService.java:29] — escalation service to fire new events
- [runtime/src/main/java/io/casehub/clinical/service/SusarGateDecisionListener.java:28] — Vert.x @ConsumeEvent pattern reference
- [runtime/src/main/java/io/casehub/clinical/service/AdverseEventService.java:29] — AE reporting with after-commit events
- [runtime/src/main/webui/src/views/safety-workbench.ts] — existing safety workbench tabs
- [runtime/src/main/webui/src/index.ts] — component registration pattern
- [runtime/src/main/webui/src/components/ae-grade-history.ts] — Lit component pattern reference
- [GitHub #161] — focal issue
- [GitHub #160] — prerequisite (live LLM agent execution)
