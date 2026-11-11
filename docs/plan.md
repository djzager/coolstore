# Migration Plan

## Goal
Migrate the Coolstore Java EE monolith (JBoss EAP, WAR) to Quarkus while reusing the existing CoolstoreDS PostgreSQL service, leaving application source unchanged in this planning stage.

## Source → Target
Java EE 7 on JBoss EAP (WAR, Java 8, javax) → Quarkus (Maven jar, Jakarta, quarkus-resteasy/hibernate-orm/artemis stack)

## Scope
- Files affected: ~35 (pom.xml, 30 Java sources, persistence.xml, web.xml/beans.xml, new application.properties)
- Estimated complexity: High
- Hardest areas: EJB/MDB services (OrderServiceMDB, InventoryNotificationMDB), EJB transactional services (OrderService, ShoppingCartService, ShoppingCartOrderProcessor), system-scoped audit-logging-library jar

## Source Facts (from repo and Konveyor analysis, baseline 32c1722)
- WAR packaging, Java 8, `javax` EE 7 APIs, tests skipped in build (`maven.test.skip=true`).
- Layers: `model/` (8 JPA/pojo classes) → `service/` (10 classes, EJB + 2 JMS MDBs) → `rest/` (4 JAX-RS classes); `persistence/Resources.java` (CDI producer), `utils/` (CDI producers, Flyway startup, WebLogic shim).
- Vendored `weblogic/*` shim classes (ApplicationLifecycleListener, i18n logging) used by startup/listener code.
- `com.enterprise:audit-logging-library` via `system` scope from `lib/`.
- Flyway 4.1.2 with `db/migration/V1_1__CreateSchema.sql`, `V1_2__AddInitialData.sql`, run at startup by `DataBaseMigrationStartup`.
- Konveyor analysis: 49 insights; hotspots OrderServiceMDB (14), ShoppingCartOrderProcessor (8), CartEndpoint (8), pom.xml (8), Transformers/DataBaseMigrationStartup/OrderService (7 each).

## Decisions Applied
- Database (human decision): **Reuse the existing Coolstore PostgreSQL service** shared with the current JBoss EAP deployment. Consequences: no new service provisioning; Quarkus `quarkus.datasource.*` must point at the same database; credentials come from environment/Quarkus config at deploy time (never hardcoded). Because the schema and Flyway history state of the shared database are **not established by this decision** (migrations currently run via a custom startup class, and the shared schema may predate or diverge from `db/migration/`), the execution stage MUST inspect the existing schema and Flyway history table first and obtain database-owner approval before any baseline is applied; never clean or automatically reapply migrations against the shared database. Risk of schema/data contention if EAP and Quarkus run concurrently during cutover — plan a stop-EAP-then-start-Quarkus cutover.
- Replace EJB `@Stateless`/`@Singleton` + JTA with CDI `@ApplicationScoped` + Quarkus Narayana JTA (`@Transactional` boundaries preserved per method).
- Messaging is an **open validation task, not a drop-in replacement**: the exact Quarkus messaging extension, its version, and its delivery/acknowledgement/transaction semantics for the two MDB consumers must be validated during execution (dependency coordinates, acknowledgement modes, redelivery behavior, and JTA interaction) before committing to an approach. Do not assume a specific connector is equivalent.
- Replace WebLogic shim usage (`StartupListener`, `DataBaseMigrationStartup`, `weblogic.*` classes) with Quarkus `@Observes StartupEvent` and the Quarkus-managed Flyway extension — with migrations disabled at startup until the schema/history review below is complete.
- Vendored frontend code under webapp/static assets is out of scope (served as static resources in `META-INF/resources` mechanically).

## Assumptions
- Quarkus 3.x with `quarkus-resteasy` (or rest) + `hibernate-orm`; plain JPA (no Panache) is acceptable. The exact Quarkus version and its `quarkus.hibernate-orm.schema-management` (or equivalent) property name/semantics must be validated against the chosen version's documentation during execution.
- The audit-logging-library system jar is binary-compatible on Java 17/21; if not, it must be rehosted into the local repo or a `install-file` step (open question below).
- Existing queues/topics and JMS broker remain available to Quarkus via Artemis JMS.

## Approach
Phases: (1) build config, (2) config/resources, (3) models, (4) persistence/utils, (5) services, (6) REST API, (7) cleanup/deletes. Each step is one file; MDBs and EJB transactional services are COMPLEX.

## Steps

### Step 1: Convert pom.xml to Quarkus build
- Phase: Build config
- File: pom.xml
- Action: MODIFY
- What to do: Change packaging war→jar; replace javaee-api/jboss specs with `quarkus-resteasy`, `quarkus-hibernate-orm`, `quarkus-jdbc-postgresql`, `quarkus-flyway`, `quarkus-arc`, and a messaging extension **to be selected after the validation task in Decisions**; validate the exact quarkus-maven-plugin goals (build/package) and all dependency coordinates against the chosen Quarkus version's documentation rather than assuming them; remove maven-war-plugin; bump compiler to 17; install `lib/audit-logging-library-1.0.0.jar` into the local repo and use a normal dependency (remove system scope).
- Why: 8 Konveyor findings on pom.xml (Java EE APIs, war packaging, system-scope dep).
- Depends on: none
- Verify: `mvn clean compile` succeeds (future, see Future migration validation).

### Step 2: Create Quarkus application.properties
- Phase: Build config
- File: src/main/resources/application.properties
- Action: CREATE
- What to do: Define default datasource to the existing PostgreSQL service (host/db from env vars, credentials via env, never hardcoded), Hibernate schema management set to none (validate the exact property name for the chosen Quarkus version), and Flyway with **`migrate-at-start=false`** — migrations stay disabled until the shared-schema/history review is complete and the database owner approves; no `baseline-on-migrate`, `clean`, or automatic reapplication is configured at this point. Broker/messaging connection properties are added only after the messaging validation task resolves the extension and its semantics.
- Why: Quarkus uses application.properties instead of JBoss datasource/standalone config.
- Depends on: Step 1
- Verify: File exists with datasource, flyway, and messaging properties.

### Step 3: Migrate persistence.xml
- Phase: Persistence configuration
- File: src/main/resources/META-INF/persistence.xml
- Action: MODIFY
- What to do: Point persistence unit at Quarkus default datasource; remove JBoss-specific properties (`jta-data-source` CoolstoreDS, hibernate.dialect for EAP).
- Why: 6 Konveyor findings; datasource is now Quarkus-managed.
- Depends on: Step 2
- Verify: No EAP-specific properties remain.

### Step 4: Remove web.xml, relocate beans.xml
- Phase: Build config
- File: src/main/webapp/WEB-INF/web.xml
- Action: DELETE
- What to do: Delete web.xml; move beans.xml to `src/main/resources/META-INF/beans.xml` (empty/discovery mode all); move static webapp assets to `src/main/resources/META-INF/resources/`.
- Why: Quarkus has no deployment descriptors.
- Depends on: Step 1
- Verify: `src/main/webapp/` no longer exists.

### Step 5: Migrate JPA model imports
- Phase: Models
- File: src/main/java/com/redhat/coolstore/model/ (8 files, one sub-step each)
- Action: MODIFY
- What to do: Replace `javax.persistence.*` with `jakarta.persistence.*`; keep entity mappings as-is.
- Why: Quarkus 3 uses Jakarta namespace.
- Depends on: Step 1
- Verify: No `javax.persistence` imports remain under model/.

### Step 6: COMPLEX — Replace Resources CDI producer
- Phase: Persistence
- File: src/main/java/com/redhat/coolstore/persistence/Resources.java
- Action: MODIFY
- What to do: BEFORE: produces `EntityManager` from `@PersistenceContext` for EAP. AFTER: `@ApplicationScoped` class exposing `@Produces` EntityManager from injected `EntityManagerFactory`, or inject Quarkus-managed EM directly at use sites; drop producer if all consumers can inject.
- Why: 5 findings; Quarkus manages persistence via Arc.
- Depends on: Steps 1, 3, 5
- Verify: Compiles; no `javax.persistence` imports.

### Step 7: COMPLEX — Rework utils (Producers, Transformers, StartupListener, DataBaseMigrationStartup)
- Phase: Services
- File: src/main/java/com/redhat/coolstore/utils/ (4 files, one sub-step each)
- Action: MODIFY / DELETE
- What to do: Convert Producers/Transformers imports to jakarta; replace `StartupListener` (WebLogic shim, `weblogic.application.*`) with `@Observes StartupEvent`; DELETE `DataBaseMigrationStartup` only after the Quarkus Flyway path is enabled post-review (until then keep migrations at `migrate-at-start=false`); DELETE the vendored `src/main/java/weblogic/**` classes once unreferenced.
- Why: ~13 findings across these files; startup lifecycle and WebLogic shims unsupported.
- Depends on: Steps 1, 2
- Verify: No `weblogic.` imports remain in coolstore sources.

### Step 8: COMPLEX — Convert EJB services to CDI/JTA
- Phase: Services
- File: src/main/java/com/redhat/coolstore/service/ (CatalogService, InventoryNotificationMDB, OrderService, OrderServiceMDB, ProductService, PromoService, ShippingService, ShippingServiceRemote, ShoppingCartOrderProcessor, ShoppingCartService — one sub-step each)
- Action: MODIFY
- What to do: BEFORE: `@Stateless`/`@Singleton`/`@MessageDriven` EJBs with container-managed JTA and JMS `MessageListener`. AFTER: `@ApplicationScoped` CDI beans; add `@Transactional` on each method that previously relied on EJB CMT to preserve transaction boundaries (notably OrderService.persist/checkout and ShoppingCartOrderProcessor); MDB consumers are redesigned per the messaging validation task (extension, acknowledgement, redelivery, and JTA semantics must be validated and covered by tests before this step completes); `@Asynchronous` EJB calls are **not** a mechanical `@Blocking`/plain-method swap — semantics differ (fire-and-forget, exception isolation, thread/context propagation), so an explicit async design is required and must be covered by transaction and redelivery tests; ShippingServiceRemote `@Remote` EJB interface → local CDI interface (no remote EJB in Quarkus; see open question on external clients).
- Why: 14+ findings on OrderServiceMDB; ~30 across service layer; EJB/MDB model unsupported in Quarkus.
- Depends on: Steps 5, 6, 7
- Verify: No `javax.ejb`, `javax.jms` MessageListener imports remain; each former-CMT method annotated `@Transactional`.

### Step 9: Migrate REST endpoints
- Phase: API
- File: src/main/java/com/redhat/coolstore/rest/ (CartEndpoint, OrderEndpoint, ProductEndpoint, RestApplication — one sub-step each)
- Action: MODIFY
- What to do: `javax.ws.rs.*` → `jakarta.ws.rs.*`; keep `@Path/@GET/@POST` etc.; RestApplication `@ApplicationPath` can remain or map to `quarkus.resteasy.path`; inject services via constructor/field CDI.
- Why: Namespace change and CDI injection; ~20 findings.
- Depends on: Step 8
- Verify: No `javax.ws.rs` imports remain.

## Verification
Checks for this planning deliverable only. The prepared checkout contains a pre-existing .vscode/extensions.json difference that is prepared-checkout provenance, not part of this plan, which is why the whole-tree check compares against the prepared checkout b5752b936f01acd858ca21fd2d3c4f03d3a6321e rather than upstream. Each check must fail (non-zero exit) on violation.
- Plan exists: test -f docs/plan.md
- Prepared source unchanged: git diff --exit-code b5752b936f01acd858ca21fd2d3c4f03d3a6321e -- . ':(exclude)docs/plan.md' ':(exclude).konveyor/**'
- Application source unchanged: git diff --exit-code 32c1722efdf883f5f400170adb770a46a9b58116 -- pom.xml src lib rules assets realm-export.json README.md
- Plan tracked: git ls-files --error-unmatch docs/plan.md
- Analysis context: test -f .konveyor/analysis.json

## Future migration validation
Pending implementation of the steps above; not yet performed.
- Build: mvn clean compile
- Package: mvn clean package
- Test: mvn test (note: tests currently skipped via maven.test.skip=true and no test suite exists — see risks)
- Runtime: mvn quarkus:dev with the shared PostgreSQL service configured via env; verify catalog/products REST endpoints respond; verify Flyway does not re-run V1_1/V1_2 against the shared schema; place an order and confirm order + inventory JMS flows complete.
- Cutover check: stop the EAP deployment before starting Quarkus against the shared database.

## Notes
- Testing risks: no existing test suite and `maven.test.skip=true` — there is no regression safety net; recommend adding Quarkus `@QuarkusTest` REST tests during execution before touching service logic.
- Transaction boundaries: EJB CMT defaults (REQUIRED per method) must be explicitly reproduced with `@Transactional`; checkout spans order persist + inventory update + JMS notification — confirm no boundary is lost or widened.
- Shared-DB risk: concurrent EAP/Quarkus writes during cutover; Flyway history state on the shared database is unknown (migrations ran via a custom startup class) — the schema/history review and database-owner approval are prerequisites for any baseline decision; never clean or automatically reapply migrations.
- Open questions: (1) Is the audit-logging-library jar source-available/compatible with Java 17+? (2) Which JMS broker hosts the order/inventory destinations, and are credentials available for Quarkus? (3) Is remote EJB access to ShippingServiceRemote used by any external client? (4) Preferred Quarkus version/LTS.
- This plan made no changes to application source, dependencies, or build configuration.
