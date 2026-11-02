# DB2 LUW → PostgreSQL: Key Differences for the Spring Boot Migration

Scope: application code (Spring Boot 4, Java 21, Maven). Schema and data movement are owned by the DBA; items that touch the schema are noted as **coordinate with DBA**.

Most JPQL/HQL and Spring Data derived queries port cleanly once the dialect changes. The real risk is in **native SQL** (`@Query(nativeQuery = true)`, `JdbcTemplate`, `@NamedNativeQuery`, `@Formula`, `@Subselect`, `columnDefinition`), **error-code handling**, and **transaction behavior**.

---

## Background: what each database is

Both are **relational database management systems (RDBMS)**. They store data in tables, use SQL, and provide full ACID transactions. They are general-purpose databases for transactional (OLTP) workloads, and both can also handle analytics. PostgreSQL is often called **object-relational**, because it lets you define your own types, operators, functions, and index methods.

### IBM Db2

**History**
- Db2 descends from **IBM System R** (1970s), the research project that put E. F. Codd's relational model into practice. System R produced SQL and the cost-based query optimizer that most databases still follow.
- It shipped in **1983 as DB2 for MVS**, on IBM mainframes. It became the standard system of record for banks, insurers, airlines, and government.
- The **LUW** product (Linux, UNIX, Windows) began in the early 1990s as DB2 for OS/2 and was later branded *DB2 Universal Database*. It is a separate codebase from Db2 for z/OS, though the two share SQL features. IBM restyled the name to **"Db2"** in 2017.
- It is a commercial product sold and supported by IBM. A free Community Edition exists, with resource limits.

**What makes it special**
- **Mainframe heritage:** decades of tuning for high-volume, mission-critical OLTP, with a strong reputation for reliability.
- **Mature cost-based optimizer** and built-in **workload management** (WLM), which can prioritize and throttle workloads.
- **pureScale:** active-active clustering on shared disk, modeled on z/OS Parallel Sysplex, giving scale-out and high availability.
- **BLU Acceleration:** in-memory columnar tables for analytics, alongside regular row tables.
- **HADR** for replication and failover, plus strong compression, pureXML, and enterprise tooling.
- **Vendor support:** a single company answers for the whole stack.

**Trade-offs**
- **Cost:** licensing and support are expensive and are often priced per core or per processor value unit (PVU).
- **Vendor lock-in** and a smaller talent pool and ecosystem.
- Less community tooling, fewer extensions, and slower adoption of newer developer conveniences.
- Differences between the LUW and z/OS codebases can surprise people.

### PostgreSQL

**History**
- It grew out of **Ingres** at UC Berkeley. Michael Stonebraker started the **POSTGRES** project ("post-Ingres") in **1986** to explore extensible types and rules.
- POSTGRES originally used its own query language, QUEL. **Postgres95** (1994–95) added SQL, and the project was renamed **PostgreSQL** in 1996.
- It has been developed ever since by the **PostgreSQL Global Development Group**, a community with no single owning company. A new major version ships roughly once a year.
- It is released under the **PostgreSQL License**, a permissive BSD/MIT-style license. It is free to use, modify, and embed, including commercially.

**What makes it special**
- **Extensibility:** you can add your own data types, operators, functions, and index types. A large extension ecosystem builds on this, including PostGIS (geospatial), pgvector (vector search), TimescaleDB (time series), Citus (sharding), and pg_stat_statements.
- **Rich types and indexes:** `jsonb`, arrays, ranges, `uuid`, full-text search, and B-tree, Hash, GIN, GiST, SP-GiST, and BRIN indexes.
- **MVCC:** readers and writers don't block each other.
- **Strict standards conformance** and transactional DDL.
- **Ubiquity:** every major cloud offers it as a managed service (RDS/Aurora, Cloud SQL/AlloyDB, Azure, and others). It has a large talent pool and first-class support in Spring, Hibernate, Flyway, Liquibase, and Testcontainers.
- **No license cost** and no vendor lock-in.

**Trade-offs**
- **VACUUM:** MVCC keeps old row versions, so tables need vacuuming. Autovacuum handles it, but heavy-update tables can bloat if it isn't tuned.
- **Process-per-connection:** each connection is an OS process, so a pooler is needed at high connection counts (HikariCP in the app, sometimes PgBouncer in front).
- **No built-in shared-disk, active-active clustering** like pureScale. High availability usually means streaming replication plus failover tooling (Patroni or a cloud-managed service). Write scale-out needs extensions such as Citus.
- **No native optimizer hints** (the `pg_hint_plan` extension adds them) and less built-in workload management.
- **Support model:** support comes from the community or a vendor you choose (EDB, Crunchy, a cloud provider), rather than a single owning company.

### Summary

| | Db2 LUW | PostgreSQL |
|---|---|---|
| Type | Relational (RDBMS) | Object-relational (ORDBMS) |
| Origin | IBM System R → DB2 (1983) | Berkeley POSTGRES (1986) → PostgreSQL (1996) |
| License | Commercial (IBM) | Open source (PostgreSQL License) |
| Governance | IBM | Community (PGDG) |
| Concurrency | Lock-based (optional Currently Committed) | MVCC |
| Scale-out / HA | pureScale, HADR | Streaming replication, Patroni/managed HA; Citus for sharding |
| Analytics | BLU columnar | Extensions (e.g., columnar options), parallel query |
| Extensibility | Limited | Core design goal |
| Strength | Mission-critical enterprise OLTP, vendor support | Flexibility, ecosystem, cost, cloud availability |

---

## 1. Driver, connection, and build config

| Area | DB2 | PostgreSQL |
|---|---|---|
| Maven dependency | `com.ibm.db2:jcc` | `org.postgresql:postgresql` |
| Driver class | `com.ibm.db2.jcc.DB2Driver` | `org.postgresql.Driver` (usually auto-detected; omit) |
| JDBC URL | `jdbc:db2://host:50000/DBNAME` | `jdbc:postgresql://host:5432/dbname?currentSchema=myschema` |
| Hibernate dialect | `DB2Dialect` | `PostgreSQLDialect` — remove any explicit dialect property and let Hibernate detect it |
| Default schema | `CURRENT SCHEMA` (defaults to auth ID) | `search_path` (defaults to `"$user", public`) — set `currentSchema` on the URL or `hibernate.default_schema` |
| Hikari test query | `SELECT 1 FROM SYSIBM.SYSDUMMY1` | Remove `connection-test-query`; JDBC4 `isValid()` is used |
| Flyway (if used) | — | Flyway 10+ needs `org.flywaydb:flyway-database-postgresql` |
| Tests | DB2 Testcontainer / H2 `MODE=DB2` | `PostgreSQLContainer` (prefer real Postgres over H2 `MODE=PostgreSQL`) |

Useful PG JDBC URL params:
- `reWriteBatchedInserts=true` — big speedup for batch inserts.
- `stringtype=unspecified` — lets string-bound params be coerced by the server (a stopgap for implicit-cast issues; see §4, fix properly where possible).
- `ApplicationName=my-app` — shows up in `pg_stat_activity`.

---

## 2. Identifiers and naming

| Topic | DB2 | PostgreSQL | Impact |
|---|---|---|---|
| Unquoted identifier folding | **UPPERCASE** | **lowercase** | `"CUSTOMER"` (quoted) ≠ `customer`. Any quoted uppercase names in native SQL, `@Table(name = "\"X\"")`, or `globally_quoted_identifiers=true` will break. |
| Max identifier length | 128 | **63** (silently truncated) | Long constraint/index/sequence names Hibernate references can collide. **Coordinate with DBA.** |
| Default sequence naming (Hibernate 6+) | `<entity>_SEQ` | same, but folded to lowercase unless quoted | Verify names match DBA's sequences. |

---

## 3. Data types and Java mappings

| DB2 | PostgreSQL | Java / JPA notes |
|---|---|---|
| `SMALLINT` / `INTEGER` / `BIGINT` | same | — |
| `DECIMAL(p,s)` / `NUMERIC` | `numeric(p,s)` | — |
| `DECFLOAT` | `numeric` | `BigDecimal` |
| `REAL` / `DOUBLE` | `real` / `double precision` | — |
| `CHAR(n)` | `char(n)` (`bpchar`) | Both pad; PG ignores trailing spaces in `=` but **not** in `LIKE` or after cast to `text`. Many teams switch to `varchar`/`text` — **coordinate with DBA**. |
| `VARCHAR(n)` | `varchar(n)` / `text` | DB2 length is usually **bytes**; PG is **characters**. `@Size`/`@Column(length)` may need review. |
| `GRAPHIC` / `VARGRAPHIC` | `varchar` / `text` | — |
| `CLOB` | `text` | **Remove `@Lob` on `String`** or map explicitly — on PG, `@Lob` can map to `oid` large objects rather than `text`. |
| `BLOB` | `bytea` | **Remove `@Lob` on `byte[]`** (or use `@JdbcTypeCode(SqlTypes.VARBINARY)`); `@Lob` → `oid` large objects, which must be read inside a transaction and aren't deleted with the row. |
| `SMALLINT`/`CHAR(1)` used as flags (`0/1`, `'Y'/'N'`) | often converted to `boolean` | If DBA converts, entity fields/`AttributeConverter`s and native-SQL literals (`= 1`, `= 'Y'`) must change to `true/false`. |
| `BOOLEAN` (DB2 11.1+) | `boolean` | — |
| `DATE` | `date` | `LocalDate` |
| `TIME` | `time` | `LocalTime` |
| `TIMESTAMP(p)` (precision up to 12) | `timestamp(p)` (max **6**, microseconds) | Sub-microsecond precision is lost. |
| *(no TZ type on LUW)* | `timestamptz` | If DBA uses `timestamptz`, prefer `Instant`/`OffsetDateTime`. Stored as UTC; displayed in session `TimeZone`. |
| `XML` | `xml` | XML functions differ completely. |
| — | `uuid`, `jsonb`, arrays | Available if you want them later. |

**Empty string vs NULL:** same in both — `''` is **not** NULL (unlike Oracle).

---

## 4. Type strictness (common runtime breakage)

PostgreSQL does **far less implicit casting** than DB2.

- `WHERE varchar_col = 123` → error: `operator does not exist: character varying = integer`.
- Binding `setString()` to an integer/date column → error: `column "x" is of type integer but expression is of type character varying`.
- `CASE`/`COALESCE`/`UNION` branches must have compatible types.
- String literals for dates must be ISO: `'2026-11-02 14:30:00'`. **DB2's `'2026-11-02-14.30.00.000000'` format does not parse in PG** — check any code that formats timestamps as strings.

Fix by binding correct Java types, or explicit casts (`CAST(? AS integer)` / `?::integer`).

---

## 5. SQL syntax and functions

### Special registers and dummy table

| DB2 | PostgreSQL |
|---|---|
| `SELECT ... FROM SYSIBM.SYSDUMMY1` | `SELECT ...` (no `FROM` needed) |
| `CURRENT DATE` / `CURRENT TIMESTAMP` (space) | `CURRENT_DATE` / `CURRENT_TIMESTAMP` (underscore required) |
| `CURRENT TIMESTAMP` = statement time | `CURRENT_TIMESTAMP`/`now()` = **transaction start** time; use `clock_timestamp()` for wall clock |
| `CURRENT USER` / `CURRENT SCHEMA` | `CURRENT_USER` / `current_schema()` |

### Sequences and generated keys

| DB2 | PostgreSQL |
|---|---|
| `NEXT VALUE FOR seq` | `nextval('seq')` |
| `PREVIOUS VALUE FOR seq` | `currval('seq')` |
| `IDENTITY_VAL_LOCAL()` | `INSERT ... RETURNING id` (or `lastval()`) |
| `SELECT ... FROM FINAL TABLE (INSERT ...)` | `INSERT ... RETURNING ...` |
| `GENERATED ALWAYS AS IDENTITY` | same syntax; explicit inserts need `OVERRIDING SYSTEM VALUE` |

### Date/time

| DB2 | PostgreSQL |
|---|---|
| `col + 1 DAY`, `- 3 MONTHS` (labeled durations) | `col + INTERVAL '1 day'`, `- INTERVAL '3 months'` |
| `DAYS(a) - DAYS(b)` | `a::date - b::date` (returns integer days) |
| `TIMESTAMPDIFF(n, CHAR(a - b))` | `EXTRACT(EPOCH FROM a - b)` / `age(a, b)` |
| `DATE(ts)` / `TIME(ts)` | `ts::date` / `ts::time` |
| `YEAR(x)`, `MONTH(x)`, `DAY(x)`, `HOUR(x)` | `EXTRACT(YEAR FROM x)` etc. |
| `TIMESTAMP('...')` | `CAST('...' AS timestamp)` / `'...'::timestamp` |
| `VARCHAR_FORMAT(ts, 'YYYY-MM-DD')` / `TO_CHAR` | `to_char(ts, 'YYYY-MM-DD')` (format codes mostly match; verify) |
| `DAYOFWEEK(x)` (1 = Sunday) | `EXTRACT(DOW FROM x)` (**0** = Sunday) |

### Strings and misc functions

| DB2 | PostgreSQL |
|---|---|
| `VALUE(a,b)`, `IFNULL(a,b)`, `NVL(a,b)` | `COALESCE(a,b)` |
| `DECODE(x, a, b, ...)` | `CASE` |
| `LOCATE(needle, hay)` / `POSSTR(hay, needle)` | `strpos(hay, needle)` / `POSITION(needle IN hay)` |
| `TRANSLATE(str, to, from)` | `translate(str, from, to)` — **argument order is reversed** |
| `CHAR(int)` / `VARCHAR(x)` | `x::text` / `CAST(x AS varchar)` |
| `INT(x)` / `INTEGER(x)` / `DECIMAL(x,p,s)` | `CAST(x AS integer)` / `CAST(x AS numeric(p,s))` |
| `DIGITS(n)` | `lpad(n::text, width, '0')` |
| `LISTAGG(col, ',')` | `string_agg(col, ',')` |
| `CONCAT(a, b)` (2 args) | `CONCAT(a, b, ...)` (variadic; NULLs treated as `''`) — `\|\|` behaves the same in both (NULL → NULL) |
| `RAND()` | `random()` |
| `SUBSTR`, `UPPER`, `LOWER`, `TRIM`, `LENGTH`, `REPLACE` | same |

### Query clauses

| DB2 | PostgreSQL |
|---|---|
| `FETCH FIRST n ROWS ONLY`, `OFFSET n ROWS` | supported as-is (also `LIMIT`/`OFFSET`) |
| `WITH UR` / `WITH CS` / `WITH RS` / `WITH RR` | **Remove** — not supported (see §6) |
| `OPTIMIZE FOR n ROWS` | **Remove** |
| `FOR READ ONLY` / `FOR FETCH ONLY` | **Remove** |
| `FOR UPDATE OF col` | `FOR UPDATE` (also `NOWAIT`, `SKIP LOCKED`) |
| Recursive CTE: `WITH x AS (...)` | `WITH RECURSIVE x AS (...)` — keyword required |
| `MERGE INTO ...` | `MERGE` (PG 15+) or `INSERT ... ON CONFLICT ... DO UPDATE` (usually simpler) |
| `DECLARE GLOBAL TEMPORARY TABLE SESSION.t` | `CREATE TEMP TABLE t` |
| `VALUES expr` | `VALUES expr` works too |

**Null ordering:** same — NULLs sort high (last in `ASC`) in both.
**Implicit row order:** queries without `ORDER BY` will return rows in different orders — tests that assume order will flake.
**Collation:** DB2 databases often use binary (`IDENTITY`) collation; PG uses the database locale (e.g., `en_US.UTF-8`), so `ORDER BY` on strings and range comparisons can differ. **Coordinate with DBA** (`COLLATE "C"` if binary order matters). Case-insensitive match: `ILIKE` is available.

---

## 6. Transactions, locking, and concurrency

| Topic | DB2 | PostgreSQL |
|---|---|---|
| Concurrency model | Lock-based (readers can block writers, unless Currently Committed is on) | MVCC — readers never block writers |
| Default isolation | Cursor Stability (CS) | Read Committed |
| Per-statement isolation | `WITH UR/CS/RS/RR` | Not supported; set at transaction level (`@Transactional(isolation = ...)`) |
| Dirty reads (UR) | Supported | Not supported (UR behaves as Read Committed) |
| **Error inside a transaction** | Statement fails; transaction continues | **Whole transaction is aborted** — every later statement fails with `current transaction is aborted, commands ignored until end of transaction block` until rollback |
| Lock timeout | `LOCKTIMEOUT` | `lock_timeout` / `statement_timeout` session settings |
| DDL | Transactional | Transactional |

**The aborted-transaction rule is the biggest behavioral change.** Patterns like "try insert → catch duplicate → update" inside one `@Transactional` method will break. Use `INSERT ... ON CONFLICT`, a savepoint, or a separate transaction (`REQUIRES_NEW`).

---

## 7. Error handling

| Case | DB2 SQLCODE | PG SQLSTATE |
|---|---|---|
| Duplicate key | -803 | `23505` |
| FK violation | -530 / -532 | `23503` |
| NOT NULL violation | -407 | `23502` |
| Deadlock / rollback | -911 | `40P01` |
| Lock timeout | -911 / -913 | `55P03` |
| Serialization failure | — | `40001` |
| Undefined table / column | -204 / -206 | `42P01` / `42703` |

- PG's driver returns `getErrorCode() == 0`; **only `getSQLState()` is meaningful**. Any code checking `getErrorCode() == -803` (or DB2 `SqlIntegrityConstraintViolationException` subclasses from the IBM driver) must change.
- Spring's exception translation (`DuplicateKeyException`, `DataIntegrityViolationException`, `CannotAcquireLockException`, etc.) works for both — prefer catching those.
- Retry logic keyed on -911 should key on `40001`/`40P01`.

---

## 8. Stored procedures and functions

| Topic | DB2 | PostgreSQL |
|---|---|---|
| Language | SQL PL | PL/pgSQL |
| Returning result sets | `DYNAMIC RESULT SETS n` + open cursor | Procedures **can't** return result sets directly; use a **function** returning `SETOF`/`TABLE` (call via `SELECT * FROM fn(...)`) or a `refcursor` |
| Calling | `CALL proc(?, ?)` | `CALL proc(...)` for procedures; `SELECT fn(...)` for functions |
| OUT params | Supported | Supported on procedures (PG 14+ for OUT on procedures); functions return values instead |

`SimpleJdbcCall` / `@Procedure` / `StoredProcedureQuery` call sites need review once the DBA defines the PG equivalents.

---

## 9. Hibernate / Spring Data specifics

- **Sequence `allocationSize`** (default 50) must match the PG sequence's `INCREMENT BY`, or IDs will collide or `ddl-auto=validate` will fail. **Coordinate with DBA.**
- **Sequences after data load** must be reset (`setval`) past the max migrated ID. **Coordinate with DBA.**
- Prefer `SEQUENCE` over `IDENTITY` generation — `IDENTITY` disables JDBC insert batching.
- Enable batching: `hibernate.jdbc.batch_size`, `hibernate.order_inserts=true`, `hibernate.order_updates=true`, plus `reWriteBatchedInserts=true`.
- Review `columnDefinition = "..."`, `@Formula`, `@Subselect`, `@ColumnTransformer`, `@Check` for DB2-specific SQL.
- Pessimistic locks (`@Lock(PESSIMISTIC_WRITE)`) map to `FOR UPDATE` automatically.
- `spring.jpa.hibernate.ddl-auto` should be `validate` or `none` (DBA owns the schema).

---

## 10. Performance / JDBC behavior

- **Result-set streaming:** the PG driver loads the **entire result set into memory** unless autocommit is off **and** `fetchSize` is set. For large reads (exports, batch jobs), run inside a transaction and set `setFetchSize(n)` / `@QueryHints(HINT_FETCH_SIZE)` / Hibernate `ScrollableResults`. DB2 cursor blocking hid this.
- Server-side prepared statements kick in after `prepareThreshold` (default 5) executions.
- Count queries (`SELECT COUNT(*)`) are slower on large PG tables than DB2 (MVCC visibility checks) — watch Spring Data `Page` queries on big tables; consider `Slice`.
- Query plans differ; `EXPLAIN ANALYZE` hot queries after cutover. Indexes are the DBA's call, but functional indexes (e.g., on `lower(col)`) may matter for app queries.

---

## 11. Codebase grep checklist

Search the app for these to find DB2-specific code:

```
SYSIBM
SYSDUMMY1
SYSCAT
NEXT VALUE FOR
PREVIOUS VALUE FOR
IDENTITY_VAL_LOCAL
FINAL TABLE
WITH UR
WITH CS
WITH RS
WITH RR
OPTIMIZE FOR
FOR READ ONLY
FOR FETCH ONLY
CURRENT DATE
CURRENT TIMESTAMP
CURRENT TIME
DAYS(
TIMESTAMPDIFF
VARCHAR_FORMAT
DECODE(
VALUE(
IFNULL(
NVL(
LOCATE(
POSSTR(
TRANSLATE(
LISTAGG
DIGITS(
DAYOFWEEK
GLOBAL TEMPORARY
SESSION.
" DAY"  " DAYS"  " MONTH"  " MONTHS"  " YEAR"  " YEARS"   (labeled durations)
getErrorCode()
-803  -911  -913  -204  -206  -530  -407
com.ibm.db2
DB2Dialect
nativeQuery = true
@NamedNativeQuery
@Formula
@Subselect
columnDefinition
@Lob
JdbcTemplate
SimpleJdbcCall
StoredProcedureQuery
```

---

## 12. Migration-time quick hits (likely first failures)

1. Driver/URL/dialect swap and `SYSDUMMY1` health checks.
2. Quoted uppercase identifiers and schema resolution (`search_path`).
3. Native SQL using DB2 functions, `WITH UR`, labeled durations, `CURRENT TIMESTAMP`.
4. Implicit-cast errors from mismatched parameter types.
5. `@Lob` mappings on `String` / `byte[]`.
6. Boolean/flag column type changes from the DBA.
7. Duplicate-key handling relying on SQLCODE or continuing after an error in the same transaction.
8. Sequence `allocationSize` vs `INCREMENT BY`, and sequence values after data load.
9. Large result sets now buffered in memory.
10. Tests depending on unordered result order or DB2/H2 behavior.
