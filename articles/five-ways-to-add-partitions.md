---
title: Five ways to add partitions
date: 2019-06-29
author: Theo Tolv
---
# Five ways to add partitions

Partitioned tables are great for optimizing performance and cost, but the question everyone has after they've created their first partitioned table is: _how do I add partitions, and how do I keep a table up to date as I add new data?_

Coming from traditional databases to Athena can be a little bit confusing. In most databases you add data and query it through the same interface. The database knows when you've added data, and it is (more or less) immediately available for queries. Athena is different, it's just a query interface to data that is organized and managed elsewhere – data is on S3, and Glue stores the metadata that describes tables, partitions, and their relationships to the data on S3. These three components are not kept in sync, and when you add data to S3 that is not in a location of an existing partition you have to tell Glue about it, so that Athena can find it when you run your next query.

There are at least five ways you can add partitions to Athena. Technically, all but one boil down to the same thing under the hood, but they can still be more or less convenient to use in different situations. Here are the five:

* Partition projection
* Using SQL with Athena
* MSCK REPAIR TABLE
* Using the Glue Data Catalog API
* Using a Glue Crawler

## Partition Projection

_When to use:_ almost always – but dates can be a bit wonky, and partition keys with values that are not known in advance work only for some use cases.

[Partition Projection](https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html) is at the time of writing a brand new feature, and the only one that doesn't involve the Glue Data Catalog API. It's also the most convenient of all of them, since you don't have to keep it up to date when you add new data on S3.

Partition Projection is a configuration on a table that tells Athena how to figure out what partitions could exist on S3, and where they are located. You describe ranges or enumerations of values, and Athena uses this information to generate the partitions relevant to the query, and their locations.

For example, you could configure a table such that the partition key `start_time` is in the range `2020-01-01,NOW`. When you run a query that includes `WHERE start_time BETWEEN '2019-09-01' AND '2020-03-05'` Athena knows that there won't be any data for the dates before 2020-01-01, and it also knows where to find the data that does exist. If the query would include dates that are after today's date, it would also know that it can skip those. Ranges of numbers work similarly, and you can also configure a partition key as an enumeration of values.

The support for date ranges can be confusing, though. If your partitions aren't ISO formatted dates like in the example, but perhaps formatted as path components, like the output from Kinesis Data Firehose; `…/2020/06/26/10/…` you must format dates in your queries to this format, e.g. `WHERE delivery_time = '2020/06/26/10'`. If your date is spread out over multiple partition keys, like many examples in the Athena documentation, e.g. `year=2020/month=06/day=26`, you will either have to configure each partition key as a integer range, or combine the whole thing into one partition key and write your queries like `WHERE dt = 'year=2020/month=06/day=26'`.

When a table has a partition key that is dynamic, e.g. an ID or other value that has many values that are not known in advance, you can still use Partition Projection if all queries include explicit values. You can specify a partition key as "injected", and Athena will use the value in the query to find the partition on S3. For example, if you have a `device_id` partition key Athena can figure out which data to include in a query that looks like `WHERE device_id in ('123', 'abc')`, but not in `WHERE device_id > '123'`.

## Using SQL with Athena

_When to use:_ interactive use and applications that use JDBC – less good when you want to write tests, or avoid multiple API calls in Lambda.

Athena inherits its partition management syntax from Hive, using `ALTER TABLE ADD PARTITION` and `ALTER TABLE DROP PARTITION` you can add and remove one or more partitions in a fairly compact way. All you need is the partition values and the corresponding locations.

When you use these commands, Athena translates them into API calls to the Glue Data Catalog API under the hood. In fact, and as you will see in the next section, it also greatly simplifies the process of adding partitions compared to using the API directly.

With this in mind, you may ask when you shouldn't use this alternative. One reason is of course that you could be using Partition Projection, but if that is not an option, there are also other drawbacks:

* I'm a TDD enthusiast, and I find writing tests for code that generates SQL to be difficult and error prone. You end up either over- or under-specifying things, and it gets really messy. In general, code that generates code is not great to work with.
* DDL statements like adding partitions run the same way as queries in Athena, you start them, and then you need to poll until they complete, and check if there were any errors. DDL statements often finish quickly, but you always have to run two API calls, `StartQueryExecution` and `GetQueryExecution`. In an environment like Lambda where you pay for the time your code is running you really want to avoid waiting, and extra network calls, plus you will need extra code to handle the two calls.

## MSCK REPAIR TABLE

_When to use:_ interactive use only with limited number of partitions – too slow and inefficient for anything else, and only works for some data sets.

This command is often used in examples, and while it works, it really only works when you use [Hive style partitioning](/articles/hive-style-partitioning) and have few partitions. With more partitions it will take a long time to run, and can even time out.

If you've just created a table and have a couple of tens of partitions with a couple of files each, it can be a convenient way to get these loaded without having to write a long `ALTER TABLE ADD PARTITION` statement. For all other cases you are better off writing a script that lists S3 and generates the SQL.

See [`MSCK REPAIR TABLE`](/articles/msck-repair-table) for a longer discussion about the command.

## Using the Glue Data Catalog API

_When to use:_ automated tools – but very tricky to get right, and can be very verbose.

Reading the Athena documentation you may be surprised to know that there are other ways to manage partitions than SQL and Partition Projection – also if you come from Redshift or other data warehousing services where all interaction with the service happens through SQL you might not even realise that there a whole other service behind Athena that manages the metadata about tables and partitions.

When you run a query in Athena, it looks in the Glue Data Catalog for metadata about the tables and partitions involved in the query. Everything you write in a `CREATE TABLE` statement ends up somewhere in a data structure in Glue. Glue is meant to be a universal data catalog – it can be used by Spark, Hive, or other services running in Elastic MapReduce, in addition to Athena. However, while there are some interoperability, it's not as easy as the documentation suggests to create tables that work in multiple services. Just creating a table that works in Athena can be a challenge.

Adding partitions is done using either [`CreatePartition`](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-api-catalog-partitions.html#aws-glue-api-catalog-partitions-CreatePartition) or [`BatchCreatePartition`](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-api-catalog-partitions.html#aws-glue-api-catalog-partitions-BatchCreatePartition), and there are corresponding calls for removing partitions again.

If you are used to adding partitions using `ALTER TABLE ADD PARTITION` in Athena you will be surprised to know that to add a partition using the Glue Data Catalog API you need to repeat almost everything that you specified when you created the parent table. The input and output formats need to be specified, the serde information, as well as all column names and types. Getting any of these wrong so that they don't match the table's metadata means that queries will most likely result in errors.

Why would you want to use this way of adding partitions, if it's so finicky and verbose? In my experience, the verbosity isn't a big deal in itself, the code that generates the storage descriptor can often be shared between creating the table and adding partitions. In cases when the table is created in one process and partitions are added in another, you can do a [`GetTable`](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-api-catalog-tables.html#aws-glue-api-catalog-tables-GetTable) call and copy the storage descriptor that way.

The benefit of using the Glue Data Catalog API directly is that it's fast and synchronous. It's much easier to write tests for code that uses the AWS SDK than code that generates SQL, and you get type checking and code completion and all other bells and whistles that come with using an SDK.

## Glue Crawlers

_When to use:_ as a last option when all other options are inconvenient – not as general or powerful as advertised.

AWS answer to the question posed in the beginning of this article, "how do I keep a table up to date…" is [Glue Crawlers](https://docs.aws.amazon.com/glue/latest/dg/add-crawler.html). A crawler discover the file types and schemas of a data set on S3, create tables, and keep those tables in sync as data is added.

Crawlers are meant to figure everything out for you. When you have a pile of data that you want organized, the idea is that you use a crawler to go through the pile and organize it into tables with usable schemas and meaningful partitioning schemes, and leave you with something you can start running queries against.

The problem is that crawlers try to be very general, with very limited configurability. Unless your data set is fairly well organized to begin with you are probably going to end up with something that is messy and only half works – or something that works for a while and then stops working. The reality is that your data set needs to be well organized and needs a fairly fixed schema for Glue Crawlers to work, but if that's the case, any of the options above is probably going to serve you better (with the exception of `MSCK REPAIR TABLE`).

If your data uses [Hive style partitioning](/articles/hive-style-partitioning), and it's schema doesn't evolve in drastic ways, you can probably use a crawler. There are definitely cases when it's less work to set up a crawler than, for example, creating a Lambda function that does a Glue Data Catalog API call in response to an S3 notification. In all my time working with Athena I have not found a case where Glue Crawlers felt like the right solution. I have used them, before [Partition Projection](https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html), as the least complicated way to keep tables for my [Cost and Usage Reports](https://docs.aws.amazon.com/cur/latest/userguide/use-athena-cf.html) up to date, for example.

There is an endless stream of questions on Stack Overflow from people who have problems getting Glue Crawlers to work for them. When the use case doesn't fit what Glue Crawlers were designed for (an unfortunately not publicly defined scope), you get surprising results like [thousands of tables being created](https://stackoverflow.com/questions/54332699/aws-glue-crawler-need-to-create-one-table-from-many-files-with-identical-schemas), [unusable tables](https://stackoverflow.com/questions/46241088/how-to-create-aws-glue-table-where-partitions-have-different-columns-hive-par), [table schemas flip-floping](https://stackoverflow.com/questions/61297671/querying-optional-nested-json-fields-in-athena), and so on, and so forth.

## Summary

In almost all situations, Partition Projection is the most convenient way to work with partitioned tables. It's simple configuration on a table that will not have to be kept up to date, or externally managed.

In situations where Partition Projections can't be used, there are multiple ways to add partitions to a table. All of them use the Glue Data Catalog API, either directly or under the hood. When using Athena interactively the most convenient way is to use `ALTER TABLE ADD PARTITION` statements, but if you are writing code and automate adding partitions, using the Glue Data Catalog API directly is faster and more testable.

Other alternatives like `MSCK REPAIR TABLE` and Glue Crawlers, that often come up in discussions about how to manage partitioned tables, should be used only if all other alternatives are more inconvenient.

Partition Projection is a new feature, and the available documentation is limited. I will write more articles that cover it in detail. While it solves almost all use cases that previously required a lot of code to handle, there are still cases where managing partitions the old way will be required, and I will also write more content on that.
