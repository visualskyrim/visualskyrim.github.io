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

# block until all async messages are sent
producer.flush()
{% endhighlight %}

Then run this script to start feeding events:

{% highlight bash %}
python producer.py
{% endhighlight %}

I recommend you also use a simple script to create a consumer to check whether the current setup is working. I highly recommend using logstash to run a simple consumer with config file like:

```
input {
    kafka {
        group_id => "flink-test"
        topics => ["flink-test"]
        bootstrap_servers => "localhost:9092"
    }
}
output {
  stdout {}
}
```

And start to consume by

{% highlight bash %}
./bin/logstash -f <your-config-file>
{% endhighlight %}

But any other kind of consumer will easily do the job.

## Setup Flink

First you will need to [download the flink](https://flink.apache.org/downloads.html) of the version you want/need.
After download the package, unpack it. Then you will have everything you need to run flink on your machine.

> Assume that `Java` and `mvn` are already installed.

### Setup local Flink cluster

This will be the tricky part.

First we need to change the config file: `./conf/flink-conf.yaml`.

In this file, you can specify basically all the configuration for the flink cluster. Like job manager heap size, task manager heap size.
But in this post, we are going to focus on save point configurations.

Add following line to this file:

{% highlight %}
state.savepoints.dir: file:///home/<username>/<where-ever-you-want>
{% endhighlight %}

With this field specified, the flink will use this folder as the storage of save point files.
In real world, I believe people usually use HDFS with `hdfs:///...` or S3 to store their save points.

In this experiment, we will use a local setup for flink cluster by running `./bin/start-local.sh`.
This script will read the configuration from `./conf/flink-conf.yaml`, which we just modified.
Then the script will start a job manager along with a task manager on localhost.

You could also change the number of slots in that task manager by modify the `./conf/flink-conf.yaml`,
but that's not something we needed for the current topic. For now we can just run:

{% highlight bash %}
./bin/start-local.sh
{% endhighlight %}
