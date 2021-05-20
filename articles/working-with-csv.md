---
title: Working with CSV
date: 2020-07-05
author: Theo Tolv
---
# Working with CSV

CSV could be the most common data interchange format there is, and since it's almost as old as computers, it's no surprise that there's no universally agreed upon way to encode data as CSV. There can be different delimiters – commas are just the character the format got its name from, and sometimes its semicolon, or tab (also known as TSV, of course).

Athena unsurprisingly has good support for reading CSV data, but it's not always easy to know what you should use as there are multiple implementations with completely different features and ways of configuration.

In this article I will cover how to use the [default CSV implementation](#the-default-implementation), what do do when you have [quoted fields](#quoted-fields), how to [skip headers](#skipping-header-lines), how to deal with [NULL and empty fields](#empty-fields-and-null-values), how [types are interpreted](#type-conversion), [column names and column order](#column-names-and-order), as well as [general guidance](#which-one-to-use).

## The default implementation

The component in Athena that is responsible for reading and parsing data is called a serde, short for serializer/deserializer. If you don't specify anything else when creating an Athena table you get a serde called [`LazySimpleSerDe`][1], which was made for delimited text such as CSV. It can be configured for different delimiters, escape characters, and line endings, among other things.

You would be forgiven for thinking that by default would be configured for some common CSV variant, but in fact the default delimiter is the somewhat esoteric `\1` (the byte with value 1), which means that you must always specify the delimiter you want to use. The default escape character is backslash.

The shortest table definition for CSV data is something like this:

```sql
CREATE EXTERNAL TABLE city_data (
  country string,
  city string,
  population int
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION 's3://example-bucket/city-data'
```

You can also configure the escape character and line endings by adding `ESCAPED BY '\\'` and `LINES TERMINATED BY '\n'` before `LOCATION`.

When creating tables in Athena, the serde is usually specified with its fully qualified class name and configuration is given as a list of properties. However, being the default, `LazySimpleSerde` has special syntax for configuration and creating a table, and that's the syntax used above. Using regular syntax common to all serdes, this is how you would create the table:

```sql
CREATE EXTERNAL TABLE city_data (
  country string,
  city string,
  population int
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES (
  'field.delim' = ',',
  'escape.delim' = '\\',
  'line.delim' = '\n'
)
LOCATION 's3://example-bucket/city-data'
```

The downside of `LazySimpleSerDe` is that it does not support quoted fields. For that you need to use the other CSV serde provided by Athena.

## Quoted fields

If your flavor of CSV includes quoted fields you must use the other CSV serde supported by Athena, [OpenCSVSerDe][2]. As the name suggests it's built on the [OpenCSV library](http://opencsv.sourceforge.net). Besides quote character, this serde also supports configuring the delimiter and escape character, but not line endings.

When you specify a quote character with `OpenCSVSerDe` fields don't all have to be quoted, it's possible to use quotes only when needed, for example when fields include the delimiter.

This is how you create a table that will use `OpenCSVSerDe` to read tab-separated values with fields optionally quoted by backticks, and backslash as escape character:

```sql
CREATE EXTERNAL TABLE city_data (
  country string,
  city string,
  population int
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
  'separatorChar' = '\\t',
  'quoteChar' = '`',
  'escapeChar' = '\\'
)
LOCATION 's3://example-bucket/city-data'
```

The default delimiter is comma, and the default quote character is double quote. The escape and quote character can be the same value, which is useful for situations where quotes in quoted fields are escaped by an extra quote as defined in RFC-4180, e.g. `1,"hello ""world""",2`.

## Skipping header lines

It's common with CSV data that the first line of the file contains the names of the columns. Sometimes files have a multi-line header with comments and other metadata. When this is the case you must tell Athena to skip the header lines, otherwise they will end up being read as regular data.

While skipping headers is closely related to reading CSV files, the way you configure it is actually through a table property called `skip.header.line.count`. It can be configured like this:

```sql
CREATE EXTERNAL TABLE city_data (
  country string,
  city string,
  population int
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://example-bucket/city-data'
TBLPROPERTIES (
  'skip.header.line.count' = '1'
)
```

For multi-line headers you can change the number to match the number of lines in your headers. Headers with a variable number of lines are not supported. This feature is supported by both `LazySimpleSerDe` and `OpenCSVSerDe`.

## Empty fields and NULL values

When `LazySimpleSerDe` and `OpenCSVSerDe` reads an empty field they interpret it differently depending on the type of the column. When the corresponding column is typed as string both will interpret an empty field as an empty string. For other data types `LazySimpleSerDe` will interpret the value as `NULL`, but `OpenCSVSerDe` will throw an error:

> HIVE_BAD_DATA: Error parsing field value '' for field 1: For input string: ""

`LazySimpleSerDe` will by default interpret the string `\N` as `NULL`, but can be configured to accept other strings (such as `-`, `null`, or `NULL`) instead with `NULL DEFINED AS '-'` or the property `serialization.null.format`.

## Type conversion

The serdes handle non-string column types differently. `OpenCSVSerDe` gets strings from the `OpenCSV` parser and then parses these strings to typed values, while `LazySimpleSerDe` converts directly from the byte stream. This results in the different interpretation of empty fields, as discussed above.

The difference in how they parse field values also means that they can interpret the same data differently. Besides how empty fields are treated, there are also differences in how timestamps and dates are parsed. `LazySimpleSerDe` expects the `java.sql.Timestamp` format similar to ISO timestamps, while `OpenCSVSerDe` expects UNIX timestamps.

## Text encodings

Both `LazySimpleSerDe` and `OpenCSVSerDe` by default assume that the data is UTF-8 encoded, and may garble non-UTF-8 data, or fail queries when the data contains byte sequences that are not valid UTF-8.

If your data is not UTF-8 you can configure `LazySimpleSerDe` with the `serialization.encoding` table property using one of Java's standard charset names (see [`java.nio.charset.Charset`](https://docs.oracle.com/javase/8/docs/api/java/nio/charset/Charset.html) for the details):

```sql
CREATE EXTERNAL TABLE city_data (
  country string,
  city string,
  population int
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION 's3://example-bucket/city-data'
TBLPROPERTIES (
  'serialization.encoding' = 'ISO-8859-1'
)
```

Unfortunately the `OpenCSVSerDe` seems to not to allow the encoding to be configured.

## Column names and order

The columns of the table must be defined in the same order as they appear in the files. Both CSV serdes read each line and map the fields of a record to table columns in sequential order. If a line has more fields than there are columns, the extra columns are skipped, and if there are fewer fields the remaining columns are filled with `NULL`.

You might think that if the data has a header the serde could use it to map the fields to columns by name instead of sequence, but this is is not supported by either serde. On the other hand, this means that the names of the columns are not constrained by the file header and you are free to call the columns of the table what you want.

Given the above you may have gathered that it's possible to evolve the schema of a CSV table, within some constraints. It's possible to add columns, as long as they are added last, and removing the last columns also works – but you can only do either or, and adding and removing columns at the start or in the middle also does not work.

In practice this means that if you at some point realize you need more columns you can add these, but you should avoid all other schema evolution. For example, if you at some point removed a column from the table, you can't later add columns without rewriting the old files that had the old column data.

## Which one to use?

In almost all cases the choice between `LazySimpleSerDe` and `OpenCSVSerDe` comes down to whether or not you have quoted fields. If you do, there is only one answer, `OpenCSVSerDe`. If you don't have quoted fields, I think it's best to follow the advice of the official Athena documentation and use the default, `LazySimpleSerDe`.

Anecdotally, and from some very unscientific testing, `LazySimpleSerDe` seems to be the faster of the two. This may or may not be due to the difference between how the two serdes parse the field values into the data types of the corresponding columns. I understand it, when using `LazySimpleSerDe` only columns used in the query are fully parsed.

## Conclusion

Overall I think it's fair to say that the state of CSV support in Athena is like the state of CSV in general: a mess. There are so many different conventions, attempts at standardization, and implementations that there just isn't one way to think about CSV. Between them, `LazySimpleSerDe` and `OpenCSVSerDe` handle a lot of what you can throw at them, but there are certainly cases where you would need features unique to both.

  [1]: https://docs.aws.amazon.com/athena/latest/ug/lazy-simple-serde.html
  [2]: https://docs.aws.amazon.com/athena/latest/ug/csv-serde.html