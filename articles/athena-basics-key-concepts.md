---
title: 'Athena Basics: Key Concepts'
date: 2020-08-21
author: Theo Tolv
---
# Athena Basics: Key Concepts

In this series of articles on Athena basics I cover the things that aren't explicit in the official documentation. I'll go beyond the bare technical details and try to explain things in more context, and how it works in practice. There won't be much in terms of code or SQL, but wherever possible I link to other articles in this guide that go into much deeper detail.

In this article I'll cover:

* [Schema-on-read](#schema-on-read)
* [The catalog: databases, tables, and views](#the-catalog-databases-tables-and-views)
* [Partitions and partitioning](#partitions-and-partitioning)

The other articles in this series cover:

* [What is Athena?](/articles/athena-basics-what-is-athena/)
* [Working With Data](/articles/athena-basics-working-with-data/)
* [Running Queries](/articles/athena-basics-running-queries/)
* [Permissions](/articles/athena-basics-permissions/)
* [Pricing Model](/articles/athena-basics-pricing-model/)

## Schema-on-read

Athena does not work like the databases you might be used to. You don't load data into Athena, you create tables that describe the data you already have. The approach taken by Athena is called "schema-on-read", which means that it's only when you run a query that table schemas are taken into consideration. You can create any number of tables with whatever columns and configuration you want, it's only when you try to use them that Athena will complain if they don't match the reality of your data. At the same time, as long as some data is found in a table's location, and it kind of looks like what you've said it would look like, Athena will do something with it.

## The catalog: databases, tables, and views

Athena uses Glue Data Catalog to look up things like databases and tables. A "catalog" in the terminology used by Athena means a place where the metadata about your data is stored, things like tables that describe the data and where it can be found.

You can create catalog entries through DDL statements (i.e. SQL) through Athena, or using the Glue Data Catalog API directly. In theory, tables created in the Glue Data Catalog from Spark on EMR or Glue ETL will also be available to Athena, although the stars need to align properly for this to actually work in practice. When you run DDL statements through Athena, it translates your request into Glue Data Catalog API calls behind the scenes.

Glue Data Catalog has two types of catalog entries: databases and tables. Databases are collections of tables, and are mostly for organizing tables into manageable namespaces. Tables describe the location of data, as well as how it's stored (e.g. file format, compression, etc.), how it's organized (partitioned or not, more on that below), the properties found on data items (i.e. columns) and their types, as well as other metadata that is used to describe and interpret the data. It's mostly up to the clients of the Glue Data Catalog how to encode this information, which is the reason that interoperability is mostly theoretical – but there is enough flexibility in this model for tools like Glue Crawler to add its metadata without interfering with Athena, for example.

Athena supports a third kind of catalog entry: views. These are encoded as tables in the Glue Data Catalog, with some extra metadata that make them appear as views to Athena, but as gibberish to other clients looking in the catalog.

Glue Data Catalog doesn't validate anything you put into it, except for basic things like required fields and the length of names, and neither does Athena. Creating a database or a table doesn't do anything to the data that it describes – in fact, it doesn't even need to describe data that actually exists. You can create tables for completely imaginary data sets, or data sets you have not yet created. Catalog entries are light-weight entities that you can throw about however you want. An important consequence of this to be aware of is that unlike most other databases, dropping a table does not affect any data – you can iterate on a table schema by creating and dropping tables without fear of changing or loosing your data.

Being able to create a table for data that already exist can be mind blowing if you've not used something like Athena before, but being able to create a table for data that doesn't yet exist might not sound like much of a feature – that's how most databases work. However, I've noticed that people sometimes think it's one or the other, and that's not the case. It's really useful to be able to set up tables as part of the infrastructure – I often do it with CloudFormation and Terraform – so that once data starts coming in it's available to be queried. If I notice I've made a mistake in the schema I update the template and the stack until I've gotten it into shape.

Never worry about recreating tables if you find that you've made a mistake. Using the Glue Data Catalog API you can even update a table without any downtime at all (although be careful if the table has partitions, you might make it unusable if the partitions no longer match the table). You can iterate on a table schema until you find something that works with the data you have, instead of transforming the data to fit into a table. Athena handles a lot of different data formats and ways of organizing data, and you should almost never have to transform your data to work with Athena.

Also don't worry about having multiple tables pointing to the same data, or parts of a data set that another table points to.

## Partitions and partitioning

You might have read that Athena doesn't have any indexes. In RDBMS' indexes makes queries fast by making it possible to find the right data, and skip all the data that is not relevant to a query. Since it doesn't have indexes, does this mean that Athena always reads all data in a table? Yes and no.

If you create a regular table for data in a format such as CSV or JSON the answer is yes, Athena will read every single byte in the table's location for every query (unless there's a `LIMIT` clause in which case it will attempt to read only as much as it needs to return the desired number of rows, but that's a special case). If the data is in in a columnar format, e.g. Parquet or ORC, Athena can make use of metadata within the files to read only the columns referenced by the query, and skip entire blocks when possible (although in my experience this doesn't work as well with ORC files, so I stick with Parquet when I can), but even if it can skip parts of files it will read every single file.

Reading all the data for a table for every query can get expensive, and slow down queries, especially if the data set grows over time. Even using Parquet will not work well enough in the long run, so what to do?

The answer, as you might have guessed from the title of this section, is partitioning. Partitioning a data set means organizing its data by one or more properties so that queries can quickly skip parts of the data set that doesn't match. A data set about sales could be partitioned by customer so that queries looking at only one or a few customers can ignore most of the data set. In practice the way you partition is simply to put data into a directory structure (prefixes for the S3 purists) where the directories are named from the property values. A very common partitioning scheme is to put data into directories named from the date so that a query looking at the most recent data quickly can skip all old data.

When you set up a table in Athena you include information about how the data is partitioned by specifying one or more partition keys, these correspond to the directory hierarchy of the data set. If the data is organized by customer and date, for example `data/acme/2020-08-01/data.json`, the partition keys could be `customer` and `date`, e.g. `PARTITIONED BY (customer string, date string)`. When querying you can say `WHERE customer = 'acme'` and Athena will know it only has to look in the `data/acme/` directory and can skip everything else – and if you also include a filter on the data it can narrow down the list of files to process even further.

When you create a regular table you can run queries against it straight away. Athena will look at the table's location field and processes all the files it finds. Partitioned tables also have a location, but that's just because it is required by the Glue Data Catalog. To be able to query a partitioned table you need to tell Athena about the partitions of the data set. This can be done in many different ways, and I cover them in more detail in [Five ways to add partitions](/articles/five-ways-to-add-partitions/). The short version is that you either create partitions in the Glue Data Catalog (through Athena DDL statements or API calls), or you configure the table with metadata that makes it possible for Athena to figure out the location of partitions (this is called [Partition Projection](https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html)).

Almost all data sets are already partitioned in some way, it's very rare in my experience to find a non-trivial data set where all files are just in a single directory without any organization to it. The most common form of partitioning is based on time. Data sets tend to grow over time, and adding new data to directories named from the date makes a lot of sense. In general, data sets are often organized in a way that makes it easy for the producing side, for example to make it possible to replace a piece of data if it changes.

[AWS CloudTrail is a good example of how data sets tend to get organized](https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-logs.html). CloudTrail delivers logs to S3 like this: `s3://some-bucket/AWSLogs/0123456789/CloudTrail/us-east-1/2020/08/21/0123456789_CloudTrail_us-east-1_20200821T2120Z_W8LDOrF9dU4NZ7Kt.json.gz`. This URI contains (after `CloudTrail` which is the root), the region, the year, month, and day, and the file name. This partitioning scheme makes it easy to find all files pertaining to a specific AWS region, and if you only want to query the last month, day, or hour's worth of logs a query can easily skip most of the data set. This data set can be said to be partitioned by region, year, month, and day (or by region and date, you don't have to have a one-to-one mapping between partition keys and path components).

A lot of what is written about partitioning in Athena uses a partitioning style that looks like `data/region=us-east-1/year=2020/month=08/day=21/data.json`. This is called [Hive style partitioning](/articles/five-ways-to-add-partitions/) and is common in the Hadoop ecosystem. It has some benefits, but even though the Athena documentation may make it look like this is _the_ way to partition data you don't have to use this scheme, and it's extremely uncommon to see outside of the Hadoop world.

You can map the partitioning of most data sets to Athena tables, but there is one situation that Athena does not handle: files with different schemas in the same directory. Someone might have decided to export data into directories named from the date, but mix files representing different kinds of data in these. For example `data/2020-08-21/orders.csv` and `data/2020-08-21/invoices.csv`. This might make sense when producing the data, but is unfortunately incompatible with the way Athena works. Athena will process all files found in a table or partition's location, and there is no way to configure it to filter based on file name.

## There is much more to Athena

The following articles continue this guide to understanding the basics of Athena:

* [What is Athena?](/articles/athena-basics-what-is-athena/)
* [Working With Data](/articles/athena-basics-working-with-data/)
* [Running Queries](/articles/athena-basics-running-queries/)
* [Permissions](/articles/athena-basics-permissions/)
* [Pricing Model](/articles/athena-basics-pricing-model/)
