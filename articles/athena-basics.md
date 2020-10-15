---
title: Athena Basics
date: 2020-08-21
author: Theo Tolv
---
# Athena Basics

Athena is not an RDBMS or a data warehouse appliance, instead it's a query engine. Unlike most databases Athena doesn't manage the storage of your data, it just provides an interface for querying data stored on S3. In this way it is more like Spark than it is like Redshift. While most of your experience with SQL and databases is transferrable to Athena, there are a few things that are distinctly different and that you need to keep in mind to get the most out of it.

In this series of articles on Athena basics I cover the things that aren't explicit in the official documentation. I'll go beyond the bare technical details and try to explain things in more context, and how it works in practice. There won't be much in terms of code or SQL, but wherever possible I link to other articles in this guide that go into much deeper detail.

Here are the articles in this series:

* [What is Athena?](/articles/athena-basics-what-is-athena/)
* [Key concepts](/articles/athena-basics-key-concepts/)
* [Working with data](/articles/athena-basics-working-with-data/)
* [Running queries](/articles/athena-basics-running-queries/)
* [Permissions](/articles/athena-basics-permissions/)
* [Pricing Model](/articles/athena-basics-pricing-model/)
