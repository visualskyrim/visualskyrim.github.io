---
layout: post
title: "BigQuery - Calculate User Access Sessions using BigQuery"
description: "Extremely complex calculation can be solved with one query in BQ"
modified: 2014-09-11
tags: BigQuery
category: Snippets
image:
  feature: abstract-3.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

Currently I was working on analytics with BigQuery.
It was quite a experience that you could accomplish any process on mass data
without considering any subtle performance issue.

The greatness and the thinking of the internal implementation of BigQuery is not going
to be discussed here.

Here I'm going to show how to use BigQuery to Calculate users' access session from
your access log with **one query**.

First, let's define the session.

> A ***session*** is a sequence of a user's access. Each access in the sequence is no more
than a specific period of time before the next access.
>
> The ***length of a session*** is the period of time after the first access in this session and
> the last access in this session.

Then, we need to describe the table which stores the data from the access log.
At least we should have following two column in this table:

1. access_time: a timestamp that represents the time when access happen. In our case, let's say 30 minute.
2. user_id.

And now we could start to make the query step by step:

***Step 1***: preprocess the log table:

In this step, we do some type converting to make the calculation in the following
steps easier:

{% highlight sql %}
SELECT
  user_id,
  TIMESTAMP_TO_SEC(access_time) AS access_time_sec, -- Convert timestamp to seconds
FROM
  ds.access_log_table -- This is the table where you put your log data into
{% endhighlight %}

***Step 2***: find the last access to the current access

After last step, we now have every access for each row.
In this step, we are going to add a column to represent last access to the access in each row.

{% highlight sql %}
SELECT
  user_id,
  access_time_sec,
  -- The lag + partition + order combination is to get the previous access to the current access in the row
  LAG(access_time_sec, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec) AS prev_access_time_sec
FROM
  (
-- previous query
-- SELECT
--   user_id,
--   TIMESTAMP_TO_SEC(access_time) AS access_time_sec, -- Convert timestamp to seconds
-- FROM
--   ds.access_log_table -- This is the table where you put your log data into
-- previous query end
  )
{% endhighlight %}


The `LAG` function is used with `PARTITION BY`, which is used to get the specific column of the previous row
in the same partition.

The partition is separated by `PARTITION BY` function, while the order within the partition is specified
by `ORDER BY` inside the `OVER` statement.

In this case, we separate all the accesses in the partitions for each user by

```
PARTITION BY user_id
```

Then, to make sure the order of time in each partition, we order each partition with:

```
PARTITION BY user_id ORDER BY access_time_sec
```

To get the access_time_sec of previous row, we do:

```
LAG(access_time_sec, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec) AS prev_access_time_sec
```

> More details about function `LAG`, you can refer to the [Doc](https://cloud.google.com/bigquery/query-reference?hl=en).

Of course, like what you might be thinking of right now, for each partition,
the first row in each partition doesn't have the `prev_access_time_sec`.
In the result, it will be `null` at this point.

Leave it like that, and we will deal with it later.

After this step, we will get the result like:

```
| user_id | access_time_sec | prev_access_time_sec |
|---------|-----------------|----------------------|
|         |                 |                      |
```


***Step 3***: Decide whether an access is the beginning of a session

In this step, we are gonna tag each access whether it is the beginning of sessions.
And as we said already, **a session will break if the next session is 30 minute after current session**.

To achieve that, we do:

{% highlight sql %}
SELECT
  user_id,
  access_time_sec,
  prev_access_time_sec,
  IFNULL -- The first access of each partition is the beginning of session by default.
    (
      access_time_sec - prev_access_time_sec >= 30 * 60,
      TRUE
    ) AS start_of_session
FROM
  (
--  previous query
--  SELECT
--    user_id,
--    access_time_sec,
--    -- The lag + partition + order combination is to get the previous access to the current access in the row
--    LAG(access_time_sec, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec) AS prev_access_time_sec
--  FROM
--    (
--    SELECT
--    user_id,
--    TIMESTAMP_TO_SEC(access_time) AS access_time_sec, -- Convert timestamp to seconds
--    FROM
--      ds.access_log_table -- This is the table where you put your log data into
--      previous query end
--    )
  )
{% endhighlight %}


As we just said, the first access in each partition can only have prev_access_time_sec as `null`.
So they're considered as the beginning of session by default.


Now we have result like:

```
| user_id | access_time_sec | prev_access_time_sec | start_of_session |
|---------|-----------------|----------------------|------------------|
|         |                 |                      |                  |
```
