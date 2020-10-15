---
title: 'Athena Basics: Pricing Model'
date: 2020-08-21
author: Theo Tolv
series:
  index: 6
---
# Athena Basics: Pricing Model

In this series of articles on Athena basics I cover the things that aren't explicit in the official documentation. I'll go beyond the bare technical details and try to explain things in more context, and how it works in practice. There won't be much in terms of code or SQL, but wherever possible I link to other articles in this guide that go into much deeper detail.

In this article I'll cover:

* [You pay for what you use](#you-pay-for-what-you-use)
* [The small print](#the-small-print)
* [Optimize for cost](#optimize-for-cost)

The other articles in this series cover:

* [What is Athena?](/articles/athena-basics-what-is-athena/)
* [Key Concepts](/articles/athena-basics-key-concepts/)
* [Working with Data](/articles/athena-basics-working-with-data/)
* [Running Queries](/articles/athena-basics-running-queries/)
* [Permissions](/articles/athena-basics-permissions/)

## You pay for what you use

In the context of AWS, Athena's pricing model is refreshingly simple, at least on the surface. It's a flat $5 per TB of data scanned, rounded to the nearest MB, with a 10 MB minimum. It's the amount of data read from S3 that is measured, which means that querying compressed data is cheaper, and using columnar formats that allow Athena to skip columns and skip blocks also lower query costs significantly.

## The small print

As will all AWS services, the devil is in the details. In addition to the Athena charges, you also pay for the Glue Data Catalog and S3 operations Athena performs. While I've never seen Glue become a big cost, I've more than once seen uses of Athena where the number of S3 operations is the cost driver.

Glue has a very generous free tier where you can store a million objects (databases, tables, partitions), and perform a million operations per month for free, and only then pay $1 per hundred thousand objects stored, and $1 per million operations, meaning it will take a while until you start paying for Glue at all, and probably only for operations unless you have tables with hundreds of thousands of partitions.

S3's pricing page has so many zeroes in the charges that it's sometimes hard to think that you'll ever end up paying anything at all. However, list operations actually cost $5 per million, and get operations $0.4 per million (in most regions, there are a few where it's slightly more expensive). If you run a query against a table with a million files you will be charged for a million get operations, and at least a thousand list operations (exactly how many depends on how the data is organized, but with S3's max page size of 1,000 it can't be done in fewer operations than that). This will cost $0.4 for the get operations and $0.005 for the list operations. For comparison, if each file is 1 MB, making the whole data set 1 GB, the Athena charges would be $0.5.

While that scenario might not be very common, I have seen setups where Athena is used as the query engine for data sets with lots of small files and where the S3 costs outweigh the Athena costs by an order of magnitude.

## Optimize for cost

When considering Athena you should take the number of files and how they are organized into account. If you can, try to limit the number of files, and keep directory hierarchies as flat as possible (Athena will traverse your data set as if it were a file system, so every directory will result in a list operation). How to optimize small data sets for costs is a separate topic I hope to be able to cover in the future.

At the other end of the spectrum, the pricing model is more predictable. If you have data sets that are hundreds of GB to multiples of TB or even larger, you probably don't have millions of small files, and it will be the cost per TB that dominates. In this scenario, finding optimal ways to partition the data set, and making use of columnar file formats will be key to managing costs. However, each query will have a real and significant cost to it, and mistakes and misconfiguration can rack up huge bills.

I really wish there were better tools available to predict the costs of queries. When you run a query you get the total bytes scanned, which means that you at least after the fact know the Athena charge for a particular query. Using CloudTrail you can also figure out what Glue and S3 operations were made. Using these two tools you can optimize the cost of queries. You can use the bytes scanned metric to make sure Athena takes advantage of your partitioning, and you can use CloudTrail to see how it lists and retrieves files from S3 to figure out if there are better ways you could organize your files to avoid S3 operations. It's a lot of hard work, but at least the data is there.

## More Athena basics

The following articles continue this guide to understanding the basics of Athena:

* [What is Athena?](/articles/athena-basics-what-is-athena/)
* [Key Concepts](/articles/athena-basics-key-concepts/)
* [Working with Data](/articles/athena-basics-working-with-data/)
* [Running Queries](/articles/athena-basics-running-queries/)
* [Permissions](/articles/athena-basics-permissions/)
