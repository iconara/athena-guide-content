---
title: 'Athena Basics: Running Queries'
date: 2020-10-15
author: Theo Tolv
series:
  index: 4
---
# Athena Basics: Running Queries

In this series of articles on Athena basics I cover the things that aren't explicit in the official documentation. I'll go beyond the bare technical details and try to explain things in more context, and how it works in practice. There won't be much in terms of code or SQL, but wherever possible I link to other articles in this guide that go into much deeper detail.

In this article I'll cover:

* [Using the API](#using-the-api)
* [Raw results](#raw-results)
* [Work groups](#work-groups)
* [JDBC & ODBC](#jdbc-odbc)
* [Concurrency limits & noisy neighbors](#concurrency-limits-noisy-neighbors)

The other articles in this series cover:

* [What is Athena?](/articles/athena-basics-what-is-athena/)
* [Key Concepts](/articles/athena-basics-key-concepts/)
* [Working with Data](/articles/athena-basics-working-with-data/)
* [Permissions](/articles/athena-basics-permissions/)
* [Pricing Model](/articles/athena-basics-pricing-model/)

## Using the API

Unlike most RDBMS', Athena has an asynchronous API. When you submit a query you get a "query execution ID" back, and the API call completes immediately. To track the progress of a query you use the ID in another API call to poll its status, and yet another API cal to retrieve results when the query has completed. The asynchronous query execution API is one of my favorite features of Athena, as it enables usage patterns that aren't possible with a synchronous model. For the situations where a synchronous connection-based model is needed, for example tools that expect a JDBC or ODBC driver, this can be built on top of the asynchronous API, and there are also offical and unofficial drivers available, more on this later.

Running a query in Athena involves three API calls: [`StartQueryExecution`](https://docs.aws.amazon.com/athena/latest/APIReference/API_StartQueryExecution.html), [`GetQueryExecution`](https://docs.aws.amazon.com/athena/latest/APIReference/API_GetQueryExecution.html), and [`GetQueryResults`](https://docs.aws.amazon.com/athena/latest/APIReference/API_GetQueryResults.html). To run a query you use `StartQueryExecution` and pass the SQL you want to run, as well as an S3 location where Athena can write the results – since queries are run asynchronously Athena needs somewhere to store the results so that you can ask for them later. The `StartQueryExecution` returns a "query execution ID", which you use with the other two API calls to identify which query you want status or results for.

It's important to understand that when you call `StartQueryExecution` you really only submit the query to Athena, and just because you don't get an error back does not mean query has succeeded, or will succeed. It performs some validation, for example checking that the syntax of the SQL is valid, but semantic SQL errors, like referring to functions, tables, or columns that don't exist, will not be caught here. Therefore it's important that you always use `GetQueryExecution` to check that the query didn't fail early because of a typo in a function or table name.

While the query is running you use `GetQueryExecution` to check its status. In most cases you need to make this API call multiple times until the query completes, and there is no API to wait, or block, until the query completes. Often what you do is that you set up a loop that makes this API with a small delay between each call. I like to implement a variant of exponential backoff so that I wait longer between calls the longer the query has been running, up to a max interval, but you can make as many calls as you want, AWS does not charge for these calls.

The result of the `GetQueryExecution` API call contains a lot of information, and for some reason the designers decided to bury the status fairly deep inside a nested structure. When submitted the status will be either `QUEUED` or `RUNNING`, the former meaning that it's waiting for resources, and the latter that resources have been allocated and the query has started executing. If the query fails it gets status `FAILED` and an error message will be available in the response from the API call. Queries can also be cancelled with the [`StopQueryExecution`](https://docs.aws.amazon.com/athena/latest/APIReference/API_StopQueryExecution.html) API call, in which case they end up in status `CANCELLED`. Hopefully your query instead ends up with status `SUCCEEDED`, in which case you can call `GetQueryResults` to retrieve the results.

`GetQueryResults` returns the result rows, as well as column metadata. All values are returned as strings, but the metadata contains type information that can be used to format the values correctly. The results are paged, with a default page size of 1,000 rows, which is also the maximum page size. Make sure you look for the token that indicates whether or not there are subsequent pages so that you don't miss any part of the result set.

Running `GetQueryResults` is only possible after a query has status `SUCCEEDED`, calling this API while a query is running, or with the ID of a failed query results in an error.

To reiterate: running a query means calling `StartQueryExecution`, and then use the ID it returns to run `GetQueryExecution` over and over again until the status is `FAILED`, `CANCELLED`, or `SUCCEEDED`. In case of the latter you finally run `GetQueryResults`, until there are no more result pages, in order to retrieve the results.

## Raw results

There is an alternative to using the `GetQueryResults` API call: as I mentioned above you must supply Athena with a location where it can write the query results, and the `GetQueryExecution` API call also contains the location of the file that Athena has written. This file is just a plain CSV that can be retrieved and processed with any tool that supports CSV. Along with the CSV there will also be a file with a `.metadata` suffix. This is a [ProtoBuf](https://developers.google.com/protocol-buffers/) encoded file containing the result metadata, including column types. There is no offical documentation for this file, but I have made some [investigations into how to interpret these files](https://gist.github.com/iconara/4969c247e8abb69600cdbe6f4b20f50b).

Reading the CSV directly from S3 instead of using the `GetQueryResults` API can in many situations give you better performance, and it can also be useful when you want to use the query results in another tool that can read CSV, such as Excel.

Using CSV as the result format for Athena was in my opinion not the best choice. CSV is the lingua franca of data science in a way, but it's also an extremely messy file format. The particular variant Athena uses means that it will [render array and map output ambiguously](https://athena.guide/articles/complex-types/#complex-types-in-results), and not casting these types to JSON can result in results that are not possible to parse correctly.

When running DDL the results are not written as CSV, but plain text, or in some cases a binary format which I have not been able to decode.

## Work groups

Athena runs queries in the context of a "work group". By default you get a work group called "primary", which you can use for everything if you want to – it's completely optional to use the features provided by the work groups.

Work groups solve a mix of different problems that existed in Athena before they were introduced. Just as Athena is one big cluster shared by all AWS customers in the same region, all applications in the same account used the same Athena service, and there was no way to determine how much of the total Athena charge on the bill that was caused by any one particular application or data scientist. There was also no way to set quotas or permissions on an application basis, or even know how much each application was using Athena.

If you create a work group per application you can set quotas, get metrics, and costs reported on a per-application basis.

The quota is unfortunately only on the total bytes scanned, and is primarily a feature designed as a circuit breaker to avoid runaway charges. Your account still has one global concurrency quota which can't be divided up between work groups.

Another feature of work groups is that they can have defaults for some of the parameters you set when you run a query. The output location can be set (and also enforced) in the work group, which can be a convenient way to avoid applications having to have know about where results are stored.

## JDBC & ODBC

Many tools expect to interact with a database using JDBC and ODBC and not a custom API. For that reason Athena have commissioned [JDBC and OBDC drivers](https://docs.aws.amazon.com/athena/latest/ug/athena-bi-tools-jdbc-odbc.html). Internally these use the same API described above (although using a currently private API for retrieving the results). The drivers are not written by the Athena team, but by a company called Simba, who have written [drivers for a lot of other RDBMS'](https://www.simba.com/drivers/).

In my experience the offical JDBC driver leaves a lot to desire, and has been plagued with bugs that don't get fixed for years. For that reason I wrote an [alternative JDBC driver](http://github.com/burtcorp/athena-jdbc/) when I was at a company that required much higher performance and quality than the offical driver could deliver. If you need a JDBC driver for your project I recommend you start with the official driver, it's going to continue to be developed and support new features as they are released, but do have a look at the alternative driver if you are interested or have had problems with the official.

## Concurrency limits & noisy neighbors

Athena runs all queries in a shared cluster. While the service goes to great lengths to ensure that your queries and data is isolated from other customers and secure, you all share the same pool of compute resources. Each account gets a quota that determines how many concurrent queries it can run, and exceeding this limit results in throttling errors when submitting queries. The default limit is 20 concurrent queries (DDL statements have the same limit, but a separate quota), and you can ask AWS for this to be raised if you have a legitimate need.

Even if you stay within your quota there is however no guarantee that your queries will run immediately. You share the cluster with all other customers and that means that sometimes there are no compute resources available when you submit a query. In this scenario your query will be queued until resources become available. Athena is used for a lot of reporting applications and these tend to be configured to run jobs at specific times of the day, almost always the top of the hour. If you run a lot of queries you can notice how the amount of time your queries spend in the queue spikes around the top of every hour. This can significantly affect performance and is something that you should be aware of, especially if you are trying to optimize your queries – always look at the statistics returned by the `GetQueryExecution` API call to check how much time your query spent in the queue before making assumptions about the performance of your query or data set.

## More Athena basics

The following articles continue this guide to understanding the basics of Athena:

* [What is Athena?](/articles/athena-basics-what-is-athena/)
* [Key Concepts](/articles/athena-basics-key-concepts/)
* [Working with Data](/articles/athena-basics-working-with-data/)
* [Permissions](/articles/athena-basics-permissions/)
* [Pricing Model](/articles/athena-basics-pricing-model/)
