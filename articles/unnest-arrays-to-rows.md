---
title: Flatten arrays into rows with UNNEST
date: 2020-07-03
author: Theo Tolv
---
# Flatten arrays into rows with UNNEST

In contrast to many relational databases, Athena's columns don't have to be scalar values like strings and numbers, they can also be arrays and maps. In fact, they can be deep structures of arrays and maps nested within each other. Queries can also aggregate rows into arrays and maps.

This makes it possible to do pretty advanced things, but it's not always easy to wrap your head around what's going on since almost everything in the SQL world was made for scalar column values. Most of the time you also want something flat as output, as Athena's CSV output format isn't really suitable for complex values, and most consumers of the output probably don't handle complex values either.

In data formats like JSON it's very common to have arrays and map properties, and one question that often comes up is how you flatten these structures to work better in a traditional tabular format – in other words, how to turn array elements into rows. The answer is the [`UNNEST`][1] operator.

In this article I will cover [how to flatten arrays to rows](#unnesting-arrays), [how to flatten maps to rows](#unnesting-maps), but also [when you should be using `UNNEST`](#when-to-unnest).

## Unnesting arrays

`UNNEST` is a bit peculiar as it is is an operator that produces a relation, unlike most functions which transform or aggregate scalar values.

Say you have an Athena table called `cities_and_countries` that is set up to read JSON data looking like this:

```json
{"country": "se", "cities": ["Stockholm", "Göteborg", "Malmö"]}
{"country": "us", "cities": ["New York", "Seattle"]}
{"country": "fr", "cities": ["Paris", "Nice", "Marseille", "Grenoble"]}
```

With `UNNEST` you can flatten this into a relation with the name of each city and its country code, like this:

```sql
SELECT
  unnested_cities.city,
  cities_and_countries.country
FROM cities_and_countries
CROSS JOIN UNNEST(cities_and_countries.cities) AS unnested_cities (city)
```

Which would give the following result:

country | city
--------|----------
se      | Stockholm
se      | Göteborg
se      | Malmö
us      | New York
us      | Seattle
fr      | Paris
fr      | Nice
fr      | Marseille
fr      | Grenoble

It's like the arrays have been pivoted (or unpivoted, depending on your point of view). Another way to think of it is that the source table has been joined with another table with all the array elements, using a join key that identifies which row they belonged to.

### Deconstructing the query

It's going to be easiest to understand this query by starting from the end.

The last line contains a lot, but it's the `UNNEST(cities_and_countries.cities) AS unnested_cities (city)` part that is the most important. It tells Athena to for each row, flatten the array `cities` into a relation called `unnested_cities` that has a column called `city`. The alias `unnested_cities` is arbitrary, but more on that later. It helps me to think of this expression as pivoting the horizontal array into a vertical column, and that there exist a hidden column that tells Athena which source row each row in this new relation came from.

Moving on to the `CROSS JOIN`, this may look a bit scary. Cross joins can produce huge results as they combine each row in each relation with every other row in the other relation. However, I think that Athena has a special case for this type of cross join that knows that it should only combine each value in the unnested relation with the current row in the relation where the array comes from – and you can't replace it with any other type of join, so I think of it as syntactic sugar to fit the feature into the SQL structure.

The result of the cross join is a relation with the source rows repeated once per element in the source row's array, and an extra column that is the element itself. If you changed the query to `SELECT * FROM …` you would get the following result, which may help you understand what's going on:

country | cities                             | city
--------|------------------------------------|----------
se      | [Stockholm, Göteborg, Malmö]       | Stockholm
se      | [Stockholm, Göteborg, Malmö]       | Göteborg
se      | [Stockholm, Göteborg, Malmö]       | Malmö
us      | [New York, Seattle]                | New York
us      | [New York, Seattle]                | Seattle
fr      | [Paris, Nice, Marseille, Grenoble] | Paris
fr      | [Paris, Nice, Marseille, Grenoble] | Nice
fr      | [Paris, Nice, Marseille, Grenoble] | Marseille
fr      | [Paris, Nice, Marseille, Grenoble] | Grenoble

You can see that every column from the source relation exists in this relation as-is, even the column containing the array.

### Simplifying the query

I was extra verbose when I wrote the query because I wanted to make it as clear as I could where each part came from. In fact, you can drop the relation aliases since there is no ambiguity as to where the columns come from:

```sql
SELECT city, country
FROM cities_and_countries
CROSS JOIN UNNEST(cities) AS t (city)
```

The alias for the unnested relation is necessary syntactically, but as it is rarely needed to disambiguate columns it usually ends up as `t` or some other single letter alias.

I said earlier that you couldn't change the `CROSS JOIN` to any other type of join, but you can drop the `CROSS JOIN` and use syntax similar to an equijoin:

```sql
SELECT city, country
FROM cities_and_countries, UNNEST(cities) AS t (city)
```

I think this further shows that Athena has a special case for `UNNEST` and knows to combine the rows of the produced relation only with the source relation.

### Multiple arrays

`UNNEST` can be used to flatten more than one array, say we had a table similar to the one above, called `country_geography`, with a `rivers` column, and the following data:

```json
{"country": "se", "cities": ["Göteborg", "Umeå"], "rivers": ["Göta älv", "Ume älv"]}
{"country": "us", "cities": ["New York", "Seattle"], "rivers": ["Hudson River", "Cedar River"]}
{"country": "fr", "cities": ["Paris", "Nice", "Marseille"], "rivers": ["Seine", "Var"]}
```

Unnesting multiple arrays is just a matter of listing the array columns in the arguments to the `UNNEST` operator:

```sql
SELECT country, city, river
FROM country_geography, UNNEST(cities, rivers) AS t (city, river)
```

But what happens when you unnest multiple arrays like this? The first time I tried this I expected it to produce the cross product of the arrays, but in fact it's more like if you used the [`zip`][2] function on the arrays before unnesting: the first elements of each array will end up on the first row, the second elements of each array will end up in the second row, etc. If the arrays have different number of elements the shorter array is padded with `NULL`.

The query above would give the following result:

country | city      | river
--------|-----------|-------------
se      | Göteborg  | Göta älv
se      | Umeå      | Ume älv
us      | New York  | Hudson River
us      | Seattle   | Cedar River
fr      | Paris     | Seine
fr      | Nice      | Var
fr      | Marseille | NULL

### Empty arrays

Unnesting an array is a form of join, and different joins deal differently with missing values. `UNNEST` can probably be said to be like an inner join, because when an array is empty no rows are produced from its row, just like when you inner join and a value for the join key does not exist in the other table.

As we saw in the example of unnest with multiple arrays it's slightly more complicated in that situation. Only when all of the arrays of a row are empty will that row be missing from the result. As long as one of the arrays have an element the query will behave as if the other arrays were padded with `NULL`.

### Including element indices

You can add the `WITH ORDINALITY` clause to an `UNNEST` expression to get the element index as a separate column. Using the table in the examples above, this is how you would use it:

```sql
SELECT city, index, country
FROM cities_and_countries, UNNEST(cities) WITH ORDINALITY t(city, index)
```

Notice that the index is added last in the list of columns of the unnested relation.

The query above would give the following result:

country | index | city
--------|-------|----------
se      | 1     | Stockholm
se      | 2     | Göteborg
se      | 3     | Malmö
us      | 1     | New York
us      | 2     | Seattle
fr      | 1     | Paris
fr      | 2     | Nice
fr      | 3     | Marseille
fr      | 4     | Grenoble

Note that array indexes are 1-based in Athena.

This feature can be used to, for example, limit the number of rows produced to just the first two from each array, for example like this:

```
SELECT city, country
FROM cities_and_countries, UNNEST(cities) WITH ORDINALITY t(city, index)
WHERE index < 3
```

Which would result in this output:

country | index | city
--------|-------|----------
se      | 1     | Stockholm
se      | 2     | Göteborg
us      | 1     | New York
us      | 2     | Seattle
fr      | 1     | Paris
fr      | 2     | Nice

## Unnesting maps

You can also unnest maps. This is actually similar to unnesting two arrays. Remember how that was like using `zip` to combine the arrays pairwise? If the first array represented the map keys and the second the map values the result would be the same as unnesting a map.

To illustrate this, let's modify the data in the `country_geography` table that we looked at before:

```json
{"country": "se", "rivers_by_city": {"Göteborg": "Göta älv", "Umeå": "Ume älv"}}
{"country": "us", "rivers_by_city": {"New York": "Hudson River", "Seattle": "Cedar River"}}
{"country": "fr", "rivers_by_city": {"Paris": "Seine", "Nice": "Var"}}
```

Using `UNNEST` on the `rivers_by_city` map column gives in a relation with two columns, unlike when unnesting an array column, which only resulted in one column:

```sql
SELECT country, city, river
FROM country_geography, UNNEST(rivers_by_city) AS t (city, river)
```

The result is identical to the previous example where the cities and rivers were in separate arrays (except for the row with Marseille that was missing a river and not included here).

## When to unnest

Being able to work with arrays and maps is very powerful, but most often you don't want these data structures in the final result. Athena's CSV output does not handle array and map data properly, and in general tools expect CSV to be flat. `UNNEST` can be a good way to flatten the output.

`UNNEST` also serves as a bridge to the relational model. Consider a data set about articles, where each article has an array of tags. If you want to count the number of articles for each tag you want to run a query like `SELECT tag, COUNT(*) FORM articles GROUP BY tag`, but if what you have is a `tags` array you can't. Using `UNNEST` you pivot the hierarchical data model into a flat model that the relational model understands.

  [1]: https://prestosql.io/docs/0.172/sql/select.html#unnest
  [2]: https://prestosql.io/docs/0.172/functions/array.html#zip
