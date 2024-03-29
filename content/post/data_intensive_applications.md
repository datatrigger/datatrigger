---
title: "Designing Data-Intensive Applications (M. Kleppmann): reading notes 1/3"
summary: "Reading through this classic about data engineering. Part 1: Foundations of Data Systems"
date: 2023-03-27
tags: ["reliability", "scalability", "maintainability", "system design", "data engineering"]
draft: false
---

*Work in progress*

![Designing Data-Intensive Applications, by Martin Kleppmann](/res/designing_data_applications/designing-data-intensive-applications-martin-kleppmann.jpeg)

# Part 1: Foundations of Data Systems

# Chapter 1: Reliable, scalable and maintainable applications

A few tools mentioned:
* [Data stores](https://en.wikipedia.org/wiki/Data_store) e.g. Redis. A datastore is an ill-defined term that design a data system in between data warehouses and data lakes. It can refer to noSQL, non-ACID semi-structured data applications like Redis.
* Caching layers e.g. Memcached (also a use-case of Redis)
* [Full-text search](https://en.wikipedia.org/wiki/Full-text_search) servers e.g. Elasticsearch or Solr

## 1. Reliability

* Distinction between *fault* (more a less a single component not working as expected) and *failure* (refers to the whole system)
* Netflix Chaos Monkey, then [Simian Army](https://netflixtechblog.com/the-netflix-simian-army-16e57fbab116): an example of [chaos engineering](https://en.wikipedia.org/wiki/Chaos_engineering)
* *Hardware* faults (MTTF) / *Software* faults / *Human* errors
* Main concepts:  
    * Prefer *tolerating* faults (easier) over *preventing* faults. Impossible in some cases though, e.g. security: there's no tolerating users' data being compromised
    * Generally less correlation in hardware faults than in software bugs: example of Linux' kernel bug with the leap second of June 30, 2012
    * Prefer *software* fault-tolerance over *hardware* redundancy. Hardware redundancy is not sustainable with data volumes and/or computing demand. Bonus: this helps with maintainability, e.g no need for planned downtimes.

## 2. Scalability

The *scalability* of a system is its ability to cope with increased load.

Scalability is linked with reliability: a common reason for a system to be less reliable is the increase of the load.

Scalability can be evaluated with respect to *load parameters*:
* requests/s (web server)
* reads/writes ratio (database)
* active users/s (chat room)
* Hit rate (cache)
* ...

### Twitter (2012)

Post tweet: 5k requests/s on average, max 12k requests/s  
Home timeline reads: 300k requests/s

Writing a tweet is an 'easy' operation, but the [**fan-out**](https://en.wikipedia.org/wiki/Fan-out_(software)) is problematic. By fan-out, the author means the one-to-many relationship between a user and its followers. Posting 1 tweet also means making it available to all the sender's followers. Twitter had two solutions at the time (combined):

1) Just insert the tweet in the central collection of tweets. Fetch it when a follower of the sender requests his timeline.
2) Write the tweet in the home timeline cache of each user following the sender

The first version is cheaper for posting tweets, but more expensive for home timeline reads. But there are far more reads than writes - almost 2 orders of magnitude - and this approach turned out not to be sustainable for Twitter. They switched to approach 2, then a mix: approach 2 is the general rule, except for users who have a large amount of followers.

In this case, the key load parameters are: writes/s (post), reads/s (home timeline) and number of followers/user.

### Performance

2 main ways to describe performance:
* Load is increased, but not system resources: how does the system behave?
* load is increased and system resources as well: how much is needed to keep performance stable?

What is performance?
* *Throughput*, e.g. batch processing systems
* *Response time*, e.g. online systems

*Note*: response time = latency + service time, even if latency and response time are often used synonymously.

The average is often used, but of course percentiles or the whole distribution provide more information.

*Example*: Amazon describes some response time requirements in terms of p999

### Coping with load

* Vertical scaling (scaling up) vs horizontal scaling (scaling out)
* *Elastic* systems: systems that can automatically add resources on load increasing
* Distributing *stateless* services is easier than for *stateful* ones
* Scaling architectures are very dependent on the application: 100 000 requests of 1kB/s and 3 requests of 2 GB/min are the same throughput, but require different scaling

## 3. Maintainability

*Operability* / *Simplicity* / *Evolvability*

# Chapter 2: Data models & Query languages

## Relational model

*Relations* and *tuples*, i.e *tables* and *rows* in the SQL world.

Weak points that drove the need for alternative ('NoSQL') models:
* Freedom in terms of data structures and queries, see [impedance mismatch](https://en.wikipedia.org/wiki/Object%E2%80%93relational_impedance_mismatch)
* Scalability
* Open-source tools

## [Object-relational impedance mismatch](https://en.wikipedia.org/wiki/Object%E2%80%93relational_impedance_mismatch)

Refers to the relative incompatibility between objects (OOP) and relational tables. **Object-relational mapping (ORM)** frameworks are tools designed to alleviate this mismatch, e.g. *Hibernate* or *ActiveRecord*.

For example, relational databases are not the best to represent one-to-many relationships: in a résumé object, there is one `first_name` and one `last_name`, but possibly many `jobs` or `education` lines. There are several ways to represent one-to-many relationships:

* Store a field (e.g. `jobs`) in its own normalized table, with a foreign key like `user_id`
* Use multi-valued types like a [structured type](https://en.wikipedia.org/wiki/Structured_type) (fixed number of values) or a JSON datatype (e.g. MySQL, PostgreSQL)
* Use a text field and store a JSON string
* Use a document-oriented database

## Document databases

E.g. MongoDB, Espresso. JSON has better **locality** than multi-table relational databases: no need to perform several queries or several joins to fetch the content of one *document* (e.g. the résumé of a given user).

## Relational vs document databases

* Nested records, i.e. **one-to-many** relationships: the case where relational and document databases differ significantly:
    * Relational: stored in a separate table
    * Document: stored within the parent record
* **Many-to-one** and **many-to-many**: *foreign keys* correspond to *document references*

### Advantages of document data **model** (not databases)

* Schema flexibility (*schema-on-read* vs *schema-on-write*)
* Locality: the data is stored in one string, not on multiple tables. Some relational databases implement locality e.g. Google Spanner
* Data structures more in line with applications (e.g. objects)

### Advantages of relational data **model** (not databases)

* Joins: better support
* Many-to-one and many-to-many: better support

## Query languages

### Declarative vs imperative

SQL is a declarative language, which has advantages over imperative ones:
* Concise
* Easy, as the implementation details are hidden inside the underlying engine
* Efficient: the underlying engine can be upgraded without changing high-level instructions
* Prone to parallelization

### MapReduce

## Graph-like Data Models

Use-case: lots of many-to-many relationships in the data, possibly heterogeneous.

Several models:
* Property graph: *Neo4j*, *Titan* - Query language: *Cypher*
* Triple store - Query languages: [Turtle](https://en.wikipedia.org/wiki/Turtle_(syntax)) (syntax), *SPARQL*

A significant advantage of graph query languages: unknown numbers of vertices can be traversed easily (e.g. `:WITHIN*0..` in Cypher) whereas *recursive CTEs* are required in SQL.

An important application of the triple store model: [Resource Description Framework (RDF)](https://en.wikipedia.org/wiki/Resource_Description_Framework), relating to the *semantic web*.

# Chapter 3: Storage and Retrieval

2 families of **storage engines** and **indexes** considered in this chapter:
* *Log-structured* indexes
* *Page-oriented* indexes

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
Trade-off: an index speeds up reads (if weel designed) and slows down writes

### Hash indexes

Assume the database is a [key-value store](https://en.wikipedia.org/wiki/Key%E2%80%93value_database) saved in a log. Then a hash index is an in-memory hash map where keys are mapped to byte offsets.
* Writes (insert/update): also create/update the offset in the hash index
* Reads: find the offset then read in O(1) time, i.e one disk seed or even better, one cache read

*Use-case*: key = URL of videos, value = views. Lots of writes (hence the log), not too many distinct keys (to fit in-memory).

*Example*: [Bitcask](https://en.wikipedia.org/wiki/Bitcask),the default storage engine of distributed NoSQL key-value store [Riak](https://en.wikipedia.org/wiki/Riak)

### Memory management

Back to the data-store (not the index): since the underlying file is append-only, how is its size kept under control?

i) *Segmentation*: break the log into chunks  
ii) *Compaction*: only retain the most recent value for each key  
iii) *Merging*: same as compaction but over several compacted segments  

Additional details:
* Segments are never modified once written: merged segments are written to new files
* This way, compaction/merging can be done in the background
    * Reads: use old segment files
    * Writes: use a new segment (not in the set of currently compacted + merged segments)
* After merging: redirect reads to the resulting merged segment + delete old segments

How to find a value? **Each segment has its own in-memory hash index:**
* Search the latest segment
* Search the second-most-recent segment
* ...

The compaction/merging process keeps the number of segments small so that lookups are fast.

### Details on log files

A few practical implementation issues:
* File format efficiency: binary encodings are preferable
* Deletion of records: see [Tombstone](https://en.wikipedia.org/wiki/Tombstone_(data_store))
* Crashes & partial writes: snapshots (segments), checksums (record)
* Concurrency: usually only one writer thread is implemented

Advantages of the log paradigm, i.e append-only/no overwrites:
* No risk of partial **over**writes: much easier concurrency/crash recovery
* Sequential writes are faster than random writes
* Less fragmentation thanks to merging

Drawbacks:
* Range queries are inefficient because the keys are stored in random order
* Hash indexes must fit in memory

### SSTables/LSM-Trees

A Sorted String Table or SSTable is a hash table with the sequences of key-value pairs **sorted by key** in the segments.

Advantages:
* Merging segments: gets simple and efficient, as a mergesort-like algorithm can be used
* Sparse index: no need to keep *all* the keys in memory
* Compression & I/O brandwith: each entry of the sparse index points to a compressed block

How it works:
* Writes: insert key and value into a *memtable* := in-memory balanced tree (e.g. AVL, red-black)
* When size(memtable) > threshold: flush memtable to disk as an SSTable
* Reads: look into memtable, then most recent segment, then second-most-recent segment, and so on...
* Merge and compact + discard delete/overwritten records once in a while

*Examples*: LevelDB, an alternative to Bitcask in Riak. Bigtable (GCP).

Implementation challenges:
* Non-existing key lookups are long (requires checking the memtable + all segments). Solution: [*bloom filters*](https://en.wikipedia.org/wiki/Bloom_filter)
* Efficient compacting/merging strategies: *size-tiered* vs *leveled*

SSTables, also known as Log-Structured Merge-Trees (LSM-Trees), can sustain high throughput because the writes are sequential.

### B-Trees

[B-trees](https://en.wikipedia.org/wiki/B-tree) also sorts they key-value pairs by key. They are a generalization of binary search trees.  

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
* Store abbreviated keys, i.e just enough information to be used ad boundaries between key ranges
* Additional data structure to link leaf pages in sequential order, or references to siblings
* Concurrency: see *B-Tree locking*

### SSTables vs B-Trees

Generally, SSTables appear to be:
    * Faster for writes: writing is sequential, less *write amplification*, less fragmentation
    * slower for reads: multiple segments to check at different stages of compaction

See detailed comparison p. 84-85

### Additional considerations on indexes

#### Secondary indexes

Secondary indexes may index **duplicate values** (in the previous section, the indexed keys were assumed to be primary keys). Log-structured indexes and B-Trees can be used as secondary indexes. Techniques to deal with duplicate indexed values:
* [inverted index / postings list](https://www.educative.io/answers/what-is-an-inverted-index)
* Append a row identifier to each entry

#### Values and indexes

Values can be either referenced by indexes and stored in an auxiliary *heap*, or be part of the index itself.
* *Clustered* indexes store the value
* *Nonclustered* indexes store references
* *Covering* indexes store the values of a subset of columns. A query is said to be covered by the index if it can be answered using the covering index alone

#### *Concatenated* indexes

The key is created by simple concatenation of several fields, e.g `last_name` + `first_name`

Use-case example: spatial data, see [R-trees](https://en.wikipedia.org/wiki/R-tree)

#### Fuzzy, full-text indexes

See [Apache Lucene](https://lucene.apache.org/)

#### In-memory databases

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

## OLTP vs OLAP

I.e. *Online Transaction Processing* vs *Online Analytical Processing*, also related to *transaction processing* vs *batch processing*.

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

Unlike traditional views in relational databases, materialized views are actual copies of a table + a query. *Data cubes* or *OLAP cubes* are examples of materialized views: a grid of aggregates grouped by different dimensions, e.g. `date` and `product_sk`. They enhance performance on given queries but lack flexibility: for example, once aggregated, no filtering is possible anymoren e.g. it's impossible to know the revenue on a given date, but only for a given location.

# Chapter 4: Encoding and Evolution

*Evolvability* was introduced in chapter 1, and changes in applications generally bring about changes in data.

Chapter 2: in relational databases, only 1 schema is valid at a given time. *Schema-on-read* or *schemaless* databases allow different formats to coexist.
