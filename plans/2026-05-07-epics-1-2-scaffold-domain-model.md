# Epics 1 + 2 — Project Scaffold and Domain Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bootstrap the casehub-clinical Maven multi-module project and implement the full clinical trial domain model (entities, enums, constants, Flyway migrations, and REST registration endpoints).

**Architecture:** Two-module layout following casehub-work convention: `api/` (pure Java enums + constants, zero JPA) and `runtime/` (Quarkus app with Panache Active Record entities, Flyway migrations, REST endpoints). Entities extend `PanacheEntityBase` and serve as domain objects directly — no separate POJO mapping layer.

**Tech Stack:** Java 21 (JVM 26), Quarkus 3.32.2, Hibernate ORM Panache, Flyway, Quarkus REST + Jackson, H2 (tests), PostgreSQL (prod), JUnit 5, REST Assured.

**Spec:** `specs/2026-05-07-epics-1-2-scaffold-domain-model-design.md`
**Issues:** casehubio/clinical#1 (Epic 1 — scaffold), casehubio/clinical#2 (Epic 2 — domain model)

---

## Pre-flight: Platform Coherence Check

Before writing any code, verify:

- [ ] Read `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md` Step 4 — confirm module structure matches the persistence module split rule (exception applies: self-contained app repo)
- [ ] Confirm `casehub-clinical` is in the dependency order table in PLATFORM.md (it is — application tier, after engine/ledger/work/qhorus)
- [ ] Confirm no capability being implemented already exists in a foundation repo

---

## File Map

```
clinical/
├── pom.xml                                                    CREATE — parent POM
│
├── api/
│   ├── pom.xml                                                CREATE — pure Java module, no Quarkus
│   └── src/
│       ├── main/java/io/casehub/clinical/api/
│       │   ├── ClinicalCapabilities.java                      CREATE — capability tag constants
│       │   ├── ClinicalTrustDimensions.java                   CREATE — trust dimension constants
│       │   └── model/
│       │       ├── AeOutcome.java                             CREATE — enum
│       │       ├── ConsentStatus.java                         CREATE — enum
│       │       ├── CtcaeGrade.java                            CREATE — enum with sla() method
│       │       ├── DeviationSeverity.java                     CREATE — enum
│       │       ├── EnrollmentStatus.java                      CREATE — enum
│       │       ├── EventActuality.java                        CREATE — enum
│       │       ├── IrbDecision.java                           CREATE — enum
│       │       ├── PiApprovalStatus.java                      CREATE — enum
│       │       ├── SiteStatus.java                            CREATE — enum
│       │       ├── TrialPhase.java                            CREATE — enum
│       │       └── TrialStatus.java                           CREATE — enum
│       └── test/java/io/casehub/clinical/api/model/
│           └── CtcaeGradeTest.java                            CREATE — unit tests for sla()
│
└── runtime/
    ├── pom.xml                                                CREATE — Quarkus module
    └── src/
        ├── main/
        │   ├── java/io/casehub/clinical/
        │   │   ├── entity/
        │   │   │   ├── ClinicalTrial.java                     CREATE — Panache entity
        │   │   │   ├── TrialSite.java                         CREATE — Panache entity
        │   │   │   ├── PatientEnrollment.java                 CREATE — Panache entity
        │   │   │   ├── AdverseEvent.java                      CREATE — Panache entity
        │   │   │   ├── ProtocolDeviation.java                 CREATE — Panache entity
        │   │   │   └── IrbApproval.java                       CREATE — Panache entity
        │   │   └── resource/
        │   │       ├── TrialResource.java                     CREATE — POST/GET /trials
        │   │       ├── SiteResource.java                      CREATE — POST/GET /trials/{id}/sites
        │   │       └── PatientResource.java                   CREATE — POST/GET .../patients
        │   └── resources/
        │       ├── application.properties                     CREATE — datasource config
        │       └── db/migration/
        │           ├── V1__clinical_trial.sql                 CREATE
        │           ├── V2__trial_site.sql                     CREATE
        │           ├── V3__patient_enrollment.sql             CREATE
        │           ├── V4__adverse_event.sql                  CREATE
        │           ├── V5__protocol_deviation.sql             CREATE
        │           └── V6__irb_approval.sql                   CREATE
        └── test/
            ├── java/io/casehub/clinical/
            │   ├── entity/
            │   │   └── ClinicalTrialPersistenceTest.java      CREATE — @QuarkusTest round-trip
            │   └── resource/
            │       ├── TrialResourceTest.java                 CREATE — REST Assured
            │       ├── SiteResourceTest.java                  CREATE — REST Assured
            │       ├── PatientResourceTest.java               CREATE — REST Assured
            │       └── ShowcaseScenarioTest.java              CREATE — end-to-end happy path
            └── resources/
                └── application.properties                     CREATE — H2 test datasource
```

---

## Task 1: Parent POM and module scaffold

**Files:**
- Create: `pom.xml`
- Create: `api/pom.xml`
- Create: `runtime/pom.xml`

- [ ] **Step 1: Create the parent POM**

`pom.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>io.casehub</groupId>
  <artifactId>casehub-clinical-parent</artifactId>
  <version>0.1-SNAPSHOT</version>
  <packaging>pom</packaging>

  <name>CaseHub Clinical :: Parent</name>
  <description>Clinical trial coordination application — GCP/FDA compliance, multi-site sub-cases</description>

  <modules>
    <module>api</module>
    <module>runtime</module>
  </modules>

  <properties>
    <java.version>21</java.version>
    <maven.compiler.release>${java.version}</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <version.quarkus.platform>3.32.2</version.quarkus.platform>
    <version.io.casehub>0.2-SNAPSHOT</version.io.casehub>
    <maven.deploy.skip>false</maven.deploy.skip>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>io.quarkus.platform</groupId>
        <artifactId>quarkus-bom</artifactId>
        <version>${version.quarkus.platform}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-clinical-api</artifactId>
        <version>${project.version}</version>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <repositories>
    <repository>
      <id>github</id>
      <url>https://maven.pkg.github.com/casehubio/*</url>
      <snapshots><enabled>true</enabled></snapshots>
    </repository>
  </repositories>

  <build>
    <pluginManagement>
      <plugins>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-compiler-plugin</artifactId>
          <version>3.15.0</version>
        </plugin>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-surefire-plugin</artifactId>
          <version>3.5.5</version>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>
</project>
```

- [ ] **Step 2: Create `api/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-clinical-parent</artifactId>
    <version>0.1-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-clinical-api</artifactId>
  <name>CaseHub Clinical :: API</name>
  <description>Domain enums, capability tags, and trust dimension constants. Zero JPA, zero Quarkus.</description>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 3: Create `runtime/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-clinical-parent</artifactId>
    <version>0.1-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-clinical</artifactId>
  <name>CaseHub Clinical :: Runtime</name>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-clinical-api</artifactId>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-orm-panache</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-flyway</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-openapi</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-health</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-postgresql</artifactId>
      <optional>true</optional>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>rest-assured</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.quarkus.platform</groupId>
        <artifactId>quarkus-maven-plugin</artifactId>
        <version>${version.quarkus.platform}</version>
        <executions>
          <execution>
            <goals>
              <goal>build</goal>
              <goal>generate-code</goal>
              <goal>generate-code-tests</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 4: Verify the reactor builds**

```bash
mvn compile -pl api,runtime --batch-mode
```
Expected: BUILD SUCCESS for both modules (no source yet, just POMs).

- [ ] **Step 5: Commit**

```bash
git add pom.xml api/pom.xml runtime/pom.xml
git commit -m "feat: multi-module Maven scaffold (api + runtime)

Refs #1"
```

---

## Task 2: `CtcaeGrade` enum with SLA — TDD first

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/model/CtcaeGrade.java`
- Create: `api/src/test/java/io/casehub/clinical/api/model/CtcaeGradeTest.java`

- [ ] **Step 1: Write the failing test**

`api/src/test/java/io/casehub/clinical/api/model/CtcaeGradeTest.java`:
```java
package io.casehub.clinical.api.model;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import static org.assertj.core.api.Assertions.assertThat;

class CtcaeGradeTest {

    @Test
    void grade1_and_2_have_no_sla() {
        assertThat(CtcaeGrade.GRADE_1.sla()).isEmpty();
        assertThat(CtcaeGrade.GRADE_2.sla()).isEmpty();
    }

    @Test
    void grade3_and_4_have_24h_sla() {
        assertThat(CtcaeGrade.GRADE_3.sla()).contains(Duration.ofHours(24));
        assertThat(CtcaeGrade.GRADE_4.sla()).contains(Duration.ofHours(24));
    }

    @Test
    void grade5_has_1h_sla() {
        assertThat(CtcaeGrade.GRADE_5.sla()).contains(Duration.ofHours(1));
    }

    @Test
    void all_grades_have_labels() {
        for (CtcaeGrade grade : CtcaeGrade.values()) {
            assertThat(grade.label()).isNotBlank();
        }
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -pl api --batch-mode
```
Expected: FAIL — `CtcaeGrade` does not exist.

- [ ] **Step 3: Implement `CtcaeGrade`**

`api/src/main/java/io/casehub/clinical/api/model/CtcaeGrade.java`:
```java
package io.casehub.clinical.api.model;

import java.time.Duration;
import java.util.Optional;

/** CTCAE v5.0 adverse event severity grades with GCP-mandated SLA durations. */
public enum CtcaeGrade {
    GRADE_1("Mild",             null),
    GRADE_2("Moderate",         null),
    GRADE_3("Severe",           Duration.ofHours(24)),
    GRADE_4("Life-threatening", Duration.ofHours(24)),
    GRADE_5("Death",            Duration.ofHours(1));

    private final String label;
    private final Duration sla;

    CtcaeGrade(String label, Duration sla) {
        this.label = label;
        this.sla = sla;
    }

    public String label() { return label; }

    /** GCP-mandated reporting SLA. Empty for grades 1 and 2 (non-serious). */
    public Optional<Duration> sla() { return Optional.ofNullable(sla); }
}
```

- [ ] **Step 4: Run test — verify it passes**

```bash
mvn test -pl api --batch-mode
```
Expected: BUILD SUCCESS, all 4 tests pass.

- [ ] **Step 5: Commit**

```bash
git add api/src/
git commit -m "feat: CtcaeGrade enum with CTCAE v5.0 SLA durations

Grade 3/4 → 24h, Grade 5 → 1h per GCP ICH E6(R3). Grades 1/2 have no SLA.

Refs #2"
```

---

## Task 3: Remaining `api/` enums and constants

**Files:**
- Create: `api/src/main/java/io/casehub/clinical/api/model/TrialPhase.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/TrialStatus.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/SiteStatus.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/ConsentStatus.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/EnrollmentStatus.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/EventActuality.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/AeOutcome.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/DeviationSeverity.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/PiApprovalStatus.java`
- Create: `api/src/main/java/io/casehub/clinical/api/model/IrbDecision.java`
- Create: `api/src/main/java/io/casehub/clinical/api/ClinicalCapabilities.java`
- Create: `api/src/main/java/io/casehub/clinical/api/ClinicalTrustDimensions.java`
- Create: `api/src/test/java/io/casehub/clinical/api/ClinicalConstantsTest.java`

- [ ] **Step 1: Write the failing test**

`api/src/test/java/io/casehub/clinical/api/ClinicalConstantsTest.java`:
```java
package io.casehub.clinical.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ClinicalConstantsTest {

    @Test
    void capability_constants_are_kebab_case() {
        assertThat(ClinicalCapabilities.ELIGIBILITY_SCREENING).isEqualTo("eligibility-screening");
        assertThat(ClinicalCapabilities.SAFETY_MONITORING).isEqualTo("safety-monitoring");
        assertThat(ClinicalCapabilities.PROTOCOL_REVIEW).isEqualTo("protocol-review");
        assertThat(ClinicalCapabilities.IRB_CONSULTATION).isEqualTo("irb-consultation");
        assertThat(ClinicalCapabilities.PI_AUTHORISATION).isEqualTo("pi-authorisation");
        assertThat(ClinicalCapabilities.DATA_SAFETY_MONITORING).isEqualTo("data-safety-monitoring");
        assertThat(ClinicalCapabilities.REGULATORY_SUBMISSION).isEqualTo("regulatory-submission");
        assertThat(ClinicalCapabilities.TRIAL_SUPERVISOR).isEqualTo("trial-supervisor");
    }

    @Test
    void trust_dimension_constants_are_kebab_case() {
        assertThat(ClinicalTrustDimensions.SAFETY_ACCURACY).isEqualTo("safety-accuracy");
        assertThat(ClinicalTrustDimensions.ELIGIBILITY_PRECISION).isEqualTo("eligibility-precision");
        assertThat(ClinicalTrustDimensions.PROTOCOL_ADHERENCE).isEqualTo("protocol-adherence");
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -pl api --batch-mode
```
Expected: FAIL — classes do not exist.

- [ ] **Step 3: Create all enum files**

`api/src/main/java/io/casehub/clinical/api/model/TrialPhase.java`:
```java
package io.casehub.clinical.api.model;

/** Clinical trial phase per FHIR ResearchStudy.phase codes. */
public enum TrialPhase {
    EARLY_PHASE_I, PHASE_I, PHASE_I_II, PHASE_II, PHASE_II_III, PHASE_III, PHASE_IV
}
```

`api/src/main/java/io/casehub/clinical/api/model/TrialStatus.java`:
```java
package io.casehub.clinical.api.model;

public enum TrialStatus {
    PLANNING, ENROLLING, ACTIVE, COMPLETED, TERMINATED
}
```

`api/src/main/java/io/casehub/clinical/api/model/SiteStatus.java`:
```java
package io.casehub.clinical.api.model;

public enum SiteStatus {
    PENDING, ACTIVE, SUSPENDED, CLOSED
}
```

`api/src/main/java/io/casehub/clinical/api/model/ConsentStatus.java`:
```java
package io.casehub.clinical.api.model;

public enum ConsentStatus {
    PENDING, OBTAINED, WITHDRAWN
}
```

`api/src/main/java/io/casehub/clinical/api/model/EnrollmentStatus.java`:
```java
package io.casehub.clinical.api.model;

/** Subject participation state per FHIR ResearchSubject.subjectState. */
public enum EnrollmentStatus {
    CANDIDATE, SCREENING, ELIGIBLE, INELIGIBLE, ENROLLED, ON_STUDY, OFF_STUDY, WITHDRAWN
}
```

`api/src/main/java/io/casehub/clinical/api/model/EventActuality.java`:
```java
package io.casehub.clinical.api.model;

/** Whether the adverse event actually occurred or was a near-miss. Per FHIR AdverseEvent.actuality. */
public enum EventActuality {
    ACTUAL, POTENTIAL
}
```

`api/src/main/java/io/casehub/clinical/api/model/AeOutcome.java`:
```java
package io.casehub.clinical.api.model;

public enum AeOutcome {
    ONGOING, RESOLVING, RESOLVED, FATAL, UNKNOWN
}
```

`api/src/main/java/io/casehub/clinical/api/model/DeviationSeverity.java`:
```java
package io.casehub.clinical.api.model;

public enum DeviationSeverity {
    MINOR, MAJOR, CRITICAL
}
```

`api/src/main/java/io/casehub/clinical/api/model/PiApprovalStatus.java`:
```java
package io.casehub.clinical.api.model;

public enum PiApprovalStatus {
    PENDING, APPROVED, REJECTED
}
```

`api/src/main/java/io/casehub/clinical/api/model/IrbDecision.java`:
```java
package io.casehub.clinical.api.model;

public enum IrbDecision {
    PENDING, APPROVED, REJECTED, DEFERRED
}
```

- [ ] **Step 4: Create constants classes**

`api/src/main/java/io/casehub/clinical/api/ClinicalCapabilities.java`:
```java
package io.casehub.clinical.api;

/** Capability tags used in CasePlanModel bindings and agent registration. */
public final class ClinicalCapabilities {
    private ClinicalCapabilities() {}

    public static final String ELIGIBILITY_SCREENING  = "eligibility-screening";
    public static final String SAFETY_MONITORING      = "safety-monitoring";
    public static final String PROTOCOL_REVIEW        = "protocol-review";
    public static final String IRB_CONSULTATION       = "irb-consultation";
    public static final String PI_AUTHORISATION       = "pi-authorisation";
    public static final String DATA_SAFETY_MONITORING = "data-safety-monitoring";
    public static final String REGULATORY_SUBMISSION  = "regulatory-submission";
    public static final String TRIAL_SUPERVISOR       = "trial-supervisor";
}
```

`api/src/main/java/io/casehub/clinical/api/ClinicalTrustDimensions.java`:
```java
package io.casehub.clinical.api;

/** Trust dimension keys for agent trust scoring via casehub-ledger. */
public final class ClinicalTrustDimensions {
    private ClinicalTrustDimensions() {}

    public static final String SAFETY_ACCURACY        = "safety-accuracy";
    public static final String ELIGIBILITY_PRECISION  = "eligibility-precision";
    public static final String PROTOCOL_ADHERENCE     = "protocol-adherence";
}
```

- [ ] **Step 5: Run tests — verify all pass**

```bash
mvn test -pl api --batch-mode
```
Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 6: Commit**

```bash
git add api/src/
git commit -m "feat: clinical domain enums and capability/trust constants

TrialPhase, TrialStatus, SiteStatus, ConsentStatus, EnrollmentStatus,
EventActuality, AeOutcome, DeviationSeverity, PiApprovalStatus, IrbDecision.
ClinicalCapabilities and ClinicalTrustDimensions constants for case engine bindings.

Refs #2"
```

---

## Task 4: Flyway migrations and `application.properties`

**Files:**
- Create: `runtime/src/main/resources/application.properties`
- Create: `runtime/src/test/resources/application.properties`
- Create: `runtime/src/main/resources/db/migration/V1__clinical_trial.sql` through `V6__irb_approval.sql`

- [ ] **Step 1: Create `runtime/src/main/resources/application.properties`**

```properties
quarkus.application.name=casehub-clinical

# Production datasource (PostgreSQL)
quarkus.datasource.db-kind=postgresql
quarkus.hibernate-orm.schema-management.strategy=validate
quarkus.flyway.migrate-at-start=true

# Dev mode: H2 in-memory
%dev.quarkus.datasource.db-kind=h2
%dev.quarkus.datasource.jdbc.url=jdbc:h2:mem:clinical;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
%dev.quarkus.hibernate-orm.schema-management.strategy=none
```

- [ ] **Step 2: Create `runtime/src/test/resources/application.properties`**

Per `quarkus-test-database.md` convention — H2 with MODE=PostgreSQL:
```properties
quarkus.http.test-port=0
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:clinical;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.schema-management.strategy=none
quarkus.flyway.migrate-at-start=true
quarkus.scheduler.enabled=false
```

- [ ] **Step 3: Create Flyway migration V1 — `clinical_trial`**

`runtime/src/main/resources/db/migration/V1__clinical_trial.sql`:
```sql
CREATE TABLE clinical_trial (
    id                 UUID         NOT NULL,
    protocol_id        VARCHAR(255) NOT NULL,
    phase              VARCHAR(50)  NOT NULL,
    sponsor            VARCHAR(255) NOT NULL,
    target_enrollment  INTEGER      NOT NULL,
    status             VARCHAR(50)  NOT NULL DEFAULT 'PLANNING',
    CONSTRAINT pk_clinical_trial PRIMARY KEY (id)
);
```

- [ ] **Step 4: Create Flyway migration V2 — `trial_site`**

`runtime/src/main/resources/db/migration/V2__trial_site.sql`:
```sql
CREATE TABLE trial_site (
    id               UUID         NOT NULL,
    trial_id         UUID         NOT NULL,
    investigator_id  VARCHAR(255) NOT NULL,
    status           VARCHAR(50)  NOT NULL DEFAULT 'PENDING',
    CONSTRAINT pk_trial_site PRIMARY KEY (id),
    CONSTRAINT fk_trial_site_trial FOREIGN KEY (trial_id) REFERENCES clinical_trial(id)
);
```

- [ ] **Step 5: Create Flyway migration V3 — `patient_enrollment`**

`runtime/src/main/resources/db/migration/V3__patient_enrollment.sql`:
```sql
CREATE TABLE patient_enrollment (
    id                UUID         NOT NULL,
    site_id           UUID         NOT NULL,
    patient_id        VARCHAR(255) NOT NULL,
    consent_status    VARCHAR(50)  NOT NULL DEFAULT 'PENDING',
    enrollment_status VARCHAR(50)  NOT NULL DEFAULT 'CANDIDATE',
    enrolled_at       TIMESTAMP,
    CONSTRAINT pk_patient_enrollment PRIMARY KEY (id),
    CONSTRAINT fk_enrollment_site FOREIGN KEY (site_id) REFERENCES trial_site(id)
);
```

- [ ] **Step 6: Create Flyway migration V4 — `adverse_event`**

`runtime/src/main/resources/db/migration/V4__adverse_event.sql`:
```sql
CREATE TABLE adverse_event (
    id            UUID        NOT NULL,
    enrollment_id UUID        NOT NULL,
    grade         VARCHAR(50) NOT NULL,
    actuality     VARCHAR(50) NOT NULL DEFAULT 'ACTUAL',
    outcome       VARCHAR(50) NOT NULL DEFAULT 'ONGOING',
    occurred_at   TIMESTAMP   NOT NULL,
    reported_at   TIMESTAMP   NOT NULL,
    sla_deadline  TIMESTAMP,
    CONSTRAINT pk_adverse_event PRIMARY KEY (id),
    CONSTRAINT fk_ae_enrollment FOREIGN KEY (enrollment_id) REFERENCES patient_enrollment(id)
);
```

- [ ] **Step 7: Create Flyway migration V5 — `protocol_deviation`**

`runtime/src/main/resources/db/migration/V5__protocol_deviation.sql`:
```sql
CREATE TABLE protocol_deviation (
    id                 UUID         NOT NULL,
    site_id            UUID         NOT NULL,
    deviation_type     VARCHAR(255) NOT NULL,
    severity           VARCHAR(50)  NOT NULL,
    pi_approval_status VARCHAR(50)  NOT NULL DEFAULT 'PENDING',
    CONSTRAINT pk_protocol_deviation PRIMARY KEY (id),
    CONSTRAINT fk_deviation_site FOREIGN KEY (site_id) REFERENCES trial_site(id)
);
```

- [ ] **Step 8: Create Flyway migration V6 — `irb_approval`**

`runtime/src/main/resources/db/migration/V6__irb_approval.sql`:
```sql
CREATE TABLE irb_approval (
    id               UUID         NOT NULL,
    site_id          UUID         NOT NULL,
    review_type      VARCHAR(255) NOT NULL,
    committee_id     VARCHAR(255) NOT NULL,
    decision_deadline TIMESTAMP   NOT NULL,
    decision         VARCHAR(50)  NOT NULL DEFAULT 'PENDING',
    CONSTRAINT pk_irb_approval PRIMARY KEY (id),
    CONSTRAINT fk_irb_site FOREIGN KEY (site_id) REFERENCES trial_site(id)
);
```

- [ ] **Step 9: Verify migrations compile and apply cleanly**

Write a minimal `@QuarkusTest` that just starts the app (Flyway runs on startup):

`runtime/src/test/java/io/casehub/clinical/FlywayMigrationTest.java`:
```java
package io.casehub.clinical;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

@QuarkusTest
class FlywayMigrationTest {

    @Test
    void migrations_apply_cleanly() {
        // If Quarkus starts without exception, all 6 migrations ran successfully.
    }
}
```

```bash
mvn test -pl runtime --batch-mode
```
Expected: BUILD SUCCESS — Quarkus starts, Flyway applies V1–V6 against H2.

- [ ] **Step 10: Commit**

```bash
git add runtime/src/main/resources/ runtime/src/test/resources/ runtime/src/test/java/io/casehub/clinical/FlywayMigrationTest.java
git commit -m "feat: Flyway migrations V1-V6 for clinical domain tables

clinical_trial, trial_site, patient_enrollment, adverse_event,
protocol_deviation, irb_approval. H2 MODE=PostgreSQL for test isolation.
sla_deadline nullable on adverse_event (null for Grade 1/2).

Refs #2"
```

---

## Task 5: JPA entities

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/entity/ClinicalTrial.java`
- Create: `runtime/src/main/java/io/casehub/clinical/entity/TrialSite.java`
- Create: `runtime/src/main/java/io/casehub/clinical/entity/PatientEnrollment.java`
- Create: `runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java`
- Create: `runtime/src/main/java/io/casehub/clinical/entity/ProtocolDeviation.java`
- Create: `runtime/src/main/java/io/casehub/clinical/entity/IrbApproval.java`
- Create: `runtime/src/test/java/io/casehub/clinical/entity/ClinicalTrialPersistenceTest.java`

- [ ] **Step 1: Write the failing persistence test**

`runtime/src/test/java/io/casehub/clinical/entity/ClinicalTrialPersistenceTest.java`:
```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class ClinicalTrialPersistenceTest {

    @Inject EntityManager em;

    @Test
    @Transactional
    void clinical_trial_round_trips() {
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = UUID.randomUUID();
        trial.protocolId = "ONCOL-001";
        trial.phase = TrialPhase.PHASE_III;
        trial.sponsor = "Acme Pharma";
        trial.targetEnrollment = 150;
        trial.status = TrialStatus.PLANNING;
        em.persist(trial);
        em.flush();
        em.clear();

        ClinicalTrial found = em.find(ClinicalTrial.class, trial.id);
        assertThat(found.protocolId).isEqualTo("ONCOL-001");
        assertThat(found.phase).isEqualTo(TrialPhase.PHASE_III);
        assertThat(found.targetEnrollment).isEqualTo(150);
        assertThat(found.status).isEqualTo(TrialStatus.PLANNING);
    }

    @Test
    @Transactional
    void trial_site_links_to_trial() {
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = UUID.randomUUID();
        trial.protocolId = "ONCOL-002";
        trial.phase = TrialPhase.PHASE_II;
        trial.sponsor = "BioTest";
        trial.targetEnrollment = 50;
        trial.status = TrialStatus.PLANNING;
        em.persist(trial);

        TrialSite site = new TrialSite();
        site.id = UUID.randomUUID();
        site.trialId = trial.id;
        site.investigatorId = "pi-alice-001";
        site.status = SiteStatus.PENDING;
        em.persist(site);
        em.flush();
        em.clear();

        TrialSite found = em.find(TrialSite.class, site.id);
        assertThat(found.trialId).isEqualTo(trial.id);
        assertThat(found.investigatorId).isEqualTo("pi-alice-001");
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -pl runtime -Dtest=ClinicalTrialPersistenceTest --batch-mode
```
Expected: FAIL — entity classes do not exist.

- [ ] **Step 3: Create all entity classes**

`runtime/src/main/java/io/casehub/clinical/entity/ClinicalTrial.java`:
```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.TrialPhase;
import io.casehub.clinical.api.model.TrialStatus;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.util.UUID;

@Entity
@Table(name = "clinical_trial")
public class ClinicalTrial extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "protocol_id", nullable = false)
    public String protocolId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public TrialPhase phase;

    @Column(nullable = false)
    public String sponsor;

    @Column(name = "target_enrollment", nullable = false)
    public int targetEnrollment;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public TrialStatus status = TrialStatus.PLANNING;
}
```

`runtime/src/main/java/io/casehub/clinical/entity/TrialSite.java`:
```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.SiteStatus;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.util.UUID;

@Entity
@Table(name = "trial_site")
public class TrialSite extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "trial_id", nullable = false)
    public UUID trialId;

    @Column(name = "investigator_id", nullable = false)
    public String investigatorId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public SiteStatus status = SiteStatus.PENDING;
}
```

`runtime/src/main/java/io/casehub/clinical/entity/PatientEnrollment.java`:
```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.ConsentStatus;
import io.casehub.clinical.api.model.EnrollmentStatus;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "patient_enrollment")
public class PatientEnrollment extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "site_id", nullable = false)
    public UUID siteId;

    @Column(name = "patient_id", nullable = false)
    public String patientId;

    @Enumerated(EnumType.STRING)
    @Column(name = "consent_status", nullable = false)
    public ConsentStatus consentStatus = ConsentStatus.PENDING;

    @Enumerated(EnumType.STRING)
    @Column(name = "enrollment_status", nullable = false)
    public EnrollmentStatus enrollmentStatus = EnrollmentStatus.CANDIDATE;

    @Column(name = "enrolled_at")
    public Instant enrolledAt;
}
```

`runtime/src/main/java/io/casehub/clinical/entity/AdverseEvent.java`:
```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.AeOutcome;
import io.casehub.clinical.api.model.CtcaeGrade;
import io.casehub.clinical.api.model.EventActuality;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "adverse_event")
public class AdverseEvent extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "enrollment_id", nullable = false)
    public UUID enrollmentId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public CtcaeGrade grade;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public EventActuality actuality = EventActuality.ACTUAL;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public AeOutcome outcome = AeOutcome.ONGOING;

    @Column(name = "occurred_at", nullable = false)
    public Instant occurredAt;

    @Column(name = "reported_at", nullable = false)
    public Instant reportedAt;

    /** Null for Grade 1 and 2 (no GCP SLA). Computed from reportedAt + grade.sla(). */
    @Column(name = "sla_deadline")
    public Instant slaDeadline;
}
```

`runtime/src/main/java/io/casehub/clinical/entity/ProtocolDeviation.java`:
```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.DeviationSeverity;
import io.casehub.clinical.api.model.PiApprovalStatus;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.util.UUID;

@Entity
@Table(name = "protocol_deviation")
public class ProtocolDeviation extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "site_id", nullable = false)
    public UUID siteId;

    @Column(name = "deviation_type", nullable = false)
    public String deviationType;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public DeviationSeverity severity;

    @Enumerated(EnumType.STRING)
    @Column(name = "pi_approval_status", nullable = false)
    public PiApprovalStatus piApprovalStatus = PiApprovalStatus.PENDING;
}
```

`runtime/src/main/java/io/casehub/clinical/entity/IrbApproval.java`:
```java
package io.casehub.clinical.entity;

import io.casehub.clinical.api.model.IrbDecision;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "irb_approval")
public class IrbApproval extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "site_id", nullable = false)
    public UUID siteId;

    @Column(name = "review_type", nullable = false)
    public String reviewType;

    @Column(name = "committee_id", nullable = false)
    public String committeeId;

    @Column(name = "decision_deadline", nullable = false)
    public Instant decisionDeadline;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public IrbDecision decision = IrbDecision.PENDING;
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
mvn test -pl runtime -Dtest=ClinicalTrialPersistenceTest --batch-mode
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/entity/
git commit -m "feat: JPA entities for clinical trial domain model

ClinicalTrial, TrialSite, PatientEnrollment, AdverseEvent,
ProtocolDeviation, IrbApproval — Panache Active Record pattern.
AdverseEvent.slaDeadline nullable (null for Grade 1/2, per CTCAE v5.0).

Refs #2"
```

---

## Task 6: `TrialResource` — register and retrieve trials

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/resource/TrialResource.java`
- Create: `runtime/src/test/java/io/casehub/clinical/resource/TrialResourceTest.java`

- [ ] **Step 1: Write the failing REST test**

`runtime/src/test/java/io/casehub/clinical/resource/TrialResourceTest.java`:
```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.model.TrialPhase;
import io.casehub.clinical.api.model.TrialStatus;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class TrialResourceTest {

    @Test
    void post_trial_returns_201_with_location() {
        given()
            .contentType("application/json")
            .body("""
                {
                  "protocolId": "ONCOL-001",
                  "phase": "PHASE_III",
                  "sponsor": "Acme Pharma",
                  "targetEnrollment": 150
                }
                """)
        .when()
            .post("/trials")
        .then()
            .statusCode(201)
            .header("Location", containsString("/trials/"));
    }

    @Test
    void get_trial_returns_200_with_fields() {
        String location =
            given()
                .contentType("application/json")
                .body("""
                    {
                      "protocolId": "ONCOL-002",
                      "phase": "PHASE_II",
                      "sponsor": "BioTest",
                      "targetEnrollment": 50
                    }
                    """)
            .when()
                .post("/trials")
            .then()
                .statusCode(201)
                .extract().header("Location");

        given()
        .when()
            .get(location)
        .then()
            .statusCode(200)
            .body("protocolId", equalTo("ONCOL-002"))
            .body("phase", equalTo("PHASE_II"))
            .body("sponsor", equalTo("BioTest"))
            .body("targetEnrollment", equalTo(50))
            .body("status", equalTo("PLANNING"));
    }

    @Test
    void get_unknown_trial_returns_404() {
        given()
        .when()
            .get("/trials/" + UUID.randomUUID())
        .then()
            .statusCode(404);
    }

    @Test
    void post_trial_missing_protocol_id_returns_400() {
        given()
            .contentType("application/json")
            .body("""
                {
                  "phase": "PHASE_III",
                  "sponsor": "Acme",
                  "targetEnrollment": 100
                }
                """)
        .when()
            .post("/trials")
        .then()
            .statusCode(400);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -pl runtime -Dtest=TrialResourceTest --batch-mode
```
Expected: FAIL — endpoint does not exist.

- [ ] **Step 3: Implement `TrialResource`**

`runtime/src/main/java/io/casehub/clinical/resource/TrialResource.java`:
```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.model.TrialPhase;
import io.casehub.clinical.api.model.TrialStatus;
import io.casehub.clinical.entity.ClinicalTrial;
import jakarta.transaction.Transactional;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;
import java.net.URI;
import java.util.UUID;

@Path("/trials")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class TrialResource {

    public record RegisterTrialRequest(
        @NotBlank String protocolId,
        TrialPhase phase,
        @NotBlank String sponsor,
        @Positive int targetEnrollment
    ) {}

    @POST
    @Transactional
    public Response register(@Valid RegisterTrialRequest req, @Context UriInfo uriInfo) {
        ClinicalTrial trial = new ClinicalTrial();
        trial.id = UUID.randomUUID();
        trial.protocolId = req.protocolId();
        trial.phase = req.phase();
        trial.sponsor = req.sponsor();
        trial.targetEnrollment = req.targetEnrollment();
        trial.status = TrialStatus.PLANNING;
        trial.persist();

        URI location = uriInfo.getAbsolutePathBuilder().path(trial.id.toString()).build();
        return Response.created(location).build();
    }

    @GET
    @Path("/{id}")
    public Response get(@PathParam("id") UUID id) {
        ClinicalTrial trial = ClinicalTrial.findById(id);
        if (trial == null) return Response.status(Response.Status.NOT_FOUND).build();
        return Response.ok(trial).build();
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
mvn test -pl runtime -Dtest=TrialResourceTest --batch-mode
```
Expected: BUILD SUCCESS, all 4 tests pass.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/resource/TrialResource.java \
        runtime/src/test/java/io/casehub/clinical/resource/TrialResourceTest.java
git commit -m "feat: POST /trials and GET /trials/{id} endpoints

Registers ClinicalTrial with validation. 201 + Location on success,
404 on unknown id, 400 on missing required fields.

Refs #2"
```

---

## Task 7: `SiteResource` — add and retrieve sites

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/resource/SiteResource.java`
- Create: `runtime/src/test/java/io/casehub/clinical/resource/SiteResourceTest.java`

- [ ] **Step 1: Write the failing test**

`runtime/src/test/java/io/casehub/clinical/resource/SiteResourceTest.java`:
```java
package io.casehub.clinical.resource;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class SiteResourceTest {

    private String createTrial() {
        return given()
            .contentType("application/json")
            .body("""
                {"protocolId":"SITE-TEST","phase":"PHASE_II","sponsor":"Test","targetEnrollment":10}
                """)
        .when()
            .post("/trials")
        .then()
            .statusCode(201)
            .extract().header("Location");
    }

    @Test
    void post_site_returns_201_with_location() {
        String trialLocation = createTrial();
        UUID trialId = UUID.fromString(trialLocation.substring(trialLocation.lastIndexOf('/') + 1));

        given()
            .contentType("application/json")
            .body("""
                {"investigatorId": "pi-alice-001"}
                """)
        .when()
            .post("/trials/{id}/sites", trialId)
        .then()
            .statusCode(201)
            .header("Location", containsString("/sites/"));
    }

    @Test
    void get_site_returns_investigator_id_and_status() {
        String trialLocation = createTrial();
        UUID trialId = UUID.fromString(trialLocation.substring(trialLocation.lastIndexOf('/') + 1));

        String siteLocation =
            given()
                .contentType("application/json")
                .body("""{"investigatorId": "pi-bob-002"}""")
            .when()
                .post("/trials/{id}/sites", trialId)
            .then()
                .statusCode(201)
                .extract().header("Location");

        given()
        .when()
            .get(siteLocation)
        .then()
            .statusCode(200)
            .body("investigatorId", equalTo("pi-bob-002"))
            .body("status", equalTo("PENDING"));
    }

    @Test
    void post_site_to_unknown_trial_returns_404() {
        given()
            .contentType("application/json")
            .body("""{"investigatorId": "pi-x"}""")
        .when()
            .post("/trials/{id}/sites", UUID.randomUUID())
        .then()
            .statusCode(404);
    }

    @Test
    void get_unknown_site_returns_404() {
        String trialLocation = createTrial();
        UUID trialId = UUID.fromString(trialLocation.substring(trialLocation.lastIndexOf('/') + 1));

        given()
        .when()
            .get("/trials/{trialId}/sites/{siteId}", trialId, UUID.randomUUID())
        .then()
            .statusCode(404);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -pl runtime -Dtest=SiteResourceTest --batch-mode
```
Expected: FAIL — endpoint does not exist.

- [ ] **Step 3: Implement `SiteResource`**

`runtime/src/main/java/io/casehub/clinical/resource/SiteResource.java`:
```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.model.SiteStatus;
import io.casehub.clinical.entity.ClinicalTrial;
import io.casehub.clinical.entity.TrialSite;
import jakarta.transaction.Transactional;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;
import java.net.URI;
import java.util.UUID;

@Path("/trials/{trialId}/sites")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class SiteResource {

    public record AddSiteRequest(@NotBlank String investigatorId) {}

    @POST
    @Transactional
    public Response add(@PathParam("trialId") UUID trialId,
                        @Valid AddSiteRequest req,
                        @Context UriInfo uriInfo) {
        if (ClinicalTrial.findById(trialId) == null)
            return Response.status(Response.Status.NOT_FOUND).build();

        TrialSite site = new TrialSite();
        site.id = UUID.randomUUID();
        site.trialId = trialId;
        site.investigatorId = req.investigatorId();
        site.status = SiteStatus.PENDING;
        site.persist();

        URI location = uriInfo.getAbsolutePathBuilder().path(site.id.toString()).build();
        return Response.created(location).build();
    }

    @GET
    @Path("/{siteId}")
    public Response get(@PathParam("trialId") UUID trialId,
                        @PathParam("siteId") UUID siteId) {
        TrialSite site = TrialSite.findById(siteId);
        if (site == null || !site.trialId.equals(trialId))
            return Response.status(Response.Status.NOT_FOUND).build();
        return Response.ok(site).build();
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
mvn test -pl runtime -Dtest=SiteResourceTest --batch-mode
```
Expected: BUILD SUCCESS, all 4 tests pass.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/resource/SiteResource.java \
        runtime/src/test/java/io/casehub/clinical/resource/SiteResourceTest.java
git commit -m "feat: POST /trials/{id}/sites and GET .../sites/{id} endpoints

Validates trial exists (404 if not). 201 + Location on success.
Site GET validates trialId ownership to prevent cross-trial access.

Refs #2"
```

---

## Task 8: `PatientResource` — enroll and retrieve patients

**Files:**
- Create: `runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java`
- Create: `runtime/src/test/java/io/casehub/clinical/resource/PatientResourceTest.java`

- [ ] **Step 1: Write the failing test**

`runtime/src/test/java/io/casehub/clinical/resource/PatientResourceTest.java`:
```java
package io.casehub.clinical.resource;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class PatientResourceTest {

    private UUID createTrialAndSite() {
        String trialLoc = given()
            .contentType("application/json")
            .body("""{"protocolId":"PAT-TEST","phase":"PHASE_I","sponsor":"T","targetEnrollment":5}""")
        .when().post("/trials").then().statusCode(201).extract().header("Location");

        UUID trialId = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));

        String siteLoc = given()
            .contentType("application/json")
            .body("""{"investigatorId":"pi-carol-003"}""")
        .when().post("/trials/{id}/sites", trialId).then().statusCode(201).extract().header("Location");

        return UUID.fromString(siteLoc.substring(siteLoc.lastIndexOf('/') + 1));
    }

    @Test
    void enroll_patient_returns_201() {
        UUID siteId = createTrialAndSite();
        String trialId = given().when().get("/trials/{id}/sites/{sid}",
            /* extract trialId from site response */ UUID.randomUUID(), siteId)
            .then().extract().path("trialId");

        // Simpler: just test via the direct URL we know
        // Re-create for simplicity in this test
        String trialLoc = given()
            .contentType("application/json")
            .body("""{"protocolId":"ENROLL-001","phase":"PHASE_I","sponsor":"T","targetEnrollment":5}""")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
        UUID tid = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));
        String siteLoc = given()
            .contentType("application/json")
            .body("""{"investigatorId":"pi-x"}""")
        .when().post("/trials/{id}/sites", tid).then().statusCode(201).extract().header("Location");
        UUID sid = UUID.fromString(siteLoc.substring(siteLoc.lastIndexOf('/') + 1));

        given()
            .contentType("application/json")
            .body("""{"patientId": "PATIENT-ALPHA-001"}""")
        .when()
            .post("/trials/{trialId}/sites/{siteId}/patients", tid, sid)
        .then()
            .statusCode(201)
            .header("Location", containsString("/patients/"));
    }

    @Test
    void get_enrollment_returns_status_fields() {
        String trialLoc = given()
            .contentType("application/json")
            .body("""{"protocolId":"ENROLL-002","phase":"PHASE_I","sponsor":"T","targetEnrollment":5}""")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
        UUID tid = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));
        String siteLoc = given()
            .contentType("application/json")
            .body("""{"investigatorId":"pi-y"}""")
        .when().post("/trials/{id}/sites", tid).then().statusCode(201).extract().header("Location");
        UUID sid = UUID.fromString(siteLoc.substring(siteLoc.lastIndexOf('/') + 1));

        String patientLoc = given()
            .contentType("application/json")
            .body("""{"patientId": "PATIENT-BETA-002"}""")
        .when()
            .post("/trials/{trialId}/sites/{siteId}/patients", tid, sid)
        .then()
            .statusCode(201).extract().header("Location");

        given().when().get(patientLoc)
        .then()
            .statusCode(200)
            .body("patientId", equalTo("PATIENT-BETA-002"))
            .body("consentStatus", equalTo("PENDING"))
            .body("enrollmentStatus", equalTo("CANDIDATE"));
    }

    @Test
    void enroll_to_unknown_site_returns_404() {
        String trialLoc = given()
            .contentType("application/json")
            .body("""{"protocolId":"ENROLL-003","phase":"PHASE_I","sponsor":"T","targetEnrollment":5}""")
        .when().post("/trials").then().statusCode(201).extract().header("Location");
        UUID tid = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));

        given()
            .contentType("application/json")
            .body("""{"patientId":"PATIENT-X"}""")
        .when()
            .post("/trials/{trialId}/sites/{siteId}/patients", tid, UUID.randomUUID())
        .then()
            .statusCode(404);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
mvn test -pl runtime -Dtest=PatientResourceTest --batch-mode
```
Expected: FAIL — endpoint does not exist.

- [ ] **Step 3: Implement `PatientResource`**

`runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java`:
```java
package io.casehub.clinical.resource;

import io.casehub.clinical.api.model.ConsentStatus;
import io.casehub.clinical.api.model.EnrollmentStatus;
import io.casehub.clinical.entity.PatientEnrollment;
import io.casehub.clinical.entity.TrialSite;
import jakarta.transaction.Transactional;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;
import java.net.URI;
import java.util.UUID;

@Path("/trials/{trialId}/sites/{siteId}/patients")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class PatientResource {

    public record EnrollPatientRequest(@NotBlank String patientId) {}

    @POST
    @Transactional
    public Response enroll(@PathParam("trialId") UUID trialId,
                           @PathParam("siteId") UUID siteId,
                           @Valid EnrollPatientRequest req,
                           @Context UriInfo uriInfo) {
        TrialSite site = TrialSite.findById(siteId);
        if (site == null || !site.trialId.equals(trialId))
            return Response.status(Response.Status.NOT_FOUND).build();

        PatientEnrollment enrollment = new PatientEnrollment();
        enrollment.id = UUID.randomUUID();
        enrollment.siteId = siteId;
        enrollment.patientId = req.patientId();
        enrollment.consentStatus = ConsentStatus.PENDING;
        enrollment.enrollmentStatus = EnrollmentStatus.CANDIDATE;
        enrollment.persist();

        URI location = uriInfo.getAbsolutePathBuilder().path(enrollment.id.toString()).build();
        return Response.created(location).build();
    }

    @GET
    @Path("/{enrollmentId}")
    public Response get(@PathParam("trialId") UUID trialId,
                        @PathParam("siteId") UUID siteId,
                        @PathParam("enrollmentId") UUID enrollmentId) {
        PatientEnrollment enrollment = PatientEnrollment.findById(enrollmentId);
        if (enrollment == null || !enrollment.siteId.equals(siteId))
            return Response.status(Response.Status.NOT_FOUND).build();
        return Response.ok(enrollment).build();
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
mvn test -pl runtime -Dtest=PatientResourceTest --batch-mode
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/clinical/resource/PatientResource.java \
        runtime/src/test/java/io/casehub/clinical/resource/PatientResourceTest.java
git commit -m "feat: POST/GET patient enrollment endpoints

patientId stored as opaque string — no PII validation. CANDIDATE/PENDING
defaults. 404 if site doesn't exist or doesn't belong to trial.

Refs #2"
```

---

## Task 9: End-to-end showcase scenario test

**Files:**
- Create: `runtime/src/test/java/io/casehub/clinical/resource/ShowcaseScenarioTest.java`

- [ ] **Step 1: Write end-to-end test for 3-site trial registration**

`runtime/src/test/java/io/casehub/clinical/resource/ShowcaseScenarioTest.java`:
```java
package io.casehub.clinical.resource;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

/**
 * End-to-end happy-path test for the 3-site oncology showcase scenario.
 * Verifies the domain layer can support the full trial registration flow
 * that Epic 3 will wire to sub-case orchestration.
 */
@QuarkusTest
class ShowcaseScenarioTest {

    @Test
    void three_site_oncology_trial_registers_correctly() {
        // Register the trial
        String trialLoc = given()
            .contentType("application/json")
            .body("""
                {
                  "protocolId": "ONCOL-PHASE3-2026-001",
                  "phase": "PHASE_III",
                  "sponsor": "Acme Oncology",
                  "targetEnrollment": 300
                }
                """)
        .when().post("/trials").then().statusCode(201).extract().header("Location");

        UUID trialId = UUID.fromString(trialLoc.substring(trialLoc.lastIndexOf('/') + 1));

        // Add 3 sites
        UUID siteAId = addSite(trialId, "pi-site-a-001");
        UUID siteBId = addSite(trialId, "pi-site-b-002");
        UUID siteCId = addSite(trialId, "pi-site-c-003");

        // Enroll a patient at each site
        UUID patientA = enrollPatient(trialId, siteAId, "PATIENT-SITE-A-001");
        UUID patientB = enrollPatient(trialId, siteBId, "PATIENT-SITE-B-001");
        UUID patientC = enrollPatient(trialId, siteCId, "PATIENT-SITE-C-001");

        // Verify trial
        given().when().get("/trials/{id}", trialId)
        .then()
            .statusCode(200)
            .body("protocolId", equalTo("ONCOL-PHASE3-2026-001"))
            .body("phase", equalTo("PHASE_III"))
            .body("targetEnrollment", equalTo(300))
            .body("status", equalTo("PLANNING"));

        // Verify all 3 sites are retrievable under the trial
        assertSiteExists(trialId, siteAId, "pi-site-a-001");
        assertSiteExists(trialId, siteBId, "pi-site-b-002");
        assertSiteExists(trialId, siteCId, "pi-site-c-003");

        // Verify all 3 patients enrolled as CANDIDATE/PENDING
        assertEnrollmentExists(trialId, siteAId, patientA, "PATIENT-SITE-A-001");
        assertEnrollmentExists(trialId, siteBId, patientB, "PATIENT-SITE-B-001");
        assertEnrollmentExists(trialId, siteCId, patientC, "PATIENT-SITE-C-001");
    }

    private UUID addSite(UUID trialId, String investigatorId) {
        String loc = given()
            .contentType("application/json")
            .body("{\"investigatorId\": \"" + investigatorId + "\"}")
        .when()
            .post("/trials/{id}/sites", trialId)
        .then()
            .statusCode(201).extract().header("Location");
        return UUID.fromString(loc.substring(loc.lastIndexOf('/') + 1));
    }

    private UUID enrollPatient(UUID trialId, UUID siteId, String patientId) {
        String loc = given()
            .contentType("application/json")
            .body("{\"patientId\": \"" + patientId + "\"}")
        .when()
            .post("/trials/{trialId}/sites/{siteId}/patients", trialId, siteId)
        .then()
            .statusCode(201).extract().header("Location");
        return UUID.fromString(loc.substring(loc.lastIndexOf('/') + 1));
    }

    private void assertSiteExists(UUID trialId, UUID siteId, String investigatorId) {
        given().when().get("/trials/{trialId}/sites/{siteId}", trialId, siteId)
        .then()
            .statusCode(200)
            .body("investigatorId", equalTo(investigatorId))
            .body("status", equalTo("PENDING"));
    }

    private void assertEnrollmentExists(UUID trialId, UUID siteId, UUID enrollmentId, String patientId) {
        given().when().get("/trials/{t}/sites/{s}/patients/{e}", trialId, siteId, enrollmentId)
        .then()
            .statusCode(200)
            .body("patientId", equalTo(patientId))
            .body("enrollmentStatus", equalTo("CANDIDATE"))
            .body("consentStatus", equalTo("PENDING"));
    }
}
```

- [ ] **Step 2: Run all tests**

```bash
mvn test -pl runtime --batch-mode
```
Expected: BUILD SUCCESS — all tests pass including the showcase scenario.

- [ ] **Step 3: Run full reactor test**

```bash
mvn test --batch-mode
```
Expected: BUILD SUCCESS across both `api` and `runtime`.

- [ ] **Step 4: Commit and close epics**

```bash
git add runtime/src/test/java/io/casehub/clinical/resource/ShowcaseScenarioTest.java
git commit -m "test: end-to-end 3-site oncology trial showcase scenario

Registers trial, adds 3 sites with independent investigators, enrolls
one patient per site. Validates all state is persisted correctly.
Domain layer ready for Epic 3 sub-case orchestration wiring.

Closes #1
Closes #2"
```

---

## Post-Implementation Checklist

- [ ] Run `superpowers:requesting-code-review` on the implementation
- [ ] Update `PLATFORM.md` capability ownership table to include `casehub-clinical` (already listed — verify)
- [ ] Update casehub-clinical `CLAUDE.md` Build Commands section with confirmed `mvn test -pl api` and `mvn test -pl runtime` commands
- [ ] Check all cross-references in the spec file are still accurate
- [ ] Verify GitHub issues #1 and #2 are closed by the final commit
