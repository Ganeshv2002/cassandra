# Technical Debt

_Assessed 2026-10-08 against commit `33feca3d84` by reading code and counting with grep. No tests or builds were run. Ratings are judgement calls about risk to someone changing this codebase, not bug reports. Paths are relative to `src/java/org/apache/cassandra/` unless they start at the repo root._

> **Update 2026-10-09:** measured against upstream: 2,008 commits behind, 0 ahead, so the sync is a fast-forward. H6 is partly fixed upstream. Details in `UPSTREAM_DELTA.md`.

## Critical

| # | Item | Evidence | Why it matters |
|---|---|---|---|
| C1 | Fork is about 21 months behind upstream | HEAD is dated 2025-01-15; upstream 6.0-alpha1 shipped spring 2026 with Accord, auto-repair, constraints | Features built here will conflict with or duplicate upstream, and the tree misses 21 months of fixes, including security fixes in dependencies |
| C2 | Uncommitted local change in a core file | `git status` shows `service/CassandraDaemon.java` modified: a stray space in `getBoolean ()` at line 119 | Harmless to behaviour, but it will fail checkstyle and ride along into unrelated commits. Left untouched by this analysis |

## High

| # | Item | Evidence | Why it matters |
|---|---|---|---|
| H1 | God classes at the centre of every flow | `DatabaseDescriptor` 5,527 lines, `StorageService` 5,473, `ColumnFamilyStore` 3,341, `StorageProxy` 3,220 | Hard to reason about, hard to test in isolation, constant merge conflicts |
| H2 | Global static singletons | 130 classes with `public static final X instance`; static config in `DatabaseDescriptor` | Prevents running multiple nodes in one classloader (hence the per-instance classloader design of `test/distributed`), hides dependencies, makes init order fragile (`CassandraDaemon.setup`, lines 225-420) |
| H3 | Two metadata control planes during transition | `tcm/` plus `gms/`, with `tcm/compatibility` and `tcm/migration`; `IEndpointSnitch` deprecated "since CEP-21" but still present | Double the surface for topology bugs until gossip-era paths are removed |
| H4 | Paxos v1 remains the default | `config/Config.java:1043` `paxos_variant = PaxosVariant.v1`; both `service/paxos/v1` and v2 are maintained; `StorageProxy.legacyCas` at line 329 | Two LWT implementations to keep correct; users get the slower one unless they opt in |
| H5 | Configuration sprawl | About 450 public fields in `config/Config.java`, 335 entries in `CassandraRelevantProperties`, many `@Replaces` aliases | Operators cannot know which settings interact; every feature adds flags |
| H6 | Ageing dependencies | JUnit 4.12, logback 1.2.12, antlr 3.5.2, airline 0.8, guava 32.0.1, jackson 2.15.3, JVector 1.0.2 (`.build/parent-pom-template.xml`); Java 11 and 17 only (`build.xml:47-48`) | Security exposure and blocked upgrades. Versions were read from the pom template; known-vulnerability status was not checked |

## Medium

| # | Item | Evidence | Why it matters |
|---|---|---|---|
| M1 | Experimental features shipped but disabled | `materialized_views_enabled`, `sasi_indexes_enabled`, `transient_replication_enabled` all default false (`Config.java:605-611`); SASI warns it is not for production (`index/sasi/SASIIndex.java:88`) | Large amounts of code (SASI is a whole index engine) that must compile and pass tests but few users run |
| M2 | Three secondary index engines | `index/internal`, `index/sasi`, `index/sai`; default is still the legacy one (`Config.java:891`) | Triple maintenance; users pick the wrong one by default |
| M3 | Two SSTable formats and two memtables | `io/sstable/format/big` and `bti`; `SkipListMemtable` and `TrieMemtable`; old formats remain the defaults in `conf/cassandra.yaml` while `conf/cassandra_latest.yaml` exists for the new ones | Test matrix growth; best performance is opt-in |
| M4 | JMX as the primary management API | 38 MBean interfaces; `nodetool` depends on them | JMX is awkward to secure and automate; virtual tables only partly replace it |
| M5 | Lock-based concurrency in hot classes | 529 uses of `synchronized` in `src/java`; `StorageService.initServer` and `joinRing` are `synchronized` methods (lines 694, 925) | Contention and deadlock risk are hard to see; counts only, not individually reviewed |
| M6 | Deprecated API surface | 344 `@Deprecated` annotations | Long compatibility tail |
| M7 | Ant build with generated Maven poms | `build.xml`, `.build/*.xml` | Unfamiliar to most Java developers, weak IDE integration, slow incremental builds |
| M8 | Vector search modelled as an index expression | `index/sai/plan/QueryController.java:199` `VSTODO` comment | Blocks general ORDER BY and hybrid ranking |

## Low

| # | Item | Evidence |
|---|---|---|
| L1 | README banner out of date | `README.asc` shows port 9160 and "Cassandra 5.0-SNAPSHOT" while the build is 5.1 |
| L2 | TODO markers | 282 `TODO`/`FIXME`/`XXX` in `src/java` |
| L3 | Stray root file | `CASSANDRA-14092.txt` documents a TTL overflow issue at the repo root |
| L4 | Compact storage remnants | `drop_compact_storage_enabled` (`Config.java:614`) and related code from the Thrift era |
| L5 | Three CI systems | `.circleci/`, `.jenkins/`, `.build/` scripts with partly duplicated logic |

## Top 10 technical improvements, in priority order

1. Sync the fork with upstream trunk (C1) and drop or commit the stray local edit (C2).
2. Upgrade security-relevant dependencies and add Java 21 support (H6). Check upstream first.
3. Make Paxos v2 the default for new clusters and plan removal of v1 (H4).
4. Finish removing gossip-era topology paths now that TCM is authoritative (H3).
5. Break `StorageService` and `StorageProxy` into focused services behind interfaces (H1), starting with code that is already separable (repair entry points, snapshot, read and write coordination).
6. Introduce an injectable context instead of new static singletons; stop adding to `DatabaseDescriptor` (H2).
7. Group and document configuration, and expose effective settings with provenance through the existing `SettingsTable` (H5).
8. Make SAI the default index for new tables and schedule SASI removal (M1, M2).
9. Move operational control from JMX toward CQL virtual tables and a stable command API (M4).
10. Give vector search its own planning abstraction and upgrade JVector (M8).

These are recommendations only. This analysis changed no production code.
