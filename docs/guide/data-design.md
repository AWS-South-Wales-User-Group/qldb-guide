---
title: "Data Design"
date: 2020-02-21T18:43:01Z
lastmod: 2020-07-11
weight: 80
draft: false
---

## Background
Amazon QLDB is designed to address the needs of high-performance online transaction processing (OLTP) workloads, with streaming support to meet OLAP requirements. This means QLDB is optimised for a specific set of query patterns, even though it supports a SQL-like query language.

It is critical to design applications and their data models to work with known constraints. Otherwise, as your tables grow, you will encounter significant performance problems, including query latency, transaction timeouts, and concurrency conflicts. In addition, QLDB supports event-driven workloads through its streams functionality, and the data design has a direct impact on the events transmitted on the stream.

So how do you go about deciding the right data model for your QLDB ledger application?

The reality is there is no one size fits all approach, as it depends upon your specific use case. This section looks to set out the important considerations you should take into account.

## Normalisation vs Denormalisation
You interact with QLDB through a SQL-like query language, and you write data into tables that you create. However, QLDB is not a traditional relational database. Traditional relational databases introduced the concept of normalisation to reduce the duplication of data, by structuring data into separate tables, and creating joins between them. This was started because of the prohibitive cost of storage when relational databases originated back in the 1970s. The past decade has seen an explosive growth in NoSQL database engines, where data can be denormalised and stored in a format used by the calling application, reducing the [`impedance mismatch`](https://en.wikipedia.org/wiki/Object%E2%80%93relational_impedance_mismatch).

QLDB writes documents, which are Amazon Ion struct objects, into tables. This makes it similar in nature to a document database. It is considered `schemaless`. When you create a table in QLDB, you do not specify any attributes or data types to it. This gives far greater flexibility, and means new fields and values can be added without amending the table structure. A more accurate term coined by Martin Kleppmann is `schema-on-read`, as the structure of the document needs to be understood and interpreted by a calling application.

The two main alternative approaches are to create separate tables for specific entities, or to created nested or embedded documents within a single table.

## Separate vs Nested Documents

In the following example, a Customer entity has an Address. This can be modeled using separate tables as follows:

<table>
<tr>
<td> <b>Customer Table</b> </td> <td> <b>Address Table</b> </td>
</tr>
<tr>
<td>  
```json
{
    "customerId": 1,
    "firstName": "Matt",
    "lastName": "Lewis"
}
```
</td>
<td>
```json
{
    "customerId": 1,
    "nickname": "home",
    "houseNo": 20,
    "street": "Bute Street",
    "city": "Cardiff"
}
```
</td>
</tr>
</table>



Where this data needs to retrieved together, you can use an inner join query using the explicit INNER JOIN clause as follows:

```
SELECT * FROM Customer AS c
INNER JOIN Address AS a
ON c.customerId = a.customerId
AND c.customerId = 1
```

This also works using the implicit syntax as follows:

```
SELECT * FROM Customer AS c, Address AS a
WHERE c.customerId = a.customerId
AND c.customerId = 1
```

The alternative approach would be to nest the Address document within the Customer table as follows:

<table>
<tr>
<td> <b>Customer Table</b> </td>
</tr>
<tr>
<td>  
```json
{
    "customerId": 1,
    "firstName": "Matt",
    "lastName": "Lewis",
    "address": {
        "nickname": "home",
        "houseNo": 20,
        "street": "Bute Street",
        "city": "Cardiff"
    }
}
```
</td>
</tr>
</table>


This approach is identical where it is a one-to-many relationship e.g. where a customer has multiple addresses. These could be modelled with multiple documents in the Address table referencing the same Customer id, or by an array of nested Address documents.

<table>
<tr>
<td> <b>Customer Table</b> </td>
</tr>
<tr>
<td>  
```json
{
    "customerId": 1,
    "firstName": "Matt",
    "lastName": "Lewis",
    "address": [{
        "nickname": "home",
        "houseNo": 20,
        "street": "Bute Street",
        "city": "Cardiff"
    },{
        "nickname": "work",
        "houseNo": 21,
        "street": "Bute Street",
        "city": "Cardiff"        
    }]
}
```
</td>
</tr>
</table>

In order to work out which is the right approach, there are a number of important factors to take into consideration:


#### Transaction Timeout
In QLDB, every PartiQL statement (including every SELECT query) is processed in a transaction and is subject to a transaction timeout limit. A transaction can run for up to 30 seconds before being committed. After this limit, QLDB rejects any work done on the transaction and discards the session that ran the transaction. This limit protects the service's client from leaking sessions by starting transactions and not committing or canceling them.


#### Types of queries

You need to look at how many and what type of reads the application makes. If an application needs to frequently retrieve related data, then it makes sense to store this data in the same document. This means applications make fewer queries. Nesting documents provides data locality for these queries. In the example with Customer and Address, not only does it result in fewer queries, but the initial insert or update can be carried out in a single database interaction. If your application has a document-like structure which is typically all loaded at once, it is likely a good idea to embed the related data in the same document. The relational technique of shredding - splitting a document-like structure into multiple tables - can lead to unnecessarily complicated application code.


#### Use of Indexes

Indexes are strongly recommended to improve query performance. You can currently create up to 5 indexes on a table. Indexes can only be created on a top level field, and not on a field in a nested document. If you have a specific requirement to query against attributes in a nested document, for example, to pull back all home addresses, you would use syntax such as:

```
SELECT a.houseNo, a.street, a.town
FROM Customer AS c, @c.address AS a
WHERE a.nickname = 'home'
```

The `@` character is optional but explicitly indicates that it is the `address` structure within `Customer`. In this example, the query cannot use any index, and will perform a full table scan. Table scans can cause transaction timeouts for queries on large tables or queries that return large result sets. 

Storing addresses in their own table would enable indexes to be created. However, the example above would be considered querying on a low-cardinality field, which can produce large result sets more likely to result on transaction timeouts or cause unintended OCC conflicts. These are the types of queries that streaming to a specific analytics solution is recommended.

> As a best practice, you should run statements with a WHERE predicate clause that filters on an indexed field or a document ID. QLDB requires an equality operator (= or IN) on an indexed field to efficiently look up a document.

Currently, QLDB only supports conjunctions and not disjunctions. This leads to the following optimised query patterns:


``` yaml
--Indexed field (VIN) lookup using the = operator
SELECT * FROM VehicleRegistration
WHERE VIN = '1N4AL11D75C109151'

--Indexed field (VIN) AND non-indexed field (City) lookup
SELECT * FROM VehicleRegistration
WHERE VIN = '1N4AL11D75C109151' AND City = 'Seattle'

--Indexed field (VIN) lookup using the IN operator
SELECT * FROM VehicleRegistration
WHERE VIN IN ('1N4AL11D75C109151', 'KM8SRDHF6EU074761')

--Document ID (r_id) lookup using the BY clause
SELECT * FROM VehicleRegistration BY r_id
WHERE r_id = '3Qv67yjXEwB9SjmvkuG6Cp'
```

> Any query that doesn't follow these patterns invokes a full table scan. Table scans can cause transaction timeouts for queries on large tables or queries that return large result sets. 

```
--No predicate clause
SELECT * FROM Vehicle

--COUNT() is not an optimized function
SELECT COUNT(*) FROM Vehicle

--Low-cardinality predicate
SELECT * FROM Vehicle WHERE Color = 'Silver'

--Inequality (>) does not qualify for indexed lookup
SELECT * FROM Vehicle WHERE "Year" > 2019

--Inequality
SELECT * FROM Vehicle WHERE VIN LIKE '1N4AL%'

--Inequality
SELECT SUM(PendingPenaltyTicketAmount) FROM VehicleRegistration
WHERE ValidToDate BETWEEN `2020-01-01T` AND `2020-07-01T`

--No predicate clause
DELETE FROM Vehicle

--No document id, and no date range for the history() function
SELECT * FROM history(Vehicle)

--Multiple indexed fields (VIN, LicensePlateNumber) lookup using the OR operator
--Disjunctions not currently supported 
SELECT * FROM VehicleRegistration
WHERE VIN = '1N4AL11D75C109151' OR LicensePlateNumber = 'LEWISR261LL'
```

#### Document Size

There are a number of [quotas and limits in QLDB](https://docs.aws.amazon.com/qldb/latest/developerguide/limits.html). The current maximum document size is 128kb. This means that if you have a one to many relationship which is unbounded, such as between a Customer and Order, there is a danger  this limit could be reached, and so should be avoided.


#### Event-Driven

QLDB supports event-driven workloads through [QLDB Streams](../qldb-streams/). QLDB Streams is a feature that allows changes made to the journal to be continuously written in near real time to a destination Kinesis Data Stream. Consumers can subscribe to the stream, and take appropriate action. For example, downstream systems can be notified that an order has been created, and create an invoice, prepare for dispatch or trigger further actions. QLDB Streams carry the full state of a document revision in the events that are streamed, using the [Event-Carried State Transfer pattern](https://martinfowler.com/articles/201701-event-driven.html).

QLDB Streams emit a separate REVISION_DETAILS record for each table that has been amended. This means that if related data are modelled in separate tables, additional work is required by the consumer.


## More information

Much of the information on this page is taken from [optimising query performance](https://docs.aws.amazon.com/qldb/latest/developerguide/working.optimize.html) in the AWS developer guide.


