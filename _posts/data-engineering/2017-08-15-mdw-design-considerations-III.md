---
layout: default
title: Design Considerations - Query Engines
description: Design considerations for modern data warehouses - Part III
categories: [data-engineering, analytics]
---

## Design Considerations III - Query Engines

This post is part of a [Series]({% post_url /data-engineering/2017-08-15-mdw-storage-query-engine-separation %}) on Modern Data Warehouse Architectures and was written in collaboration with with [Will Fleury](http://www.willfleury.com/).

### Query Engines

#### Overview

As we have mentioned numerous times in this series, the goal of the data architecture we discuss in this series is to remove the burden of having to pick a single query engine or analytics tool. We need the ability to switch between engines as the use case arises and technology advances. If engine A isn’t working well for this particular workload and engine B is better suited, with the storage and query concerns separated, it’s easy switch or use both. When the storage and query engine are collocated (such as [Vertica](Vertica), [Oracle](https://www.oracle.com), etc) it becomes much more difficult and orders of magnitude more time consuming and expensive to test alternative query engines and is typically not even considered an option. This severely restricts business flexibility. 

There are a baffling number of database solutions out there. It is impossible to give a thorough review of each of them or say which is the best for your particular use case. This said, we will provide an overview of some points to be aware of when looking at a suitable engine. 

#### Multi Data Source Support

While the general goal of a Data Lake is to contain all of your data in one location, there are times when some data such as application metadata or inventories are located in different systems or storage. The ability to connect to these disparate systems and storage and join with our main data warehouse is very valuable. Take something as simple as the metadata for an Organisation identifier. Instead of maintaining a copy of this data in the data lake, some tools allow reading directly from the metastore which holds this data and joining with behavioural or transactional data stored in the Data Lake. This completely removes some pointless [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load) tasks and data synchronization issues. The data source may even be another [AWS S3](https://aws.amazon.com/s3/) bucket. Tools such as [Spark](https://spark.apache.org/), [Drill](https://drill.apache.org/), [Impala](https://impala.apache.org/) and [Presto](https://prestodb.io/) support many data sources. In many organisations, such support can remove thousands of unnecessary ETL tasks and synchronization headaches. 

Naturally one must be careful with an analytics engine talking directly to an application database, and the correct controls should be in place. However, correct usage means there is no more strain placed on the database than an ETL job to extract changes. We are talking about application metadata here, not transactional or behavioural data and so the read volumes are small (potentially to the extent that it should constitute a broadcast join in most engines). 

#### Schema Support

Up front schema modelling of data warehouse solutions is no longer a mandatory task. Many tools are built to infer the schema from the data sources they are pointed to. Drill in particular is built with this in mind and is marketed as "schema free SQL". This means it can infer the schema by either analysing a sample of the files (such as JSON), or by examining some other schema information collocated with the data files (such as [Parquet](https://parquet.apache.org/), [Avro](https://avro.apache.org/), [ORC](https://orc.apache.org/) schemas etc). [Spark](https://spark.apache.org/) and many other tools can perform similar schema inference and merging. 

When it comes to analytics, we feel that the data format used should be schema based. Most efficient data formats require a schema and all columnar formats we are aware of require one. However these schemas, and how they evolve are very different to a traditional schema design for a data warehouse application. There is no need for a team of business analysts to spend weeks, months or years designing a data model **up-front**, which analysis has shown will suffice for all requirements. Many times the analysis is out of date before it has been implemented. This form of up-front schema modelling is not scalable and should simply be forgotten about. We do not want 6 month data modelling and new ETL process turnaround times for new use cases. 

As some storage formats we have discussed allow for storing **all** the data received using the same schema it was received with, there is no need for a separate schema modelling task. The only conversion that takes place is related to the storage format. For instance, if an upstream system is sending order records in avro data format, we would ensure that when we ingest the records into the data lake, they are written in parquet format, but with the same schema as the avro record. Similarly, if an upstream system was sending JSON records, we would write them in parquet and ensure we produced the corresponding parquet schema to match (extra care is required when dealing with JSON as backward incompatible changes can be introduced and more strict schema evolution and validation procedure should be employed). As the schema for the parquet files is actually written alongside the data files, we have an automated schema management solution also. This collocation of the schema and data is important. 

The benefits of this approach should be apparent. We do not require expensive schema analysis and the schema management process is essentially automated. We have all of the data we received stored and available to query. That means we are never "missing" an attribute when we look to deliver a new use case and can skip the dreaded process of “adding” new attributes to our data warehouse. New attributes and entities all naturally flow from their sources. This does of course mean that validation at source is important in many cases, as we are trusting that new attributes we encounter are to be automatically added and queryable in our data warehouse. Hence, it is desirable to have schema validation performed at the point of capture (API) where possible and feasible. Where not feasible, adequate automation rules should be in place to deal with nonconforming data (e.g. where an existing attribute changes data type etc). 

A very good read on this topic and others which one should be aware of called "The Seven Tenets of Data Unification" from Michael Stonebraker is available [here](https://www.tamr.com/landing-pages/dr-michael-stonebrakers-7-tenets-data-unification-3/). Section 2.2 is especially applicable to what we have just discussed.

#### Nested Structures

We have discussed some of the benefits of nested structures already. We must be able to then query this data. The first time we came across the concept of querying nested structures within an SQL environment was when using [BigQuery](https://cloud.google.com/bigquery/). While slightly confusing at the beginning, we noticed how much safer from a data lineage perspective they are when compared to more classical data storage patterns, where the data belonging to a single logical entity is spread across multiple entity tables. There are varying levels of support and varying syntax for working with and "unflattening" such nested structures. In particular for dealing with nested arrays of structures (e.g. order items). Hive and Spark call this a `LATERAL VIEW EXPLODE` operation while other engines such as Presto and Impala call it an `UNNEST` operation. 

One can then easily create a "view" in the given query engine to provide access to the child entity as if it was denormalized into its own table. Taking the order items example

```sql
CREATE VIEW orderItems AS 
    SELECT T.* FROM orders 
        CROSS JOIN UNNEST(orderItems) AS T
```


From an analytics perspective, there are some important steps to consider when querying nested data structures which we have covered in a previous [post](https://docs.google.com/document/d/1v2X0JqA66vCJUBRmCW3coFya2oSGe5G4C-ioEqQfWoU/edit). 

One additional point to be aware of with Parquet format and nested structures, is that not all query engines support projection and predicate pushdown past the first level of nesting. This is an implementation detail and open issue with both [Presto](https://github.com/prestodb/presto/issues/2508) and [Spark](https://issues.apache.org/jira/browse/SPARK-4502). However, Presto supports predicate push down on nested items while Spark does not as of the time of writing. As with everything, check the support and validate it with the given version of the tool you are thinking of using. 

#### ANSI SQL

Reproducibility of the analysis over time is a key factor in data science. As data engineers and data scientists, we want to be able to rerun the same logic on the same data structure, on a different database engine, with no change if possible.

Given the current trend towards separating the storage from the query engine, using standard ANSI SQL becomes even more important. It allows for switching from one query engine to another by simply launching the respective cluster. As there is no need to move any data into the cluster as both can query it where it is stored, such as on S3, we can immediately run the same queries on both engines. The power of this cannot be underestimated. Different query and execution engines will have different strengths and this ability to switch between is priceless. It also reduces lock in over time which can make changing engine extremely expensive, quite often prohibitively expensive in the analytics space.

Most engines have additional support for different types of SQL based analysis than that supported by pure ANSI SQL. While these can be useful for certain use cases, we feel their usage should be kept to a minimum. One clear exception to that is the UNNEST statement (or equivalent) which is used to work with nested structures. Of course there are many ANSI SQL standards from SQL-86 to SQL-16. Which level you will need to look out for is based on your requirements but at a minimum, one should look for SQL-03 when window functions were added to the standard.

#### Query Isolation

Some engines such as Spark are not designed for managing multiple concurrent queries. Apart from resulting in very poor concurrent query performance, it also means they have some rather nasty failure semantics for rogue queries. A rogue Spark query will take down the entire cluster. Basically there is no isolation. Considering the typical use case for Spark is ETL this makes sense, but for an SQL engine it is a rather unpleasant feature. Contrast that to engines such as Presto which is designed for many concurrent, interactive queries, and the isolation is on the individual query. Which is best will again depend on the use case. 



