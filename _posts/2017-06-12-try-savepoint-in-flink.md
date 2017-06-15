---
layout: post
title: "Check out Flink's fancy save point functionality in your local machine"
description: "Is it easy to use and can we count on it?"
modified: 2017-06-12
tags: [flink, kafka]
category: Experiment
image:
  feature: abstract-12.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

In this post, we will use a **Flink local setup** with **savepoint configured**, consuming from a **local kafka instance**.
We also will have a very simple kafka producer to **feed sequential numbers to kafka**.

To check whether the savepointing is actually working, we will crucially stop the flink program, and restore it from the last savepoint, then check the consumed events is in consistency with the producer.


## Setup local Kafka

To setup a local Kafka is pretty easy. Just follow the [official guide](https://kafka.apache.org/08/documentation.html#quickstart), then you will have a zookeeper and Kafka instance in your local machine.

Start zookeeper:

{% highlight bash %}
bin/zookeeper-server-start.sh config/zookeeper.properties
{% endhighlight %}

Start Kafka:

{% highlight bash %}
bin/kafka-server-start.sh config/server.properties
{% endhighlight %}

Create our testing topic:

{% highlight bash %}
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic flink-test
{% endhighlight %}
