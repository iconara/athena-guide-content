---
title: Why is MSCK REPAIR TABLE so slow?
date: 2019-06-26
author: Theo Tolv
---
# Why is MSCK REPAIR TABLE so slow?

Many guides, including the official Athena documentation, suggest using the command [`MSCK REPAIR TABLE`][1] to load partitions into a partitioned table.

_You should almost never use this command._

The main problem is that this command is very, very inefficient. You won't notice when you have only a few partitions, but as the number grows this command will run slower and slower. The [Athena documentation for the command][1] even mentions how this command sometimes times out and that you "should run the statement on the same table until all partitions are added".

The command also only works if your data set is organized using [Hive style partitioning](/articles/hive-style-partitioning). You can't use it with, for example, Kinesis Data Firehose output, CloudTrail logs, or most other data sets that are produced by tools outside of the Hive ecosystem.

## Why is it so slow?

What the command does is that it recursively lists your table's location on S3, compares every prefix to the list of partitons, and even looks at and validates the key of each individual object. This is a lot more work than you would really need to find all partitions for a table, and you may wonder why it does all of this work. The reason is that the command pre-dates Athena, and even Presto (the open source project that Athena builds on), and was made to work with any Hadoop-compatible file system, where S3 is just one of many. Where an S3 optimized tool would know to do listings efficiently, and use the metadata retrieved that way, this command will traverse S3 like a file system, and do operations on individual objects.

Every time you run the command it will do a full traversal of your table's location, which means it only gets slower and slower with time as you add more data. Very quickly, almost all of the work it does will be going through prefixes that haven't changed, but since it can't know where you've added, removed, or changed objects it has to go through it all every time.

If you want to know more about what makes this command so inefficient, you can find some more information in the comments and answers to the Stack Overflow question [What does MSCK REPAIR TABLE do behind the scenes and why it's so slow?](https://stackoverflow.com/q/53667639/1109).

## When to use it

I don't like to be categorical and say you should absolutely never do something. If you've just created a table in the Athena console, and there are a few partitions that you just quickly want to add to test something out, by all means, run `MSCK REPAIR TABLE`, or use the "Load partitions" feature of the console. If it works, it works – and with just a few partitions it will even run quickly.

## What to use instead

The best solution is to use [Partition Projection](https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html), to avoid having to manage partitions at all. If that is not possible, the best thing is if you can add code to the process that produces the table's data that adds partitions after it's uploaded the data to S3. That way the table will be up to date as soon as the data is on S3. Alternatively you can trigger a Lambda function that is triggered by S3 notifications and add partitions when new objects are created.

I've written more about different ways to add partitions in [Five ways to add partitions](/articles/five-ways-to-add-partitions).

  [1]: https://docs.aws.amazon.com/athena/latest/ug/msck-repair-table.html
