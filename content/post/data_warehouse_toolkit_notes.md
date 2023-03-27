---
title: "The Data Warehouse Toolkit (R. Kimball): reading notes"
summary: "It's been 10 years since I read the first few chapters of Kimball's data warehousing bible. It would have been smart to take notes at the time. Better late than never!"
date: 2022-06-15
tags: ["data modeling", "warehouse", "dimensional data", "fact table", "data engineering"]
draft: true
---

*\*Work in progress\**

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

Characteristics:
* metrics/aggregation oriented
* Usually dealing with lots of transaction at a time

*A DW is definitely not a copy of operational databases.*

Fundamentals requirements of a DW/BI system:
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

### Relational vs multidimensional databases

A dimensional model implemented in:
* a relational database is referred to as a **star schema**
* a multidimensional database is referred to as a **OLAP** (OnLine Analytical Processing, not to be confused with OLTP - OnLine Transactional Processing)

Either way, the logic is the same (a dimensional model), but since the implementation differs, performance and deployment differ too. OLAP systems have a higher barrier to entry but are more efficient.

## *Fact tables*

A fact table stores low-level measurements data from business process events at a given *grain*.

A *fact* represents a business measure, each row of a fact table represents a measurement event (e.g. a scan in a supermarket). Each measurement row must be at the same grain. This ensures consistency and avoids double counting for example.

A fact should be, if possible, numeric and additive.
* Sales amounts are numeric and additive
* Account balances are said to be semi-additive: they cannot be summed over the time dimension
* Unit prices are not additive

A fact table is typically narrow and very long, i.e many rows and a few columns. But they make up most of the dimensional model space, since they aim at exhaustivity.

A fact table has 2 or more foreign keys.

*Referential integrity*: when all the keys' values in a fact table lie in the corresponding primary key's set of values.

> in a relational database, any column in a base table that is declared a foreign key can only contain either null values or values from a parent table's primary key or a candidate key. ([Wikipedia](https://en.wikipedia.org/wiki/Referential_integrity))

*Composite key:*

> The fact table generally has its own primary key composed of a subset of the for-
eign keys. This key is often called a composite key. Every table that has a composite
key is a fact table.

## *Dimension tables*

> Dimension tables are integral companions to a fact table. The dimension tables con-
tain the textual context associated with a business process measurement event. They
describe the “who, what, where, when, how, and why” associated with the event.

Each dimension is defined by a single primary key.

## Fact vs dimension

> When triaging operational source data, it is sometimes unclear whether a
numeric data element is a fact or dimension attribute. You often make the decision
by asking whether the column is a measurement that takes on lots of values and
participates in calculations (making it a fact) or is a discretely valued description
that is more or less constant and participates in constraints and row labels (making
it a dimensional attribute). For example, the standard cost for a product seems like
a constant attribute of the product but may be changed so often that you decide it
is more like a measured fact. Occasionally, you can’t be certain of the classiﬁcation;
it is possible to model the data element either way (or both ways) as a matter of the
designer’s prerogative.

## Hierarchy

> Dimension tables often represent hierarchical relationships.

They are also rather de-normalized, favoring star schema over snowflake schemas. This incurs a minor loss of storage (dimension tables are not that big compared to fact tables)

## Performances

> A database engine can make strong assumptions about ﬁ rst constraining the heavily
indexed dimension tables, and then attacking the fact table all at once with the
Cartesian product of the dimension table keys satisfying the user’s constraints.
Amazingly, using this approach, the optimizer can evaluate arbitrary n-way joins
to a fact table in a single pass through the fact table’s index.

# Surrogate keys

Say your primary key is product code, then after a few years a new product is given a product code that already exists. The consistency of the key is compromised. To avoid this kind of problem, using a meaningless integer surrogate key is recommended.


