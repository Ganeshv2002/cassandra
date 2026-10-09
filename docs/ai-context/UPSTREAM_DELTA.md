# Upstream Delta

_Checked 2026-10-09 by fetching `apache/cassandra` trunk into a read-only `upstream` remote and inspecting it with `git ls-tree`, `git show` and `git grep`. Nothing was merged, checked out, built or run. Fork point: `33feca3d84` (2025-01-15). Upstream head: `36d2433dfb` (2026-10-08)._

## How far behind

| Measure | Fork | Upstream trunk |
|---|---|---|
| Version (`build.xml`) | 5.1-SNAPSHOT | 7.0 (6.0 is at alpha2 in `CHANGES.txt`) |
| Commits | 0 ahead | 2,008 behind |
| `src/java` Java files | 2,706 | 3,271 |
| Diff in `src/java` | | 2,364 files changed, +185,412 / -20,786 lines |
| Supported Java | 11, 17 | 11, 17, 21 |

The fork has no commits of its own on trunk, so catching up is a fast-forward with no conflicts (the docs branch adds only new files under `docs/ai-context/`).

## What upstream has that this tree lacks

| Area | Upstream evidence | Effect on the earlier analysis |
|---|---|---|
| General transactions (Accord, CEP-15) | `service/accord/` (194 files), `modules/accord`, new `journal/` package, `formalise/accord` | Confirms "saturated" |
| Automated repair (CEP-37) | `repair/autorepair/` (12 files) | Confirms "saturated" |
| Constraints (CEP-42), including `NOT NULL` and SPI-loaded custom constraints | `cql3/constraints/` (19 files) | Confirms "saturated" |
| Slow query table | `db/virtual/SlowQueriesTable.java`, `utils/logging/SlowQueriesAppender.java` (CASSANDRA-13001) | Partly covers the "doctor" idea |
| Partition key statistics, uncaught exceptions, schema comments and security labels as virtual tables | `PartitionKeyStatsTable`, `ExceptionsTable`, `SchemaCommentsTable`, `SchemaSecurityLabelsTable` | Partly covers the "doctor" idea |
| Client warnings on large-partition writes | CASSANDRA-17258 | Partly covers the "doctor" idea |
| Built-in async profiling | `profiler/AsyncProfilerMBean.java` (CASSANDRA-20854) | Diagnosis tooling exists |
| More guardrails | misprepared statements, client driver versions, Zstd level, CMS size, disk usage across replicas, `durable_writes` | Guardrail framework is actively extended |
| Zstd dictionary compression; hardware-accelerated compression (CEP-49) | CASSANDRA-17021, CASSANDRA-20975 | New, not in the earlier gap list |
| Cursor-based compaction; direct I/O for compaction reads and background writes | CASSANDRA-20918, 19987, 21134 | New |
| SAI: scored vector iterators, blob type, frozen collection elements, sharded in-memory indexes | `index/sai/disk/v1/vector/*WithScore*`, CASSANDRA-20012, 18492, 18216 | SAI moves quickly, as warned |
| logback 1.5.18 and slf4j 2.0.17 | CASSANDRA-20429 | Part of debt item H6 is fixed upstream |
| AI contributor guidance | root `AGENTS.md`, `CLAUDE.md`, and `.claude/skills/` (in-JVM dtest, review, bug archaeology, TLA+ skills) | See "Effect on these documents" |

## What upstream still does not have

Checked by searching upstream trunk; absence of a match is evidence, not proof.

| Item | Check | Result |
|---|---|---|
| `EXPLAIN` for CQL queries | `git grep -i EXPLAIN` in `src/antlr/Parser.g`; changelog search for "explain" and "query plan" | No match |
| Tombstone diagnostics as a virtual table | file names under `db/virtual/` | None dedicated |
| Hybrid lexical plus vector ranking, BM25, vector quantization | file names under `index/sai/`; changelog search | No match |
| Per-tenant quotas or fairness | file-name search for tenant and quota | No match |
| Tiered or object storage for SSTables | file-name search | No match |
| Paxos v2 as default | `config/Config.java:1193` still `PaxosVariant.v1` (a yaml option was documented in CASSANDRA-21316) | Still v1 |
| SAI as default index | `config/Config.java:1012` still `CassandraIndex.NAME` | Still legacy |
| Materialized views and SASI enabled by default | `Config.java:707,717` | Still off |

## Revised gap ratings

| Gap | Earlier | Now | Why |
|---|---|---|---|
| Operator diagnosis and explainability | Strong | **Strong, but narrower** | Slow queries, partition stats and large-partition warnings exist upstream. What is still missing is the part that explains: a CQL `EXPLAIN` showing the chosen index plan and replicas, and a tombstone and partition "why is this slow" view |
| Hybrid search and vector quality | Strong to moderate | **Strong to moderate (unchanged)** | Upstream added scoring plumbing but no hybrid ranking or quantization |
| Per-tenant quotas | Moderate | **Moderate (unchanged)** | Nothing upstream |
| Tiered or object storage | Moderate | **Moderate (unchanged)** | Nothing upstream; large effort |
| Transactions, auto-repair, constraints | Saturated | **Saturated (confirmed)** | Present in upstream code |

## Recommended first feature

A read-only **`EXPLAIN` for SELECT statements**: report the read command type, the index query plan chosen (or that the query filters without an index), the replica plan for the consistency level, and applicable guardrail thresholds. It is small, additive, touches the parser, `SelectStatement` and SAI plan classes without changing execution, and is testable with `CQLTester`. It needs a Jira ticket and probably a short CEP because it adds CQL syntax. Search the Cassandra Jira and dev list for existing proposals before starting; that was not done here.

## Effect on these documents

- Upstream now ships its own agent guidance (`AGENTS.md`, `CLAUDE.md`, `.claude/skills/`). After syncing, those are authoritative for build, test and style rules; `docs/ai-context/` should stay as the architectural and market context and must not duplicate or contradict them.
- Line numbers in `DATA_FLOWS.md` and counts in `TECH_DEBT.md` describe the fork point and will be wrong after a sync. They need a refresh pass then.
- `TECH_DEBT.md` H6 is partly resolved upstream (logback, Java 21); the upstream dependency file moved to `.build/cassandra-deps-maven-pom.xml`, and exact versions there were not read successfully in this pass.
