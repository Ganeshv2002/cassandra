# Project Overview

_Analysed 2026-10-08 against commit `33feca3d84` (trunk, committed 2025-01-15, version `5.1-SNAPSHOT` per `build.xml:base.version`). Code is the source of truth; every claim below was checked in the tree unless marked "inferred"._

## What this is

Apache Cassandra is a distributed, masterless, partitioned row store. Clients speak CQL (a SQL-like language) over a binary protocol. Every node can accept any request; data is spread across nodes by hashing the partition key onto a token ring and replicated to several nodes per datacenter. Consistency is tunable per request (ONE, QUORUM, ALL, SERIAL and the local/each variants in `db/ConsistencyLevel.java`).

This repository is a personal fork (`github.com/Ganeshv2002/cassandra`) of `apache/cassandra` trunk.

## Important caveat: the snapshot is old

The checkout is trunk as of **January 2025**. Upstream has since moved a long way: Cassandra 6.0-alpha1 was published in spring 2026 and includes Accord transactions (CEP-15), an automated repair scheduler (CEP-37) and a constraints framework (CEP-42). None of those are in this tree (verified: no `service/accord`, `repair/autorepair` or `cql3/constraints` packages). Any "gap" analysis must be read against upstream before building anything, or the work will duplicate what already exists. See `MARKET_ANALYSIS.md`.

## Size and shape

| Measure | Value |
|---|---|
| Production Java files (`src/java`) | 2,706 files, about 569k lines |
| Test Java files (`test/`) | 2,342 (unit 1,518; distributed 466; simulator 139; harry 117; microbench 57) |
| Largest packages | `db` 471, `index` 260, `cql3` 243, `io` 213, `utils` 212, `tools` 194, `service` 148, `tcm` 122 |
| Build | Apache Ant (`build.xml`), dependencies resolved through Maven resolver tasks in `.build/` |
| Java | 11 (default) and 17 (`build.xml:47-48`) |
| Client tooling | `cqlsh` in Python (`pylib/cqlshlib`), `nodetool` and sstable tools in Java (`tools/`) |
| Docs | Antora/AsciiDoc in `doc/` |

## Major capabilities present in this tree

- CQL native protocol v3 to v5 server on Netty (`transport/`).
- LSM storage engine: commit log, memtables (skip list and trie), SSTables in two formats, `big` and `bti` (`io/sstable/format/`).
- Compaction strategies: size-tiered, leveled, time-window and unified (UCS) (`db/compaction/`).
- Transactional Cluster Metadata (TCM, CEP-21): schema and topology changes go through a linearizable log owned by a Cluster Metadata Service (`tcm/`). Gossip remains for liveness and state dissemination (`gms/`).
- Lightweight transactions on Paxos, v1 and v2 (`service/paxos/`); default is still `v1` (`config/Config.java:1043`).
- Secondary indexes: legacy 2i (`index/internal`), SASI (experimental, off by default), and Storage Attached Indexes, SAI (`index/sai`), including vector ANN search.
- Anti-entropy: hinted handoff (`hints/`), read repair (`service/reads/repair/`), full, incremental and preview repair (`repair/`).
- Streaming for bootstrap, rebuild, repair and decommission (`streaming/`), including zero-copy whole-SSTable streaming.
- Security: pluggable authN/authZ, mTLS identities, CIDR filtering, audit log, dynamic data masking, password validation (CEP-24).
- Guardrails (`db/guardrails/`) and virtual tables for observability (`db/virtual/`).

## Who uses it and why

Teams that need always-on writes across datacenters at very large scale with predictable linear scaling and no single point of failure, and that can model data around known query patterns. Typical uses: time series, messaging, user and device state, feature stores, and increasingly vector retrieval for AI applications.

## Where to read next

- `ARCHITECTURE.md` for layers and diagrams.
- `DATA_FLOWS.md` for traced request paths with file and line references.
- `CODEBASE_MAP.md` and `REPO_INDEX.md` for navigation.
- `WALKTHROUGH.md` for the long explanation in engineering and plain language.
