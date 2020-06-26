---
title: MSCK REPAIR TABLE
date: 2019-06-26
author: Theo Tolv
---
# MSCK REPAIR TABLE

Many guides, including the official Athena documentation, suggest using the command `MSCK REPAIR TABLE` to load partitions into a partitioned table. _You should almost never use this command._

The command only works if your data is organized using [Hive style partitioning](hive_style_partitioning.md). If this is the case you can use this command to load partitions after you have created a table, provided the number of partitions is farily low – but avoid this command in all other situations.

The main problem is that this command is very, very inefficient. You won't notice when you have only a few partitions, but as the number grows this command will run slower and slower. The [Athena documentation for the command](https://docs.aws.amazon.com/athena/latest/ug/msck-repair-table.html) even mentions how this command sometimes times out and that you "should run the statement on the same table until all partitions are added".

If you want to know more about what makes this command so inefficient, you can find some more information in the comments and answers to the Stack Overflow question [What does MSCK REPAIR TABLE do behind the scenes and why it's so slow?](https://stackoverflow.com/questions/53667639/what-does-msck-repair-table-do-behind-the-scenes-and-why-its-so-slow)
