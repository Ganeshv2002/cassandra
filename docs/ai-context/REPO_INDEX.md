# Repository Index

_Curated 2026-10-08 for commit `33feca3d84`. Lightweight on purpose: entry points and directories, not every file. Excludes generated sources (`src/gen-java`), dependencies (`lib/`, `~/.m2`), build output (`build/`), IDE files and test data fixtures (`test/data`)._

`S/` means `src/java/org/apache/cassandra/`.

## I want to understand...

| Topic | Open these |
|---|---|
| How a node starts | `S/service/CassandraDaemon.java`, `S/service/StartupChecks.java`, `S/tcm/Startup.java`, `S/service/StorageService.java` (`initServer`) |
| How a CQL request arrives | `S/transport/CQLMessageHandler.java`, `S/transport/Dispatcher.java`, `S/transport/messages/` |
| How CQL is parsed and executed | `src/antlr/Parser.g`, `S/cql3/QueryProcessor.java`, `S/cql3/statements/` |
| Writes | `S/cql3/statements/ModificationStatement.java`, `S/service/StorageProxy.java` (`mutate*`, `performWrite`, `sendToHintedReplicas`), `S/db/MutationVerbHandler.java`, `S/db/Keyspace.java` (`applyInternal`), `S/db/commitlog/CommitLog.java`, `S/db/ColumnFamilyStore.java` (`apply`) |
| Reads | `S/cql3/statements/SelectStatement.java`, `S/service/StorageProxy.java` (`read`, `fetchRows`), `S/service/reads/`, `S/db/ReadCommand.java`, `S/db/SinglePartitionReadCommand.java`, `S/db/PartitionRangeReadCommand.java` |
| Replica selection | `S/locator/ReplicaPlans.java`, `S/locator/NetworkTopologyStrategy.java`, `S/tcm/ownership/` |
| Internode messaging | `S/net/MessagingService.java`, `S/net/Verb.java` (all message types and their handlers), `S/net/OutboundConnection.java`, `S/net/InboundMessageHandler.java` |
| Cluster metadata | `S/tcm/TCM_implementation.md`, `S/tcm/ClusterMetadata.java`, `S/tcm/ClusterMetadataService.java`, `S/tcm/transformations/`, `S/tcm/sequences/`, `S/tcm/log/LocalLog.java` |
| Gossip and failure detection | `S/gms/Gossiper.java`, `S/gms/FailureDetector.java` |
| Memtables and flush | `S/db/memtable/Memtable_API.md`, `S/db/memtable/TrieMemtable.java`, `S/db/memtable/Flushing.java` |
| SSTables | `S/io/sstable/format/SSTableReader.java`, `S/io/sstable/format/big/`, `S/io/sstable/format/bti/`, `S/io/sstable/Descriptor.java` |
| Compaction | `S/db/compaction/CompactionManager.java`, `S/db/compaction/CompactionStrategyManager.java`, `S/db/compaction/UnifiedCompactionStrategy.md` |
| Lightweight transactions | `S/service/paxos/Paxos.md`, `S/service/paxos/Paxos.java`, `S/service/paxos/v1/` |
| Indexes and vector search | `S/index/Index.java`, `S/index/SecondaryIndexManager.java`, `S/index/sai/README.md`, `S/index/sai/plan/`, `S/index/sai/disk/v1/vector/` |
| Repair | `S/repair/RepairCoordinator.java`, `S/repair/RepairSession.java`, `S/repair/Validator.java`, `S/repair/consistent/`, `S/utils/MerkleTree.java` |
| Streaming | `S/streaming/StreamSession.java`, `S/streaming/StreamPlan.java`, `S/db/streaming/` |
| Hints and batches | `S/hints/HintsService.java`, `S/batchlog/BatchlogManager.java` |
| Schema model | `S/schema/TableMetadata.java`, `S/schema/KeyspaceMetadata.java`, `S/schema/Schema.java`, `S/cql3/statements/schema/` |
| Types | `S/db/marshal/AbstractType.java`, `S/db/marshal/VectorType.java`, `S/cql3/CQL3Type.java` |
| Security | `S/auth/`, `S/security/`, `S/audit/`, `S/cql3/functions/masking/` |
| Configuration | `conf/cassandra.yaml`, `S/config/Config.java`, `S/config/DatabaseDescriptor.java`, `S/config/CassandraRelevantProperties.java` |
| Guardrails | `S/db/guardrails/Guardrails.java`, `S/db/guardrails/GuardrailsConfig.java` |
| Observability | `S/db/virtual/SystemViewsKeyspace.java`, `S/metrics/`, `S/tracing/`, `S/diag/` |
| Operator tools | `S/tools/NodeTool.java`, `S/tools/NodeProbe.java`, `S/tools/nodetool/`, `bin/` |
| Thread pools | `S/concurrent/Stage.java`, `S/concurrent/SharedExecutorPool.java` |

## Tests

| Kind | Location | Run |
|---|---|---|
| Unit | `test/unit/org/apache/cassandra/` (mirrors `S/`) | `ant test -Dtest.name=<Class>` |
| CQL-level unit | `test/unit/org/apache/cassandra/cql3/` (base class `CQLTester`) | same |
| In-JVM distributed | `test/distributed/org/apache/cassandra/distributed/test/` | `ant test-jvm-dtest-some -Dtest.name=<Class>` |
| Simulator | `test/simulator/` | `ant test-simulator-dtest` |
| Fuzz (Harry) | `test/harry/` | see `ci/harry_simulation.sh` |
| Benchmarks | `test/microbench/` | `ant microbench -Dbenchmark.name=<Class>` |

Test commands are taken from `build.xml` target names and `TESTING.md`; they were not executed during this analysis.

## Build and process

| Need | File |
|---|---|
| Build targets | `build.xml`, `.build/*.xml` |
| Dependency versions | `.build/parent-pom-template.xml` |
| Style | `.build/checkstyle.xml` |
| Contribution rules | `CONTRIBUTING.md`, `.github/pull_request_template.md` |
| Release notes | `CHANGES.txt`, `NEWS.txt` |
| CI | `.circleci/config_template.yml`, `.jenkins/` |

## The other AI context documents

`PROJECT_OVERVIEW.md`, `ARCHITECTURE.md`, `CODEBASE_MAP.md`, `DATA_FLOWS.md`, `AI_ARCHITECTURE.md`, `MARKET_ANALYSIS.md`, `TECH_DEBT.md`, `DECISIONS.md`, `WALKTHROUGH.md`, `UPSTREAM_DELTA.md`.
