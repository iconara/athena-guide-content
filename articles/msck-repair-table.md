---
title: Avoid MSCK REPAIR TABLE
date: 2019-06-26
author: Theo Tolv
---
# Avoid MSCK REPAIR TABLE

Many guides, including the official Athena documentation, suggest using the command `MSCK REPAIR TABLE` to load partitions into a partitioned table. _You should almost never use this command._

The main problem is that this command is very, very inefficient. You won't notice when you have only a few partitions, but as the number grows this command will run slower and slower. The [Athena documentation for the command](https://docs.aws.amazon.com/athena/latest/ug/msck-repair-table.html) even mentions how this command sometimes times out and that you "should run the statement on the same table until all partitions are added".

The command also only works if your data set is organized using [Hive style partitioning](hive-style-partitioning). You can't use it with, for example, Kinesis Data Firehose output, CloudTrail logs, or most other data sets that are produced by tools outside of the Hive ecosystem.

If you want to know more about what makes this command so inefficient, you can find some more information in the comments and answers to the Stack Overflow question [What does MSCK REPAIR TABLE do behind the scenes and why it's so slow?](https://stackoverflow.com/questions/53667639/what-does-msck-repair-table-do-behind-the-scenes-and-why-its-so-slow).

## When to use it

I don't like to be categorical and say you should absolutely never do something. If you've just created a table in the Athena console, and there are a few partitions that you just quickly want to add to test something out, by all means, run `MSCK REPAIR TABLE`, or use the "Load partitions" feature of the console. If it works, it works – and with just a few partitions it will even run quickly.

## What to use instead

See [five ways to add partitions](five-ways-to-add-partitions).
