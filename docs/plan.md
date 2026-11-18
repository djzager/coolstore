# Migration Plan

## Goal
Produce a focused, planning-only Java EE → Quarkus plan for Coolstore's PostgreSQL/persistence portion; this stage changes no application source, dependency manifest, or build configuration.

## Source → Target
Java EE 7 on JBoss EAP 7.4 (javax, WAR) → Quarkus (version open) using Agroal, Hibernate ORM, and Flyway against the existing PostgreSQL database.

## Scope
- Files affected: 0 now; future execution targets pom.xml, persistence.xml, Resources.java, DataBaseMigrationStartup.java, plus a new application.properties (5 files).
- Estimated complexity: Medium; hardest areas: EJB @Startup Flyway bootstrap, JNDI datasource → Agroal config, transaction-boundary semantics.

## Source Facts (from the five allowed reads)
- pom.xml: WAR packaging, provided javaee-api 7.0, flyway-core 4.1.2, Java 1.8, maven.test.skip=true, system-scoped lib/audit-logging-library-1.0.0.jar.
- src/main/resources/META-INF/persistence.xml: PU "primary", JTA datasource java:jboss/datasources/CoolstoreDS, schema-generation none, hibernate show/format/comments, use_get_generated_keys=false.
- persistence/Resources.java: @Dependent CDI bean producing the @PersistenceContext EntityManager.
- utils/DataBaseMigrationStartup.java: @Singleton @Startup EJB with BEAN-managed transactions, @Resource(mappedName="java:jboss/datasources/CoolstoreDS") DataSource, Flyway baseline()+migrate() in @PostConstruct; FlywayException is only logged — startup continues on failure.
- README.md: one PostgreSQL container (postgresUser/postgresPW, DB postgresDB, port 5432); Keycloak 20.0.5 is also required to run the app.

## Key Decisions Applied
- Human decision: reuse the existing PostgreSQL instance/database (postgresDB) rather than provisioning a separate database for migration testing.
- No human acceptance of the shared-database schema-drift/dirty-data risk has been given; that risk remains open and unmitigated in this plan.

## Assumptions and Open Questions
- Flyway scripts assumed at the Flyway 4.x default src/main/resources/db/migration; unverified in this focused pass — confirm before execution.
- Quarkus version and exact extension set (quarkus-jdbc-postgresql, quarkus-hibernate-orm, quarkus-flyway vs. plain flyway-core) are open.
- javax → jakarta namespace shift and its ripple into services are deferred to the broader migration.
- Whether Quarkus runs Flyway at startup (replacing the EJB) or only validates is open; broader REST/JMS/Keycloak/web-tier/audit-jar details remain open questions.

## Risks (database reuse)
- Schema ownership: Flyway owns the shared schema (schema-generation=none); Quarkus Flyway baseline/migrate or Hibernate validate runs against the reused DB can alter or reject the live Java EE app's schema — keep Hibernate generation off and decide validate vs. migrate-at-start first.
- Transaction boundaries: the EJB migrator commits BEAN-managed transactions directly to the reused DB while services rely on container-managed JTA; Quarkus/Narayana @Transactional semantics must be audited per service so partial migrations don't leave the shared DB inconsistent.
- Migration testing: no test safety net (maven.test.skip=true), Flyway failures are only logged, and there is no isolated database — Quarkus migration testing would mutate the shared schema/data with no rollback plan defined.

## Approach
Future phases, none executed now: (1) Build config — pom.xml to Quarkus with persistence extensions; (2) Persistence config — persistence.xml superseded by application.properties (Agroal URL to the reused DB, Hibernate ORM generation=none, Flyway settings); (3) Resources — CDI EntityManager producer; (4) Startup — EJB migrator becomes a Quarkus startup observer running Flyway.

## Steps
### Step 1: Migrate pom.xml to Quarkus (future)
- Phase: Build config | File: pom.xml | Action: MODIFY
- What to do: Replace the WAR/Java EE 7 build with Quarkus, adding quarkus-jdbc-postgresql, quarkus-hibernate-orm, and a Flyway extension (exact set/version open).
- Why: Quarkus does not build or deploy as a JBoss EAP WAR.
- Depends on: none | Verify: pending implementation
### Step 2: Create Quarkus persistence configuration (future)
- Phase: Persistence config | File: src/main/resources/application.properties | Action: CREATE
- What to do: Agroal datasource for the reused PostgreSQL DB, Hibernate ORM (PU "primary", generation=none), Flyway baseline/migrate vs. validate (open).
- Why: Quarkus configures persistence via application.properties instead of a JNDI-referenced persistence.xml.
- Depends on: Step 1 | Verify: pending implementation
### Step 3: Migrate Resources.java (future)
- Phase: Persistence resources | File: src/main/java/com/redhat/coolstore/persistence/Resources.java | Action: MODIFY
- What to do: Replace the javax @PersistenceContext/@Produces producer with Quarkus-compatible (jakarta) EntityManager exposure.
- Why: Quarkus CDI injects the EntityManager directly; the javax producer pattern does not carry over.
- Depends on: Step 2 | Verify: pending implementation
### Step 4: COMPLEX — Convert DataBaseMigrationStartup.java (future)
- Phase: Persistence startup | File: src/main/java/com/redhat/coolstore/utils/DataBaseMigrationStartup.java | Action: MODIFY
- What to do: Replace @Singleton @Startup EJB + @Resource JNDI lookup with a Quarkus startup observer (@Observes StartupEvent) running Flyway via the Agroal DataSource; define failure behavior (today only logged).
- Why: EJB lifecycle and JNDI mappedName lookups do not exist in Quarkus.
- Depends on: Step 2 | Verify: pending implementation
### Step 5: Remove persistence.xml (future)
- Phase: Persistence config | File: src/main/resources/META-INF/persistence.xml | Action: DELETE
- What to do: Delete once application.properties fully supersedes PU "primary" configuration.
- Why: Quarkus reads persistence configuration from application.properties.
- Depends on: Steps 2-4 | Verify: pending implementation

## Verification
Plan exists: test -f docs/plan.md
Human decision recorded: grep -q 'Human decision:' docs/plan.md
Tracked application/build baseline clean: git diff --quiet 32c1722efdf883f5f400170adb770a46a9b58116 -- . ':(exclude)docs/plan.md' ':(exclude).konveyor/**'
Untracked application/build baseline clean: test -z "$(git ls-files --others --exclude-standard -- . ':(exclude)docs/plan.md' ':(exclude).konveyor/**')"

## Future migration validation
- Build (mvn clean compile / Quarkus build): pending implementation
- Tests (none today; maven.test.skip=true): pending implementation
- Runtime startup against the reused PostgreSQL with Flyway baseline/migrate: pending implementation
- Business-flow smoke check per README: pending implementation

## Notes
- Planning-only stage: only docs/plan.md is created; application, manifest, and build files stay at baseline 32c1722efdf883f5f400170adb770a46a9b58116.
