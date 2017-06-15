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

## Create a script to feed event

As mentioned above, the producer is very simple.
I'm going to use python to create this producer.

First, the Kafka dependency:

{% highlight bash %}
pip install kafka-python
{% endhighlight %}

Then create a python script `producer.py` like this:

{% highlight python %}
from kafka import KafkaProducer
from kafka.errors import KafkaError
import json
import time

producer = KafkaProducer(bootstrap_servers=['localhost:9092'], value_serializer=lambda m: json.dumps(m).encode('ascii'))
# Asynchronous by default
future = producer.send('flink-test', b'raw_bytes')

# Block for 'synchronous' sends
try:
    record_metadata = future.get(timeout=10)
except KafkaError:
    # Decide what to do if produce request failed...
    log.exception()
    pass

# Successful result returns assigned partition and offset
print (record_metadata.topic)
print (record_metadata.partition)
print (record_metadata.offset)

# Feed sequential data to kafka
i = 1
while (True):
    producer.send('flink-test', {'idx': i})
    i = i + 1
    time.sleep(1)
