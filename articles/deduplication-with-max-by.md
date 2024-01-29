---
title: Deduplicate faster with MAX_BY and ARBITRARY
date: 2023-10-09
author: Theo Tolv
---
# Deduplicate faster with MAX_BY and ARBITRARY

I often work with data sets that contain duplicate rows. In order to get the correct results I need to deduplicate the rows by selecting the row that contains the latest data. You may find this familiar if you've ever worked with change data capture (CDC), or perhaps an IoT data set where each row is an update from a sensor and you want the latest state.

Deduplicating rows can be surprisingly difficult in SQL, but in this post I will show you a few features of Athena that can help.

Imagine we have an IoT data set that looks something like this:

time             | sensor_id     | location        | battery_charge
-----------------|---------------|-----------------|--------------:
2023-10-09 10:45 | humidity-1    | office-west-1   |             65
2023-10-09 10:50 | humidity-1    | office-west-1   |             64
2023-10-09 10:56 | humidity-1    | office-west-1   |             64
2023-10-09 11:00 | humidity-1    | office-west-1   |             62
2023-10-09 10:50 | temperature-1 | office-east-1   |             33
2023-10-09 10:55 | temperature-1 | office-east-1   |             31
2023-10-09 11:00 | temperature-1 | office-east-1   |             29
2023-10-09 10:45 | humidity-2    | office-east-1   |             95
2023-10-09 10:49 | humidity-2    | office-east-1   |             95
2023-10-09 11:00 | humidity-2    | office-east-1   |             94

Each row is an event sent from a sensor, with the time, the sensor ID, the sensor location, and it's current battery charge. The sensors send an event roughly every five minutes. Sometimes an update fails to be sent for some reason, or it's delayed. The real world is messy, after all.

Let's say we want to build a monitoring application that displays the latest known battery charge for each sensor. This feels like it should be easy, we should be able to order by the time and sensor ID and just pick the most recent row, something like `ORDER BY time DESC LIMIT 1` – but how to do that for each sensor ID and not for the whole data set? What if we use `GROUP BY sensor_id`? No, the sorting and limiting still applies to the whole result, hm… it's more complicated than it first appears.

## The hard way, using a window function

If you just want to know how to do it the easy way you can skip this section. I'm going to show how this is done in standard SQL, and it's something you might have seen before. In the next section I'll show you how to do it in Athena with less complex SQL.

In most SQL engines, the way you solve this is with window functions. Window functions are very powerful and let you describe operations to apply on arbitrary subsets of all rows, and also define how these should be ordered. Explaining window functions is out of scope for this guide, but [Wikipedia does a decent job](https://en.wikipedia.org/wiki/Window_function_(SQL)).

Someone experienced with PostgreSQL or SQL Server may suggest we use the following SQL to find the last event for each sensor:

```sql
SELECT sensor_id,
       battery_charge
FROM (
  SELECT ROW_NUMBER() OVER (PARTITION BY sensor_id ORDER BY time DESC) AS row_number,
         sensor_id,
         battery_charge
  FROM events
)
WHERE row_number = 1
```

It's not _that_ much SQL, but it's far from straight forward. The short explanation is that the `ROW_NUMBER() OVER …` expression finds the row number for this row in the set of rows with the same value of `sensor_id`, when ordered by `time` in descending order. This means that the row with the most recent value of `time` will get row number 1. We use this fact in the outer `SELECT` to pick only rows with row number 1. There will exactly one of these per `sensor_id` value, and it will be the one with the most recent information, and therefore most up to date value for `battery_charge`.

The detail that makes using window functions extra complex, and cause queries to be unnecessarily complex is that they often require multiple steps like this. There needs to be an inner query that computes the row number, and and outer that filters by the row number, because `WHERE` clauses are not allowed to have window functions.

I find queries with window functions much more difficult to understand than queries without them. They often require multiple steps, and the relationship between the steps aren't always clear, if you don't know the pattern. They are also computationally and memory intensive to process. There are problems that can only be solved by window functions, but when I can I prefer to write my queries without them.

## The quick and easy way, using `MAX_BY`

Athena has a set of powerful aggregate functions that replace some common uses of window functions. For the battery charge problem we can use my favourite: [`MAX_BY`](https://trino.io/docs/current/functions/aggregate.html#max_by). It's a cousin of `MAX`, but with a trick up it's sleeve: instead of returning the greatest value of a column in a group, it can return the value from _another_ column from the group where the greatest value of a column was found. For example, `MAX_BY(battery_charge, time)` returns the value of `battery_charge` from the row with the greates value of `time` within the group.

This means that all we need to find the most up to date battery charge information for each sensor is this:

```sql
SELECT sensor_id,
       MAX_BY(battery_charge, time) AS battery_charge
FROM events
GROUP BY sensor_id
```

Running this on our IoT data returns the following:

sensor_id     | battery_charge
--------------|--------------:
temperature-1 | 29
humidity-2    | 94
humidity-1    | 62

Not only does using `MAX_BY` result in simpler SQL, for the queries I run where I need to deduplicate rows it is faster, and much more rarely exhaust the maximum resources Athena can allocate.

## Including metadata using `ARBITRARY`

Now that I've shown you `MAX_BY`, let me also introduce you to its siblings [`MIN_BY`](https://trino.io/docs/current/functions/aggregate.html#min_by) and [`ANY_VALUE`](https://trino.io/docs/current/functions/aggregate.html#any_value), a.k.a. `ARBITRARY`. `MIN_BY` is just the opposite of `MAX_BY`, but `ARBITRARY` is different.

`ARBITRARY`, is at first glance an odd aggregate function. Like `MAX`, `MIN`, `MAX_BY`, and `MIN_BY`, it returns a value of a column from the group, but not any particular value. Why is this useful?

When I run my queries that require deduplication, some properties work like `battery_charge` where each update has a newer value, but other properties don't change. They're like the `location` column in the IoT dataset we're working with in this guide. If we want to include the location of the sensor in our battery charge report, we could use `MAX_BY` for that column too. Or we could use `MIN`, `MAX`, or even `MIN_BY`. They would all return the same value, because the value is the same on all rows. We could also use `ARBITRARY`, and tell the engine that we don't care. If we don't need the greatest value, why ask the engine to do the calculations?

At this point we perhaps need a small aside about why we need any aggregate function at all. It goes back to the SQL standard, which says that if there is a `GROUP BY` clause, an expression has to be in that clause, or be an aggregate expression. In other words, if we say `SELECT sensor_id, location` we also must say `GROUP BY sensor_id, location`. If we don't, the engine would know which value within the group to use. We don't want to add an unnecessary expression to the `GROUP BY` clause, which means we need to use an aggregate function. Luckily, Athena has the perfect aggregate function to express the semantics we're after.

```sql
SELECT sensor_id,
       ARBITRARY(location) AS location
       MAX_BY(battery_charge, time) AS battery_charge
FROM events
GROUP BY sensor_id
```

Running this on our IoT data returns the following:

sensor_id     | battery_charge | location
--------------|---------------:|--------------
temperature-1 | 29             | office-east-1
humidity-2    | 94             | office-east-1
humidity-1    | 62             | office-west-1

## Conclusion

The most simple and efficient way to deduplicate rows in a CDC data set, or finding the latest event for a device, is to use plain `GROUP BY` with the `MAX_BY` and `ARBITRARY` aggregate functions. `ARBITRARY` can also come in handy to work around that pesky SQL constraint that expressions that aren't in the `GROUP BY` clause need to use an aggregate function.

I find myself using these functions almost on a daily basis, and I hope knowing about them will save you from having to look up the syntax for window functions, and the incantations needed to make them work.
