# TESTING.md — Spock Testing in Moqui Framework

## Overview

Moqui uses **Spock Framework** for testing, running on **JUnit 5 Platform**. Tests are written in Groovy and extend `spock.lang.Specification`. The framework provides a full in-process test environment — no external server required.

---

## Test Directory Structure

Tests live under `src/test/groovy/` in each component:

```
runtime/component/YourComponent/
├── src/
│   └── test/
│       └── groovy/
│           ├── YourComponentSuite.groovy    # Test suite (entry point)
│           ├── EntityTests.groovy            # Entity CRUD tests
│           ├── ServiceTests.groovy           # Service logic tests
│           ├── ScreenRenderTests.groovy      # Screen rendering tests
│           └── RestApiTests.groovy           # REST API tests
├── build.gradle                              # Test configuration
└── component.xml
```

---

## build.gradle for Testing

Every component with tests needs a `build.gradle`:

```groovy
apply plugin: 'groovy'

def moquiDir = file(projectDir.absolutePath + '/../../..')
def frameworkDir = file(moquiDir.absolutePath + '/framework')
def componentNode = parseComponent(project)
version = componentNode.'@version'

repositories {
    flatDir name: 'localLib', dirs: frameworkDir.absolutePath + '/lib'
    mavenCentral()
}

tasks.withType(JavaCompile) { options.compilerArgs << "-proc:none" }
tasks.withType(GroovyCompile) { options.compilerArgs << "-proc:none" }

dependencies {
    implementation project(':framework')
    testImplementation project(':framework').configurations.testImplementation.allDependencies
}

// Don't run tests on build, only when explicitly requested
check.dependsOn.clear()

test {
    useJUnitPlatform()
    testLogging { events "passed", "skipped", "failed" }
    testLogging.showStandardStreams = true
    testLogging.showExceptions = true
    maxParallelForks 1

    dependsOn cleanTest
    include '**/YourComponentSuite.class'

    systemProperty 'moqui.runtime', moquiDir.absolutePath + '/runtime'
    systemProperty 'moqui.conf', 'conf/MoquiDevConf.xml'
    systemProperty 'moqui.init.static', 'true'

    classpath += files(sourceSets.main.output.classesDirs)
    classpath = classpath.filter { it.exists() }

    beforeTest { descriptor -> logger.lifecycle("Running test: ${descriptor}") }
}
```

---

## Test Suite

Group all test classes into a JUnit Platform Suite. Moqui has **two-level cleanup** — individual test classes call `ec.destroy()` in `cleanupSpec()` to release the per-thread ExecutionContext, while the Suite's `@AfterAll` destroys the factory itself via `Moqui.destroyActiveExecutionContextFactory()`:

```groovy
import org.junit.jupiter.api.AfterAll
import org.junit.platform.suite.api.SelectClasses
import org.junit.platform.suite.api.Suite
import org.moqui.Moqui

@Suite
@SelectClasses([ EntityTests.class, ServiceTests.class,
        ScreenRenderTests.class, RestApiTests.class ])
class YourComponentSuite {
    @AfterAll
    static void destroyMoqui() {
        Moqui.destroyActiveExecutionContextFactory()
    }
}
```

---

## Test Patterns

### 1. Entity CRUD Tests

Test entity create, read, update, delete operations:

```groovy
import spock.lang.*
import org.moqui.context.ExecutionContext
import org.moqui.entity.EntityValue
import org.moqui.Moqui

class EntityTests extends Specification {
    @Shared ExecutionContext ec

    def setupSpec() {
        ec = Moqui.getExecutionContext()
    }

    def cleanupSpec() {
        ec.destroy()
    }

    def setup() {
        ec.artifactExecution.disableAuthz()
        ec.transaction.begin(null)
    }

    def cleanup() {
        ec.artifactExecution.enableAuthz()
        ec.transaction.commit()
    }

    def "create and find MyEntity"() {
        when:
        ec.entity.makeValue("yourpkg.MyEntity")
                .setAll([myEntityId: "TEST1", description: "Test Record"])
                .create()
        EntityValue found = ec.entity.find("yourpkg.MyEntity")
                .condition("myEntityId", "TEST1").one()

        then:
        found != null
        found.description == "Test Record"
    }

    def "update MyEntity"() {
        when:
        EntityValue myEntity = ec.entity.find("yourpkg.MyEntity")
                .condition("myEntityId", "TEST1").one()
        myEntity.description = "Updated Record"
        myEntity.update()
        EntityValue updated = ec.entity.find("yourpkg.MyEntity")
                .condition("myEntityId", "TEST1").one()

        then:
        updated.description == "Updated Record"
    }

    def "delete MyEntity"() {
        when:
        ec.entity.find("yourpkg.MyEntity")
                .condition("myEntityId", "TEST1").one().delete()
        EntityValue deleted = ec.entity.find("yourpkg.MyEntity")
                .condition("myEntityId", "TEST1").one()

        then:
        deleted == null
    }
}
```

### 2. Service Tests

Test service calls including business logic services:

```groovy
import spock.lang.*
import org.moqui.context.ExecutionContext
import org.moqui.Moqui

class ServiceTests extends Specification {
    @Shared ExecutionContext ec
    @Shared String testEntityId

    def setupSpec() {
        ec = Moqui.getExecutionContext()
        // Set sequenced IDs to avoid collisions with demo data
        ec.entity.tempSetSequencedIdPrimary("yourpkg.MyEntity", 90000, 10)

        // IMPORTANT: Clean up leftover data from previous failed runs (idempotent setup)
        ec.artifactExecution.disableAuthz()
        ["90000", "90001", "90002", "90003", "90004"].each { id ->
            ec.entity.find("yourpkg.MyEntity").condition("myEntityId", id).one()?.delete()
        }
        ec.artifactExecution.enableAuthz()
    }

    def cleanupSpec() {
        // Robust cleanup — delete by known ID range, not by test step
        ec.artifactExecution.disableAuthz()
        ["90000", "90001", "90002", "90003", "90004"].each { id ->
            ec.entity.find("yourpkg.MyEntity").condition("myEntityId", id).one()?.delete()
        }
        ec.artifactExecution.enableAuthz()

        ec.entity.tempResetSequencedIdPrimary("yourpkg.MyEntity")
        ec.destroy()
    }

    def setup() {
        ec.artifactExecution.disableAuthz()
    }

    def cleanup() {
        ec.artifactExecution.enableAuthz()
    }

    def "create MyEntity via service"() {
        when:
        Map result = ec.service.sync().name("yourpkg.MyServices.create#MyEntity")
                .parameters([description: "Service Test", statusId: "MyActive"])
                .call()
        testEntityId = result.myEntityId  // Assign in when: block, NOT in then: block

        then:
        testEntityId != null
        !ec.message.hasError()
        def entity = ec.entity.find("yourpkg.MyEntity")
                .condition("myEntityId", testEntityId).one()
        entity.description == "Service Test"
        entity.statusId == "MyActive"
    }

    def "verify data after service create"() {
        when:
        List<String> dataCheckErrors = []
        long fieldsChecked = ec.entity.makeDataLoader().xmlText("""<entity-facade-xml>
            <yourpkg.MyEntity myEntityId="${testEntityId}" description="Service Test" statusId="MyActive"/>
        </entity-facade-xml>""").check(dataCheckErrors)

        then:
        dataCheckErrors.size() == 0
        if (ec.message.hasError()) logger.warn(ec.message.getErrorsString())
    }
}
```

### 3. Business Flow Tests (Mantle Pattern)

Test end-to-end flows like Order-to-Cash. Use `@Shared` fields to pass data between ordered test methods:

```groovy
import spock.lang.*
import org.moqui.context.ExecutionContext
import org.moqui.Moqui
import org.slf4j.Logger
import org.slf4j.LoggerFactory

class OrderFlowTests extends Specification {
    @Shared protected final static Logger logger = LoggerFactory.getLogger(OrderFlowTests.class)
    @Shared ExecutionContext ec
    @Shared String orderId, orderPartSeqId

    def setupSpec() {
        ec = Moqui.getExecutionContext()
        ec.user.loginUser("joe@public.com", "moqui")

        // Set sequenced IDs to avoid collisions
        ec.entity.tempSetSequencedIdPrimary("mantle.order.OrderHeader", 90000, 10)
        ec.entity.tempSetSequencedIdPrimary("mantle.shipment.Shipment", 90000, 10)
        ec.entity.tempSetSequencedIdPrimary("mantle.account.invoice.Invoice", 90000, 10)
        ec.entity.tempSetSequencedIdPrimary("mantle.account.payment.Payment", 90000, 10)
    }

    def cleanupSpec() {
        ec.entity.tempResetSequencedIdPrimary("mantle.order.OrderHeader")
        ec.entity.tempResetSequencedIdPrimary("mantle.shipment.Shipment")
        ec.entity.tempResetSequencedIdPrimary("mantle.account.invoice.Invoice")
        ec.entity.tempResetSequencedIdPrimary("mantle.account.payment.Payment")
        ec.destroy()
    }

    def setup() {
        ec.artifactExecution.disableAuthz()
    }

    def cleanup() {
        ec.artifactExecution.enableAuthz()
    }

    def "create Sales Order"() {
        when:
        Map createOut = ec.service.sync().name("mantle.order.OrderServices.create#Order")
                .parameters([vendorPartyId: "ORG_ZIZI_RETAIL", customerPartyId: "CustJqp"])
                .call()
        // IMPORTANT: Assign @Shared fields in the when: block, NEVER in then: block
        orderId = createOut.orderId
        orderPartSeqId = createOut.orderPartSeqId

        ec.service.sync().name("mantle.order.OrderServices.create#OrderItem")
                .parameters([orderId: orderId, orderPartSeqId: orderPartSeqId,
                    productId: "DEMO_1_1", quantity: 2, unitAmount: 16.99])
                .call()

        then:
        orderId != null
        orderPartSeqId != null
    }

    def "place Order"() {
        when:
        ec.service.sync().name("mantle.order.OrderServices.place#Order")
                .parameters([orderId: orderId]).call()
        def orderHeader = ec.entity.find("mantle.order.OrderHeader")
                .condition("orderId", orderId).one()

        then:
        orderHeader.statusId == "OrderPlaced"
    }
}
```

### 4. Screen Render Tests

Test that screens render without errors:

```groovy
import spock.lang.*
import org.moqui.context.ExecutionContext
import org.moqui.screen.ScreenTest
import org.moqui.screen.ScreenTest.ScreenTestRender
import org.moqui.Moqui
import org.slf4j.Logger
import org.slf4j.LoggerFactory

class ScreenRenderTests extends Specification {
    protected final static Logger logger = LoggerFactory.getLogger(ScreenRenderTests.class)

    @Shared ExecutionContext ec
    @Shared ScreenTest screenTest

    def setupSpec() {
        ec = Moqui.getExecutionContext()
        ec.user.loginUser("john.doe", "moqui")
        screenTest = ec.screen.makeTest().baseScreenPath("apps/marble")
    }

    def cleanupSpec() {
        long totalTime = System.currentTimeMillis() - screenTest.startTime
        logger.info("Rendered ${screenTest.renderCount} screens " +
                "(${screenTest.errorCount} errors) in ${ec.l10n.format(totalTime/1000, '0.000')}s")
        ec.destroy()
    }

    def setup() {
        ec.artifactExecution.disableAuthz()
    }

    def cleanup() {
        ec.artifactExecution.enableAuthz()
    }

    @Unroll
    def "render screen #screenPath (#containsText)"() {
        setup:
        ScreenTestRender str = screenTest.render(screenPath, null, null)

        expect:
        !str.errorMessages
        containsText ? str.assertContains(containsText) : true

        where:
        screenPath                          | containsText
        "Customer/FindOrder"                | ""
        "Customer/EditOrder?orderId=55500"  | "55500"
        "Configure/FindProduct"             | ""
    }
}
```

### 5. REST API Tests

Test REST endpoints using `ScreenTest` with the `rest` base path.

**IMPORTANT**: `ScreenTest.render()` only reliably supports **GET** requests for REST endpoints. POST returns empty output, and PATCH/DELETE require CSRF tokens that ScreenTest cannot provide. For write operations, create test data via direct entity operations or service calls in `setupSpec()`, then test only GET endpoints via ScreenTest:

```groovy
import spock.lang.*
import org.moqui.context.ExecutionContext
import org.moqui.screen.ScreenTest
import org.moqui.screen.ScreenTest.ScreenTestRender
import org.moqui.Moqui
import groovy.json.JsonSlurper
import org.slf4j.Logger
import org.slf4j.LoggerFactory

class RestApiTests extends Specification {
    protected final static Logger logger = LoggerFactory.getLogger(RestApiTests.class)

    @Shared ExecutionContext ec
    @Shared ScreenTest screenTest
    @Shared String testEntityId = "TEST_REST_01"  // Use explicit IDs for reliability

    def setupSpec() {
        ec = Moqui.getExecutionContext()
        ec.user.loginUser("john.doe", "moqui")
        screenTest = ec.screen.makeTest().baseScreenPath("rest")

        // Create test data via direct entity operations (NOT via REST POST)
        ec.artifactExecution.disableAuthz()
        ec.entity.makeValue("yourpkg.MyEntity").setAll([
                myEntityId: testEntityId, description: "REST Test", statusId: "MyActive"
        ]).createOrUpdate()
        ec.artifactExecution.enableAuthz()
    }

    def cleanupSpec() {
        ec.artifactExecution.disableAuthz()
        ec.entity.find("yourpkg.MyEntity").condition("myEntityId", testEntityId).one()?.delete()
        ec.artifactExecution.enableAuthz()
        ec.destroy()
    }

    def setup() {
        ec.artifactExecution.disableAuthz()
    }

    def cleanup() {
        ec.artifactExecution.enableAuthz()
    }

    def "GET entity list"() {
        when:
        ScreenTestRender str = screenTest.render("s1/yourpkg/entities", null, null)

        then:
        !str.errorMessages
        str.output.contains("entityList")
    }

    def "GET entity by ID returns correct data"() {
        when:
        ScreenTestRender str = screenTest.render("s1/yourpkg/entities/${testEntityId}", null, null)
        def json = new JsonSlurper().parseText(str.output)

        then:
        !str.errorMessages
        json.myEntityId == testEntityId
        json.description == "REST Test"
    }
}
```

---

## Key APIs for Testing

### ExecutionContext (ec)

```groovy
// Entity operations
ec.entity.find("EntityName").condition("fieldName", value).one()
ec.entity.find("EntityName").condition("fieldName", value).list()
ec.entity.makeValue("EntityName").setAll(map).create()

// Service calls
Map result = ec.service.sync().name("package.Services.verb#Noun")
        .parameters([key: value]).call()

// User management
ec.user.loginUser("username", "password")
ec.user.setEffectiveTime(new Timestamp(millis))

// Authorization (disable for tests)
ec.artifactExecution.disableAuthz()
ec.artifactExecution.enableAuthz()

// Transactions
ec.transaction.begin(null)
ec.transaction.commit()
ec.transaction.rollback(null, null, null)

// Sequenced IDs (avoid collisions with demo data)
ec.entity.tempSetSequencedIdPrimary("entity.Name", startValue, bankSize)
ec.entity.tempResetSequencedIdPrimary("entity.Name")

// Messages/errors (diagnostic logging — mantle-usl standard)
ec.message.hasError()                    // Boolean: are there errors?
ec.message.getErrorsString()             // Formatted error string for logging
// if (ec.message.hasError()) logger.warn(ec.message.getErrorsString())
ec.message.getErrors()                   // List<String> of error messages
ec.message.clearErrors()                 // Use sparingly — only after testing error conditions

// Screen testing
ScreenTest screenTest = ec.screen.makeTest().baseScreenPath("apps/marble")
ScreenTestRender str = screenTest.render("path/to/screen", paramsMap, null)
str.errorMessages   // list of errors
str.output          // rendered output string
str.assertContains("text")

// Wait for async operations
ec.factory.waitWorkerPoolEmpty(50)  // 50 ticks = ~5 seconds
```

---

## Spock Features Used in Moqui

### Blocks

| Block | Purpose |
|-------|---------|
| `setup:` / `given:` | Prepare test state |
| `when:` | Execute the action under test |
| `then:` | Assert outcomes (implicit assertions) |
| `expect:` | Combined when + then (for simple cases) |
| `where:` | Data-driven parameterization |
| `cleanup:` | Per-test teardown |

### Annotations

| Annotation | Purpose |
|-----------|---------|
| `@Shared` | Share field across all test methods in a Specification |
| `@Unroll` | Generate separate test per `where:` row (shows params in test name) |
| `@Stepwise` | Run tests in declaration order (implicit in Spock for `@Shared` state) |
| `@Ignore` | Skip a test |
| `@IgnoreIf` | Conditionally skip |

### Data-Driven Tests

```groovy
@Unroll
def "render screen #screenPath"() {
    expect:
    ScreenTestRender str = screenTest.render(screenPath, null, null)
    !str.errorMessages

    where:
    screenPath          | _
    "Customer/FindOrder" | _
    "Customer/Dashboard" | _
    "Configure/FindProduct" | _
}
```

---

## Running Tests

### Prerequisites

Before running any tests, the database must be loaded with seed/demo data:

```bash
# First-time setup: build and load data
./gradlew load
```

### Run Tests for a Single Component

Gradle subprojects in Moqui use colon-separated paths matching the `settings.gradle`
registration pattern `runtime:<directory>:<ComponentName>`. For components under
`runtime/component/`, the Gradle project path is `:runtime:component:ComponentName`.

```bash
# Load data then run tests for a specific component (full clean run)
./gradlew cleanAll load :runtime:component:YourComponent:test

# Run tests only (if data is already loaded)
./gradlew :runtime:component:YourComponent:test

# Example: EmployeeManagement component
./gradlew cleanAll load :runtime:component:EmployeeManagement:test
```

### Run All Component Tests

```bash
# Run all component tests (components under runtime/component/ that have build.gradle)
./gradlew compTest

# Full clean, pull, load, then run all component tests
./gradlew cleanPullCompTest
```

### Run Framework and Mantle Tests

```bash
# Framework tests
./gradlew :framework:test

# Mantle-usl tests (under runtime/mantle/, NOT runtime/component/)
./gradlew cleanAll load :runtime:mantle:mantle-usl:test

# Run ALL tests (framework + all components)
./gradlew cleanPullTest
```

### Quick Run with Saved DB (for Repeated Test Runs)

```bash
# First time: clean, build, load, then save the database state
./gradlew loadSave

# Subsequent runs: reload from saved DB snapshot and run tests (much faster)
./gradlew reloadSave :runtime:component:YourComponent:test
```

---

## Available Demo Party IDs for Foreign Keys

When writing tests or demo data that reference `partyId` (a foreign key to `mantle.party.Party`), you **must** use party IDs that already exist in the loaded seed/demo data. Using non-existent IDs like `EXP_PUBLIC` causes referential integrity constraint violations.

### Valid partyId Values (from mantle-usl ZbaOrganizationDemoData.xml)

| partyId | Type | Description |
|---------|------|-------------|
| `ORG_ZIZI_CORP` | Organization | Ziziwork Industries (main org) |
| `ORG_ZIZI_SALES` | Organization | Ziziwork Sales |
| `ORG_ZIZI_HR` | Organization | Ziziwork Human Resources |
| `ORG_ZIZI_SERVICES` | Organization | Ziziwork Services |
| `ORG_ZIZI_RETAIL` | Organization | Ziziwork Retail |
| `ORG_ZIZI_DEVA` | Organization | Ziziwork Dev Team A |
| `ORG_ZIZI_DEVB` | Organization | Ziziwork Dev Team B |
| `ORG_ZIZI_JD` | Person | Joe Developer (Ziziwork employee) |
| `ORG_ZIZI_BD` | Person | Bob Developer (Ziziwork employee) |
| `ORG_ACME` | Organization | Another Company Making Everything |
| `ORG_ACME_BA` | Person | Bob Acme |
| `ORG_ACME_SA` | Person | Sheldon Acme |
| `ORG_BLUTH` | Organization | Bluth Corp |
| `ORG_BLUTH_GOB` | Person | Gob Bluth |
| `CustJqp` | Person | Customer John Q. Public |
| `EX_JOHN_DOE` | Person | John Doe |
| `ZiddlemanInc` | Organization | Ziddleman & Sons Suppliers Inc. |
| `JoeDist` | Organization | Joe Distributors |

### Carriers (from mantle-udm CarrierData.xml)

| partyId | Description |
|---------|-------------|
| `UPS` | United Parcel Service |
| `FedEx` | FedEx Corporation |
| `DHLX` | DHL Express |
| `USPS` | United States Postal Service |

### Recommended Usage in Tests & Demo Data

- **For employee/person records**: Use `ORG_ZIZI_JD`, `ORG_ZIZI_BD`, `EX_JOHN_DOE`, `CustJqp`
- **For vendor/customer org records**: Use `ORG_ZIZI_RETAIL`, `ORG_ACME`, `ZiddlemanInc`
- **For internal org records**: Use `ORG_ZIZI_CORP`, `ORG_ZIZI_HR`
- **NEVER invent partyId values** — always reference IDs from the tables above

---

## Critical Pitfalls

These are common mistakes that cause hard-to-debug test failures. Read this section carefully before writing tests.

### 1. Always call `ec.destroy()` in `cleanupSpec()` — but NEVER call `destroyActiveExecutionContextFactory()` there

Moqui has two-level cleanup. `ec.destroy()` releases the per-thread ExecutionContext and is **safe and recommended** in every class's `cleanupSpec()` — the factory remains alive and creates a fresh EC when the next test class calls `Moqui.getExecutionContext()`. This is the standard mantle-usl pattern (used in OrderToCashBasicFlow, OrderProcureToPayBasicFlow, ReturnToResponseBasicFlow, WorkPlanToCashBasicFlow, RestApiTests, etc.).

`Moqui.destroyActiveExecutionContextFactory()` destroys the entire factory and must **only** be called in the Suite's `@AfterAll`. Calling it in a test class would break all subsequent classes.

```groovy
// CORRECT — ec.destroy() cleans up the per-thread EC (mantle-usl standard)
def cleanupSpec() {
    ec.entity.tempResetSequencedIdPrimary("yourpkg.MyEntity")
    ec.destroy()
}

// WRONG — destroys the factory, breaks subsequent test classes
def cleanupSpec() {
    ec.entity.tempResetSequencedIdPrimary("yourpkg.MyEntity")
    Moqui.destroyActiveExecutionContextFactory()  // DO NOT DO THIS
}
```

The Suite class handles factory destruction:
```groovy
class YourComponentSuite {
    @AfterAll
    static void destroyMoqui() {
        Moqui.destroyActiveExecutionContextFactory()
    }
}
```

### 2. NEVER put assignments in Spock `then:` blocks

Spock treats **every statement** in a `then:` block as a boolean assertion. Variable assignments like `testId = result.id` are interpreted as comparisons and cause a compilation error: *"Expected a condition, but found an assignment."*

```groovy
// WRONG — compilation error
def "create entity"() {
    when:
    Map result = ec.service.sync().name("pkg.Svc.create#Entity")
            .parameters([name: "Test"]).call()
    then:
    testId = result.entityId  // ERROR: assignment in then: block
    testId != null
}

// CORRECT — assign in when: block, assert in then: block
def "create entity"() {
    when:
    Map result = ec.service.sync().name("pkg.Svc.create#Entity")
            .parameters([name: "Test"]).call()
    testId = result.entityId  // Assignment in when: block

    then:
    testId != null
    !ec.message.hasError()
}
```

### 3. ALWAYS add idempotent cleanup in both `setupSpec()` and `cleanupSpec()`

If a test run fails mid-way, stale data persists in the H2 database. The next run fails with "record already exists" errors. Always clean up by known ID range in both setup and cleanup:

```groovy
def setupSpec() {
    ec = Moqui.getExecutionContext()
    ec.entity.tempSetSequencedIdPrimary("yourpkg.MyEntity", 90000, 10)

    // Clean leftover data from previous failed runs
    ec.artifactExecution.disableAuthz()
    ["90000", "90001", "90002", "90003", "90004"].each { id ->
        ec.entity.find("yourpkg.MyEntity").condition("myEntityId", id).one()?.delete()
    }
    ec.artifactExecution.enableAuthz()
}

def cleanupSpec() {
    // Same cleanup — ensures data is removed even if tests fail
    ec.artifactExecution.disableAuthz()
    ["90000", "90001", "90002", "90003", "90004"].each { id ->
        ec.entity.find("yourpkg.MyEntity").condition("myEntityId", id).one()?.delete()
    }
    ec.artifactExecution.enableAuthz()
    ec.entity.tempResetSequencedIdPrimary("yourpkg.MyEntity")
    ec.destroy()
}
```

### 4. REST ScreenTest only supports GET — use direct entity ops for test data

`ScreenTest.render()` only reliably supports **GET** requests for REST endpoints:
- **POST** returns empty output (no response body)
- **PATCH/DELETE** require CSRF tokens that ScreenTest cannot provide

Create test data via direct entity operations with explicit IDs in `setupSpec()`, then test only GET endpoints:

```groovy
// WRONG — REST POST via ScreenTest returns empty output
Map params = [name: "Test"]
ScreenTestRender str = screenTest.render("s1/pkg/entities", params, null)  // empty response

// CORRECT — create data directly, test only GET
def setupSpec() {
    ec.entity.makeValue("yourpkg.MyEntity").setAll([
            myEntityId: "TEST_REST_01", name: "Test"
    ]).createOrUpdate()
}

def "GET entity by ID"() {
    when:
    ScreenTestRender str = screenTest.render("s1/pkg/entities/TEST_REST_01", null, null)
    then:
    !str.errorMessages
}
```

### 5. Use `hasError()` + `getErrorsString()` for diagnostic logging — avoid blanket `clearErrors()`

Mantle-usl tests **never** use `clearErrors()`. Instead, they use `ec.message.hasError()` paired with `ec.message.getErrorsString()` for **diagnostic logging** after data validation checks. This pattern logs warnings when unexpected errors occur, rather than silently swallowing them:

```groovy
// CORRECT — mantle-usl standard: log errors as diagnostics after data validation
List<String> dataCheckErrors = []
long fieldsChecked = ec.entity.makeDataLoader().xmlText("""<entity-facade-xml>
    <yourpkg.MyEntity myEntityId="${testId}" description="Expected Value"/>
</entity-facade-xml>""").check(dataCheckErrors)
if (dataCheckErrors) for (String dataCheckError in dataCheckErrors) logger.info(dataCheckError)
if (ec.message.hasError()) logger.warn(ec.message.getErrorsString())

then:
dataCheckErrors.size() == 0
```

**Why not `clearErrors()`?** If your tests regularly need `clearErrors()`, it usually means errors are leaking from broken test logic. Fix the root cause instead. Mantle-usl tests call services that should succeed, then validate the resulting data — they don't test for expected service failures.

If you **must** test error conditions (e.g., duplicate rejection), call `clearErrors()` immediately after the assertion, not as blanket setup/cleanup:

```groovy
// ACCEPTABLE — testing a specific error condition, clear immediately after
when:
ec.service.sync().name("pkg.Svc.create#Entity")
        .parameters([:]).call()
boolean hadError = ec.message.hasError()
ec.message.clearErrors()  // Clear immediately, don't leave for cleanup

then:
hadError
```

### 6. Use `createOrUpdate()` or explicit IDs for test data reliability

Service calls in `setupSpec()` can silently fail (returning null) if there are leftover errors or transaction issues from previous test classes. Prefer direct entity creation with explicit IDs:

```groovy
// FRAGILE — service call may return null, causing NPE on .entityId
Map result = ec.service.sync().name("pkg.Svc.create#Entity")
        .parameters([name: "Test"]).call()
testId = result.entityId  // NPE if result is null

// ROBUST — direct entity creation with explicit IDs
@Shared String testId = "TEST_MY_01"

def setupSpec() {
    ec.artifactExecution.disableAuthz()
    ec.entity.makeValue("yourpkg.MyEntity").setAll([
            myEntityId: testId, name: "Test", statusId: "MyActive"
    ]).createOrUpdate()  // createOrUpdate handles reruns gracefully
    ec.artifactExecution.enableAuthz()
}
```

---

## Best Practices

1. **Always use `@Shared ExecutionContext ec`** — Initialize in `setupSpec()` via `Moqui.getExecutionContext()`, call `ec.destroy()` in `cleanupSpec()`
2. **NEVER call `Moqui.destroyActiveExecutionContextFactory()` in individual test classes** — Only the Suite's `@AfterAll` should destroy the factory; `ec.destroy()` in `cleanupSpec()` is safe and recommended
3. **Disable authz in `setup()`** — Re-enable in `cleanup()` to avoid permission errors in tests
4. **Use `tempSetSequencedIdPrimary()`** — Prevents ID collisions with seed/demo data; always reset in `cleanupSpec()`
5. **Idempotent cleanup** — Clean up by known ID range in both `setupSpec()` and `cleanupSpec()` to handle stale data from failed runs
6. **Transaction wrapping** — For entity CRUD tests, begin in `setup()` and commit/rollback in `cleanup()`
7. **Login a user** — Many services require an authenticated user; call `ec.user.loginUser()` in `setupSpec()`
8. **Test data independence** — Use unique ID ranges (90000+) that don't overlap with demo data (55000-55999 used by mantle-usl)
9. **Flow tests use `@Shared` state** — Pass IDs between sequential test methods via `@Shared` fields; assign in `when:` blocks only
10. **Screen tests use `@Unroll` with `where:`** — Efficiently test many screens with parameterized data
11. **Check for errors diagnostically** — Assert `!str.errorMessages` for screen tests; for service tests use `if (ec.message.hasError()) logger.warn(ec.message.getErrorsString())` for diagnostic logging, and assert `dataCheckErrors.size() == 0` for data validation
12. **Avoid blanket `clearErrors()`** — Mantle-usl never uses `clearErrors()` in tests. If you must test error conditions, call `clearErrors()` immediately after the assertion in the `when:` block, never as routine setup/cleanup
13. **REST tests are GET-only** — Use `ScreenTest.render()` only for GET endpoints; create test data via direct entity ops
14. **Wait for async** — If services trigger async work, call `ec.factory.waitWorkerPoolEmpty(50)` before assertions

## Verify

After writing or running a Spock test:
1. `cd moqui-framework && ./gradlew :runtime:component:landmark-hsto-core:test` — exit code 0, and the test count in the summary matches what you wrote.
2. For a failing test, the failure output should include the assertion, the actual value, and the data state. If it doesn't, improve diagnostics with `then:` block prints.
3. Run the test in isolation (`--tests "ClassName.featureName"`) to confirm it doesn't depend on test-order side effects.
4. For data-validation assertions (`dataCheckErrors.size() == 0`): print the errors on failure for diagnostics.

Common failures: order-dependent tests (use proper fixtures); blanket `clearErrors()` masking real failures; async services not awaited (`waitWorkerPoolEmpty`).

See [`../SKILL.md`](../SKILL.md) §"Verify After Each Pattern" for the cross-cutting table.

<!-- begin: related-lessons (auto-generated by scripts/regen-skill-lesson-links.sh — do not edit by hand) -->

## Related Lessons

Cross-references from [`docs/ai/lessons.md`](../../docs/ai/lessons.md) that apply to this area. Filter tags: `Process,Moqui-Services`. Regenerate with `scripts/regen-skill-lesson-links.sh`.

- 2026-04-15 — HSTO backend directory name differs from component name  `[Process,Moqui-Entities]`

<!-- end: related-lessons -->
