# Decisions

_Written 2026-10-08. Part A records architectural decisions observable in the code at commit `33feca3d84`; the rationale is inferred from code and in-tree design docs unless a document is cited. Part B records decisions made while producing these AI context documents._

## Part A: decisions embodied in the code

| # | Decision | Where it shows | Consequence |
|---|---|---|---|
| A1 | Masterless peer-to-peer data plane with tunable consistency | `service/StorageProxy.java`, `db/ConsistencyLevel.java`, `locator/ReplicaPlans.java` | Any node coordinates; availability over strict consistency by default |
| A2 | Log-structured merge storage | `db/commitlog`, `db/memtable`, `io/sstable`, `db/compaction` | Fast writes; reads and space depend on compaction; deletes become tombstones |
| A3 | Last-write-wins by cell timestamp | `db/rows/Cells.java`, `db/LivenessInfo.java` | No read-before-write; clock skew can lose updates |
| A4 | Linearizable metadata through TCM, replacing gossip for schema and ownership (CEP-21) | `tcm/`, `tcm/TransactionalClusterMetadata.md` | Safe concurrent topology and schema changes; a CMS quorum is needed to change metadata |
| A5 | Gossip kept for liveness only | `gms/Gossiper.java`, `gms/FailureDetector.java` | Failure detection stays decentralised |
| A6 | Lightweight transactions on Paxos, v2 opt-in | `service/paxos/Paxos.md`, `config/Config.java:1043` | Single-partition linearizability only in this tree |
| A7 | Pluggable memtable, SSTable format, index and compaction | interfaces listed in `ARCHITECTURE.md` | New engines can ship next to old ones; defaults lag |
| A8 | SAI as the strategic index, with vectors inside it | `index/sai/`, JVector dependency | One index framework for scalar and vector search |
| A9 | Staged thread pools plus Netty event loops | `concurrent/Stage.java`, `transport/Dispatcher.java` | Simple isolation between work types; not thread-per-core |
| A10 | Process-wide singletons and static configuration | 130 `instance` fields, `config/DatabaseDescriptor.java` | Simple access everywhere; hard to test and embed |
| A11 | Guardrails and feature flags instead of removing risky features | `db/guardrails/`, `*_enabled` flags in `config/Config.java` | Operators opt in; dead-by-default code remains |
| A12 | Correctness through simulation and model-based fuzzing | `test/simulator`, `test/harry`, `test/distributed` | Heavy test infrastructure in the same repo |
| A13 | Ant build, JUnit 4, Jira-driven process with `CHANGES.txt` | `build.xml`, `.github/pull_request_template.md` | Stable but dated contributor experience |
| A14 | JMX first for management, virtual tables growing | `*MBean.java`, `db/virtual/` | Two management surfaces |

## Part B: decisions made for this documentation

| # | Decision | Reason |
|---|---|---|
| B1 | Documents live in `docs/ai-context/` | Requested location. The upstream Antora docs live in `doc/` (singular) and were not touched |
| B2 | No production code changed, no refactoring | Requested. The pre-existing uncommitted edit to `CassandraDaemon.java` was left as found and not committed |
| B3 | `REPO_INDEX.md` is a hand-curated Markdown index | No index existed. A generated index of 5,000+ files would go stale and add noise; the curated one skips generated files, dependencies (`lib/`) and build output (`build/`) |
| B4 | Line numbers are cited but marked as tied to commit `33feca3d84` | They make flows checkable now and will drift |
| B5 | Market findings carry source dates and vendor flags | Most available material is vendor-written |
| B6 | Nothing was built or run | Analysis was static: reading code, grep counts and web search. Performance and correctness claims are therefore from code reading only |
| B7 | The repository was not pulled or rebased before analysis | The analysis describes the checkout as found; the staleness is recorded as debt item C1 |
