# Codebase Map

_Analysed 2026-10-08 against commit `33feca3d84`. File counts are `.java` files under each directory._

## Top level

| Path | What it is |
|---|---|
| `src/java/org/apache/cassandra/` | Server code (2,706 files) |
| `src/antlr/` | CQL grammar: `Cql.g`, `Lexer.g`, `Parser.g` |
| `src/resources/` | Bundled resources |
| `test/unit` | JUnit 4 unit tests (1,518 files) |
| `test/distributed` | In-JVM multi-node tests (466) |
| `test/simulator` | Deterministic simulation framework and tests (139) |
| `test/harry` | Model-based fuzz testing (117) |
| `test/microbench` | JMH benchmarks (57) |
| `test/burn`, `test/long`, `test/memory` | Stress, long-running and memory tests |
| `test/conf`, `test/data`, `test/resources` | Test configuration and legacy SSTable fixtures |
| `tools/stress`, `tools/fqltool`, `tools/bin` | `cassandra-stress`, full query log tool, tool launchers |
| `bin/` | `cassandra`, `nodetool`, `cqlsh`, sstable tool launchers |
| `conf/` | `cassandra.yaml`, `cassandra_latest.yaml`, JVM options, logback |
| `pylib/cqlshlib` | `cqlsh` implementation (Python) |
| `doc/` | Antora docs, native protocol specs v3 to v5 |
| `.build/` | Ant sub-builds, checkstyle, OWASP, docker and CI scripts |
| `.circleci/`, `.jenkins/`, `ci/` | CI definitions |
| `debian/`, `redhat/` | Packaging |
| `ide/` | IDE project templates |
| `examples/` | Trigger and SSTable examples |
| `docs/ai-context/` | These AI context documents (not part of upstream) |

## Server packages (`src/java/org/apache/cassandra/`)

| Package | Files | Role | Start reading at |
|---|---|---|---|
| `db` | 471 | Storage engine: keyspaces, tables, commit log, memtables, compaction, rows, filters, virtual tables, guardrails | `Keyspace`, `ColumnFamilyStore`, `ReadCommand`, `Mutation` |
| `index` | 260 | Secondary indexes: `internal` (2i), `sasi`, `sai` | `Index`, `SecondaryIndexManager`, `sai/StorageAttachedIndex` |
| `cql3` | 243 | CQL statements, restrictions, selection, functions, types | `QueryProcessor`, `statements/SelectStatement`, `statements/ModificationStatement` |
| `io` | 213 | SSTable formats, file utilities, compression | `sstable/format/SSTableReader`, `sstable/format/big`, `sstable/format/bti` |
| `utils` | 212 | BTree, bloom filters, Merkle trees, concurrency, memory | `btree/BTree`, `MerkleTree`, `concurrent/OpOrder` |
| `tools` | 194 | `nodetool` commands and offline sstable tools | `NodeTool`, `NodeProbe` |
| `service` | 148 | Daemon, coordinator, node lifecycle, Paxos, read executors, pagers | `CassandraDaemon`, `StorageProxy`, `StorageService` |
| `tcm` | 122 | Transactional Cluster Metadata | `ClusterMetadata`, `ClusterMetadataService`, `TCM_implementation.md` |
| `repair` | 88 | Anti-entropy repair | `RepairCoordinator`, `RepairSession`, `Validator` |
| `net` | 75 | Internode messaging | `MessagingService`, `Verb`, `OutboundConnection` |
| `locator` | 71 | Replication strategies, replica plans, snitches | `ReplicaPlans`, `NetworkTopologyStrategy` |
| `auth` | 68 | Authentication, roles, permissions, mTLS, CIDR | `PasswordAuthenticator`, `CassandraRoleManager` |
| `streaming` | 62 | Bulk SSTable transfer | `StreamSession`, `StreamPlan` |
| `metrics` | 54 | Metric definitions | `TableMetrics`, `ClientRequestMetrics` |
| `transport` | 47 | Native protocol server | `Dispatcher`, `CQLMessageHandler`, `messages/` |
| `schema` | 47 | Schema model and distributed schema | `TableMetadata`, `Schema` |
| `concurrent` | 37 | Stages and executors | `Stage`, `SEPExecutor` |
| `dht` | 35 | Tokens, partitioners, ranges | `Murmur3Partitioner`, `Range` |
| `hints` | 31 | Hinted handoff | `HintsService` |
| `config` | 31 | Configuration | `Config`, `DatabaseDescriptor`, `CassandraRelevantProperties` |
| `serializers`, `db/marshal` | 31 + | CQL type serialization, including `VectorType` | `db/marshal/AbstractType` |
| `gms` | 27 | Gossip and failure detection | `Gossiper`, `FailureDetector` |
| `cache` | 20 | Key, row, counter and chunk caches | `CacheService` (in `service`) |
| `security` | 18 | TLS, encryption contexts | `SSLFactory` |
| `audit`, `fql`, `diag`, `tracing` | 15, 3, 8, 6 | Audit log, full query log, diagnostic events, tracing | `AuditLogManager` |
| `batchlog`, `triggers`, `notifications` | 5, 4, 14 | Logged batches, triggers, SSTable lifecycle events | `BatchlogManager` |

## Design documents that live next to the code

- `tcm/TCM_implementation.md`, `tcm/TransactionalClusterMetadata.md`
- `db/compaction/UnifiedCompactionStrategy.md`
- `db/memtable/Memtable_API.md`
- `service/paxos/Paxos.md`
- `index/sai/README.md`
- `doc/SASI.md`

## Hot spots (large, central, risky to change)

| File | Lines |
|---|---|
| `config/DatabaseDescriptor.java` | 5,527 |
| `service/StorageService.java` | 5,473 |
| `utils/btree/BTree.java` | 4,220 |
| `db/ColumnFamilyStore.java` | 3,341 |
| `service/StorageProxy.java` | 3,220 |
| `cql3/functions/types/TypeCodec.java` | 3,173 |

## Conventions

- Commits follow `<summary>\n\npatch by X; reviewed by Y for CASSANDRA-NNNNN` (`.github/pull_request_template.md`). Each change adds a line to `CHANGES.txt`; operator-visible changes go in `NEWS.txt`.
- Checkstyle rules in `.build/checkstyle.xml`. Run `ant check` before committing.
- Build with `ant jar`; run one test with `ant test -Dtest.name=ClassName`. See `TESTING.md` and `CONTRIBUTING.md`.
