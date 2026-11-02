# DB2 → PostgreSQL: Application Code Migration Best Practices
 
**Stack:** Spring Boot (3.x/4.x), Java 21, Maven, JPA/Hibernate and/or JdbcTemplate
**Scope:** Application code only. DBAs own the schema, data migration, sequences, indexes, and database configuration.
**End state:** PostgreSQL only. No dual-database support is needed.
 
---
 
## 1. Guiding Principles
 
1. **Treat the DBA's Postgres schema as the source of truth.** Do not let the app create or alter schema objects. Use `spring.jpa.hibernate.ddl-auto=validate` (or `none`) everywhere.
2. **Prefer portable abstractions over native SQL.** JPQL, Spring Data derived queries, and `Pageable` are mostly database-agnostic. Native SQL is where nearly all the migration risk lives.
3. **Inventory first, change second.** Find every DB2-specific construct before you start editing.
4. **Test against real Postgres.** Use the same major version as production (via Testcontainers). Don't use H2, and don't use a "DB2 compatibility mode."
5. **Prove equivalence, not just compilation.** Compare query results between DB2 and Postgres using the DBA's migrated data.
6. **Small, reviewable commits.** Organize them by category: config, then native SQL, then types, then error handling, and so on.
---
 
## 2. Coordinate With the DBAs First
 
Get written answers to these before changing code. Each answer affects your entities and SQL.
 
| Question | Why it matters to the app |
|---|---|
| Target Postgres major version? | Determines whether `MERGE` (15+) and `MERGE ... RETURNING` (17+) are available. Pin Testcontainers to this version. |
| Schema name(s) and `search_path`? | DB2 often uses the auth ID as the default schema. Postgres uses `search_path` and `currentSchema`. |
| Identifier casing? Will they quote anything? | DB2 folds unquoted names to UPPER; Postgres folds them to lower. Quoted mixed-case names break unquoted SQL. |
| Data type mapping per column | Needed to align entity types for timestamps, booleans, CLOB/BLOB, CHAR vs VARCHAR, and DECIMAL precision. |
| How are booleans stored? (`BOOLEAN`, `SMALLINT`, `CHAR(1)` Y/N) | Determines entity field types and whether you need `AttributeConverter`s. |
| `TIMESTAMP` or `TIMESTAMPTZ`? DB server timezone? | Affects how `Instant`, `OffsetDateTime`, and `LocalDateTime` are mapped. |
| Identity columns or sequences? Sequence names and `INCREMENT BY`? | Must match `@GeneratedValue` / `@SequenceGenerator` and `allocationSize`. |
| Will sequences be reset to `MAX(id)+1` after the data load? | Missing this causes duplicate-key errors on the first insert after cutover. |
| Stored procedures/functions: rewritten in PL/pgSQL? New signatures? | Changes `CALL` syntax, OUT params, and result-set handling. |
| Collation and case sensitivity of text comparisons | Affects `ORDER BY` output and equality and `LIKE` behavior. |
| `max_connections`, pooler (PgBouncer?), and pool mode | Affects Hikari sizing and prepared-statement settings. |
| SSL requirements and credentials delivery | Affects JDBC URL `sslmode` and secrets config. |
| Who owns Flyway/Liquibase going forward (if the app uses one)? | Decides whether to disable app-run migrations or rebaseline scripts for Postgres. |
 
---
 
## 3. Dependencies & Configuration
 
### 3.1 Maven
 
```xml
<!-- REMOVE -->
<dependency>
  <groupId>com.ibm.db2</groupId>
  <artifactId>jcc</artifactId>
</dependency>
 
<!-- ADD (version managed by the Spring Boot BOM) -->
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <scope>runtime</scope>
</dependency>
```
 
If the app runs Flyway, also add `org.flywaydb:flyway-database-postgresql`; Flyway 10+ needs it. In Spring Boot 4, Flyway and Liquibase auto-configuration moved into their own modules, so use `spring-boot-starter-flyway` or `spring-boot-starter-liquibase`.
 
Also remove any DB2-only libraries, custom dialects, or license jars (`db2jcc_license_cu`, etc.).
 
### 3.2 `application.yml`
 
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}?currentSchema=${DB_SCHEMA}&reWriteBatchedInserts=true&ApplicationName=my-app
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10        # Postgres connections are processes; size deliberately
      # Remove connection-test-query (e.g. "SELECT 1 FROM SYSIBM.SYSDUMMY1"); JDBC4 isValid() is used
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
    properties:
      hibernate:
        # Do NOT set hibernate.dialect. Hibernate 6+/7 auto-detects PostgreSQL.
        jdbc:
          time_zone: UTC
          batch_size: 50
        order_inserts: true
        order_updates: true
```
 
**Remove or replace:**
- `hibernate.dialect: org.hibernate.dialect.DB2Dialect` (or any custom DB2 dialect subclass)
- `spring.datasource.driver-class-name: com.ibm.db2.jcc.DB2Driver`. Spring infers the driver from the URL.
- DB2 JDBC properties such as `currentSchema` in DB2 format, `progressiveStreaming`, `retrieveMessagesFromServerOnGetMessage`, `currentLockTimeout`, and similar.
- Any `sql-error-codes.xml` override containing DB2 codes.
---
 
## 4. Inventory DB2-Specific Code
 
Search the codebase, including `src/main/resources`, MyBatis mappers, `.sql` files, and test fixtures:
 
```bash
rg -n -i \
  'SYSIBM|SYSDUMMY1|WITH (UR|CS|RS|RR)\b|NEXT ?VAL(UE)? FOR|PREV ?VAL(UE)? FOR|IDENTITY_VAL_LOCAL|FINAL TABLE|CURRENT (DATE|TIME|TIMESTAMP|SCHEMA|SQLID)|\bVALUE\(|DECODE\(|LOCATE\(|POSSTR|DIGITS\(|\bDAYS\(|TIMESTAMPDIFF|VARCHAR_FORMAT|TIMESTAMP_FORMAT|LISTAGG|OPTIMIZE FOR|FOR READ ONLY|nativeQuery|com\.ibm\.db2|DB2Dialect|SQLCODE|getErrorCode|-803|-911|-913' \
  --glob '*.{java,xml,yml,yaml,properties,sql}'
```
 
Then catalog every hit in a tracking sheet with these columns: file, query, construct, fix, test coverage.
 
Also look for:
- `@Query(nativeQuery = true)`, `@NamedNativeQuery`, `@SqlResultSetMapping`
- `JdbcTemplate`, `NamedParameterJdbcTemplate`, `JdbcClient`, `SimpleJdbcCall`, and `StoredProcedure` usage
- `@Formula`, `@Subselect`, `@Where`/`@SQLRestriction`, and `columnDefinition` attributes
- Quoted identifiers in annotations, such as `@Table(name = "\"MY_TABLE\"")`
- Code that inspects `SQLException.getErrorCode()` or SQLSTATE strings
---
 
## 5. SQL Dialect Conversion Reference
 
| DB2 | PostgreSQL | Notes |
|---|---|---|
| `SELECT x FROM SYSIBM.SYSDUMMY1` | `SELECT x` | No dummy table needed. |
| `CURRENT DATE` / `CURRENT TIMESTAMP` | `CURRENT_DATE` / `CURRENT_TIMESTAMP` | Postgres requires the underscore. `now()`/`CURRENT_TIMESTAMP` returns the **transaction** start time; use `clock_timestamp()` for wall-clock time. |
| `NEXT VALUE FOR seq` / `NEXTVAL FOR seq` | `nextval('seq')` | |
| `PREVIOUS VALUE FOR seq` | `currval('seq')` | |
| `IDENTITY_VAL_LOCAL()` | `INSERT ... RETURNING id` | Prefer `RETURNING` over `lastval()`. |
| `SELECT id FROM FINAL TABLE (INSERT ...)` | `INSERT ... RETURNING id` | |
| `FETCH FIRST n ROWS ONLY` | Same, or `LIMIT n` | Supported as-is. |
| `... WITH UR` / `WITH CS` / `WITH RR` | *(remove)* | Postgres uses MVCC; set isolation via `@Transactional(isolation=...)` if truly needed. |
| `OPTIMIZE FOR n ROWS`, `FOR READ ONLY` | *(remove)* | |
| `VALUE(a, b)` / `NVL(a, b)` | `COALESCE(a, b)` | |
| `DECODE(x, a, r1, r2)` | `CASE x WHEN a THEN r1 ELSE r2 END` | |
| `LOCATE(sub, str)` / `POSSTR(str, sub)` | `strpos(str, sub)` | Note the argument order. |
| `DIGITS(n)` | `lpad(n::text, width, '0')` | |
| `CHAR(n)` / `VARCHAR(n)` used as a cast | `n::text` or `CAST(n AS varchar)` | |
| `INT(x)` / `INTEGER(x)` / `DECIMAL(x, p, s)` | `CAST(x AS integer)` / `CAST(x AS numeric(p,s))` | |
| `YEAR(d)`, `MONTH(d)`, `DAY(d)` | `EXTRACT(YEAR FROM d)` or `date_part(...)` | |
| `DATE(ts)` | `ts::date` | |
| `d + 30 DAYS` / `ts - 2 HOURS` | `d + INTERVAL '30 days'` / `ts - INTERVAL '2 hours'` | |
| `DAYS(d1) - DAYS(d2)` | `d1 - d2` | Date minus date yields an integer number of days. |
| `TIMESTAMPDIFF(2, CHAR(t1 - t2))` | `EXTRACT(EPOCH FROM (t1 - t2))` | Returns seconds. |
| `VARCHAR_FORMAT(ts, 'YYYY-MM-DD')` | `to_char(ts, 'YYYY-MM-DD')` | Verify format masks; they differ in places. |
| `TIMESTAMP_FORMAT(s, fmt)` / `TO_DATE` | `to_timestamp(s, fmt)` / `to_date(s, fmt)` | |
| `LISTAGG(col, ',')` | `string_agg(col, ',')` | |
| `RAND()` | `random()` | |
| `CONCAT(a, CONCAT(b, c))` | `a \|\| b \|\| c` or `concat(a, b, c)` | Postgres `concat()` is variadic and ignores NULLs; `\|\|` returns NULL if any operand is NULL. |
| `MERGE INTO ...` | `MERGE` (PG 15+) or `INSERT ... ON CONFLICT ... DO UPDATE` | `ON CONFLICT` requires a unique constraint or index on the conflict target. |
| `TRUNCATE TABLE t IMMEDIATE` | `TRUNCATE t` | |
| `CURRENT SCHEMA` | `current_schema` | |
 
---
 
## 6. JPA / Hibernate Entity Changes
 
### 6.1 Identifiers & Naming
- Remove quoted identifiers and `hibernate.globally_quoted_identifiers=true` unless the DBA intentionally created quoted mixed-case names.
- Spring's default naming strategy already produces lowercase snake_case, which matches Postgres conventions.
- Set the default schema with `currentSchema` in the URL or `hibernate.default_schema`. Avoid hard-coding the schema in `@Table(schema=...)` unless it is required.
### 6.2 ID Generation
- Prefer `GenerationType.SEQUENCE` over `IDENTITY`. `IDENTITY` disables Hibernate's insert batching.
- `allocationSize` **must equal** the sequence's `INCREMENT BY` (Hibernate defaults to 50). A mismatch causes duplicate keys or gaps.
```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
@SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50) // match DBA INCREMENT BY
private Long id;
```
 
- If the DBA uses `GENERATED ... AS IDENTITY`, use `GenerationType.IDENTITY` and accept the loss of insert batching.
### 6.3 Data Types
 
| Concern | Action |
|---|---|
| **CLOB / BLOB** | Postgres uses `text` / `bytea`. Avoid `@Lob` on these fields: on PostgreSQL it can map to large objects (`oid`), which won't match `text`/`bytea` columns. Use plain `String`/`byte[]` and let `ddl-auto=validate` confirm the mapping. |
| **Booleans** | If the DBA converted `CHAR(1)`/`SMALLINT` to `boolean`, change fields to `boolean`/`Boolean` and remove the old converters. If not, keep or add an `AttributeConverter`. |
| **Timestamps** | Postgres stores at most microsecond precision; DB2 allows up to 12 fractional digits. Truncate nanos in code or tests that compare `Instant`s exactly. |
| **Time zones** | If columns are `timestamptz`, map to `Instant`/`OffsetDateTime`. If they are `timestamp`, use `LocalDateTime`. Set `hibernate.jdbc.time_zone=UTC`. |
| **`CHAR(n)`** | Padding semantics differ, and `char(n)` loses trailing spaces when cast to `text`. Prefer `varchar`, and `trim()` in Java where code relied on DB2 padding. |
| **`DECIMAL` / `NUMERIC`** | Usually maps directly. Verify that precision and scale match `@Column(precision, scale)`. |
| **`columnDefinition`** | Remove or update any DB2 type names (`CLOB(1M)`, `GRAPHIC`, `VARGRAPHIC`, `DBCLOB`). |
 
### 6.4 Validate the Mapping
Run the app with `ddl-auto=validate` against the DBA's schema early and often. Hibernate will fail fast on type and column mismatches.
 
---
 
## 7. JDBC & Query Behavior Differences
 
### 7.1 Strict Typing
Postgres doesn't implicitly cast between strings and numbers. Comparing a `varchar` column to an integer parameter fails with `operator does not exist: character varying = integer`.
- Bind parameters with the correct Java type for each column.
- For nulls, use typed nulls (`setNull(i, Types.VARCHAR)`). Spring's `SqlParameterValue` helps here.
- Avoid the `stringtype=unspecified` URL flag as a blanket fix. It can hide real bugs.
### 7.2 IN Lists & Bind Limits
- The Postgres JDBC driver allows at most 32,767 bind parameters per statement. Chunk large `IN` lists, or use `= ANY(?)` with a SQL array:
```java
jdbcClient.sql("SELECT * FROM orders WHERE id = ANY(?)")
    .param(connection.createArrayOf("bigint", ids.toArray()))  // or use a JdbcTemplate PreparedStatementSetter
    .query(Order.class).list();
```
 
### 7.3 Result Set Streaming / Fetch Size
- The Postgres driver **loads the entire result set into memory** unless autocommit is off **and** a fetch size is set.
- For large reads, run inside a `@Transactional(readOnly = true)` method and set a fetch size (`jdbcTemplate.setFetchSize(...)`, or the JPA hint `org.hibernate.fetchSize`).
### 7.4 Ordering
- Without `ORDER BY`, row order is undefined, and Postgres order differs from DB2's. Add explicit `ORDER BY` anywhere code or tests depend on order.
- Collation can change sort order for mixed-case and accented text.
### 7.5 Case Sensitivity
- `LIKE` is case-sensitive. Use `ILIKE` or `lower(col) LIKE lower(?)` where case-insensitive matching is intended. Ask the DBA for a matching expression index.
- Result-set column labels come back **lowercase**. `rs.getString("CUSTOMER_ID")` still works because label lookup is case-insensitive. However, anything that serializes column names will change, such as `Map` keys copied into JSON, CSV headers, or report generators.
### 7.6 Batching
- `reWriteBatchedInserts=true` on the URL lets the driver rewrite batched inserts into multi-row inserts.
- Check `batchUpdate` return values. With rewriting enabled, the driver may return `Statement.SUCCESS_NO_INFO` (-2) instead of actual row counts.
---
 
## 8. Transactions, Locking & Error Handling
 
### 8.1 Aborted-Transaction Behavior (most common runtime surprise)
In Postgres, **any error inside a transaction aborts the entire transaction**. Every subsequent statement then fails with `current transaction is aborted, commands ignored until end of transaction block`. DB2 lets you continue after a failed statement.
 
Patterns that break:
- Try an insert, catch the duplicate-key error, then update, all in the same transaction.
- Swallowing a `DataAccessException` in a loop and continuing.
Fixes:
- Use `INSERT ... ON CONFLICT DO UPDATE/NOTHING` for upserts.
- Run the risky statement in its own transaction with `@Transactional(propagation = REQUIRES_NEW)`.
- Use savepoints with `TransactionTemplate` and a nested propagation, or explicitly through the `Connection`.
### 8.2 Locking & Isolation
- In Postgres, readers never block writers. Code that relied on DB2 read locks for coordination needs explicit `SELECT ... FOR UPDATE`, or `@Lock(PESSIMISTIC_WRITE)` in JPA.
- `FOR UPDATE SKIP LOCKED` is available and works well for queue-style processing.
- Lock-timeout errors differ. Postgres raises SQLSTATE `55P03` for lock not available and `40P01` for deadlock; DB2 used SQLCODE -911/-913. Update any retry logic accordingly.
- Prefer `@Version` optimistic locking where possible.
### 8.3 Error Codes
Replace DB2 SQLCODE checks with Spring's exception hierarchy, which is portable:
 
| DB2 | PostgreSQL SQLSTATE | Spring exception |
|---|---|---|
| -803 (duplicate key) | `23505` | `DuplicateKeyException` |
| -530 / -532 (FK violation) | `23503` | `DataIntegrityViolationException` |
| -407 (NOT NULL) | `23502` | `DataIntegrityViolationException` |
| -911 / -913 (deadlock/timeout) | `40P01` / `55P03` | `PessimisticLockingFailureException` / `CannotAcquireLockException` |
| -204 (object not found) | `42P01` | `BadSqlGrammarException` |
 
Catch Spring exceptions, not vendor codes. If you must check codes, use `SQLException.getSQLState()`, never `getErrorCode()`. Postgres always returns 0 for `getErrorCode()`.
 
---
 
## 9. Stored Procedures & Functions
 
- Confirm with the DBA whether each routine became a Postgres `PROCEDURE` (invoked with `CALL`, PG 11+) or a `FUNCTION` (invoked with `SELECT`).
- Functions that return result sets typically return `SETOF`/`TABLE`. Query them like tables: `SELECT * FROM fn(?, ?)`.
- `refcursor` results require autocommit off, so call them inside a transaction.
- Re-test `SimpleJdbcCall` metadata lookups. Explicitly declare parameters with `.withoutProcedureColumnMetaDataAccess()` if detection is unreliable.
- Look for logic that belongs in the app rather than in PL/pgSQL. Simple procs are often easier to move into Java than to port.
---
 
## 10. Testing Strategy
 
### 10.1 Integration Tests on Real Postgres
- Remove H2 and DB2 test configurations.
- Use Testcontainers with the **same Postgres major version as production**, plus `spring-boot-testcontainers` and `@ServiceConnection`:
```java
@TestConfiguration(proxyBeanMethods = false)
class PostgresTestConfig {
    @Bean
    @ServiceConnection
    PostgreSQLContainer postgres() {
        // Match prod major version. The package and generics differ between Testcontainers 1.x and 2.x.
        return new PostgreSQLContainer("postgres:17");
    }
}
```
 
- Load the **DBA's DDL** into the container, through an init script or by having the DBAs provide a schema dump, so tests validate against the real target schema.
- Make sure every native query and every repository method is exercised by at least one test.
### 10.2 Parity / Golden-Master Testing
- Run critical queries and API endpoints against DB2 (current prod-like data) and against Postgres (the DBA-migrated copy of the same data). Diff the results.
- Focus on ordering, timestamps (precision and time zones), numeric rounding, trimmed `CHAR` values, NULL handling in concatenation, and case-sensitive filters.
### 10.3 Performance Testing
- Capture the top-N queries by frequency and latency on DB2. Run them on Postgres with `EXPLAIN (ANALYZE, BUFFERS)`.
- Postgres has no optimizer hints. Fix slow plans with indexes (request these from the DBAs), query rewrites, or fresh statistics (`ANALYZE` after the data load).
- Expression filters like `lower(col)` or `date(col)` need matching expression indexes.
- Load-test with production-like pool sizes. Postgres connection count is a hard resource limit.
### 10.4 Things to Assert Explicitly
- [ ] App starts with `ddl-auto=validate` against the DBA schema
- [ ] First insert after data load succeeds (sequences reset)
- [ ] Upserts and duplicate handling work without aborting the transaction
- [ ] Large exports and reports stream without out-of-memory errors (fetch size)
- [ ] Batch jobs complete within time windows
- [ ] Date and time values round-trip with the expected time zone and precision
---
 
## 11. Suggested Order of Work
 
1. **Kickoff with DBAs.** Get the target DDL, type mapping, sequence plan, and Postgres version (Section 2).
2. **Inventory.** Grep and catalog every DB2-specific construct (Section 4).
3. **Test harness.** Set up Testcontainers with the DBA schema, and make existing tests run against Postgres (failures are expected).
4. **Dependencies & config.** Swap the driver, update the URL and Hikari settings, and remove dialect and DB2 properties (Section 3).
5. **Entities.** Fix IDs, types, and naming until `ddl-auto=validate` passes (Section 6).
6. **Native SQL.** Convert queries by category using the reference table (Section 5).
7. **Behavioral fixes.** Address typing, fetch size, ordering, transactions, locking, and error handling (Sections 7–8).
8. **Stored procedures** (Section 9).
9. **Parity and performance testing** against DBA-migrated data (Section 10).
10. **Cutover rehearsal.** Run it in a lower environment with the DBAs, using the same steps and timings planned for production.
11. **Production cutover.** Deploy the Postgres build, run smoke tests, and monitor errors, latency, and connections.
---
 
## 12. Cutover & Rollback
 
- Keep all connection details externalized (env vars or a secrets manager). The cutover should be a deploy plus a config change, with no code edits.
- Tag the last DB2-compatible release so rollback is a redeploy.
- Agree with the DBAs on the write-freeze window, data-sync verification, and the point of no return. After that point, writes land only in Postgres and a DB2 rollback means data loss.
- Have post-cutover monitoring ready: error rate by SQLSTATE, slow-query log or `pg_stat_statements` (with the DBAs), and pool metrics (Hikari active, pending, and timeouts).
- Once stable, delete the DB2 leftovers: dead config, profiles, converters, and docs.
---
 
## 13. Definition of Done
 
- [ ] No DB2 driver, dialect, or properties remain in the code or build
- [ ] Inventory sheet is 100% resolved, with each native query covered by a test
- [ ] All tests pass against Postgres Testcontainers using the DBA schema
- [ ] `ddl-auto=validate` passes in every environment
- [ ] No `getErrorCode()` / SQLCODE logic remains; Spring exceptions are used
- [ ] Parity diffs are reviewed and signed off
- [ ] Performance meets or beats the DB2 baseline for critical paths
- [ ] Cutover rehearsed successfully in a lower environment
- [ ] Runbook and README updated for Postgres (local dev setup, connection config)
 
