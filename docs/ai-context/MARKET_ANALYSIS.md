# Market Analysis

_Researched 2026-10-08 by web search. Each finding carries the date of its source. Vendor sources are marked; treat their claims as marketing. Nothing here was verified by running the products._

## State of Apache Cassandra itself

| Finding | Source date | Source |
|---|---|---|
| Production line is 5.0.x; 5.0.8 was released 2026-04-16 | July 2026 | [youngju.dev status post](https://www.youngju.dev/blog/2026-07-17-cassandra-6-accord-alpha1-status.en) |
| Cassandra 6.0-alpha1 published March to April 2026; still alpha as of July 2026. No later status was found | July 2026 | same; [Instaclustr (vendor)](https://www.instaclustr.com/blog/apache-cassandra-6-accord-transactions-what-you-need-to-know/) |
| 6.0 bundles Accord general-purpose transactions (CEP-15), Transactional Cluster Metadata, an automated repair scheduler (CEP-37) and a constraints framework (CEP-42) | 2026 | [The New Stack](https://thenewstack.io/apache-cassandra-6-features/), [Ksolves (vendor)](https://www.ksolves.com/blog/big-data/apache-cassandra/whats-new-in-apache-cassandra-6) |
| DB-Engines ranks Cassandra first among wide column stores at about 102.6 points (April 2026); Cosmos DB 22.4, HBase 18.8, ScyllaDB 4.5 | April 2026 | [DB-Engines](https://db-engines.com/en/ranking/wide+column+store) |

**Consequence for this fork:** this checkout is from January 2025 and has TCM but not Accord, auto-repair or constraints. Three of the most obvious gaps are already closed upstream.

## Competitors

| Competitor | Type | Position against Cassandra | Finding and date |
|---|---|---|---|
| **ScyllaDB** | CQL-compatible C++ rewrite | Performance per node, tablets, lower ops burden | Moved to a source-available license from release 2025.1; AGPL 6.2 was the last open source release; free tier capped at 50 vCPU and 10 TB (announced 2024-12-18, [ScyllaDB (vendor)](https://www.scylladb.com/2024/12/18/license/)). 2026.2 adds Cassandra SAI compatibility for LangChain and LlamaIndex, DynamoDB-compatible streams GA, and experimental strongly consistent tables (2026, [ScyllaDB forum (vendor)](https://forum.scylladb.com/t/release-scylladb-2026-2/5426)) |
| **IBM / DataStax (Astra DB, HCD)** | Commercial Cassandra distribution and DBaaS | Managed service, vector and GenAI tooling, enterprise support | IBM closed the acquisition 2025-05-28; Astra DB becomes part of watsonx.data ([DBTA](https://www.dbta.com/Editorial/News-Flashes/IBM-Officially-Closes-Acquisition-of-DataStax-169711.aspx), [IBM community, 2025-10-02](https://community.ibm.com/community/user/blogs/datastax-pm-team-datastax-pm-team/2025/10/02/ibm-datastax)) |
| **Amazon DynamoDB** | Proprietary serverless key-value / wide column | Zero operations, pay per request | Cloud only, different data model ([DB-Engines](https://db-engines.com/en/system/Amazon+DynamoDB%3BCassandra%3BHyperSQL%3BMicrosoft+Azure+Cosmos+DB%3BYugabyteDB)) |
| **Amazon Keyspaces** | CQL-compatible proprietary service | Serverless CQL | Not Apache Cassandra underneath; ignores compaction, compression, caching and gc_grace settings; IAM-only auth (2026, [AxonOps (vendor)](https://axonops.com/blog/managed-cassandra-vs-self-hosted-2026/)) |
| **Azure Cosmos DB (Cassandra API) and Azure Managed Instance for Apache Cassandra** | Proprietary API and managed real Cassandra | Azure-native | Same AxonOps overview, 2026 |
| **YugabyteDB (YCQL) / CockroachDB / TiDB** | Distributed SQL | ACID transactions and SQL with horizontal scale | YugabyteDB is Apache 2.0 with a CQL-rooted API and a PostgreSQL-compatible API ([DB-Engines](https://db-engines.com/en/system/Amazon+DynamoDB%3BCassandra%3BHyperSQL%3BMicrosoft+Azure+Cosmos+DB%3BYugabyteDB)). CockroachDB and TiDB were not researched in this pass |
| **Apache HBase / Google Bigtable** | Wide column on HDFS / managed | Hadoop ecosystem, GCP-native | HBase score slowly declining (DB-Engines, January to April 2026). Bigtable was not found in this search |
| **Dedicated vector databases** (Pinecone, Milvus, Weaviate, pgvector) | Vector search | AI retrieval | Not researched in this pass; listed from general knowledge |

Top direct competitors: **ScyllaDB, DynamoDB, DataStax/IBM Astra**, then the distributed SQL group.

## Recurring user pain (dated)

- Operational burden: repair, compaction and tombstone management must be scheduled and monitored ([Instaclustr (vendor)](https://www.instaclustr.com/blog/top-reasons-apache-cassandra-projects-fail-and-how-to-overcome-them/), undated; [tombstones guide](https://www.instaclustr.com/support/documentation/cassandra/using-cassandra/managing-tombstones-in-cassandra/)).
- Large operators build their own automation for restarts, replacement and decommission ([Uber, 2023](https://www.uber.com/en-US/blog/how-uber-optimized-cassandra-operations-at-scale/)).
- Latency variance and node sprawl ([ScyllaDB white paper, competitor](https://lp.scylladb.com/cassandra-falls-short-wp-offer)).
- Kubernetes operation depends on third-party operators ([K8ssandra docs](https://docs.k8ssandra.io/components/k8ssandra-operator/)).

No 2026 user survey was found; this list leans on vendor material.

## Gaps and how promising they are

Ratings: **strong** (real demand, little competition inside open source Cassandra), **moderate**, **weak**, **saturated** (already solved upstream or by many vendors).

| Gap | Rating | Reasoning |
|---|---|---|
| Diagnosis and self-explanation for operators: why is this query slow, which tombstones or partitions are the cause, what should I change. Built on existing virtual tables, guardrails and diagnostic events | **Strong** | Pain is well documented, the code already exposes the raw signals (`db/virtual`, `db/guardrails`, `diag`), and upstream work has focused on transactions and repair rather than explanation |
| Hybrid search: lexical plus vector ranking, general `ORDER BY` on SAI, newer JVector with quantization | **Strong to moderate** | ScyllaDB is now copying SAI for AI frameworks, so demand is real. Check upstream trunk first because SAI moves quickly |
| Workload-aware guardrails and tenant-level rate limiting and quotas | **Moderate** | Guardrail framework exists; per-tenant fairness does not. Useful for platform teams |
| Open source migration and compatibility tooling (Scylla after its license change, Keyspaces, DynamoDB to Cassandra) | **Moderate** | The Scylla license change created a population looking to move; tooling lives outside the server |
| First-party Kubernetes story | **Moderate to weak** | K8ssandra and cass-operator are established |
| Tiered or object storage for cold SSTables | **Moderate** | Common request, large engineering effort, upstream discussions exist (not verified in this pass) |
| General ACID transactions | **Saturated** | Accord is in 6.0 alpha |
| Automated repair | **Saturated** | CEP-37 is in 6.0 alpha |
| Schema constraints | **Saturated** | CEP-42 is in 6.0 alpha |
| Managed Cassandra hosting | **Saturated** | IBM, Instaclustr, Aiven, Azure, AWS |
| Raw single-node performance race with ScyllaDB | **Weak** | Architectural (JVM, thread-per-core); not winnable by incremental features |

## Recommended direction

1. **Rebase onto current upstream trunk first.** Building on a January 2025 snapshot guarantees conflict and duplication.
2. Target **operator-facing explainability and safety**: a query and table "doctor" exposed through virtual tables and `nodetool`, with new guardrails driven by what it finds. It is additive, uses stable extension points, is testable without large clusters, and fits the Apache contribution process (small CEP or Jira-sized pieces).
3. As a second track, **SAI and vector quality** (hybrid ranking, library upgrade), after confirming what upstream has already shipped.
