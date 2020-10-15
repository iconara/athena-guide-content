---
title: 'Athena Basics: Working with Data'
date: 2020-08-21
author: Theo Tolv
---
# Athena Basics: Working with Data

In this series of articles on Athena basics I cover the things that aren't explicit in the official documentation. I'll go beyond the bare technical details and try to explain things in more context, and how it works in practice. There won't be much in terms of code or SQL, but wherever possible I link to other articles in this guide that go into much deeper detail.

In this article I'll cover:

* [What is a "serde"?](#what-is-a-serde)
* [Reading data](#reading-data)
* [Writing data](#writing-data)
* [What's best?](#whats-best)

The other articles in this series cover:

* [What is Athena?](/articles/athena-basics-what-is-athena/)
* [Key Concepts](/articles/athena-basics-key-concepts/)
* [Running Queries](/articles/athena-basics-running-queries/)
* [Permissions](/articles/athena-basics-permissions/)
* [Pricing Model](/articles/athena-basics-pricing-model/)

## What is a "serde"?

To achieve the goal of activating the data you already have, Athena has support for a lot of different file and data formats. Though Hadoop and Hive, Presto has a concept called "serde", short for serializer/deserializer. Serdes are plugins that provide support for reading and writing different file and data formats. Athena does not allow you to add your own, but the available serdes cover most situations.

Tables specify a serde so that Athena knows how to read the data during query execution. Using the [CTAS](https://docs.aws.amazon.com/athena/latest/ug/ctas.html) feature mentioned above, Athena can also write some of these formats.

## Reading data

Athena supports serdes for well defined formats such as [JSON][json-serde], [Avro][avro-serde], [Parquet][parquet-serde], and [ORC][orc-serde], as well as a [specialized serde for reading CloudTrail][cloudtrail-serde] logs (which are JSON, but formatted in a way that is not suitable for the standard JSON serde). The serdes have a few configuration properties you can set for specific situations, but with the exception of the Avro serde they can be used more or less straight out of the box with no configuration necessary. To use the Avro serde you must provide a schema as a configuration property.

Meanwhile, CSV, which is not exactly well defined, but arguably the most popular data format in history, is covered by two different serdes, and even those two don't really cover all the bases when it comes to the weird variants that people come up with. They do provide configuration for delimiters, quoting, and escapes, though. I cover as many details as I can about using CSV in Athena in [Working with CSV](/articles/working-with-csv/).

Finally there's a couple of serdes that can be used to parse custom line-based text formats: [one where you specify a regular expression][regex-serde] with capture groups that are mapped to table columns, and one slightly less lethal [based on Grok][grok-serde]. These are very useful for working with different kinds of unstructured and semi-structured logs, like VPC flow logs, or application logs.

One common requirement for all the text-based formats supported, including JSON, is that they work on a line-by-line basis. There is no support for multi-line formats.

## Writing data

Some of the serdes can be used as output formats when using the CTAS feature. These are JSON, Parquet, ORC, and CSV. There is not much to configure, you can really only specify the delimiter of CSV, and the compression of Parquet and ORC. JSON and CSV are always compressed with gzip.

You can control the number of output files by (ab)using the [bucketing feature](https://docs.aws.amazon.com/athena/latest/ug/bucketing-vs-partitioning.html), by default you get one file per partition.

### The result is always CSV

One question that is often asked by people new to Athena is how you change the output format from CSV to something less problematic. The answer is that you can't. Athena will always write the result as a single CSV file. The only way to get the result in another way is to use CTAS, but that has a lot of overhead.

The particular flavor of CSV that Athena uses for results is also unfortunate in that it can't represent the serialization of complex types unambiguosly – see [Working with Complex Types](https://athena.guide/articles/complex-types/#complex-types-in-results).

## What's best?

When it comes to reading data the best format to use is the one that you have. Unless you run into a situation where performance and/or cost is really hurting you I think you should try use the data you have. Implementing an ETL process to transform data can be just as expensive, both in implementation and in running it – but of course each situation is unique.

If you have control over the process creating the data and no other considerations constrain you, I think Parquet is the way to go. Athena works really well with Parquet, reading only as much of the files as it needs, skipping columns and whole blocks when possible. It also supports complex types (i.e. lists, maps, and structs) schema evolution (adding and removing columns, and changing types), which can save a lot of work when you must make changes.

In theory ORC has provides the same benefits, but I have had less success with it. In my tests Athena seems to always read the whole files, not just the columns used by the query. Operations like `COUNT(*)` are also read the whole file, while with Parquet only the file footer is read. With ORC, columns are mapped by index, like CSV, rather than by name like JSON and Parquet, which is more limiting for schema evolution (unlike CSV this behavior [can be changed](https://docs.aws.amazon.com/athena/latest/ug/handling-schema-updates-chapter.html#index-access), though).

I qualified the Parquet recommendation slightly, because it's very rare that you don't have any constraint. A common problem that you almost certainly will run into is that it's not easy to produce Parquet files in the first place. You almost have to run something like Spark to do it, and running Spark is way more complicated than running Athena, and it doesn't make sense to do just to produce optimized files for Athena, right?

For that reason I would say JSON is a much more reasonable format to default to. It's ubiquitous, it's self-describing, it supports schema evolution, complex types, and it's human readable. The downside with JSON is that repeating all the property names for each record often more than doubles the file sizes, but gzip compression usually negates that and more.

If you have any way to avoid CSV, do it. CSV is even more ubiquitous than JSON, but there's not much good about it as a data storage format. It doesn't support schema evolution other than adding columns at the end (or removing, but never both), nor does it support complex types (it doesn't even have the concept of type). If you use it, at least opt for tabs as separators, or even `0x01` (which is the default delimiter used by Athena's default CSV serde) or `0x1f` (the ASCII code for field separator), using commas or other charachters that are likely to appear in fields will lead you down a very unpleasant path of quoting and escaping.

### Performance considerations

When it comes to data and performance, Athena prefers fewer larger files to lots of small files. There are many reasons for this, and this topic deserves an article in itself to get into all the details (you should read [Top 10 Performance Tuning Tips for Amazon Athena](https://aws.amazon.com/blogs/big-data/top-10-performance-tuning-tips-for-amazon-athena/), but I don't think it goes into enough detail).

The short story is that you should aim for as few files as possible per table or partition, in the case of partitioned tables. Files should be around 100 MB in size at most (if they aren't splittable like Parquet files, but it's a good target for those as well), because you want each compute resource to work as efficiently as possible, 100 files of 1 MB is 100x the overhead of getting the data from S3.

You should also avoid deeply nested directory structures and too many files in a directory. The query planner will list S3 like it were a file system, so each level becomes another list operation. S3's list operation also has a max page size of 1000, which means that if there are more files than that in a directory another operation needs to be made, and it can't be done in parallel. The planning stage of a query execution can sometimes dominate the total running time because of this.

  [avro-serde]: https://docs.aws.amazon.com/athena/latest/ug/avro-serde.html
  [regex-serde]: https://docs.aws.amazon.com/athena/latest/ug/regex-serde.html
  [cloudtrail-serde]: https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-serde.html
  [grok-serde]: https://docs.aws.amazon.com/athena/latest/ug/grok-serde.html
  [json-serde]: https://docs.aws.amazon.com/athena/latest/ug/json-serde.html
  [orc-serde]: https://docs.aws.amazon.com/athena/latest/ug/orc-serde.html
  [parquet-serde]: https://docs.aws.amazon.com/athena/latest/ug/parquet-serde.html

## More Athena basics

The following articles continue this guide to understanding the basics of Athena:

* [What is Athena?](/articles/athena-basics-what-is-athena/)
* [Key Concepts](/articles/athena-basics-key-concepts/)
* [Running Queries](/articles/athena-basics-running-queries/)
* [Permissions](/articles/athena-basics-permissions/)
* [Pricing Model](/articles/athena-basics-pricing-model/)
