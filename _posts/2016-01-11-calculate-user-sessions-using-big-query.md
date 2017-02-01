---
layout: post
title: "BigQuery - Calculate User Access Sessions using BigQuery"
description: "Extremely complex calculation can be solved with one query in BQ"
modified: 2016-01-11
tags: BigQuery
category: Snippets
image:
  feature: abstract-12.jpg
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

## Step 1: preprocess the log table:

In this step, we do some type converting to make the calculation in the following
steps easier:

{% highlight sql %}
SELECT
  user_id,
  TIMESTAMP_TO_SEC(access_time) AS access_time_sec, -- Convert timestamp to seconds
FROM
  ds.access_log_table -- This is the table where you put your log data into
{% endhighlight %}

## Step 2: find the last access to the current access

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

{% highlight sql %}
PARTITION BY user_id
{% endhighlight %}

Then, to make sure the order of time in each partition, we order each partition with:

{% highlight sql %}
PARTITION BY user_id ORDER BY access_time_sec
{% endhighlight %}

To get the access_time_sec of previous row, we do:

{% highlight sql %}
LAG(access_time_sec, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec) AS prev_access_time_sec
{% endhighlight %}

> More details about function `LAG`, you can refer to the [Doc](https://cloud.google.com/bigquery/query-reference?hl=en).

Of course, like what you might be thinking of right now, for each partition,
the first row in each partition doesn't have the `prev_access_time_sec`.
In the result, it will be `null` at this point.

Leave it like that, and we will deal with it later.

After this step, we will get the result like:

~~~ html
| user_id | access_time_sec | prev_access_time_sec |
|---------|-----------------|----------------------|
|         |                 |                      |
~~~


## Step 3: Decide whether an access is the beginning of a session

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

~~~ html
| user_id | access_time_sec | prev_access_time_sec | start_of_session |
|---------|-----------------|----------------------|------------------|
|         |                 |                      |                  |
~~~

## Step 4: Decide whether the access is the end of session

Things become complex from here.

To achieve the goal of this step, we take two `select`s.

First we tag each row(access) with **whether the next access is the beginning of the session** in the partition with the same `user_id`:

{% highlight sql %}
SELECT
  user_id,
  access_time_sec,
  prev_access_time_sec,
  LEAD(start_of_session, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec, prev_access_time_sec) is_next_access_sos
FROM
  (
-- previous query
-- SELECT
--   user_id,
--   access_time_sec,
--   prev_access_time_sec,
--   IFNULL -- The first access of each partition is the beginning of session by default.
--     (
--       access_time_sec - prev_access_time_sec >= 30 * 60,
--       TRUE
--     ) AS start_of_session
-- FROM
--   (
--   SELECT
--     user_id,
--     access_time_sec,
--     -- The lag + partition + order combination is to get the previous access to the current access in the row
--     LAG(access_time_sec, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec) AS prev_access_time_sec
--   FROM
--     (
--     SELECT
--     user_id,
--     TIMESTAMP_TO_SEC(access_time) AS access_time_sec, -- Convert timestamp to seconds
--     FROM
--       ds.access_log_table -- This is the table where you put your log data into
--       previous query end
--     )
--   )
  )
{% endhighlight %}

Now we know for each access of each user, whether the next access is the beginning of the next session.

The reason why we need to know this, is, let's consider this:

Let's say there is one partition, which **is already ordered by access_time**, for a user like this following


~~~ html
| user_id | access_time_sec | prev_access_time_sec | start_of_session | is_next_access_sos |
|---------|-----------------|----------------------|------------------|--------------------|
|         |                 |                      | true             | false              |
|         |                 |                      | false            | true               |
|         |                 |                      | true             | true               |
|         |                 |                      | false            | false              |
|         |                 |                      | false            | false              |
|         |                 |                      | false            | true               |
|         |                 |                      | true             | null               |
~~~

The combination of `start_of_session` and `is_next_access_sos` and the meaning behind at this point must by one of the following:

~~~ html
| start_of_session | is_next_access_sos | this_access_must_be                                                                                        |
|------------------|--------------------|------------------------------------------------------------------------------------------------------------|
| true             | false              | the first access in the session with number of access >= 2                                                 |
| true             | true               | the only access in the session                                                                             |
| true             | null               | the only access in the session, and the last access in the partition                                       |
| false            | true               | the last access in the session with number of access >= 2                                                  |
| false            | false              | this access is not the first access nor the last in the session with number of access >= 3                 |
| false            | null               | the last access in the session with number of access >=2, and this access is the last one in the partition |
~~~

Knowing this, we can easily get whether an access is the last access in the partition.

{% highlight sql %}
SELECT
  user_id,
  access_time_sec,
  prev_access_time_sec,
  start_of_session,
  ISNULL(is_next_access_sos, TRUE) AS end_of_session -- if an access is the end of the partition, it must be the end of the session
FROM
  (
  SELECT
    user_id,
    access_time_sec,
    prev_access_time_sec,
    start_of_session,
    LEAD(start_of_session, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec, prev_access_time_sec) is_next_access_sos
  FROM
    (
--  previous query
--  SELECT
--    user_id,
--    access_time_sec,
--    prev_access_time_sec,
--    IFNULL -- The first access of each partition is the beginning of session by default.
--      (
--        access_time_sec - prev_access_time_sec >= 30 * 60,
--        TRUE
--      ) AS start_of_session
--  FROM
--    (
--    SELECT
--      user_id,
--      access_time_sec,
--      -- The lag + partition + order combination is to get the previous access to the current access in the row
--      LAG(access_time_sec, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec) AS prev_access_time_sec
--    FROM
--      (
--      SELECT
--      user_id,
--      TIMESTAMP_TO_SEC(access_time) AS access_time_sec, -- Convert timestamp to seconds
--      FROM
--        ds.access_log_table -- This is the table where you put your log data into
--        previous query end
--      )
--    )
    )
  )
WHERE
  start_of_session OR is_next_access_sos IS NULL OR is_next_access_sos -- only get the start or the end of session in the result
{% endhighlight %}


After this step, we get all the accesses that are **either the start or the end of a session** in the result.

~~~ html
| user_id | access_time_sec | prev_access_time_sec | start_of_session | end_of_session |
|---------|-----------------|----------------------|------------------|----------------|
|         |                 |                      |                  |                |
~~~

## Step 5: Get sessions

We are really close now.

We have all the start and end access of sessions in our result set (some of them are both start and end).

It's already very clear about how to get all the sessions now:

- A session must have a start access
- Then we rule out accesses that are not start access
- We get all the sessions

But that's not enough, nor fun.
Now let's say we also need to know the duration of sessions. That sounds fun enough.

Let's do the same trick again.
Remember, now we only have start and end access in the result of previous query.

So let's get the access time of previous access for each access in each partition with same user_id:

{% highlight sql %}
SELECT
  user_id,
  access_time_sec,
  LAG(access_time_sec, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec, prev_access_time_sec) AS prev_access_time_sec_2
  start_of_session,
  end_of_session
FROM
  (
--  previous query
--  SELECT
--    user_id,
--    access_time_sec,
--    prev_access_time_sec,
--    start_of_session,
--    ISNULL(is_next_access_sos, TRUE) AS end_of_session -- if an access is the end of the partition, it must be the end of the session
--  FROM
--    (
--    SELECT
--      user_id,
--      access_time_sec,
--      prev_access_time_sec,
--      start_of_session,
--      LEAD(start_of_session, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec, prev_access_time_sec) is_next_access_sos
--    FROM
--      (
--      SELECT
--        user_id,
--        access_time_sec,
--        prev_access_time_sec,
--        IFNULL -- The first access of each partition is the beginning of session by default.
--          (
--            access_time_sec - prev_access_time_sec >= 30 * 60,
--            TRUE
--          ) AS start_of_session
--      FROM
--        (
--        SELECT
--          user_id,
--          access_time_sec,
--          -- The lag + partition + order combination is to get the previous access to the current access in the row
--          LAG(access_time_sec, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec) AS prev_access_time_sec
--        FROM
--          (
--          SELECT
--          user_id,
--          TIMESTAMP_TO_SEC(access_time) AS access_time_sec, -- Convert timestamp to seconds
--          FROM
--            ds.access_log_table -- This is the table where you put your log data into
--            previous query end
--          )
--        )
--      )
--    )
--  WHERE
--    start_of_session OR is_next_access_sos IS NULL OR is_next_access_sos -- only get the start or the end of session in the result
  )
{% endhighlight %}


Because of `we only have start and end access in the result of previous query`, the previous access time of each access must be either of following:

- If the access is the start of session: this previous access time must be the end time of the previous session
- If the access is not the start of session: this previous access time must be the the start time of the current session <-- this is what we need

> Be careful when think this through, there might be access that is both the start and the end of the session.

When we are clear with this, we can get all the sessions with duration:

{% highlight sql %}
SELECT
  user_id,
  CASE
    WHEN NOT start_of_session AND end_of_session THEN access_time_sec - session_start_sec
    WHEN start_of_session AND end_of_session THEN 0  -- for sessions that only have one access
  END AS duration,
  CASE
    WHEN NOT start_of_session AND end_of_session THEN SEC_TO_TIMESTAMP(session_start_sec)
    WHEN start_of_session AND end_of_session THEN SEC_TO_TIMESTAMP(access_time_sec)  -- for sessions that only have one access
  END AS session_start_time
FROM
  (
  previous query
  SELECT
    user_id,
    access_time_sec,
    LAG(access_time_sec, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec, prev_access_time_sec) AS session_start_sec
    start_of_session,
    end_of_session
  FROM
    (
    SELECT
      user_id,
      access_time_sec,
      prev_access_time_sec,
      start_of_session,
      ISNULL(is_next_access_sos, TRUE) AS end_of_session -- if an access is the end of the partition, it must be the end of the session
    FROM
      (
      SELECT
        user_id,
        access_time_sec,
        prev_access_time_sec,
        start_of_session,
        LEAD(start_of_session, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec, prev_access_time_sec) is_next_access_sos
      FROM
        (
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
          SELECT
            user_id,
            access_time_sec,
            -- The lag + partition + order combination is to get the previous access to the current access in the row
            LAG(access_time_sec, 1) OVER (PARTITION BY user_id ORDER BY access_time_sec) AS prev_access_time_sec
          FROM
            (
            SELECT
            user_id,
            TIMESTAMP_TO_SEC(access_time) AS access_time_sec, -- Convert timestamp to seconds
            FROM
              ds.access_log_table -- This is the table where you put your log data into
              previous query end
            )
          )
        )
      )
    WHERE
      start_of_session OR is_next_access_sos IS NULL OR is_next_access_sos -- only get the start or the end of session in the result
    )
  )
WHERE NOT (start_of_session AND NOT end_of_session) -- rule out the accesses that are only start of the sessions
{% endhighlight %}


Now we have all the sessions with **user_id**, **duration** and **session_start_time**.

The main points in constructing this query are:

- use partition function to folder the table.
- rule out the intervals between start and end of the session.
- use partition function again to calculate the duration.
