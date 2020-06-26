---
title: Hive style partitioned data
date: 2019-06-26
author: Theo Tolv
---
# Hive style partitioned data

You have probably seen file listings like this:

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

You can probably guess that the data is partitioned by day and country just by looking at the file paths. This is Hive style partitioning, a convention from [Apache Hive](https://hive.apache.org). The benefit of this style is that it is more or less self documenting and self describing. Both humans and computers can see how the data is partitioned, and select only the files relevant for a query that includes conditions on `day` and `country`.

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

This listing has the same files, and if you were presented with this listing you will still probably figure it out. This is probably how most people would organize this data if given no specific instructions. As long as there is some documentation on what the path components mean it's also not really any worse than the Hive style, except that Hive-aware tools can't use it directly.

Another variant that is fairly common, and used by, for example, CloudTrail and Kinesis Firehose is using separate path components for the date parts, e.g. `data/2020/04/20/fr/7fc6742c.json`.

## Hive style partitioned data and Athena

Athena and Glue, being born out of the Hadoop ecosystem, understand and can make use of the Hive style. However, they don't require it, and work equally well without it. In Athena you can run [`MSCK REPAIR TABLE my_table`](msck_repair_table.md) to automatically load new partitions into a partitioned table if the data uses the Hive style, and Glue Crawler works similarly. However, both of these tools have severe problems and limitations, and I very much discourage you to use them. When you work with partitioned tables the way I recommend in this guide, there is no benefit to using the Hive style. There is also no downside, so if all else is equal you can use the Hive style, just know that you don't have to.
