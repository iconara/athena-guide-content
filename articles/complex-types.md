---
title: Working with complex types
date: 2020-07-08
author: Theo Tolv
---
# Working with complex types

Most of the data formats that Athena supports have support for complex types in the form of lists and maps. It even supports lists and maps in CSV files, if you really want it to.

The relational model wasn't made with complex types in mind, but modern data is very rarely flat. Developers like data formats like JSON because they allow for expressing things like one-to-many relationships naturally and in a self-contained manner. When you try to describe the world you often end up with lists of things, the one-or-more type of relationship is very common, and things like lists of tags, or key/value pairs of metadata is more or less standard.

Before Athena I worked a lot with Redshift, and was often frustrated with the lack of complex types. The data I worked with almost always contained lists of strings and other similarly "simple" complex types. With Athena this is almost never an issue.

In this article I'm going to cover how to use complex types in three different contexts: how to create tables when you have [complex types in data](#complex-types-in-data), how to work with [complex types in queries](#complex-types-in-queries), and how to deal with [complex types in results](#complex-types-in-results).

First, let's define complex types so that we're on the same page.

## Complex types defined

Simple types, or "scalars", are things like `number`, `boolean`, `string`, and `timestamp`. Complex types are types that refer to other types – or more concretely in the case of Athena, `array` (lists of elements), `map` (key/value associations), and `struct` (key/value associations with a fixed schema).

Complex types can be for example `array<string>` (an array of strings), `map<string,boolean>` (a map with string keys and boolean values), or `struct<name:string,age:tinyint>` (a record with a string property called "name" and an integer property called "age"). Complex types can have arbitrarily complex structure, for example `array<struct<title:string,author:struct<name:string,email:string>,tags:map<string,string>>>` could be a way to describe a list of article metadata.

## Complex types in data

If you are using Athena to query JSON data you have most likely already worked with complex types in your data in the form of an array property or an object property. Being able to describe most JSON data in table form is one of the most powerful features of Athena.

### JSON

In my experience, most JSON data isn't very hierarchical. Most of the properties are usually scalar, with one or two lists of either strings or fairly simple objects with one or two properties each.

The [official Athena documentation](https://docs.aws.amazon.com/athena/latest/ug/json-serde.html) makes a fairly good job at describing how to create tables for JSON data.

What's good to know is that Athena is fairly forgiving when it comes to describing complex types. If you define a column as `array<string>` and the JSON document has a list of numbers, that works too, as long as type coercion works there is usually no problem. More importantly, if you define a struct column, the fields of the struct work just like the columns of a table: if they're not found in the data the value defaults to `NULL`, and if there are more properties in the data than in the table definition, those properties are ignored.

### Schemaless or schemafree?

JSON doesn't require a schema, although most JSON data probably has some kind of informal schema defined by the code that produces it. This is roughly what the term "schemaless" means; there's no formal schema, and it's possible for documents to have slightly different schemas, but when zooming out and looking at all documents together they look more alike than not.

If you don't control the code that produces a data set it can take some time to figure out what its schema is. Experimenting with different table designs in Athena is cheap, though. If you get it wrong you can just drop the table and recreate it with a new schema that captures some aspect you had previously missed. There's very rarely need to fix the data, Athena is all about making it possible to read the data you have.

Lots of tools exist that are aimed at helping you figure out the schema of JSON data. Glue Crawlers, for example, will read a sample of your data and figure out which properties exist, and their types, and at re:Invent 2019, AWS launched a feature that [detects the schemas of events sent through EventBridge](https://aws.amazon.com/about-aws/whats-new/2019/12/introducing-amazon-eventbridge-schema-registry-now-in-preview/). In some situations tools like these might be helpful, but in other situations they might just make it worse. Glue Crawlers can, for example, [flip-flop the schema of a table](https://stackoverflow.com/q/61297671/1109) from run to run if the sample they look at each time is different enough. Say your documents have a property that contains arbitary key/value pairs from your users. Glue won't figure out that there is really no schema there, and instead find different schemas on different days.

One example of free-form properties like these can be found in CloudTrail logs. There are properties like `requestParameters` and `additionalEventData` that are service specific. Even if these properties could potentially be described using the union of what all AWS services put into them today, we can't tell how future services will use them. These properties both have and don't have schemas, depending on your point of view. From a global perspective where you want to consume events from all services, and future services, you'll have to think of them as free-form structures.

Luckily, there's a solution for that, and I'm going to describe it in detail when discussing how to work with [complex types in queries and free form structures](#working-with-free-form-structures). For now, you can do as the [Athena documentation about CloudTrail][cloudtrail] suggests and use the type `string` for columns where the type can't be pinned down. You can also use `map<string,string>` when you know a property is an object, but not what the values are.

### Parquet, ORC, and Avro

Parquet, ORC, and Avro are also data formats that support hierarchical structures with lists, maps, and/or structs. One important difference between these and JSON is that these formats have formal schemas, either embedded or in side-cars. Each document in a file will conform to the same schema, and creating a table is often just a matter of translating this schema to Athena compatible types.

### Map keys

JSON and Avro both require keys in objects/maps to be strings, but Parquet and ORC are more relaxed and allow keys to be of other types. Athena supports this fully and even though your data is probably the limiting factor, you can declare tables with maps where the keys are any scalar type. Complex type keys are not supported, though.

### Struct or map?

From the table modelling perspective the struct and map types overlap. In many situations where you have an object property in JSON, for example, you could use either. My general advice is that if you know the names of the fields of the object property you should use a struct in your table. However, if the property is free form, as in the CloudTrail example above, a map is more suitable.

Map functions won't work on struct columns, so if you are planning to use map functions a lot then using the map type will make that easier.

Again, experimenting with table design in Athena is cheap, don't worry about getting it right the first time. You can always drop the table and create a new one that works better with no or minimal downtime.

## Complex types in queries

SQL wasn't designed with complex types in mind, and most of the support in Athena comes in the form of aggregate functions, and what's called ["lambda expressions"](https://prestosql.io/docs/0.172/functions/lambda.html). These are pieces of functional code that can look very out of place in a SQL statement, but can be very powerful when working with complex types. I'll show you some examples of lambda expressions below.

### Structs

The complex type that looks the least out of place in SQL is probably `struct`. Struct fields are accessed using dot notation, so if there's a column `author` defined as `struct<name:string,email:string>` you would access them something like this:

```sql
SELECT author.name, author.email
FROM my_table
```

Fields in nested structs are accessed the same way, just like how you would access properties in a JavaScript object.

### Arrays

Array access looks like it does in most programming languages with literal lists, you use square brackets to access the elements by index – but in SQL indexes start at 1!

The first element of a column called `tags` defined as `array<string>` can be accessed like this:

```sql
SELECT
  tags[1] AS first_tag,
  tags[2] AS second_tag
FROM my_table
```

Elements can also be accessed with [`element_at`][array_element_at], e.g. `element_at(tags, 1)`. This variant also allows access from the end with indexes from 0 and negative numbers, e.g. `element_at(tags, 0)` is the last element of the array.

Be aware that trying to access an element that doesn't exist results in a runtime error. The query above would only succeed if _every_ row had two or more elements in their `tags` array. If you access array elements by index you can wrap the expression in `try` to avoid the error, e.g. `try(tags[99])`, or select only rows that have enough elemenents using something like `WHERE cardinality(tags) >= 2`.

Accessing array elements by index is rare, though. More common is to flatten the array into a single value, similar to what you do in a `GROUP BY` query. There are in fact [a whole set of functions that operate on arrays][array] with names similar to the common aggregate functions: [`array_max`][array_max], [`array_min`][array_min], [`array_distinct`][array_distinct], as well as [`join`][join] to create a string, and [`cardinality`][array_cardinality] to count the numer of elements. When a simple aggregate function isn't enough there's also [`reduce`][reduce] which can be used together with lambda expressions to implement almost any aggregation over an array – more on that below.

There are also functions that produce new arrays, like [`concat`][concat] for combining arrays (which can also be done with `||`, just as with strings), or through set operations like [`array_union`][array_union] and [`array_intersect`][array_intersect], as well as [`reverse`][reverse], [`slice`][slice], and [`zip`][zip], which combines pairs of elements of multiple arrays – all that can be used in arbitrary combinations to create new arrays, either before aggregation, or as means to produce new arrays for subsequent processing steps.

#### Creating arrays

Sometimes you don't have an array but want to create one. One way is with the `ARRAY` constructor which can create arrays from literals or column references, e.g. `SELECT ARRAY[col1, 'hello', col2]` – just make sure that all elements are of the same type.

A really powerful way to create arrays is the `array_agg` aggregate function. It can be used to create an array from a group of rows in a `GROUP BY`. Say you have a data set created in a database without support for arrays, where you would model a list as a table with an ID column and a value column. Using `array_agg` you can create a relation with the values as an array, which can then be processed further as needed:

```sql
SELECT article_id, ARRAY_AGG(tag) AS tags
FROM article_tags
GROUP BY article_id
```

#### Flattening arrays

Other times you want to go in the other direction, you have an array but you want a row per element in the array. For this you can use the `UNNEST` operation, which I have written [a separate article about](/articles/unnest-arrays-to-rows/). In many ways you can think of `UNNEST` as the reverse of `array_agg`.

  [array]: https://prestosql.io/docs/0.172/functions/array.html
  [array_element_at]: https://prestosql.io/docs/0.172/functions/array.html#element_at
  [array_max]: https://prestosql.io/docs/0.172/functions/array.html#array_max
  [array_min]: https://prestosql.io/docs/0.172/functions/array.html#array_min
  [array_distinct]: https://prestosql.io/docs/0.172/functions/array.html#array_distinct
  [join]: https://prestosql.io/docs/0.172/functions/array.html#join
  [array_cardinality]: https://prestosql.io/docs/0.172/functions/array.html#cardinality
  [reduce]: https://prestosql.io/docs/0.172/functions/array.html#reduce
  [concat]: https://prestosql.io/docs/0.172/functions/array.html#concat
  [array_union]: https://prestosql.io/docs/0.172/functions/array.html#array_union
  [array_intersect]: https://prestosql.io/docs/0.172/functions/array.html#array_intersect
  [reverse]: https://prestosql.io/docs/0.172/functions/array.html#reverse
  [slice]: https://prestosql.io/docs/0.172/functions/array.html#slice
  [zip]: https://prestosql.io/docs/0.172/functions/array.html#zip
  [array_agg]: https://prestosql.io/docs/0.172/functions/aggregate.html#array_agg

### Maps

Just as with arrays, map access looks a lot like it does in programming languages with literal maps (e.g. dictionaries in Python, objects in JavaScript, hashes in Ruby).

A column called `params` with type `map<string,string>` can be accessed like this:

```sql
SELECT
  params['id'] AS id,
  params['value'] AS value
FROM my_table
```

The [`element_at`][map_element_at] function also works with maps, e.g. `element_at(tags, 'id')`. However, both square brackets and `element_at` returns `NULL` when an element is not found, in contrast to the array versions that result in an error.

There are not as many [map functions][map] as there are array functions, and except for `element_at` and [`cardinality`][map_cardinality], they are all about transforming maps.

You can create a new map with transformed keys with [`transform_keys`][transform_keys], or values with [`transform_values`][transform_values], create a subset of a map with [`map_filter`][map_filter], or combine maps with [`map_concat`][map_concat].

#### Flattening maps

Although less common, maps can be used with `UNNEST` to pivot them into rows. See [the article `UNNEST` for more information](/articles/unnest-arrays-to-rows/).

  [map]: https://prestosql.io/docs/0.172/functions/map.html
  [map_element_at]: https://prestosql.io/docs/0.172/functions/map.html#element_at
  [map_cardinality]: https://prestosql.io/docs/0.172/functions/map.html#cardinality
  [transform_keys]: https://prestosql.io/docs/0.172/functions/map.html#transform_keys
  [transform_values]: https://prestosql.io/docs/0.172/functions/map.html#transform_values
  [map_filter]: https://prestosql.io/docs/0.172/functions/map.html#map_filter
  [map_concat]: https://prestosql.io/docs/0.172/functions/map.html#map_concat

### Lambda expressions

Many times you want to process the elements of arrays and maps. In many programming languages there is the concept of lambda functions, anonymous functions, or blocks, that are often used to process lists in very compact ways. Athena has something similar, and many [functions that process complex types support them][lambda_functions].

Say you have an array of strings and want to uppercase them, this can be done like this:

```sql
SELECT transform(tags, tag -> upper(tag))
FROM my_table
```

The lambda expression is in this case `tag -> upper(tag)`. The symbol before `->` is the argument and the expression after is the body. The body can use almost all functions, and in the example above I use the [`upper`](https://prestosql.io/docs/0.172/functions/string.html#upper) string function.

Functions that take lambda expressions act on each row individually, they are not aggregate functions in the SQL sense. It helps me to think of a regular aggregate function like [`max`](https://prestosql.io/docs/0.172/functions/aggregate.html#max) as operating vertically on a table, and an array function like [`array_max`][array_max] as operating horizontally on the elements of a column of a single row.

Functions that operate on arrays usually have one argument, the element, and functions that operate on maps two, the key and the value. When there is more than one argument the argument list must be enclosed in parenthesis, with one argument the parenthesis are optional. This is, for example, how you create a new map with all values in upper case:

```sql
SELECT transform_values(params, (key, value) -> upper(value))
FROM my_table
```

  [lambda_functions]: https://prestosql.io/docs/0.172/functions/lambda.html

### Working with free form structures

Sometimes it's not possible to describe the shape of a data set definitely. There may be parts of it that is free form, like the CloudTrail event properties I discussed above. For these situations, and others, Athena has a type that is only available at runtime called `JSON`. You can't declare a column to be of type `JSON`, you have to use the `string` type and cast to `JSON` in queries, or use one of Athena's [JSON functions][json].

The `requestParameters` property of [CloudTrail][cloudtrail] events is almost always a map with string keys and complex values, though sometimes it's `NULL` and sometimes it's the string "null". Because of the occational string value you have to use the type `string` for the column, otherwise `map<string,string>` could have worked too.

When something decrypts using KMS it generates a CloudTrail event with these request parameters:

```json
{"encryptionContext":{"aws:lambda:FunctionArn":"arn:aws:lambda:eu-west-1:1234567890:function:some-function"},"encryptionAlgorithm":"SYMMETRIC_DEFAULT"}
```

Say you want to do some statistics on this, for example count the number of decryptions with different encryption algorithms, or which Lambda functions are doing the most decryptions. Athena comes with a library of [JSON functions][json] that let you write [JSONPath](https://jsonpath.com) expressions to extract values out of JSON strings. To extract a scalar value you use the [`json_extract_scalar`][json_extract_scalar] function:

```sql
SELECT
  json_extract_scalar(requestparameters, '$.encryptionAlgorithm') AS encryption_algorithm,
  COUNT(*) AS "count"
FROM cloudtrail_events
```

In JSONPath, `$` represents the document and you access properties more or less like you would in JavaScript. Doing the same as above but with the Lambda function ARN requires using square brackets since the property name contains colons:

```sql
SELECT
  json_extract_scalar(requestparameters, '$.encryptionContext["aws:lambda:FunctionArn"]') AS encryption_algorithm,
  COUNT(*) AS "count"
FROM cloudtrail_events
```

In these cases the value extracted was a scalar, but you can extract arrays and objects too, using [`json_extract`][json_extract]. The type of the result of this function is JSON.

The JSON type and functions are useful for working with arbitrary structures and columns where the schema is different from row to row. I've also come across [situations where casting to the JSON type can be a way to get around tricky situations where types don't exactly match](https://stackoverflow.com/questions/62664578/athena-union-do-i-need-to-define-struct-columns). It's sort of a wildcard type, and can both be used and abused.

### Hide the sausage making with views

When your queries become really complicated due to long expressions that extract values from deeply nested structures, or multiple levels of aggregation and flattening, it's a good idea to create a view to hide all the messy sausage making. It's going to be much easier for the people and code that will query the data set if it's clean and neat.

  [json]: https://prestosql.io/docs/0.172/functions/json.html
  [json_extract_scalar]: https://prestosql.io/docs/0.172/functions/json.html#json_extract_scalar
  [json_extract]: https://prestosql.io/docs/0.172/functions/json.html#json_extract

## Complex types in results

So far I've described how to work with complex types in the data, how you can work with them in queries, and now it's time to discuss how to deal with them in the result of a query.

Athena stores query results as CSV files on S3. Regardless of whether you use the console, the API, or the JDBC driver, the results end up as CSV on S3.

CSV is not exactly known for it's great support for complex types, so what do you do when you want to return an array or a map from a query?

Athena won't stop you from having arrays and maps in the result, it will dutifully serialize these values into CSV – and make a proper mess out of things. Its serialization format for lists and maps does not quote the elements, keys, or values, which means that it's very easy to produce output that is ambiguous. If you see `"[hello, world]"` in an Athena output file there is no way to tell if the value was an array with one element ("hello, world") or two elements ("hello" and "world"). You can also not tell numbers and strings apart, and Athena's query metadata also doesn't contain that information, it only specifies if a column is an array or map, not the types of the elements, keys, or values.

Because of this you should _never_ return raw arrays or maps from queries. It may appear to work for some test cases, but in general it is completely unsafe.

So what to do instead? The answer is is the `JSON` type. The serialization format for that type is, you guessed it, JSON. JSON serialization is unambigious and retains enough type information for most situations. It's not perfect, but it's a lot better than the alternative.

What you do is that in any query that returns complex types, you wrap those expressions in `CAST(… AS JSON)`, for example:

```sql
SELECT
  article_id,
  CAST(array_agg(tag) AS JSON) AS tags
FROM my_table
```

Athena's result metadata will indicate that the `tags` column is a string, and you will have to parse it in the code that reads the result data – but in contrast to returning a raw array you will be able to parse it!

## Summary

You can model very elaborate complex types in Athena tables, just look at the [CloudTrail schema][cloudtrail], with it's arrays of structs, and structs within structs. Even when properties are completely free form you won't get stuck because there's the [JSON type and functions][json] that let you unpack and work with them at query time.

When queries get too complicated you can create views to hide the details of how to extract and flatten the values from hierarchical and messy data models.

Even when you don't have complex types in the source data you can benefit a lot from Athena's support for complex types. Being able to aggregate rows into arrays can make things that have always been clunky in the relational model much more elegant.

The only problem with Athena and complex types is really how to deal with them in results, and that the consumers of the results may not expect or know how to deal with complex types. It's unfortunate that the Athena team didn't think through the consequences of using CSV as an output format when it comes to complex types, but luckily we can work around it using the JSON data type.

  [cloudtrail]: https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-logs.html
