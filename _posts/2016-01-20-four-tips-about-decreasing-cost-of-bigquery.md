---
layout: post
title: "BigQuery - Four Tips about Decreasing Cost of BigQuery"
description: "Design carefully or it will make you poor"
modified: 2016-02-03
tags: BigQuery
category: Snippets
image:
  feature: abstract-14.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

> **UPDATE 2016-02-03:** Change of the opinion of deviding table into smaller ones.

BigQuery is a very powerful data analytics service.
A very heavy query on massive data may only take seconds in BigQuery.
Among all the interesting features of BigQuery, one thing you should most keep in mind,
is the way BigQuery charge your money.

## How does BQ charge you

In general, it **charges you for the size of the data you evaluate in your query**.
The more data your query *evaluate*, the poorer BQ will get you.

One thing that will draw attention is **what kind of operation will be considered as EVALUATE?**

Basically, **every column mentioned in your query will be considered as evaluated data.**

Let's see. For example, I have a table with around **1,000,000 rows**' data, which is around **500 MB** in total.

First, let's fetch all the data from the table:

{% highlight sql %}
SELECT * FROM [ds.fat_table]
{% endhighlight %}

Because this query directly fetches all the data from the table, the size of data will be charged is, without doubt, **500MB**.

If we only fetch a part of the columns, the cost will dramatically fall.

{% highlight sql %}
SELECT col1, col2 FROM [ds.fat_table]
{% endhighlight %}

According to the actual total size of data of `col1` and `col2`, BQ will just charges you the size of data you fetched.
If the data of `col1` and `col2` in this table is 150MB only, this query will charge you **150MB**.

But the **size of evaluated data is not only the size of fetched data**.
Any column appears in `WHERE`, `GROUP BY`, `ORDER BY`, `PARTITION BY` or any other functions,
will be considered as "evaluated".

{% highlight sql %}
SELECT col1, col2 FROM [ds.fat_table]
WHERE col3 IS NOT NULL
{% endhighlight %}

The charged size of data is not just the total size of data in `col1` and `col2`,
but the total size of `col1`, `col2`, `col3`, because `col3` is in the `WHERE` clause.
If the data in `col3` is 30MB, then the query will charge you **180MB** in total.

> Note all the data in `col3` will be charged, not just the part that `IS NOT NULL`.

Actually, BigQuery did optimize the pricing for us.
If there is some columns is not necessary in the `SELECT` list of a sub-query,
BigQuery will not charge you for that.

{% highlight sql %}
SELECT col1, col2
FROM
  (
  SELECT * FROM [ds.fat_table]
  ) AS sub_table
WHERE col3 IS NOT NULL
{% endhighlight %}

As you can see, even if we selected all the columns in the table,
BQ will not charge all of them.
In this case, only the data of `col1`, `col2`, `col3` will be charged.

## Tips on optimizing charge

After knowing how BigQuery would charge us,
it will be clear on how to save our money.

Here is some tips I concluded from the experience of using BigQuery.

### Tip 0: Make sure you need BigQuery

This is not actually a tip, but a very important point before you start to use BQ.

In general, BigQuery is designed to deal with massive data,
and of course designed to charge from processing the massive data.

If you don't have the data that can't be processed by your current tech,
you could just keep using *MySQL*, *SPSS*, *R*, or even *Excel*.

Because, well, they're relatively cheap.

### Tip 1: Do use wildcard function

This will be your very first step to save money on BigQuery.

Let's say, we have a `log_table` to store all the access log lines on the server.

If I want to do some query on only one or two day's log, for instance,
get the users who accessed to the server in certain days,
there is nothing I can do rather that:

{% highlight sql %}
SELECT user_id
FROM
  ds.log_table
WHERE DATE(access_time) = '2016-01-01' or DATE(access_time) = '2016-01-02'
GROUP BY user_id
{% endhighlight %}

Well, never do this.

If you collected years of logs in this table,
this one single query will make you cry.

The better solution, the correct one, is to import the log of each day
into different tables with a date suffix in the first place, like following:

~~~ html
log_table20160101
log_table20160102
log_table20160103
log_table20160104
~~~

By doing this, the [table wildcard function](https://cloud.google.com/bigquery/query-reference?hl=en#tablewildcardfunctions) will allow you to select from a
certain range of table:

{% highlight sql %}
SELECT user_id
FROM
  TABLE_DATE_RANGE(ds.log_table, TIMESTAMP('2016-01-01'), TIMESTAMP('2016-01-02'))
GROUP BY user_id
{% endhighlight %}

Now you will not be charged of the data of dates that you don't want,
and even the data of `access_time` will not be charged.

***DO USE THIS PATTERN***, this is really import if you have some date will be kept importing day by day.

### Tip 2: Don't use query when you can actually use meta data

This is the most specific tip, but really should be considered before you design the tables.

For example, if you have a table to store the created users,
and you want to know the number of created users in each day,
don't do following:


{% highlight sql %}
SELECT created_date, COUNT(user_id) AS user_number
FROM
  (
  SELECT DATE(created_time) AS created_date, user_id
  FROM ds.users
  )
GROUP BY  created_date
{% endhighlight %}

This would cost you tons of coin if your website have, sadly, millions of users,
while you can actually achieve this free.

You can just:

1. Import newly created users of each day into different tables following [wildcard](https://cloud.google.com/bigquery/query-reference?hl=en#tablewildcardfunctions) pattern.
2. Call API to get the number of rows in each table, which is free.

There are many usecase just like this:
the sessions of each day, the transaction of each day, the activities of each day.

They are all free.


### Tip 3: Separate tables carefully

Maybe after the first tip, you may think: *"Yeah, query against the data that I only need as much as possible right? Let's separate our table as tiny as possible lol."*

***That's a trap.***

BigQuery does charge of the size of data we use in the query, BUT, **in the unit of 10MB**.
Meaning, even if I have query query that only uses 1kb of data, this query will be charged 10MB.

For example, if you are running a B2B service which consists hundreds of shops in your system.
So, you don't just separate your table by date, you also separate the table by shop with shop_id.

So you have:

~~~ html
access_log_shop001_20160101
access_log_shop002_20160101
access_log_shop001_20160102
access_log_shop002_20160102
~~~

But in each table you only have less than 1MB data.
If you have to use this access_log table to generate another kind of tables separated by shop,
you will be charged by `10MB * <num-of-shops> * <number-of-days>`,
while if you don't separate access_log table by shop you will be charged `1MB * <num-of-shops> * <number-of-days>` in the worst case.

So, is it bad idea to separate tables other than dates?
NO, it depends.

As you can see, if the size of access_log for each shop on each day is around 10MB or even bigger,
this design is actually perfect.

So, whether separate further depends on the size of your data.
The point is to know the **granularities** of your data before you make the decision.

**Update 2016-02-03**:

Actually, dividing table by not only date but also some other id will make BigQuery not functioning well.

According to my own experience, I have 20000+ tables in one dataset separated by both dates and a id.
The tables look like `[table_name]_[extra_id]_[date_YYmmdd]`.

And following issues will truly happen:

- BigQuery's API call will fail with InternalException occasionally. This will happen to the API call used to create empty table with given schema.
- There are following three kinds of time record in each query you have fired. Usually the interval between first one and second one is less than 1 second. When you have a lot of tables like my case, the gap between `creationTime` and `startTime` could last 20 seconds.
  - `creationTime`: The time when your request arrives the BQ server.
  - `startTime`: The time BQ server to start execute the Query.
  - `endTime`: The time that BQ server finishes you query or finds it invalid.
- BigQuery start to be not able to find your table. There will be query failing because of errors like ***FROM clause with table wildcards matches no table*** or ***Not found: Table ***, while you know exactly the tables do exist. And when you run the failed query again without fixing anything, the query will dramatically succeed.

So, I **STRONGLY** recommend not to use too many tables in the same dataset.


### Tip 4: Think, before use mid-table

The mid-table is the table you use to store the temporary result.

***Do use it when:***

*There will be more than one queries going to use this mid-table.*

Let's say, by doing query A, you get the result you want.
But there is another query B using the sub-query C within query A.

If don't use mid-table, you will be charged by:

- sub-query C
- remaining query in A after sub-query C
- sub-query C
- remaining query in B after sub-query C

But if you use mid-table to store the result of sub-query C,
you will be charged by:

- sub-query C
- remaining query in A after sub-query C
- remaining query in B after sub-query C

By doing this, you can save at least the money cost by the data of the size of the mid-table.


***Do NOT use it when:***

*If you only fear your query is too long, while there is no other query using the mid-table you generate.*

Not all people will agree on this.
Some people may start to say "This is too nasty", "God, this query is too fat." or even "WTF is this?".

Well, generate mid-table in this situation surely charging you more.
I will just to choose save more of my money.

And if you format your query well, no matter how long your query grows,
it will still looks nice.



_


So, these tips are some thing I concluded these days, hope they help you,
and save your money.
