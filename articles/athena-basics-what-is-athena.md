---
title: 'Athena Basics: What is Athena?'
date: 2020-08-21
author: Theo Tolv
---
# Athena Basics: What is Athena?

Athena is not an RDBMS or a data warehouse appliance, instead it's a query engine. Unlike most databases Athena doesn't manage the storage of your data, it just provides an interface for querying data stored on S3. In this way it is more like Spark than it is like Redshift. While most of your experience with SQL and databases is transferrable to Athena, there are a few things that are distinctly different and that you need to keep in mind to get the most out of it.

In this series of articles on Athena basics I cover the things that aren't explicit in the official documentation. I'll go beyond the bare technical details and try to explain things in more context, and how it works in practice. There won't be much in terms of code or SQL, but wherever possible I link to other articles in this guide that go into much deeper detail.

In this article I'll cover:

* [The origins and components of Athena](#the-origins-and-components-of-athena)
* [What problems Athena solves](#what-problems-athena-solves)

The other articles in this series cover:

* [Key concepts](/articles/athena-basics-key-concepts/)
* [Working with data](/articles/athena-basics-working-with-data/)
* [Running queries](/articles/athena-basics-running-queries/)
* [Permissions](/articles/athena-basics-permissions/)
* [Pricing Model](/articles/athena-basics-pricing-model/)

## The origins and components of Athena

You've probably heard that Athena is built on the open source query engine [Presto](https://prestosql.io). Unlike AWS' other similar services like their managed ElasticSearch, Kafka, and Cassandra offerings, Athena is not just a managed Presto cluster. In fact, AWS has another service that's is closer to managed Presto: Elastic MapReduce (EMR), which can be configured to run Presto clusters.

The fact that Athena is based on Presto shines through here and there, and AWS makes no secret of it – but for the most part Athena is Athena more than it is Presto. A big part of that is that it's an AWS service much more than hosted software. Instead of running a Presto cluster for you, it's a completely serverless setup where you just submit your queries, without worrying about anything else. It's also much more integrated into the AWS ecosystem than Presto is out of the box, but of course, also more limited – one USP of Presto is that it can connect to a lot of different backends, for example joining tables in MySQL with data in Google Sheets, while Athena only works with data on S3. _There is currently a preview of a feature called [Athena Federated Query](https://docs.aws.amazon.com/athena/latest/ug/connect-to-a-data-source.html) that makes it possible to query other backendds, but requires a lot of work on your part and is completely different from Presto's connectors_.

Athena is based on a fairly old version of Presto – 0.172 at the time of writing this, while the latest version is 340 (they dropped the "0." part between 0.215 and 300). However, the Athena team backports fixes, and passes their own fixes upstream, and the version number is probably useful mostly for looking up the documentation for Presto SQL syntax and functions at this point.

Athena was initially launched with its own data catalog, the component that keeps track of your databases and tables, but when Glue was launched it migrated to using Glue Data Catalog. Using Athena is really using three distinct AWS services; Athena itself, Glue, and S3 – and also IAM for permissions (there is also Lake Formation, but that's a topic for another post). This is something that you are aware of at almost all times, and something that trips up many new users, especially when it comes to permissions. There's not that many other AWS services where you need to know a lot about other services to use it.

## What problems Athena solves

I like to say that Athena activates all the data you already have on S3. Most companies that have used AWS for a while have probably accumulated a lot of random data on S3, logs of different kinds, backups, user data, etc. Perhaps you call this your "data lake", like so many startups do to inflate their valuation and enterprises do to seem to be on top of things. Whatever your pet name, before Athena you would either run jobs in EMR or on some home grown ETL "platform" to crunch the data into some useful format, which you stored in Redshift, some other RDBMS, or as new files on S3. There's a ton of tools in EMR for this, some of them are even good – but running clusters 24/7 is expensive, while running on demand is finicky, especially if you want to maintain state between runs.

Athena is an always-on query interface to all that data on S3, it makes it available and accessible, without hassle and with a fairly simple pricing model. Athena gets you quite a long way towards that dream of a data lake where the data you already have can be queried and combined without a lot of preprocessing or complex ETL.

Athena makes the data you have accessible and useful without forcing you to spend a lot of time making it accessible and useful.

While Athena is primarily a query engine, it has grown a few ETL features since its launch. You can [create new tables from query results](https://docs.aws.amazon.com/athena/latest/ug/ctas.html), which means that you don't need to use another service like EMR or Glue ETL for anything but the more complex ETL tasks. If all you want to do is some cleaning, transformation, and data format conversion, Athena and a little bit of management code will take you a very long way.

When you run a query in most RDBMS' you need to stick around until the response is ready, because their driver protocols are synchronous and stateful. If you disconnect your query might get cancelled, and at least you'll have no way to retrieve the results. In contrast, Athena's API is asynchronous and stateless; when you start a query you get a query ID, which you can use at any point in time (up to a couple of months later) to check the query status. This means that your code doesn't need to run while Athena is processing a query – you can start a query as a reaction to some event, put the ID somewhere and at a much later point retrieve the results. This fits really well into the event driven serverless model where your code, for example a Lambda function, only runs while it's actually doing useful work._Since writing this, the [Redshift Data API](https://docs.aws.amazon.com/redshift/latest/mgmt/data-api.html) has been launched, which provides a very Athena-like API for interacting with Redshift._

I'll talk more about the pricing model below, but it's worth mentioning here too: Athena is priced very differently from services like Redshift, EMR, and Glue ETL. Instead of capacity/duration-based pricing Athena charges by the amount of data you process. For many use cases this is the problem that Athena solves: the cost of _not_ using Athena is zero. You don't have to think about how your Redshift cluster is burning a hole in your budget on weekends when no one is using it, for example.

## More Athena basics

The following articles continue this guide to understanding the basics of Athena:

* [Key concepts](/articles/athena-basics-key-concepts/#key-concepts)
* [Working with data](/articles/athena-basics-working-with-data/#working-with-data)
* [Running queries](/articles/athena-basics-running-queries/#running-queries)
* [Permissions](/articles/athena-basics-permissions/#permissions)
* [Pricing Model](/articles/athena-basics-pricing-model/#pricing-model)
