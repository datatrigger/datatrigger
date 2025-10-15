---
title: "Designing Data-Intensive Applications: reading notes 2/3"
summary: "Reading through this backend/data engineering classic. Part 2: Distributed Data"
date: 2023-09-03
tags: ["backend", "databases", "system design", "distributed computing", "replication", "partitioning"]
draft: false
---

![Designing Data-Intensive Applications, by Martin Kleppmann](/res/designing_data_applications/designing-data-intensive-applications-martin-kleppmann.jpeg)

# Part 2 - Distributed Data

Benefits of distributed architectures:
* Scalability
* Fault-tolerance/availability
* Latency

Distributed data architectures may involve:
* Replication
* Partitioning a.k.a. Sharding

# Chapter 5: Replication

There are multiple reasons to replicate data, including: scalability, reliability/availability and latency.

The main 3 algorithms to implement replication are:
* Single-leader replication
* Multi-leader replication
* Leaderless replication

## Single-leader replication

When only one node - the leader - processes writes. The replicas maintain a copy of the leader's data to serve read requests only.

### Synchronous vs asynchronous replication

100% synchronous replication is not really used. Even if it guarantees consistency with the leader's data across replicas, the constraints are too restrictive: it takes one replica down to fail the entire system.

Instead, *semi-synchronous* replication is implemented: only 1 replica is synchronous, not the other ones. If the synchronous replica fails, another one replaces it.

Fully asynchronous replication is widespread, even if there is a risk of data loss.

### Failures

Replicated databases make an extensive use *log* files to handle failures gracefully.  
Followers do **catch-up recovery**. They make a diff between their own log and the leader's log to fetch the latest data changes since failure.  
When a leader fails, the process of promoting a follower to replace the lost leader is called  **failover**. Failover is much more complicated for a bunch of reasons:
* A follower may have data that the new leader does not have. This data may be discarded, but systems may have read it in the meantime.
* *Split brain*: two nodes think they are the sole leader
* Deciding when the leader is actually down is difficult. What is an appropriate timeout?

### Replicating logs

**Statement-based replication** is not widely used because it can cause a lot of problems, e.g. nondeterministic functions like `NOW()`, out of sync sequences or side effects...  Instead, **physical-log-based replicaton** uses the actual log of the storage engine: the WAL for B-tree based engines, or SSTables for LSM-tree bases engines. In this case the leader "writes twice" its log: first locally, then sending it over the network to the followers. A big downside: as a low-level object, the WAL is tightly coupled to the storage engine. As a consequence, database version mismatches between nodes are risky. This prevents zero-downtime rolling upgrades. As a middle ground, higher-level **logical log** objects are often used to avoid coupling replication and storage engines.

### Replication lag

Leader-based replication scales when data access is mostly reads, not writes: just add followers. Realistically, replication is asynchronous (see above) and brings about **eventual consistency** and **replication lag**. Some examples:

#### Read-your-writes consistency

The problem: first, the user submits data (to the leader, under the hood). Then, the user requests the new data (to a follower, under the hood). Because of lag, the follower serving the user's data does not have the new data.  
There are many ways to provide *read-your-writes consistency*, i.e. consistency only for the user's data:
* Identify user's data specifically and serve it only from the leader
* Serve data from the leader only when the current time is in the interval [last_update, last_update + SOME_TIME]
* Store timestamps on the client's side to be able to identify lag

#### Monotonic reads

The problem: the user reads data from a replica, then again from another replica with greater lag. The user sees *older* data e.g. deleted records. This can be addressed by always routing the same user's queries to the same replica. This guarantee, weaker than strong consistency, is called monotonic reads.

#### Prefix reads

Another type of consistency that preserves the order of writes. The problem of returning records in a shuffled order is particularly common in partitioned/sharded databases.

#### Monotonic reads vs Prefix reads

**Monotonic Reads**: Focuses on one client’s sequential reads. Ensure there is no going backwards for a given item.  
**Consistent Prefix Reads**: Focuses on global write order. Ensure there is no newer writes without earlier ones, even across different keys.*

## Multi-leader replication

Multi-leader replication is the combination of two things:
1) Single-leader replication, replicated \\( n \\) times
2) Each leader is a follower of the other leaders

Multi-leader replication is much harder than single-leader replication and should be avoided when possible. The reason for that is **write conflicts** between leaders. Now, the benefits are latency (e.g. multiple datacenters), reliability and write throughput.

The typical use case is a multi-datacenter setup. Other use cases include multi-device offline applications or real-time collaborative applications, where each device/user is both a leader and a follower (no strict followers).

### Write conflicts resolution strategies

#### Avoidance

One way to avoid conflicts is to map subsets of records to a unique leader, e.g. attributing a given datacenter to some user.

#### Convergence

Converge to a consistent state with more or less risky approaches. An approach prone to data loss is to uniquely identify each record, then elect one of the conflicting pieces of data as the *winner*: *last write wins* if the ID is a timestamp, highest UUID, random selection, majority vote by the replicas... A somewhat less dangerous approach is to algorithmically merge the values (e.g. concatenate). The safest but most demanding solution is to keep all conflicting records and delegate conflict resolution (to an application layer, the user or whatever).

### Multi-leader replication topologies

* Circular
* Star
* All-to-all

All-to-all topologies are more fault-tolerant, but require more work to guarantee prefix read consistency (i.e. data coming in in the right order).

## Leaderless replication

Leaderless replication is not multi-leader replication with only leaders (like one of the examples of multi-leader setups). There is no concept of any leader node replicating data to followers. Instead, write request are sent altogether to all nodes, either by the client itself or by a coordinator gateway node. Read requests are also sent to all nodes. In case of conflicts, the reader retains the most up-to-date record, e.g. with the highest version number.

## Catch-up mechanisms

### Read repair

Say the client fetches a record version 2 from replicas A and B, and version 1 from replica C. The client writes back version 2 on replica C.

### Anti entropy

A background process regularly scans the nodes for missing data and makes the appropriate writes.

## Quorum consistency

Assume the following setup:
* \\( n \\) replicas
* A write is successful if \\( w \\) nodes confirm it
* A read is successful if \\( r \\) nodes confirm it

To ensure up-to-date reads, \\( w + r > n \\) must hold. To adjust the parameters, consider the application's needs in terms of writes vs reads, node availability and so on. The condition is necessary, but not sufficient. On page 181, the author enumerates a few ways stale records can be returned despite quorum consistency. Monitoring staleness is encouraged.

# Chapter 6: Partitioning i.e. Sharding

## Partitioning

### How to?

The problem you want to avoid is a **skewed** partition causing some nodes to be **hot spots**. But assigning nodes randomly is not really an option since it would take all nodes to be queried for each request. Instead, there are mainly 2 strategies:
* Partitioning by key range (much like an encyclopedia): efficient with range queries, but can lead to hot spots
* Partitioning by hash key: avoids skewed partitions, but does not work for range queries

A *compound primary key* implements a hybrid approach where the first part of the key is hashed and the second part is used for sorting: e.g. `(user_id, update_timestamp)`.

## Secondary indexes for partitioning

### **Local** vs **Global** secondary indexes

Local secondary indexes are easier to implement and maintain: each partition has its own secondary index for its data. On the other hand, they are not efficient for reads because they require *all* partitions to be queried, unless a specific partitioning has been enforced by the user.

A global secondary index is unique for the entire dataset, and it is partitioned itself. It is *term-partitioned* because each key of the index is attributed a partition. The reads are more efficient but the writes are slower and harder to implement, because each write requires the update of the global index, most likely on a separate partition.

## Rebalancing

## Routing