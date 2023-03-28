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
* Netflix Chaos Monkey, then [Simian Army](https://netflixtechblog.com/the-netflix-simian-army-16e57fbab116): an example of [chaos engineering]()