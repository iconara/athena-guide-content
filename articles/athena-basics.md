---
title: Athena basics
date: 2020-08-21
author: Theo Tolv
---
# Basic concepts in Athena

Athena is not an RDBMS or a data warehouse appliance, instead it's a query engine. Unlike most databases Athena doesn't manage the storage of your data, it just provides an interface for querying data stored on S3. In this way it is more like Spark than it is like Redshift. While most of your experience with SQL and databases is transferrable to Athena, there are a few things that are distinctly different and that you need to keep in mind to get the most out of it.

## The origins and components of Athena

You've probably heard that Athena is built on the open source query engine [Presto](https://prestosql.io). Unlike AWS' other similar services like their managed ElasticSearch, Kafka, and Cassandra offerings, Athena is not just a managed Presto cluster. In fact, AWS has another service that's is closer to managed Presto: Elastic MapReduce (EMR), which can be configured to run Presto clusters.

The fact that Athena is based on Presto shines through here and there, and AWS makes no secret of it – but for the most part Athena is Athena more than it is Presto. A big part of that is that it's an AWS service much more than hosted software. Instead of running a Presto cluster for you, it's a completely serverless setup where you just submit your queries, without worrying about anything else. It's also much more integrated into the AWS ecosystem than Presto is out of the box, but of course, also more limited – one USP of Presto is that it can connect to a lot of different backends, for example joining tables in MySQL with data in Google Sheets, while Athena only works with data on S3. _There is currently a preview of a feature called [Athena Federated Query](https://docs.aws.amazon.com/athena/latest/ug/connect-to-a-data-source.html) that makes it possible to query other backendds, but requires a lot of work on your part and is completely different from Presto's connectors_.

Athena is based on a fairly old version of Presto – 0.172 at the time of writing this, while the latest version is 340 (they dropped the "0." part between 0.215 and 300). However, the Athena team backports fixes, and passes their own fixes upstream, and the version number is probably useful mostly for looking up the documentation for Presto SQL syntax and functions at this point.

Athena was initially launched with its own data catalog, the component that keeps track of your databases and tables, but when Glue was launched it migrated to using Glue Data Catalog. Using Athena is really using three distinct AWS services; Athena itself, Glue, and S3 – and also IAM for permissions (there is also Lake Formation, but that's a topic for another post). This is something that you are aware of at almost all times, and something that trips up many new users, especially when it comes to permissions. There's not that many other AWS services where you need to know a lot about other services to use it.

## What problems does Athena solve

I like to say that Athena activates all the data you already have on S3. Most companies that have used AWS for a while have probably accumulated a lot of random data on S3, logs of different kinds, backups, user data, etc. Perhaps you call this your "data lake", like so many startups do to inflate their valuation and enterprises do to seem to be on top of things. Whatever your pet name, before Athena you would either run jobs in EMR or on some home grown ETL "platform" to crunch the data into some useful format, which you stored in Redshift, some other RDBMS, or as new files on S3. There's a ton of tools in EMR for this, some of them are even good – but running clusters 24/7 is expensive, while running on demand is finicky, especially if you want to maintain state between runs.

Athena is an always-on query interface to all that data on S3, it makes it available and accessible, without hassle and with a fairly simple pricing model. Athena gets you quite a long way towards that dream of a data lake where the data you already have can be queried and combined without a lot of preprocessing or complex ETL.

Athena makes the data you have accessible and useful without forcing you to spend a lot of time making it accessible and useful.

While Athena is primarily a query engine, it has grown a few ETL features since its launch. You can [create new tables from query results](https://docs.aws.amazon.com/athena/latest/ug/ctas.html), which means that you don't need to use another service like EMR or Glue ETL for anything but the more complex ETL tasks. If all you want to do is some cleaning, transformation, and data format conversion, Athena and a little bit of management code will take you a very long way.

When you run a query in Redshift or almost any other database you need to stick around until the response is ready, because their protocols are stateful. If you disconnect your query might get cancelled, and at least you'll have no way to retrieve the results. In contrast, Athena's API is asynchronous; when you start a query you get a query ID, which you can use at any point in time (up to a couple of months later) to check the query status. This means that your code doesn't need to run while Athena is processing a query – you can start a query as a reaction to some event, put the ID somewhere and at a much later point retrieve the results. This fits really well into the event driven serverless model where your code, for example a Lambda function, only runs while it's actually doing useful work. _Since writing this, the [Redshift Data API](https://docs.aws.amazon.com/redshift/latest/mgmt/data-api.html) has been launched, which provides a very Athena-like API for interacting with Redshift._

If you want a synchronous connection model you can use the [JDBC and OBDC drivers](https://docs.aws.amazon.com/athena/latest/ug/athena-bi-tools-jdbc-odbc.html) that are written on top of the stateless asynchronous API. There is also an [alternative JDBC driver](http://github.com/burtcorp/athena-jdbc/) written by yours truly that provides higher performance and fewer quirks.

I'll talk more about the pricing model below, but it's worth mentioning here too: Athena is priced very differently from services like Redshift, EMR, and Glue ETL. Instead of capacity/duration-based pricing Athena charges by the amount of data you process. For many use cases this is the problem that Athena solves: the cost of _not_ using Athena is zero. You don't have to think about how your Redshift cluster is burning a hole in your budget on weekends when no one is using it, for example.

## Key concepts

Athena does not work like the databases you might be used to. You don't load data into Athena, you create tables that describe the data you already have. The approach taken by Athena is called "schema-on-read", which means that it's only when you run a query that table schemas are taken into consideration. You can create any number of tables with whatever columns and configuration you want, it's only when you try to use them that Athena will complain if they don't match the reality of your data. At the same time, as long as some data is found in a table's location, and it kind of looks like what you've said it would look like, Athena will do something with it.

### The catalog: databases, tables, and views

Athena uses Glue Data Catalog to look up things like databases and tables. A "catalog" in the terminology used by Athena means a place where the metadata about your data is stored, things like tables that describe the data and where it can be found.

You can create catalog entries through DDL statements (i.e. SQL) through Athena, or using the Glue Data Catalog API directly. In theory, tables created in the Glue Data Catalog from Spark on EMR or Glue ETL will also be available to Athena, although the stars need to align properly for this to actually work in practice. When you run DDL statements through Athena, it translates your request into Glue Data Catalog API calls behind the scenes.

Glue Data Catalog has two types of catalog entries: databases and tables. Databases are collections of tables, and are mostly for organizing tables into manageable namespaces. Tables describe the location of data, as well as how it's stored (e.g. file format, compression, etc.), how it's organized (partitioned or not, more on that below), the properties found on data items (i.e. columns) and their types, as well as other metadata that is used to describe and interpret the data. It's mostly up to the clients of the Glue Data Catalog how to encode this information, which is the reason that interoperability is mostly theoretical – but there is enough flexibility in this model for tools like Glue Crawler to add its metadata without interfering with Athena, for example.

Athena supports a third kind of catalog entry: views. These are encoded as tables in the Glue Data Catalog, with some extra metadata that make them appear as views to Athena, but as gibberish to other clients looking in the catalog.

Glue Data Catalog doesn't validate anything you put into it, except for basic things like required fields and the length of names, and neither does Athena. Creating a database or a table doesn't do anything to the data that it describes – in fact, it doesn't even need to describe data that actually exists. You can create tables for completely imaginary data sets, or data sets you have not yet created. Catalog entries are light-weight entities that you can throw about however you want. An important consequence of this to be aware of is that unlike most other databases, dropping a table does not affect any data – you can iterate on a table schema by creating and dropping tables without fear of changing or loosing your data.

Being able to create a table for data that already exist can be mind blowing if you've not used something like Athena before, but being able to create a table for data that doesn't yet exist might not sound like much of a feature – that's how most databases work. However, I've noticed that people sometimes think it's one or the other, and that's not the case. It's really useful to be able to set up tables as part of the infrastructure – I often do it with CloudFormation and Terraform – so that once data starts coming in it's available to be queried. If I notice I've made a mistake in the schema I update the template and the stack until I've gotten it into shape.

Never worry about recreating tables if you find that you've made a mistake. Using the Glue Data Catalog API you can even update a table without any downtime at all (although be careful if the table has partitions, you might make it unusable if the partitions no longer match the table). You can iterate on a table schema until you find something that works with the data you have, instead of transforming the data to fit into a table. Athena handles a lot of different data formats and ways of organizing data, and you should almost never have to transform your data to work with Athena.

Also don't worry about having multiple tables pointing to the same data, or parts of a data set that another table points to.

### Partitions and partitioning

You might have read that Athena doesn't have any indexes. In RDBMS' indexes makes queries fast by making it possible to skip all the data that is not relevant to a query. Since it doesn't have indexes, does this mean that Athena always reads all data in a table? Yes and no.

If you create a regular table for data in a format such as CSV or JSON the answer is yes, Athena will read every single byte in the table's location for every query (unless there's a `LIMIT` clause in which case it will attempt to read only as much as it needs to return the desired number of rows, but that's a special case). If the data is in in a columnar format, e.g. Parquet or ORC, Athena can make use of metadata within the files to read only the columns referenced by the query, and skip entire blocks when possible (although in my experience this doesn't work as well with ORC files, so I stick with Parquet when I can).

Reading all the data for a table for every query can get expensive, and slow down queries, especially if the data set grows over time. Even using Parquet will not work well enough in the long run, so what to do?

The answer, as you might have guessed from the header, is partitioning. Partitioning a data set means organizing its data by one or more properties so that queries can quickly skip parts of the data set that doesn't match. A data set about sales could be partitioned by customer so that queries looking at only one or a few customers can ignore most of the data set. In practice the way you partition is simply to put data into a directory structure (prefixes for the S3 purists) where the directories are named from the property values. A very common partitioning scheme is to put data into directories named from the date so that a query looking at the most recent data quickly can skip all old data.

When you set up a table in Athena you include information about how the data is partitioned by specifying one or more partition keys, these correspond to the directory hierarchy of your data set. If the data is organized by customer and date, for example `data/acme/2020-08-01/data.json`, the partition keys are `customer` and `date`, e.g. `PARTITIONED BY (customer string, date string)`.

There are two ways 


, and also add each partition (i.e. the "physical" partitions of your data set) to the table's metadata in Glue Data Catalog so that Athena knows where to find it (see Five ways to add partitions](/articles/five-ways-to-add-partitions) for the details).


Athena will then be able to use constraints on partition keys to filter the list of partitions before it starts reading any files. For example, if your table is partitioned by `date` and `customer` and the query contains `WHERE customer = 123 AND "date" > '2020-08-01'` Athena will use this information when it lists the partitions in Glue Data Catalog, and then list the 

Larger data sets, especially those that grow over time, tend to already be partitioned. Data sets tend to be organized in a way that's easy for the producing side, for example to make it possible to replace a piece of data if it changes. [AWS CloudTrail is a good example of this](https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-logs.html), it delivers logs to S3 like this: `s3://some-bucket/AWSLogs/0123456789/CloudTrail/us-east-1/2020/08/21/0123456789_CloudTrail_us-east-1_20200821T2120Z_W8LDOrF9dU4NZ7Kt.json.gz`. This URI contains (after `CloudTrail` which is the root), the region, the year, month, and day, and the file name. This partitioning scheme makes it easy to find all files pertaining to a specific AWS region, and if you only want to query the last month, day, or hour's worth of logs a query can easily skip most of the data set.

The CloudTrail data set can be said to be partitioned by region, year, month, and day – or by region and date, you don't have to have a one-to-one mapping between partition keys and path components.

## Running queries

### Work groups

## Permissions

Permissions in Athena are managed through IAM, unless you use Lake Formation (which is a topic in itself and not covered here). As I've mentioned above, Athena is not an isolated service, and running a query involves at least three AWS services: Athena, Glue Data Catalog, and S3. This is reflected in the permissions model too: Athena is the query engine, and to be allowed to start a query you need permissions for the Athena service, but Athena will use the Glue Data Catalog on your behalf to look up table metadata, which requires permissions to the Glue service, and it will read data on S3, which of course requires the right permissions to S3.
by 
Because there are multiple services involved, IAM policies for Athena often have a lot of statements, and they can be hard to get right in the beginning. Each service has its resources and ways of specifying and limiting permissions.

Athena is probably the simplest of them, you really only need to make sure the principal (i.e. user or role) has permission to the API calls involved in running a query, which means the actions `athena:StartQueryExecution`, `athena:GetQueryExecution`, and `athena:GetQueryResults` for the [workgroup][workgroups] that the query runs in. Next is S3, where `s3:ListBucket`, `s3:GetObject` are needed for the bucket and objects that will be read. Finally, Glue's IAM permissions are probably the hardest to get right, partly because it's hard to know which API calls Athena makes behind the scenes and therefore needs permissions for, and partly because Glue requires you to specify permissions on all levels of its catalog hierarchy – granting permission to a table is not enough, you also need to grant permission to the database the table is in, and the catalog the database is in. Luckily the [Athena documentation has example policies for the most common use cases](https://docs.aws.amazon.com/athena/latest/ug/fine-grained-access-to-glue-resources.html).

The permissions model is far from perfect, and it has a very steep learning curve, but I think there are benefits to it too. I like that it's transparent that Athena uses the other two services, and that it makes the API calls to them in the same way, with the same permissions, as if the principal had done it themselves – and that it shows up in CloudTrail in that way too.

calledVia https://aws.amazon.com/blogs/security/how-to-define-least-privileged-permissions-for-actions-called-by-aws-services/

## Pricing model

## Serverless and multitenant


  [workgroups](https://docs.aws.amazon.com/athena/latest/ug/manage-queries-control-costs-with-workgroups.html)