---
title: What is Hive style partitioning?
date: 2019-06-26
author: Theo Tolv
---
# What is Hive style partitioning?

If you look at the Athena documentation you often see data organized with paths that contain key value pairs, like `country=fr/…` or `year=2020/month=06/day=26/…`. This is Hive style (or format) partitioning. The paths include both the names of the partition keys and the values that each path represents. It can be convenient and self documenting, but it's also uncommon to see it outside outside the Hadoop and Hive ecosystem, where it originated.

## A self-documenting scheme

Consider a file listing like this one:

```
data/day=2020-04-20/country=fr/7fc6742c.json
data/day=2020-04-20/country=ir/80af7e0c.json
data/day=2020-04-20/country=se/f76c0d8f.json
data/day=2020-04-20/country=us/cf7e76b9.json
data/day=2020-04-21/country=af/cb81aacf.json
data/day=2020-04-21/country=ch/6b8ea57f.json
data/day=2020-04-21/country=fr/0c8e74a2.json
data/day=2020-04-21/country=us/68af2b83.json
```

You can probably guess that the data is partitioned by day and country just by looking at the file paths. The benefit of this style is that it is more or less self-documenting. Both humans and computers can see how the data is partitioned, and select only the files relevant for a query that includes conditions on `day` and `country`.

## Should you use Hive style partitioning?

If you are starting from scrach and have full control over how the data in your data lake is organized, this is a good convention to follow, and it is understood by many tools in the Hadoop ecosystem.

Often, though, you don't fully control the way data is produced and organized, and most tools outside of the Hadoop world don't organize data using the Hive style. Instead, you probably get files organized like this:

```
data/2020-04-20/fr/7fc6742c.json
data/2020-04-20/ir/80af7e0c.json
data/2020-04-20/se/f76c0d8f.json
data/2020-04-20/us/cf7e76b9.json
data/2020-04-21/af/cb81aacf.json
data/2020-04-21/ch/6b8ea57f.json
data/2020-04-21/fr/0c8e74a2.json
data/2020-04-21/us/68af2b83.json
```

This listing has the same files, and if you were presented with this listing you would still figure out how it was organized. This is probably also how most people would organize this data set if given no specific instructions. As long as there is some documentation on what the path components mean it's also not really any worse than the Hive style, except that Hive-aware tools can't use it directly.

Another variant that is fairly common, and used by, for example, CloudTrail and Kinesis Firehose is using separate path components for the date parts, e.g. `data/2020/04/20/fr/7fc6742c.json`.

The Hive style can look a bit messy, and even if it's self documenting and logical, non-programmers may be put off since it doesn't look like how they are used to organize files in file systems.

## Hive style partitioned data and Athena

Athena and Glue, being born out of the Hadoop ecosystem and using a lot of code from Hive under the hood, understand and can make use of the Hive style. However, they don't require it, and work equally well without it. In Athena you can for example run `MSCK REPAIR TABLE my_table` to automatically load new partitions into a partitioned table if the data uses the Hive style (but if that's slow, read [Why is MSCK REPAIR TABLE so slow](msck-repair-table)), and Glue Crawler figures out the names for a table's partition keys if the data is partitioned in the Hive style.
