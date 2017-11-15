---
layout: post
title: "Create external Hive table in JSON with partitions"
description: "Most common usage in Hive"
modified: 2017-11-10
tags: [hive]
category: Snippets
image:
  feature: abstract-13.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

> Hive provides a good way for you to evaluate your data on HDFS.
> It is the common case where you create your data and then want to use hive to evaluate it.
> In that case, creating a external table is the approach that makes sense.

In this post, we are going to discuss a more complicated usage where we need to include **more than one partition fields** into this external table. And the original data on HDFS is **in JSON**.


> For the difference between ***managed table*** and ***external table***, please refer to [this SO post](https://stackoverflow.com/questions/17038414/difference-between-hive-internal-tables-and-external-tables).

## Here is your data

In this post, we assume your data is saved on HDFS as `/user/coolguy/awesome_data/year=2017/month=11/day=01/\*.json.snappy`.

The data is well divided into daily chunks. So, we definitely want to keep `year`, `month`, `day` as the partitions in our external hive table.

## Make hive be able to read JSON

Since every line in our data is a JSON object, we need to tell hive how to comprehend it as a set of fields.

To achieve this, we are going to add an external jar.
There are two jars that I know of could do the job:

- [json-serde-X.Y.Z-jar-with-dependencies.jar](https://github.com/rcongiu/Hive-JSON-Serde)
- [hive-hcatalog-core-X.Y.Z.2.4.2.0-258.jar](org.apache.hive.hcatalog.data.JsonSerDe)

To add the jar you choose to hive, simply run `ADD JAR` in the hive console:

{% highlight sql %}
ADD JAR /home/coolguy/hive/lib/json-udf-1.3.8-jar-with-dependencies.jar
{% endhighlight %}

***Note:*** The path here is the path to your jar on the local machine. But you can still specify a path on HDFS by specifying `hdfs://` prefix.


## Create the external table

By now, all the preparation is done. The rest of the work is pretty straight forward:

- Tell hive where to look for the data.
- Tell hive which ones are the fields for partitions.
- Tell hive which library to use for JSON parsing.

So, the HQL to create the external table is something like:

{% highlight sql %}
create external table awesome_data (
-- <field-list>
)
PARTITIONED BY (
year string,
month string,
day string
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
LOCATION '/user/coolguy/awesome_data/';
{% endhighlight %}

> This HQL uses `hive-hcatalog-core-X.Y.Z.2.4.2.0-258.jar` to parse JSON. For the usage of `json-serde-X.Y.Z-jar-with-dependencies.jar`, change `ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'` to `ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'`.

There are two things you want to be careful about:

- The fields for partition shouldn't be in the `<field-list>`.
- The part of `/year=2017/month=11/day=01/` in the path shouldn't be in the `LOCATION`.

And here you go, you get yourself an external table based on the existing data on HDFS.

## However...

Then soon enough, you will find this external table doesn't seem to contain any data.
That is because we need to manually add partitions into the table.

{% highlight sql %}
ALTER TABLE awesome_data ADD PARTITION(year='2017', month='11', day='01');
{% endhighlight %}

When you finish the ingestion of `/user/coolguy/awesome_data/year=2017/month=11/day=02/`, you should also run

{% highlight sql %}
ALTER TABLE awesome_data ADD PARTITION(year='2017', month='11', day='02');
{% endhighlight %}


## Update: For ORC external table

For orc external table it is even simpler.
You don't need external library for this. Only thing you need to specify the ORC file format is `STORED AS ORC`:

{% highlight sql %}
create external table table_name (
... fields ...
)
PARTITIONED BY (
... partition fields ...
)
STORED AS ORC
LOCATION ...
TBLPROPERTIES ("orc.compress"="SNAPPY");
{% endhighlight %}
