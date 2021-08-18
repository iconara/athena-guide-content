---
title: Stitching together tables with SymlinkTextInputFormat
date: 2021-08-17
author: Theo Tolv
---
# Stitching together tables with SymlinkTextInputFormat

A unique selling point of Athena is that you can use it to query data that you already have, data that was not created specifically to be consumed by Athena. This applies to different file formats as well as different ways of organizing the data set on S3. In this article I will show you how to deal with situations when your data set is not neatly organized in the strict, non-overlapping hierarchies.

## Everyone has a different idea about how to organize a data set

Athena tables have a `LOCATION` property that points to where the data for the table is located, or one `LOCATION` per partition for partitioned tables. During query execution, _all_ objects prefixed by the `LOCATION` will be processed. This means that there can't be any other objects there, or the results will not be what you want – if the query doesn't fail outright. Data sets must be neatly organized in a strict hierarchy and there can't be any overlap. This is not necessarily how everyone thinks data sets should be organized.

You could have an application that produces a few different kinds of output every day. It does this by creating a directory named from the date and puts each type of output in a separate file in the directory. It's neat and tidy, it's easy to move around and to archive. It can look something like this:

```
data/2021-08-16/electricity-usage.csv
data/2021-08-16/production-faults.csv
data/2021-08-16/widgets-produced.csv
data/2021-08-17/electricity-usage.csv
data/2021-08-17/production-faults.csv
data/2021-08-17/widgets-produced.csv
```

You and I can look at that and figure out what goes together, and it's not hard to write a shell command that outputs all of the data for one of the types for a given time period. But point a Glue crawler at this directory structure and you will have to spend a lot of time cleaning up afterwards.

You could also have data produced by a previous version of an application that's in a different place, and organized differently than the output of the new version. Or there are two applications creating compatible data, but in different places and with different directory structures. There are ways to stich these together in a view from multiple tables, but sometimes it would be convenient to have just one table.

Whenever you can it's better to organize your data sets the way Athena prefers them, each in a prefix of its own. However, in situations where that's not possible, or inconvenient, you can use a special input format called `SymlinkTextInputFormat` to stitch things together.

### What's an input format?

An _input format_ is what reads data from S3 and feeds the _serde_ (serializer/deserializer), and these two in combination are how objects on S3 end up as rows and columns in Athena (see [What is the difference between InputFormat and OutputFormat in Hive?](https://stackoverflow.com/q/42416236/1109)). You've probably seen `TextInputFormat` when you've created tables for JSON or CSV data and perhaps `MapredParquetInputFormat` for Parquet backed tables.

The default input formats you get when you create tables in Athena are used to read objects from the table or partition's `LOCATION`, but the `SymlinkTextInputFormat` is different: it expects the table or partition's `LOCATION` to contain a list of URIs pointing to the actual objects. This is like a [symlink in the file system](https://en.wikipedia.org/wiki/Symbolic_link), it points to the location where the real data is located. Unlike a file system symlink, though, it can point to multiple objects.

## Stitching together overlapping data sets

Let's look at a situation similar to what I described above. Say you have data of different kinds organized by region in a bucket called `example-bucket`. It looks something like this:

```
data/amer/taxes1.csv
data/amer/taxes2.csv
data/amer/unicorn-inventory.csv
data/apac/taxes1.csv
data/apac/unicorn-inventory.csv
data/emea/taxes1.csv
data/emea/unicorn-inventory.csv
```

You and I can figure out that the objects with "taxes" in the name probably go together, and the objects with "unicorn-inventory" in the name are a different data set. Out of the box neither Glue nor Athena can figure this out, and both will make a terrible mess if you ask them to. There is no one prefix that captures only the tax data but no unicorn inventory data, nor vice versa.

### You can always move the data

One way to solve the problem would of course be to move or copy the objects into a structure that worked better with Athena. If you copied them into this structure it would be straightforward to create tables:

```
taxes/amer/data1.csv
taxes/amer/data2.csv
taxes/apac/data1.csv
taxes/emea/data1.csv
unicorn-inventory/amer/data.csv
unicorn-inventory/apac/data.csv
unicorn-inventory/apac/data.csv
unicorn-inventory/emea/data.csv
```

In this listing the objects with the same schema have prefixes that don't overlap: `taxes/` and `unicorn-inventory/`. A table for the taxes data set in the listing above could look something like this:

```sql
CREATE EXTERNAL TABLE taxes (…)
PARTITIONED BY (region string)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION 's3://example-bucket/taxes/'
```

If you have control over the thing that produces the data the easiest solution is often to make sure it produces data in a structure like this. However, there are situations where it's not possible to change the application that produces the data, or it's inconvenient to move the objects after they've been produced. For these situations there is `SymlinkTextInputFormat`.

### Creating symlinks

Instead of copying or moving the objects into the structure above, you create a similar structure but put an object in each partition that points to the actual objects:

```
taxes/amer/symlink.txt
taxes/emea/symlink.txt
taxes/apac/symlink.txt
unicorn-inventory/amer/symlink.txt
unicorn-inventory/emea/symlink.txt
unicorn-inventory/apac/symlink.txt
```

This looks very similar to how it would look if we had copied the object, but instead of the data objects there is an object called `symlink.txt` in each partition (this object can be called anything and you can have more than one per partition if you want, but the convention seems to be to use this name).

The symlink objects are very basic, they just contain the full S3 URIs of the objects for the corresponding partition, e.g. the contents of `taxes/amer/symlink.txt` would be just this:

```
s3://example-bucket/data/amer/taxes1.csv
s3://example-bucket/data/amer/taxes2.csv
```

To tell Athena to expect partitions to contain symlinks you create your table with the `SymlinkTextInputFormat` instead of the default:

```sql
CREATE EXTERNAL TABLE taxes (…)
PARTITIONED BY (region string)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS
  INPUTFORMAT 'org.apache.hadoop.hive.ql.io.SymlinkTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://example-bucket/taxes/'
```

When you specify the input format you need to specify the output format too, and `HiveIgnoreKeyTextOutputFormat` is the default output format.

You add partitions to the table just like you would any other table (but using [partition projection](https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html) is always preferable):

```sql
ALTER TABLE taxes ADD
  PARTITION (region = 'amer') LOCATION 's3://example-bucket/taxes/amer/'
  PARTITION (region = 'apac') LOCATION 's3://example-bucket/taxes/apac/'
  PARTITION (region = 'emea') LOCATION 's3://example-bucket/taxes/emea/'
```

## Stitching together data in different locations

In the previous example all the objects were in the same place, but what if the data for a table or partition is scattered over multiple locations and even multiple buckets?

To `SymlinkTextInputFormat` it doesn't matter if the URIs point to objects in the same or in different buckets. It will also list prefix URIs, which means that you don't have to include each object's URI if there is a prefix that contains objects that all should be included (and no objects that shouldn't) – in other words it's like being able to specify multiple values for the `LOCATION` property of a table or partition.

For example, this could be the contents of a symlink object:

```
s3://example-bucket/data/amer/taxes.csv
s3://example-bucket/data/amer/taxes-additional.csv
s3://example-bucket/data/amer2/taxes.csv
s3://another-example-bucket/random-stuff/taxes-old.csv
s3://example-bucket-with-old-data/historical-taxes/
```

There is just one caveat: while it's ok for a table or partition's `LOCATION` to not contain anything, all the URIs in a symlink object must exist or not be empty when listed.

## Repartitioning and unpartitioning data sets

Using `SymlinkTextInputFormat` you can change the partitioning of a data set. The example above is organized in directories by region (e.g. "amer", "emea") and the file names signify the data set (e.g. "taxes"). If your data is organized differently, perhaps the value you want to partition by is in the file name, or the files are partitioned and you don't want to create a partitioned table, you use `SymlinkTextInputFormat` in many different ways to create the structure you need.

For example, I could ignore the region directories in the example above and create unpartitioned tables. The listing would look like this:

```
taxes/symlink.txt
unicorn-inventory/symlink.txt
```

With `taxes/symlink.txt` containing all URIs:

```
s3://example-bucket/data/amer/taxes1.csv
s3://example-bucket/data/amer/taxes2.csv
s3://example-bucket/data/apac/taxes1.csv
s3://example-bucket/data/emea/taxes1.csv
```

And the table DDL looking something like this:

```sql
CREATE EXTERNAL TABLE taxes (…)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS
  INPUTFORMAT 'org.apache.hadoop.hive.ql.io.SymlinkTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://example-bucket/taxes/'
```

## Closing thoughts

Curiously the Athena documentation doesn't mention `SymlinkTextInputFormat`, and it's hard to find any documentation on how to use it or how it works. Even the Hive documentation is very sparse on the subject.

I came across `SymlinkTextInputFormat` when I read [the documentation on how to query S3 Inventory with Athena](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory-athena-query.html) and I've reverse engineered most of what I know about it from there.

`SymlinkTextInputFormat` is a useful tool to solve some difficult situations when you really want to avoid having to reorganize your data sets. It creates a level of indirection that needs to be maintained, though, and I don't know how it affects performance. It's not for every situation, but I'm very happy that it exists the times when I need it.
