---
layout: default
title: Why Separate Storage and Compute?
description: Why separate storage and compute (query) concerns for a modern data warehouse architecture?
categories: [data-engineering, analytics]
---

## Why Separate Storage and Query Engine Concerns?

This post is part of a series on Modern Data Warehouse Architectures and was written in Collaboration with [Will Fleury](http://www.willfleury.com/).

### Overview

Over the last number of years, there has been a clear trend in the separation of the storage and query layers for data warehouse solutions by web scale organisations. These include but are not limited to Google, Facebook, Netflix, Amazon, etc. The progress and penetration of cloud providers has also greatly facilitated the adoption of such solutions by increasing the availability to cheap on-demand compute resources, providing essentially infinite scale object storage solutions such as [AWS S3](https://aws.amazon.com/s3/) and [Google Cloud Storage](https://cloud.google.com/storage/) and lightning fast internal network speeds. There are a number of reasons why we believe the time is right for many organisations to start considering such an architecture more seriously which we will discuss in this post.

The business drivers for such an architectural change arise from the continuing need to increase the flexibility of analytics solutions, reduce data model fragmentation which in turn reduces expensive and error prone [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load) stages, and at the same time reduce operational overhead and costs. For years, there was only ever a single analytics data warehouse ([OLAP](https://en.wikipedia.org/wiki/Online_analytical_processing)) solution in organisations. This was fraught with issues when it comes to web scale companies and is discussed in great detail in this O’Reilly article - "[In Search Of Database Nirvana](https://www.oreilly.com/ideas/in-search-of-database-nirvana)". While many organisations have adopted a more modern approach to managing data, there is still typically a single centralised OLAP system which is far too slow and expensive when it comes to adapting to new use cases. While there is no such thing as one size fits all in the world of big data analytics, there are solutions which can greatly help reduce the burden of providing and running an analytics solution at scale. At the same time, such solutions can provide the flexibility to use the right tool for the job, without an expensive multi-year project to simply change your query engine or add an additional analytics capability. 

The [NoSQL](https://en.wikipedia.org/wiki/NoSQL) trend which spiraled out of control and dominated big analytics for a number of years has died down and while many NoSQL solutions are still ideally suited to certain use cases, they are not being treated as the jack of all trades anymore. Similar to previous generation systems, most (if not all) of these solutions relied on custom storage formats. While these formats helped them meet certain goals, they were often extremely difficult to work with and required managing many database systems with their own storage and operational overhead. The ETL and operational overhead of keeping all of these databases in sync became a tortuous experience for everyone involved. The trend has now turned to providing many different query engines on top of a single storage format, thereby solving the one size fits all problem of the end users but unifying on a suitable storage solution for these disparate tools. 

The return of ANSI SQL compatible query engines is extremely pleasing to witness and allows users to once again start unifying analytics capabilities across solutions without product or vendor lock in. We distinguish ANSI SQL compatible query engines such as [Presto](https://prestodb.io/), [Impala](https://impala.apache.org/), etc from the [NewSQL](https://en.wikipedia.org/wiki/NewSQL) solutions such as [MemSQL](http://www.memsql.com/), [VoltDB](https://www.voltdb.com/), [Spanner](https://cloud.google.com/spanner/), etc which require caution. As we will discuss later on, some of the NewSQL solutions are attempting to fulfill the [one-fits-all](http://dl.acm.org/citation.cfm?id=1054024) systems which tend to end badly. That said, while researching for this blog series, we discovered some very interesting changes being introduced into Google Spanner later this year (see the [blog](https://cloudplatform.googleblog.com/2017/06/from-NoSQL-to-New-SQL-how-Spanner-became-a-global-mission-critical-database.html) entry and the corresponding SIGMOID ‘17 [publication](http://dl.acm.org/citation.cfm?id=3056103)). We feel this deserves its own discussion based on the views we put forward in this series. Therefore, we will review Spanner separately at the end of the series.

There are many decisions that one must make when introducing this new type of architecture and there are already many implementations to choose from. In this post we provide an argument for choosing such a data warehouse architecture, along with an overview of some of the driving factors and decisions which need to be made along the way. In subsequent blog posts we will delve into more technical details on many of these decisions and how to design data pipelines to deliver such a solution. 

### Why Separate Storage and Execution?

Historically, both storage and execution (query) layers have been tightly coupled into a single database system such as [Oracle](https://www.oracle.com/database/index.html), [DB2](https://www.ibm.com/analytics/us/en/db2/), [Vertica](https://www.vertica.com/), [MySQL](https://www.mysql.com/), etc. Each has their own storage format for myriad reasons from vendor lock in, to specific query engine optimisations to simply no need or desire to standardise. Many of these tools did not, and still do not support many of the concepts we take for granted when dealing with big data. That is the ability to scale out horizontally. Queries should be answerable by distributing the data and the workload across many commodity machines. This distribution of the data and query across a cluster is generally known as [MPP](https://en.wikipedia.org/wiki/Massively_parallel) (Massively Parallel Processing) in the database world. Vertica is an example of an MPP solution. Database Warehouse Appliances (DWA) were commonly required for such MPP solutions and vendors such as Teradata, Oracle were happy to supply both the engine and the specialised DWA hardware required to run it at extortionate prices. 

The data in these solutions and architectures was always co-located with the engine. Traditional databases, even when running on distributed filesystems such as [NFS](https://en.wikipedia.org/wiki/Network_File_System), create narrow interfaces, where the only way to get at the data is through the database interface.  However, very often people need to access the data in different ways and it is difficult for a single solution to specialise for each. This meant that the use cases which could be built from this data were very limited and often painful to create. When web scale companies started appearing, these problems were amplified by orders of magnitude. Even with solving the data capture scaling issues, the solutions that existed simply did not allow access to and use of the data they had captured in an efficient and cost effective manner. This is around the time map-reduce (via [Hadoop](http://hadoop.apache.org/)) started to appear. It allowed to distribute and run any kind of algorithm (execution) against the data stored a variety of formats in HDFS. 

In many ways, MPP and Hadoop are similar concepts. There is a lot of fuzziness in various articles when it comes to describing what MPP is and how it is different to Hadoop or SQL on hadoop solutions. There are many articles which try and compare the two. Potentially, the distinction was made with Hadoop v1, before the YARN resource manager appeared in Hadoop v2. However we have been unable to determine any clear difference in the underlying computational concept. MPP is typically used to refer to database engines, where they are answering structured SQL queries, whereas map-reduce may refer to any distributed computation running a potentially more unstructured data task or a structure SQL query. Some authors claim MPP means data partitioning and shared nothing architecture (disk, cpu, network, etc). These concepts can also be easily delivered on Hadoop which means we could view MPP functionality as a subset of Hadoop when framed in this way. In fact solutions such as [HAWQ](http://hawq.incubator.apache.org/docs/userguide/2.2.0.0-incubating/overview/HAWQArchitecture.html) claim to do just this. For this reason, in the remainder of this article, when we mention MPP, it can be a custom distributed database and engine, an engine running on Hadoop or any other distributed architecture. 

Hadoop has no doubt been the biggest player in the Big Data ecosystem over the last 6 years. Hadoop itself was based on a [paper](https://research.google.com/archive/mapreduce.html) by Google which described the map-reduce architecture they had been using for half a decade. It was designed to distribute processing on commodity hardware instead of expensive custom hardware that many of the existing MPP solutions required. At the beginning, data locality was a very important feature in this architecture and so the architecture naturally had a strong focus on collocating the execution with the storage. Even though this co-location was an original design decision, Hadoop marked the beginning of separating the execution layer from the data layer. As more and more execution frameworks were written to run on top of YARN, the conceptual separation grew larger and larger. Combined with the ability to read data over the network as fast as, and in some setups faster than from a local disk, the importance of data locality started to fade. While locality is still critical for many applications and use cases such [OLTP](https://en.wikipedia.org/wiki/Online_transaction_processing) systems, when a given problem requires the reading and processing of MBs, GBs or TBs of data, the relevance of locality diminishes so long as the network bandwidth is sufficient.  

The improvements in storage have largely been driven by faster, cheaper SSD storage and super fast networking combined with the extreme commoditization of storage by cloud providers such as AWS, Google and Microsoft. When operating within these providers data centers, the bandwidth of the network card is typically the limiting factor for read operations from the cloud storage to a compute node within the same network. This means that node level data locality isn’t so important anymore, while data center and region level is still important. This locality indifference is extremely important, because  if you must co-locate a query with the data, then there is a direct vertical scaling limit imposed on the operations on this copy of the data. In these situations, avoiding contention and starvation issues become more and more challenging. 

### The Evolution of the Data Lake

Most organisations have embraced the concept of the "Data Lake". However, the term “Data Lake” is a poorly defined concept. Does it simply mean a storage medium, where all an organization's raw data is available, transactional and behavioural? Are there any constraints on the usability and format of this data for it to be classified as a Data Lake? This lack of clarity has in many ways caused a lot of pain for organisations trying to develop Big Data strategies. They put a lot of time and money into building a Data Lake without a clear strategy for how to use it to derive valuable business insight. Most of the time, the ability to deliver this insight is treated as an afterthought, and a more classic data mart and ETL process is still used for analytics and reporting purposes. So instead of making life easier, it actually increased the complexity due to an increased number of ETL steps, each one with its own tools, schemas, headaches and inevitable drift. Of course having a data lake is important for many reasons and can still provide many benefits, but this feels fragmented. This is where the separation of storage and execution is paying dividends again and can be seen as the start of the current trend. As the Data Lake contains **all the data**, a user working against this data will never be “missing an attribute” that was captured. This means faster insight cycles and less overhead on data engineering to constantly fix and add new attributes to countless data marts and tables for each new use case. Often, this needless cycle results in use cases becoming too timely and expensive to implement. 

Another key driver towards the separation between storage and execution has been the increasing importance of AI, machine learning and data science in many organisations. This separation becomes natural in machine learning and data science because on one side they  require a unified view of the all the data, which the Data Lake can provide, however vastly different execution engines and tooling must be supported on the same data. 

Given all these reasons, the thinking has started shifting towards, "if we have all our data in the Data Lake, **how can we make the data lake queryable"**. This question is leading to a unification of storage location, schemas and formats for the data lake and an explosion of storage neutral query engines. This concept does not prevent one from keeping all of their raw data in the data lake, even under a separate namespace, it simply begins to merge the data lake and data warehouse storage concepts. 

Clearly, there are many benefits to enabling fast, interactive analytics, directly against the data lake. While there are many design decisions and implementation challenges in delivering such a solution, we feel the time is right at least start considering a transition to such a data architecture. Over the next few months, we will bring you through many problems and solutions available to solving these design decisions. 

Next in the series - [Design Considerations]({% post_url /data-engineering/2017-08-15-mdw-design-considerations %})