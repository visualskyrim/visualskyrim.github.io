---
layout: post
title: "Check out Flink's fancy save point in your local machine"
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

{% highlight yaml %}
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

### Prepare the job

Consider flink cluster is just distributed worker system with enpowered streaming ability.
It is just a bunch of starving workers (and their supervisors) waiting for the job assignment.

Although we have set up the cluster, we still need to give it a job.

Here is a simple Flink job we will use to check out the save point functionality:

It will consume the events from our kafka, and write it to local files divided by minute.

{% highlight java %}
import org.apache.flink.api.common.restartstrategy.RestartStrategies;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.CheckpointingMode;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.CheckpointConfig;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.fs.bucketing.BucketingSink;
import org.apache.flink.streaming.connectors.fs.bucketing.DateTimeBucketer;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer08;
import org.apache.flink.streaming.util.serialization.SimpleStringSchema;

import java.util.Properties;


public class DataTransfer {


    public static void main(String[] args) throws Exception {
      // Properties for Kafka
      Properties kafkaProps = new Properties();
      kafkaProps.setProperty("topic", "flink-test");
      kafkaProps.setProperty("bootstrap.servers", "localhost:9092");
      kafkaProps.setProperty("zookeeper.connect", "localhost:2181");
      kafkaProps.setProperty("group.id", "flink-test");

      // Flink environment setup
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.getConfig().disableSysoutLogging();
      env.getConfig().setRestartStrategy(RestartStrategies.fixedDelayRestart(4, 10000));

      // Flink check/save point setting
      env.enableCheckpointing(30000);
      env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
      env.getCheckpointConfig().setMinPauseBetweenCheckpoints(10000);
      env.getCheckpointConfig().setCheckpointTimeout(10000);
      env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

      env.getCheckpointConfig().enableExternalizedCheckpoints(
              CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION
      );

      // Init the stream
      DataStream<String> stream = env
              .addSource(new FlinkKafkaConsumer08<String>(
                      "flink-test",
                      new SimpleStringSchema(),
                      kafkaProps));

      // Path of the output
      String basePath = "<some-place-in-your-machine>"; // Here is you output path

      BucketingSink<String> hdfsSink = new BucketingSink<>(basePath);
      hdfsSink.setBucketer(new DateTimeBucketer<>("yyyy-MM-dd--HH-mm"));
      stream.print();
      stream.addSink(hdfsSink);

      env.execute();
    }
}
{% endhighlight %}

## Start experiment

To submit our job, first build the above project by:

{% highlight bash %}
mvn clean install -Pbuild-jar
{% endhighlight %}

There will be a fat jar under your target folder: `<project-name>.<version>.jar`

To submit this jar as our job, run:

{% highlight bash %}
./bin/flink run <project-name>.<version>.jar
{% endhighlight %}

Once after it starts running, you will find files start to be generated in the `basePath`.

#### Restart the job without save point

After the job runs for a while, cancel the job in the flink UI.

Check **the lastest finished file before cancellation**, and find **the last line** of this file. In my experiment, it's `{"idx": 2056}`.

Then start the job again:

{% highlight bash %}
./bin/flink run <project-name>.<version>.jar
{% endhighlight %}

After a few minute, check **first finished file after restart**, and find **the first line** of this file. In my experiment, it's `{"idx": 2215}`.

This means, there are events missing when we restart job without savepoin.

> Finished file is the file that have been checked by flink's check point.
> It is file that without prefix `in-progress` or suffix `pending`
> Initially, a file under writing is in `in-progress` state.
> When this file stop being written for a while(can be specified in config), this file become `pending`.
> A checkpoint will turn `pending` files to the finished files.
> Only finished file should be considered as the consistant output of flink. Otherwise you will get duplication.
> For more info about checkpointing, please check their [official document](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/stream/checkpointing.html).

#### Restart with save point

Let's try save pointing.
This time, we will create a save point before cancel the job.

Flink allows you to make save point by executing:

{% highlight bash %}
bin/flink savepoint <job-id>
{% endhighlight %}

The `<job-id>` can be found at the header of the job page in flink web UI.

After you run this command, flink will tell you the path to your save point file. ***Do record this path***.

Then, we cancel the job, and check **the lastest finished file before cancellation**, and find **the last line** of this file. In my experiment, it's `{"idx": 4993}`.

Then we restart the job.

Because we want to restart from a save point, we need to specify the save point when we start the job:

{% highlight bash %}
./bin/flink run <project-name>.<version>.jar -s <your-save-point-path>
{% endhighlight %}

After a few minute, check **first finished file after restart**, and find **the first line** of this file. In my experiment, it's `{"idx": 4994}`, which is consistant with the number before cancellation.

## General thoughts

Flink's save pointing is much easier than what I expect.
The only thing I should constantly keep in mind is that we need record the save point file path carefully when we use crontab to create save points.
Other than that, flink seems to handle all the data consistancy.

As part of the further experiment and research, I think it could be very useful if we try flink's save point in the cluster setup with multiple task managers.
