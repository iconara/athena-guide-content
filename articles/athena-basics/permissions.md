---
title: 'Athena Basics: Permissions'
date: 2020-10-15
author: Theo Tolv
series:
  index: 5
---
# Athena Basics: Permissions

In this series of articles on Athena basics I cover the things that aren't explicit in the official documentation. I'll go beyond the bare technical details and try to explain things in more context, and how it works in practice. There won't be much in terms of code or SQL, but wherever possible I link to other articles in this guide that go into much deeper detail.

In this article I'll cover:

* [IAM and Athena](#iam-and-athena)

The other articles in this series cover:

* [What is Athena?](/articles/athena-basics-what-is-athena/)
* [Key Concepts](/articles/athena-basics-key-concepts/)
* [Working with Data](/articles/athena-basics-working-with-data/)
* [Running Queries](/articles/athena-basics-running-queries/)
* [Pricing Model](/articles/athena-basics-pricing-model/)

## IAM and Athena

Permissions in Athena are managed through IAM, unless you use Lake Formation (which is a topic in itself and not covered here). As I've mentioned above, Athena is not an isolated service, and running a query involves at least three AWS services: Athena, Glue Data Catalog, and S3. This is reflected in the permissions model too: to run a query, Athena will use the Glue Data Catalog on your behalf, as well as list and read files on S3, and you will need permissions for all of this in order for the query to succeed. This is unlike invoking a Lambda function where the function has its own set of permissions that govern the actions of the function. Athena instead proxies your permissions when it performs actions on other services (again, catalogs managed by Lake Formation have a different model, more similar to that of Lambda).

Because there are multiple services involved, IAM policies for Athena often have a lot of statements, and they can be hard to get right in the beginning. Each service has its resources and ways of specifying and limiting permissions.

Athena is probably the simplest of them, you really only need to make sure the principal (i.e. user or role) has permission to the API calls involved in running a query, which means the actions `athena:StartQueryExecution`, `athena:GetQueryExecution`, and `athena:GetQueryResults` for the [workgroup](https://docs.aws.amazon.com/athena/latest/ug/manage-queries-control-costs-with-workgroups.html) that the query runs in. Next is S3, where `s3:ListBucket` and `s3:GetObject` are needed for the bucket and objects that will be read, and `s3:PutObject` and `s3:GetObject` where the results will be written. Finally, Glue's IAM permissions are probably the hardest to get right, partly because it's hard to know which API calls Athena makes behind the scenes and therefore needs permissions for, and partly because Glue requires you to specify permissions on all levels of its catalog hierarchy – granting permission to a table is not enough, you also need to grant permission to the database the table is in, and the catalog the database is in. Luckily the [Athena documentation has example policies for the most common use cases](https://docs.aws.amazon.com/athena/latest/ug/fine-grained-access-to-glue-resources.html).

The permissions model is far from perfect, and it has a very steep learning curve, but I think there are benefits to it too. I like that it's transparent that Athena uses the other two services, and that it makes the API calls to them in the same way, with the same permissions, as if the principal had done it themselves – and that it shows up in CloudTrail in that way too.

A side effect of the permissions model is that a principal that is allowed to query a table will also be allowed to download all the files belonging to that table. In most cases this is not really an issue, the same data can after all be downloaded by making SQL queries, but there may be situations where the principal is only allowed to query views that aggregate the data or tables where some properties present in the data are not mapped to columns, or situations where you just don't want to provide access to the raw data. In these cases you can use the [`aws:calledVia`](https://aws.amazon.com/blogs/security/how-to-define-least-privileged-permissions-for-actions-called-by-aws-services/) condition on the S3 statements to say that they are only allowed to be performed by the Athena service, not by the principal directly.

## More Athena basics

The following articles continue this guide to understanding the basics of Athena:

* [What is Athena?](/articles/athena-basics-what-is-athena/)
* [Key Concepts](/articles/athena-basics-key-concepts/)
* [Working with Data](/articles/athena-basics-working-with-data/)
* [Running Queries](/articles/athena-basics-running-queries/)
* [Pricing Model](/articles/athena-basics-pricing-model/)
