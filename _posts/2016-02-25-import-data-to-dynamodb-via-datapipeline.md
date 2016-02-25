---
layout: post
title: "DynamoDB - Import data via Data Pipeline"
description: ""
modified: 2016-02-25
tags: [aws,dynamodb,aws data pipeline]
category: aws
image:
  feature: abstract-14.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

In this post I'm going to show how to import data into DynamoDB table.

Before we start, you show

- Have a AWS account with access to the services we are gonna need.
- Massive data that you badly want to import into a DynamoDB table.


Roughly, the steps to achieve our goal is preparing a file containing the data you want to insert into the table onto S3, and using [Amazon Data Pipeline](https://console.aws.amazon.com/datapipeline/) to import this S3 file to your DynamoDB Table.

Pretty simple and straight.

## Step 1: Prepare the data.

Basically, your data is the content of items you want to insert into the table.
So, for each item you wanna insert you put one line in json style to a file like this:

```
{"<key>":{"<type>":"<value>"},"<key>":{"<type>":"<value>"},"<key>":{"<type>":"<value>"}...}
{"<key>":{"<type>":"<value>"},"<key>":{"<type>":"<value>"},"<key>":{"<type>":"<value>"}...}
```

For example:

```
{"BicycleType":{"s":"Hybrid"},"Brand":{"s":"Brand-Company C"},"Price":{"n":"100000"}}
{"BicycleType":{"s":"Road"},"Brand":{"s":"Brand-Company B"},"Price":{"n":"19000"}}
```

The data you offer here must match the schema of existing DynamoDB table you want to import.

> **Note**: the data format has been changed recently. The correct data format is like the one offered above, not the one shown at [official website](http://docs.aws.amazon.com/datapipeline/latest/DeveloperGuide/dp-importexport-ddb-pipelinejson-verifydata2.html).


## Step 2: Upload this file to your S3

This is simple enough, but you better put this file into a folder under the bucket.

## Step 3: Create a data pipeline to import

Go to the [AWS Data Pipeline Console](https://console.aws.amazon.com/datapipeline/), and click *Create new pipeline* or *Get started now* if you're new to data pipeline.

Then you will be guided to the pipeline creation page.

In this page:

1. Input ***Name*** of your pipeline.
2. In ***Source***, choose **Import DynamoDB backup data from S3**.
3. Under the section of *Parameters*, in ***Input S3 folder*** select the **folder containing your data file** you just uploaded.
4. In ***Target DynamoDB table name***, input your table name.
5. For ***DynamoDB write throughput ratio*** input how much you want to consume your capacity. (Recommend 0.9).
6. In ***Region of the DynamoDB table***, input the region of your table.
7. For the section of ***Schedule***, change the setting to meet your need. If you just want to import data for once, choose **on a schedule** for ***Run***.
8. Change the ***Logging*** setting.
9. Hit the button ***Activate***, and wait for execution.

The data pipeline will create a ec2 server to execute the import task, which will cause cost on you bill, be aware of that :)

> The import will take some time, you can check the status of your import task in pipeline list.
>
> The task will fail if the format of your data file is incorrect.
