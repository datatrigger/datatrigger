---
title: "Designing Data-Intensive Applications: reading notes, part 1"
summary: "Reading through this backend/data engineering classic. Part 1: Foundations of Data Systems"
date: 2023-03-27
tags: ["backend", "databases", "system design", "data engineering", "indexes", "data structures"]
draft: false
---

![Designing Data-Intensive Applications, by Martin Kleppmann](/res/designing_data_applications/designing-data-intensive-applications-martin-kleppmann.jpeg)

# Part I - Foundations of Data Systems

Premise: modern applications tend to be data-intensive rather than compute-intensive.

# Chapter 1: Reliable, scalable and maintainable applications

A few tools mentioned:
* in-memory data store *Redis*
* in-memory caching layer *Memcached*
* [Full-text search](https://en.wikipedia.org/wiki/Full-text_search) servers e.g. Elasticsearch or Solr

Modern data tools often serve multiple overlapping use cases: for example, Redis can function as a message broker, while Apache Kafka offers database-like durability guarantees. At the same time, modern applications tend to have so many requirements that multiple tools need to be architected together.

This chapter covers three key variables that data system design often optimizes for: reliability, scalability, and maintainability.

## 1. Reliability

A reliable system basically keeps working when things go wrong. A single thing going wrong is called a *fault*, while a *failure* means the whole system stops working. Faults can be made unlikely but they are inevitable. So, in general, implementing fault-tolerance is more realistic than no fault at all. Unless the error cannot be tolerated (e.g. security leaks). The book focuses on tolerable faults.

Testing fault-tolerance with [chaos engineering](https://en.wikipedia.org/wiki/Chaos_engineering): Netflix Chaos Monkey, then [Simian Army](https://netflixtechblog.com/the-netflix-simian-army-16e57fbab116).

### Hardware faults (MTTF) vs. Software faults vs. Human errors

Prefer *software* fault-tolerance over *hardware* redundancy. Hardware redundancy is not sustainable with data volumes and/or computing demand. Bonus: this helps with maintainability, e.g no need for planned downtimes. Also, there is generally less correlation among hardware faults than among software bugs: example of Linux' kernel bug with the leap second of June 30, 2012.

## 2. Scalability

The *scalability* of a system is its ability to cope with increased load.

### Load

Load can be evaluated with respect to *load parameters*:
* requests/s (web server)
* reads/writes ratio (database)
* active users/s (chat room)
* Hit rate (cache)
* ...

#### Twitter (2012)

Post tweet: 5k requests/s on average, max 12k requests/s  
Home timeline reads: 300k requests/s

Writing a tweet is an 'easy' operation, but the [**fan-out**](https://en.wikipedia.org/wiki/Fan-out_(software)) is problematic. By fan-out, the author means the one-to-many relationship between a user and its followers. Posting 1 tweet also means making it available to all the sender's followers. Twitter had two solutions at the time (combined):

1) Just insert the tweet into the central collection of tweets. Fetch it when a follower of the sender requests his timeline.
2) Write the tweet in the home timeline cache of each user following the sender

The first version is cheaper for posting tweets, but more expensive for home timeline reads. But there are far more reads than writes - almost 2 orders of magnitude - and this approach turned out not to be sustainable for Twitter. They switched to approach 2, then a mix: approach 2 is the general rule, except for users who have a large amount of followers.

In this case, the key load parameters are: writes/s (post), reads/s (home timeline) and number of followers/user.

### Performance

2 main ways to describe scalability with regard to load, by looking at the functions:
* Fixed resources: $$f_{resource}(load) = performance$$
* Fixed performance: $$f_{performance}(load) = resource$$

What is performance?
* *Throughput*, e.g. batch processing systems
* *Response time*, e.g. online systems

*Note*: response time = latency + service time, even if latency and response time are often used synonymously.

The average is often used, but of course percentiles or the whole distribution provide more information.

*Example*: Amazon describes some response time requirements in terms of p999

### Coping with load

* Vertical scaling (scaling up) vs. horizontal scaling (scaling out)
* *Elastic* systems: systems that can automatically add resources on load increasing
* Distributing *stateless* services is easier than for *stateful* ones
* Scaling architectures are very dependent on the application: 100 000 requests of 1kB/s and 3 requests of 2 GB/min are the same throughput, but require different scaling

## 3. Maintainability

*Operability* / *Simplicity* / *Evolvability*

# Chapter 2: Data models & Query languages

Data models used to store data structures: relational model, document model, graph-based models...

## Relational model

*Relations* and *tuples* = i.e *tables* and *rows* in the SQL world.

Weak points that drove the need for alternative (*NoSQL*) models:
* Greater freedom in terms of data structures and queries, see [impedance mismatch](https://en.wikipedia.org/wiki/Object%E2%80%93relational_impedance_mismatch)
* Scalability
* Open-source tools availability

## [Object-relational impedance mismatch](https://en.wikipedia.org/wiki/Object%E2%80%93relational_impedance_mismatch)

Refers to the relative incompatibility between objects (OOP) and relational tables. **Object-relational mapping (ORM)** frameworks are tools designed to alleviate this mismatch, e.g. *Hibernate* or *ActiveRecord*.

For example, relational databases are not the best to represent one-to-many relationships: in a résumé object, there is one `first_name` and one `last_name`, but possibly many `jobs` or `education` lines. There are several ways to represent one-to-many relationships:

* Store a field (e.g. `jobs`) in its own normalized table, with a foreign key like `user_id`
* Use multi-valued types like a [structured type](https://en.wikipedia.org/wiki/Structured_type) (fixed number of values) or a JSON datatype (e.g. MySQL, PostgreSQL)
* Use a text field and store a JSON string
* Use a document-oriented database

## Document databases

E.g. MongoDB, Espresso. JSON has better **locality** than multi-table relational databases: no need to perform several queries or several joins to fetch the content of one *document* (e.g. the résumé of a given user).

## Relational vs. document databases

* **one-to-many** relationships: the approach is significantly different
    * Relational: the many entities are stored in a separate table
    * Document: the many entities are stored within the parent record
* **Many-to-one** and **many-to-many**: conceptually similar approach as *foreign keys* correspond to *document references*

### Advantages of document data **model** (not databases)

* Schema flexibility (*schema-on-read* vs. *schema-on-write*)
* Locality: the data is stored in one string, not on multiple tables. Some relational databases implement locality e.g. Google Spanner
* Data structures more in line with applications (e.g. objects)

### Advantages of relational data **model** (not databases)

* Joins: better support
* Many-to-one and many-to-many: better support

## Query languages

### Declarative vs. imperative

SQL is a declarative language, which has advantages over imperative ones:
* Concise
* Easy, as the implementation details are hidden inside the underlying engine
* Efficient: the underlying engine can be upgraded without changing high-level instructions
* Prone to parallelization

### MapReduce

## Graph-like Data Models

Use-case: lots of many-to-many relationships in the data, possibly heterogeneous. *Neo4j* or *Titan* are graph databases, *Cypher* is a query language

A significant advantage of graph query languages: unknown numbers of vertices can be traversed easily (e.g. `:WITHIN*0..` in Cypher) whereas *recursive CTEs* are required in SQL.

*Datalog* is another example of programming language that is very efficient for recursive queries:

```
// Data
parent(alice, bob).
parent(bob, carol).

// Rules: base case, recursion
ancestor(X, Y) :- parent(X, Y).
ancestor(X, Y) :- parent(X, Z), ancestor(Z, Y).
```

*Datalog* is used in graph databases but also IAM, Data Lineage or Static Program analysis systems.

# Chapter 3: Storage and Retrieval

2 families of **storage engines**/**indexes** considered in this chapter:
* *Log-structured*
* *Page-oriented*

## Log-structured databases and indexes

*log*: append-only sequence of records

### A fundamental example

```bash
#!/bin/bash

db_set() {
    echo "$1,$2" >> database
}

db_get() {
    grep "^$1" database | sed -e "s/^$1,//" | tail -n 1
}
```

Apart from the lack of concurrency control, disk space reclaiming mechanisms, error/partial writes handling... The performance of this log as a database is very unbalanced:
* Writes: O(1)
* Reads: O(n)

### Indexes

An **index** is an additional data structure that aims at increasing the performance of queries.  
Trade-off: an index speeds up reads (if well designed) and slows down writes

### Hash indexes

E.g. `HashMap<Key, Offset>`  
*Example*: [Bitcask](https://en.wikipedia.org/wiki/Bitcask),the default storage engine of distributed NoSQL key-value store [Riak](https://en.wikipedia.org/wiki/Riak)

### Memory management

Back to the data-store (not the index): since the underlying file is append-only, how is its size kept under control?

i) *Segmentation*: break the log into chunks  
ii) *Compaction*: only retain the most recent value for each key  
iii) *Merging*: same as compaction but over several compacted segments  

### Details on log files

A few practical implementation issues:
* File format efficiency: binary encodings are preferable
* Deletion of records: see [Tombstone](https://en.wikipedia.org/wiki/Tombstone_(data_store))
* Crashes & partial writes: snapshots (segments), checksums (record)
* Concurrency: usually only one writer thread is implemented

Why not just use the hashmap itself to store values? Because the log paradigm (append-only/no overwrites) brings benefits:
* No risk of partial overwrites: much easier concurrency/crash recovery
* Sequential writes are faster than random writes
* Less fragmentation thanks to merging

Drawbacks:
* Range queries are inefficient because the keys are stored in random order
* Hash indexes must fit in memory

### SSTables/LSM-Trees

A *Sorted String Table* or *SSTable* is a log with the entries **sorted by key**.

Advantages:
* Merging segments: gets simple and efficient, as a mergesort-like algorithm can be used
* Sparse index: no need to keep *all* the keys in memory (if you know where keys 1 and 10 are, then you know 5 is somewhere in between)
* Compression & I/O brandwith: each entry of the sparse index points to a compressed block

How it works:
* Writes: insert key and value into a *memtable* := in-memory balanced tree (e.g. AVL, red-black)
* When size(memtable) > threshold: flush memtable to disk as an SSTable
* Reads: look into memtable, then most recent segment, then second-most-recent segment, and so on...
* Merge and compact + discard delete/overwritten records once in a while

*Examples*: LevelDB, an alternative to Bitcask in Riak. Bigtable (GCP).

Implementation challenges:
* Non-existing key lookups are long (requires checking the memtable + all segments). Solution: [*bloom filters*](https://en.wikipedia.org/wiki/Bloom_filter)
* Efficient compacting/merging strategies: *size-tiered* vs. *leveled*

SSTables are an implementation of Log-Structured Merge-Trees (LSM-Trees). They can sustain high throughput because the writes are sequential.

## B-Trees

[B-trees](https://en.wikipedia.org/wiki/B-tree) also sort they key-value pairs by key. They are a generalization of binary search trees.  

The database is split into fixed-sized chunks called *pages* or *blocks*. Only 1 page is read/written at a time. Files are overwritten and modified in-place, with no restriction to appending data as with SStables.

Overwrites are risky and complicated. Example: writing to an already-full page requires the page to be updated and spit into two children. What if the database crashes in the middle of the operation? *Orphan pages* may be created.

Structure:
* *Root*: the first page of the B-tree, where all lookups start.
* Root and subsequent pages contain:
    * Keys
    * References: a reference surrounded by key X and Y leads to a page with X < keys < Y
* *Leaves*: pages with only references to values, or keys + values

*Branching factor*: number of references in child pages

Implementation challenges/optimizations:
* Crash recovery: *Write-Ahead Log* (WAL) a.k.a. *redo log* / copy-on-write schemes (also helps with concurrency)
* Store abbreviated keys, i.e just enough information to be used as boundaries between key ranges
* Additional data structure to link leaf pages in sequential order, or references to siblings
* Concurrency: see *B-Tree locking*

## SSTables vs. B-Trees

Generally, SSTables appear to be:
* Faster for writes: writing is sequential, less *write amplification*, less fragmentation
* slower for reads: multiple segments to check at different stages of compaction

See detailed comparison p. 84-85

## Additional considerations on indexes


### Primary/secondary indexes

Unlike primary (generally clustered) indexes, secondary indexes may index **duplicate values**. Log-structured indexes and B-Trees can be used as secondary indexes. Techniques to deal with duplicate indexed values:
* [inverted index / postings list](https://www.educative.io/answers/what-is-an-inverted-index)
* Append a row identifier to each entry

### Clustered/Nonclustered indexes

Values can be either referenced by indexes and stored in an auxiliary *heap*, or be part of the index itself:
* *Clustered* indexes store the value (there can be only 1)
* *Nonclustered* indexes store references
* *Covering* indexes store the values of a subset of columns. A query is said to be covered by the index if it can be answered using the covering index alone

### Concatenated indexes

The key is created by simple concatenation of several fields, e.g `last_name` + `first_name`

Use-case example: spatial data, see [R-trees](https://en.wikipedia.org/wiki/R-tree)

### Fuzzy, full-text indexes

See [Apache Lucene](https://lucene.apache.org/)

## In-memory databases

Disks have 2 significant advantages:
* Durability
* Low cost per GB

The second advantage is less and less significant over time and now many datasets fit in the RAM, hence the rise of in-memory databases.

*Examples*: Redis, VoltDB, MemSQL...

Some in-memory databases provide multiple levels of durability (none, weak...) by several means:
* Log (on disk)
* Snapshots (on disk)
* Replication

Counterintuitive:  in-memory databases are not really faster thanks to not reading/writing to disk. Traditional databases rely on the OS caching disk blocks, which can lead to similar results in terms of speed. In-memory databases are actually faster because they avoid the overhead of encoding data structures to writeable formats.

## OLTP vs. OLAP

I.e. *Online Transaction Processing* vs. *Online Analytical Processing*, also related to *transaction processing* vs. *batch processing*.

* OLTP: Low-latency reads/writes, small number of records per query, fetched by some key and using an index
* OLAP: large number of rows fetched per query, but only a few columns, and aggregated

### Data warehousing

The various OLTP systems in an organization (e.g. logistics, sales, employees, suppliers...) need high availability + low latency, thus they are not suited for complex analytics queries fetching many rows. The general idea of the data warehouse is to centralize read-only copies of the various OLTP systems, through ETLs. Also, they are optimized for these queries and use different indexing strategies than what was presented above.

Key ideas: facts, dimensions, star schema, snowflake schema

### Columnar storage

Most OLTP systems store the values of a given row next to each other. Instead, OLAP systems tend to use column-oriented storage. This allows reads of only a subset of columns, as often needed with analytics queries.

Columnar storage can be coupled with several technique for even more efficient data retrieval:

* Column compression
    * *Bitmap encoding* (similar to One-Hot encoding in Data Science), which allows *vectorized processing* downstream
    * *Run-length encoding* on top of bitmap encoding
* *Sort order* on one or several columns, e.g. `date_key` if date ranges are often targeted, or even several different sort orders (one sort order per replica, Vertica does this)

Column compression and ordering are more easily implemented with LSM-Trees with their sorted in-memory structure and incremental merges with files on disk. On the contrary, B-Trees require in-place updates, making them inappropriate for these use-cases.

### Materialized views

Unlike traditional views in relational databases, materialized views are actual copies of a table + a query. *Data cubes* or *OLAP cubes* are examples of materialized views: a grid of aggregates grouped by different dimensions, e.g. `date` and `product_sk`. They enhance performance on given queries but lack flexibility: for example, once aggregated, no filtering is possible anymore e.g. it's impossible to know the revenue on a given date, but only for a given location.

# Chapter 4: Encoding and Evolution

*Evolvability* was introduced in chapter 1, and changes in applications generally bring about changes in data.

Chapter 2: in relational databases, only 1 schema is valid at a given time. *Schema-on-read* or *schemaless* databases allow different formats to coexist.

## Encoding formats

Encoding or serialization is about writing in-memory objects to self-contained byte sequences, for storage or transmission.

### Built-in programming languages encoding libraries

**Not recommended**: typically language-specific, unsafe, without proper versioning support and inefficient.

### JSON, XML, CSV...

Broadly supported, but inherently ambiguous and without native support for binary data.

### Binary encodings

Much more compact than human-readable formats, but require a schema. Appropriate for more "internal" data transfers.

#### Protobuf

*Protocol Buffers* or *Protobuf* is the most popular and widely spread binary encoding.

* Forward compatibility: newer fields are ignored by older schema
* Backward compatibility: works if newer fields are optional

Beware with data type evolution. It may work, but it can be risky. E.g. 32-bit -> 64-bit is backward-compatible but not necessarily forward-compatible as 64-bit numbers might get truncated when read with the old 32-bit type.

#### Avro

Avro is even more lightweight than Protobuf because it externalizes the schema. The *writer's schema* and the *reader's schema* may be different, but compatible according to Avro's specifications. There are different ways to enforce a schema:
* Write the schema at the beginning of an Avro file
* Maintain schemas separately and identify them by version numbers, then each record can reference a schema version
* Negotiate the schema with the Avro RPC protocol

#### Dynamically-generated schemas

Avro is more convenient for dynamically generated schemas because fields are identified by name. It automatically handles schema reconciliation between the writer’s and reader’s schemas. In contrast, Protobuf and Thrift use numeric field tags, and the user is responsible for maintaining tag consistency to ensure compatibility.

## Dataflow

* Data might be lost when an older version of an app updates records previously written by a newer version of the app
* Discussion about REST, RPC
* Message brokers