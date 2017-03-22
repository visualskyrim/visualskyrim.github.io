---
layout: post
title: "Stream Process - Simple Get-Start with Kafka and Spark"
description: "Build a simple stream processing stack with Kafka and Spark in you local env."
modified: 2017-01-23
tags: [stream process,kafka,spark]
category: Snippets
image:
  feature: abstract-14.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

***This post is for those who already understand the concept of stream processing and basic functionality of Kafka and Spark.***
***And this post is also for people who are just dying to try Kafka and Spark right away. :)***

# Purpose

- Setup a simple pipeline for stream processing in your local machine.
- Integrate Spark consumer to the Kafka.
- Implement a word frequency processing pipeline.

The versions of components in this post will be:

- Kafka: 0.10.1.0 with scala 2.11
- Spark streaming: 2.10
- Spark streaming kafka: 2.10

# Setup

***Step 1*** Install Kafka:

```
wget http://ftp.meisei-u.ac.jp/mirror/apache/dist/kafka/0.10.1.0/kafka_2.11-0.10.1.0.tgz
tar -xzf kafka_2.11-0.10.1.0.tgz
cd kafka_2.11-0.10.1.0
```

***Step 2*** Start a zookeeper for kafka

```
bin/zookeeper-server-start.sh config/zookeeper.properties
```

***Step 3*** Start kafka

```
bin/kafka-server-start.sh config/server.properties
```

***Step 4*** Create a topic *test* on kafka

```
bin/kafka-topics.sh \
--create \
--zookeeper localhost:2181 \
--replication-factor 1 \
--partitions 1 \
--topic test
```

***Step 5*** Program a spark consumer for word frequency using scala:

sbt file:

{% highlight scala %}
name := "spark-playground"

version := "1.0"

scalaVersion := "2.11.8"


libraryDependencies += "org.apache.spark" %% "spark-core" % "2.1.0"

libraryDependencies += "org.apache.spark" %% "spark-streaming" % "2.1.0"

libraryDependencies += "org.apache.spark" %% "spark-streaming-kafka-0-10" % "2.1.0"

{% endhighlight %}


main file:

{% highlight scala %}
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.spark.streaming.kafka010._
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe


object AQuickExample extends App {
  val conf = new SparkConf().setMaster("local[2]").setAppName("AQuickExample")

  val kafkaParams = Map[String, Object](
    "bootstrap.servers" -> "localhost:9092",
    "key.deserializer" -> classOf[StringDeserializer],
    "value.deserializer" -> classOf[StringDeserializer],
    "group.id" -> "spark-playground-group",
    "auto.offset.reset" -> "latest",
    "enable.auto.commit" -> (false: java.lang.Boolean)
  )
  val ssc = new StreamingContext(conf, Seconds(1))


  val inputStream = KafkaUtils.createDirectStream(ssc, PreferConsistent, Subscribe[String, String](Array("test"), kafkaParams))
  val processedStream = inputStream
    .flatMap(record => record.value.split(" "))
    .map(x => (x, 1))
    .reduceByKey((x, y) => x + y)

  processedStream.print()
  ssc.start()
  ssc.awaitTermination()

}

{% endhighlight %}


***Step 6*** Start the Spark consumer you wrote in step 5.

```
sbt run
```

***Step 7*** Start a console kafka producer and fire some message to the kafka using the topic *test*.

Start the console producer:

```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
```

Send some message to the kafka:

```
hello world
```

***Step 8*** Once you send the message, there will be something showing up in your Spark consumer terminal representing the word frequency:

```
-------------------------------------------
Time: 1485135467000 ms
-------------------------------------------
(hello,1)
(world,1)

...
```
