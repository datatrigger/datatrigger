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

**Statement-based replication** is not widely used because it can cause a lot of problems, e.g. nondeterministic functions like `NOW()`, out of sync sequences or side effects...  Instead, **physical-log-based replicaton** uses the actual log of the storage engine: the WAL for B-tree based engines, or SSTables for LSM-tree bases engines. In this case the leader "Writes twice" its log: first locally, then sending it over the network to the followers. A big downside: as a low-level object, the WAL is tightly coupled to the storage engine. As a consequence, database version mismatches between nodes are risky. This prevents zero-downtime rolling upgrades. As a middle ground, higher-level **logical log** objects are often used to avoid coupling replication and storage engines.

### Replication lag

Leader-based replication scales when data access is mostly reads, not writes: just add followers. Realistically, replication is asynchronous (see above) and brings about **eventual consistency** and **replication lag**. Some examples:

#### Read-your-writes consistency

The problem: first, the user submits data (to the leader, under the hood). Then, the user requests the new data (to a follower, under the hood). Because of lag, the follower serving the user's data does not have the new data.  
There are many ways to prove *read-your-writes consistency*, i.e. consistency only for the user's data:
* Identify user's data specifically and serve it only from the leader
* Serve data from the leader only when the current time is in the interval [last_update, last_update + SOME_TIME]
* Store timestamps on the client's side to be able to identify lag

#### Monotonic reads

The problem: the user reads data from a replica, then again from another replica with greater lag. The user showed *older* data e.g. deleted records. This can be addressed by always routing the same user's queries to the same replica. This guarantee, weaker than strong consistency, is called monotonic reads.

#### Prefix reads

Another type of consistency that preserves the order of writes. The problem of returning records in a shuffled order is particularly common in partitioned/sharded databases.

## Multi-leader replication