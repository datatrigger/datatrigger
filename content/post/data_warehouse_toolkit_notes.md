---
title: "The Data Warehouse Toolkit (R. Kimball): reading notes"
summary: "It's been 10 years since I read the first few chapters of Kimball's data warehousing bible. It would have been smart to take notes at the time. Better late than never!"
date: 2022-07-21
tags: ["data modeling", "warehouse", "dimensional data", "fact table", "data engineering"]
draft: true
---

# Chapter 1: Data Warehousing

A DW/BI system starts from the business needs, then works backwards to logical then physical considerations.

## Operational vs analytical data worlds

### Operational systems

E.g orders, new customers, status of operational activities, logs...

Characteristics:
* Speed/performance
* One transaction at a time
* Usually repetitive entries
* Usually no historical data, but current operational state

### DW/BI systems

E.g orders, new customers, status of operational activities, logs...

Characteristics:
* metrics/aggregation oriented
* Usually dealing with lots of transaction at a time

*A DW is definitely not a copy of operational databases.*

Fundamentals requirements of a DW/I system:
* Understandability/Accessibility from the user's standpoint
* Consistency: data definition, labels...
* Maintainability, handling change
* Security
* **Trustworthiness**
* **User acceptance**

## Editor-in-chief Metaphor

## Dimensional modeling

Provides:
* Understandable data
* Fast query performance

### Dimensional modeling vs 3NF

*Third normal form models (3NF)* a.k.a. *entity-relationship models (ER)* seek to eliminate data redundancies, splitting data into (many) relational tables. Dimensional models also consist in joined relational tables, but less normalized to improve usability and performance addressing complex queries. 3NF is more indicated for operational processing.

### Relation vs multidimensional databases



