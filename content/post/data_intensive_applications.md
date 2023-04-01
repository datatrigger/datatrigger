---
title: "Designing Data-Intensive Applications (M. Kleppmann): reading notes"
summary: "A classic about low-level data engineering"
date: 2022-03-27
tags: ["reliability", "scalability", "maintainability", "system design", "data engineering"]
draft: true
---

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
    * Generally less correlation in hardware faults than in software bugs: example of Linux' kernel bug with the leap seconde of June 30, 2012
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