# Cassandra Walkthrough

_Written 2026-10-08 for commit `33feca3d84` (trunk of January 2025, version 5.1-SNAPSHOT). Each section has two parts: **In plain language**, for someone who has never run a database cluster, and **For engineers**, with the classes that do the work. Paths are relative to `src/java/org/apache/cassandra/`. Exact line references are in `DATA_FLOWS.md`._

## Contents

1. What Cassandra is
2. How data is spread across machines
3. How the cluster agrees on who owns what
4. How a node starts
5. How a request gets in
6. Writing data
7. Storing data on disk
8. Reading data
9. Deleting data
10. Keeping copies in sync
11. Transactions
12. Indexes and vector search
13. Adding and removing nodes
14. Operating and observing a cluster
15. How the project tests itself
16. Quality assessment
17. Main risks
18. Where to go from here

---

## 1. What Cassandra is

**In plain language.** Imagine a filing system spread over many offices in several cities. Every office can take a request to file or fetch a document. Each document is copied to a few offices, so if one office burns down, or a whole city loses power, the others keep working. Cassandra is that filing system for application data. Its speciality is never refusing a write and growing by simply adding more machines.

**For engineers.** A partitioned row store with a masterless data plane. Tables have a primary key made of a partition key (decides placement) and clustering columns (sort order inside the partition). Clients use CQL over the native binary protocol. Consistency is chosen per request. The server is one JVM per node, about 569k lines of Java in 2,706 files.

## 2. How data is spread across machines

**In plain language.** Every row has a key. Cassandra runs the key through a scrambling function that produces a number. The whole range of possible numbers is drawn as a circle, and each machine is responsible for some arcs of it. The row lives on the machine whose arc contains its number, plus the next few machines, so there are several copies.

**For engineers.** `dht/Murmur3Partitioner` hashes partition keys to tokens. Replication strategies in `locator/` (`NetworkTopologyStrategy`, `SimpleStrategy`) turn token ownership into replica sets per datacenter. In this version the computed placements are stored inside `tcm/ClusterMetadata` (`tcm/ownership/`), and coordinators build a `ReplicaPlan` for each operation through `locator/ReplicaPlans`.

## 3. How the cluster agrees on who owns what

**In plain language.** Older versions let machines spread news by chatting with random neighbours, like office gossip. That is resilient but people can briefly disagree, which is dangerous when the news is "I now own this data". This version keeps a single official logbook of changes. A small committee of machines signs off each new entry, and every machine reads the logbook in order, so everyone reaches the same picture in the same sequence. Gossip is still used, but only to notice who is alive.

**For engineers.** This is Transactional Cluster Metadata (CEP-21) in `tcm/`. State is an immutable `ClusterMetadata` versioned by `Epoch`. A change is a `Transformation` (`tcm/transformations/`: `AlterSchema`, `Register`, `PrepareJoin`, `PrepareLeave`, `PrepareMove`, `PrepareReplace`...). `ClusterMetadataService.commit` hands it to a `Processor`: CMS members use `PaxosBackedProcessor` to append to the distributed log table, other nodes use `RemoteProcessor` to forward. `tcm/log/LocalLog` applies entries in order and notifies listeners. Long operations are `MultiStepOperation`s in `tcm/sequences/` that survive restarts. `gms/Gossiper` runs every second for the failure detector and for states like load. The in-tree `tcm/TCM_implementation.md` is the best design reference.

## 4. How a node starts

**In plain language.** On start-up a machine reads its settings, checks the environment is sane, catches up on the official logbook, replays any notes it had not finished filing before it was switched off, announces itself, and only then opens the door to customers.

**For engineers.** `service/CassandraDaemon`: `activate` then `applyConfig` (`config/DatabaseDescriptor` loads `conf/cassandra.yaml`), `setup` runs `StartupChecks`, `tcm/Startup.initialize`, `CommitLog.instance.recoverSegmentsOnDisk`, `StorageService.instance.initServer` (gossip, messaging, join sequence), schedules background tasks, then `start` opens the native transport via `NativeTransportService`.

## 5. How a request gets in

**In plain language.** A program connects and sends a sentence such as "store this" or "give me that". The front desk checks it is not overloaded, hands the sentence to a worker, the worker works out what the sentence means, checks the caller is allowed, and carries it out.

**For engineers.** Netty pipeline in `transport/`. For protocol v5, `CQLMessageHandler` decodes frames and applies byte and rate limits (`ClientResourceLimits`), then `Dispatcher.dispatch` queues work on the request executor. `transport/messages/QueryMessage`, `ExecuteMessage` and `BatchMessage` call `cql3/QueryProcessor`. Text is parsed by the ANTLR grammar in `src/antlr/`; prepared statements are cached with Caffeine. The result is a `CQLStatement` whose `authorize`, `validate` and `execute` methods run in turn.

## 6. Writing data

**In plain language.** The machine that received the request (the coordinator) works out which machines hold copies, sends the write to all of them at once, and replies to the caller as soon as enough of them confirm. "Enough" is chosen by the caller: one, a majority, or all. If a machine is down, the coordinator keeps a note and delivers it when the machine comes back. Each machine that receives the write first jots it in a journal on disk, so it cannot be lost in a crash, then puts it in memory.

**For engineers.** `ModificationStatement.execute` builds `Mutation`s and calls `StorageProxy.mutateWithTriggers`. Logged batches go through `mutateAtomically` and the batchlog; tables with materialized views go through `mutateMV`; the rest through `mutate` and `performWrite`. `sendToHintedReplicas` sends `MUTATION_REQ` via `net/MessagingService`, applies locally on the `MUTATION` stage, and writes hints (`hints/HintsService`) for down replicas. A response handler (`AbstractWriteResponseHandler` and subclasses) blocks until the consistency level is met. On each replica `MutationVerbHandler` calls `Keyspace.applyInternal`, which appends to `db/commitlog/CommitLog` and then applies to the memtable and indexes through `ColumnFamilyStore.apply`, all under the `Keyspace.writeOrder` `OpOrder` so flushes know which writes are in flight.

## 7. Storing data on disk

**In plain language.** Memory fills up, so every so often the machine writes the contents of memory to a new file, sorted by key, and never edits that file again. Over time there are many such files, so a background job merges them into fewer, bigger ones and throws away superseded versions. Never editing files in place is what makes writes fast.

**For engineers.** A log-structured merge tree. Memtables (`db/memtable/TrieMemtable`, `SkipListMemtable`) are flushed by `ColumnFamilyStore.switchMemtable` and its inner `Flush` class to SSTables. Two formats exist: `io/sstable/format/big` (partition index plus summary) and `bti` (trie index). Compaction (`db/compaction/`) offers size-tiered, leveled, time-window and unified strategies; `CompactionManager` runs tasks and `CompactionIterator` merges and purges. Everything streams through the iterator model in `db/rows`, `db/partitions` and `db/transform`. `conf/cassandra_latest.yaml` turns on the newer choices (trie memtable, `bti`, UCS, SAI as default index); `conf/cassandra.yaml` keeps the old defaults.

## 8. Reading data

**In plain language.** The coordinator asks the fastest copy holder for the full answer and asks others for just a fingerprint of their answer. If the fingerprints match, it replies. If not, it fetches the full data from everyone, keeps the newest version of each value, replies with that, and sends the correction to whoever was out of date. On each machine, the answer has to be assembled from memory plus possibly several files; small lookup aids tell it which files can be skipped.

**For engineers.** `SelectStatement.execute` builds a `ReadQuery`. `StorageProxy.read` then `fetchRows` create an `AbstractReadExecutor` per partition (never, percentile or always speculating, chosen from the table's `speculative_retry`). `DigestResolver` compares digests; on mismatch `service/reads/repair/BlockingReadRepair` reconciles. Locally, `ReadCommand.executeLocally` either uses an index searcher or `queryStorage`. `SinglePartitionReadCommand.queryMemtableAndDisk` merges memtable and SSTable iterators, skipping SSTables by bloom filter, min/max clustering and timestamps, with a timestamp-ordered fast path for named rows. Range reads go through `service/reads/range/`. Short-read and replica-filtering protection classes in `service/reads/` handle correctness corner cases. Paging lives in `service/pager/`.

## 9. Deleting data

**In plain language.** Because files are never edited, a delete is recorded as a new note saying "this is gone". The note has to be kept long enough for every copy holder to hear about it; otherwise a machine that missed it could bring the deleted data back. Too many of these notes slow reads down, which is one of the best-known ways to get into trouble with Cassandra.

**For engineers.** Tombstones: cell, row, range (`db/RangeTombstone`, `db/DeletionTime`) and partition level. They are purgeable only after `gc_grace_seconds` and only when compaction can prove no older data lives in other SSTables (`db/compaction/CompactionController`). `ReadCommand.withoutPurgeableTombstones` drops them at read time. Guardrails and thresholds warn or fail queries that scan too many.

## 10. Keeping copies in sync

**In plain language.** Three safety nets: the notes kept for machines that were down, the corrections made while reading, and a periodic full comparison where machines summarise their data, compare summaries, and exchange only what differs.

**For engineers.** Hinted handoff (`hints/`), read repair (`service/reads/repair/`), and anti-entropy repair (`repair/`): `RepairCoordinator` creates `RepairSession`s and `RepairJob`s; replicas build Merkle trees (`repair/Validator`, `utils/MerkleTree`); differences are streamed with `streaming/`. Incremental repair tracks repaired SSTables through `repair/consistent/`. In this tree repair must be started by the operator; the automated scheduler (CEP-37) arrived upstream later.

## 11. Transactions

**In plain language.** Ordinary writes do not check anything first. For "only do this if that is still true", the copy holders run a voting procedure so that exactly one competing request wins. It is slower, and in this version it only works within a single group of rows.

**For engineers.** Lightweight transactions via Paxos. `StorageProxy.cas` dispatches to `service/paxos/Paxos` (v2) or `legacyCas` (v1); default is v1 (`config/Config.java:1043`). Paxos state lives in system tables with its own repair (`service/paxos/cleanup`, `uncommitted`). TCM itself uses Paxos for the metadata log. Multi-partition ACID transactions (Accord) are not in this tree.

## 12. Indexes and vector search

**In plain language.** Normally you can only look things up by their key. An index is like the index at the back of a book: it lets you find rows by other values. The newest kind can also find "things similar to this one", which is what AI applications need to look up relevant passages.

**For engineers.** Three engines behind `index/Index`: legacy table-backed 2i (`index/internal`), SASI (experimental, disabled by default) and SAI (`index/sai`). SAI attaches per-SSTable index components and memtable indexes, supports numeric ranges, text analyzers, collections and `vector<float, n>` with ANN ordering through JVector. See `AI_ARCHITECTURE.md`.

## 13. Adding and removing nodes

**In plain language.** A new machine registers in the logbook, is told which arcs of the circle it will take, copies that data from current owners while they keep serving, and then officially takes over. Leaving is the reverse.

**For engineers.** `tcm/sequences/BootstrapAndJoin`, `BootstrapAndReplace`, `Move`, `UnbootstrapAndLeave`, each a sequence of committed steps (prepare, start, mid, finish) with `LockedRanges` preventing conflicting concurrent operations and `ProgressBarrier` waiting for enough nodes to acknowledge an epoch. Data moves through `streaming/StreamSession`, including zero-copy whole-file transfer in `db/streaming/`.

## 14. Operating and observing a cluster

**In plain language.** Administrators have a command-line tool to ask a machine how it is doing and tell it to do maintenance. They can also query special read-only tables that show internal status. Safety limits can warn or block clearly dangerous usage.

**For engineers.** `tools/NodeTool` talks JMX to 38 MBean interfaces through `tools/NodeProbe`. Virtual tables in `db/virtual/` expose settings, clients, caches, streaming, repairs, snapshots, queries, gossip, TCM log and metrics through CQL. Metrics are Dropwizard (`metrics/`). `db/guardrails/` holds configurable limits. Audit logging (`audit/`) and full query logging (`fql/`) use Chronicle Queue. `tracing/` records per-request traces.

## 15. How the project tests itself

**In plain language.** Besides ordinary tests, the project can run a whole pretend cluster inside one program, and can replay the same simulated chaos (delays, crashes, reordered messages) exactly, so a rare bug can be reproduced on demand.

**For engineers.** `test/unit` (JUnit 4, `CQLTester` base class), `test/distributed` (in-JVM dtests with one classloader per node, needed because of the static singletons), `test/simulator` (deterministic simulation with bytecode weaving), `test/harry` (model-based fuzzing), `test/microbench` (JMH). Python dtests live in a separate repository.

## 16. Quality assessment

This is a judgement from reading code, not from running it.

**Strengths**
- Mature core algorithms with unusually strong correctness tooling (simulator, Harry, in-JVM dtests).
- Clear extension interfaces for storage, indexes, compaction, auth.
- Design documents kept next to the code for the newest subsystems (TCM, UCS, Paxos v2, memtable API, SAI).
- Careful resource handling: explicit backpressure on client and internode paths, reference counting, off-heap buffer pools.
- Disciplined process: every change tied to a Jira ticket and recorded in `CHANGES.txt`.

**Weaknesses**
- A handful of very large central classes and pervasive static singletons.
- Old and new implementations coexist in many areas (gossip and TCM, Paxos v1 and v2, three index engines, two SSTable formats, two memtables), and defaults favour the old ones.
- Very large configuration surface.
- Dated build and test stack (Ant, JUnit 4).

**Overall:** high-quality, battle-tested distributed systems code with heavy accumulated legacy. Safe to extend at its plug-in points; expensive to change at its core.

## 17. Main risks

1. **Stale fork.** This checkout is 21 months behind upstream; see `TECH_DEBT.md` C1.
2. **Change risk in core classes.** `StorageService`, `StorageProxy`, `ColumnFamilyStore`, `DatabaseDescriptor`.
3. **Transition complexity.** Mixed gossip and TCM paths, mixed Paxos versions.
4. **Operational complexity for users.** Repair, compaction and tombstones remain the main reasons deployments fail.
5. **Dependency age.**
6. **Competitive pressure.** Managed and serverless alternatives remove operational work entirely; ScyllaDB competes on performance and is now also courting AI frameworks.

## 18. Where to go from here

Read `MARKET_ANALYSIS.md` for the gap ratings and recommended direction, and `TECH_DEBT.md` for the prioritised improvement list. Before writing any feature code, bring the fork up to date with upstream trunk and re-check each gap there.
