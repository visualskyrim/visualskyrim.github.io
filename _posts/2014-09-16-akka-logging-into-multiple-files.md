---
layout: post
title: "Akka - Logging into multiple files"
description: "Just about everything you'll need to style in the theme: headings, paragraphs, blockquotes, tables, code blocks, and more."
modified: 2014-09-16
tags: Akka Logging
category: Snippets
image:
  feature: abstract-4.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---


*Ref:*

[1] - [Defining 1 logback file per level](http://stackoverflow.com/a/9344385/1105455)

[2] - [logback - manual](http://logback.qos.ch/manual/filters.html#levelFilter)

[3] - [Akka - Log into files](http://qiita.com/visualskyrim/items/8aa73b1136180660234e)

***

In my [last memo](http://qiita.com/visualskyrim/items/8aa73b1136180660234e) about how to use logback in Akka to log into file system, I showed the way to config `logback.xml`.

However, recently I find it killing me when I use this log file for bug-shooting, because all the log, including *debug*, *info*, *warning*, *error* are all in one file. And what I am really looking for is *error*, which is really rare in the log file.

So, it seems reasonable to place logs of different level into different files. In my case, which would also cover most cases, I will place all the *info* log line into **info.log**, and all the *error* log and above to the **error.log**.

Since we are place log into different files, we are going to use multiple appenders.

#### To put info log one file

{% highlight xml %}
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>log/info.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- Daily rollover with compression -->
            <fileNamePattern>process-log-%d{yyyy-MM-dd}.gz</fileNamePattern>
            <!-- keep 30 days worth of history -->
            <maxHistory>90</maxHistory>
        </rollingPolicy>
        <append>true</append>
        <!-- encoders are assigned the type
             ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
        <encoder>
            <pattern>%d,%msg%n</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
{% endhighlight %}

The key point is in the ***filter*** element. This filter uses [LevelFilter](http://logback.qos.ch/manual/filters.html#levelFilter) to decide which level of log should be logged.


#### To put error and all levels above to another file

{% highlight xml %}
  <appender name="ERROR_FILE" class = "ch.qos.logback.core.rolling.RollingFileAppender">
        <file>
            log/error.log
        </file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- daily rollover with compression -->
            <fileNamePattern>error-log-%d{yyyy-MM-dd}.gz</fileNamePattern>
            <!-- keep 1 week worth of history -->
            <maxHistory>100</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%date{yyyy-MM-dd HH:mm:ss ZZZZ} %message%n</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
    </appender>
{% endhighlight %}

In this example, we use [ThresholdFilter](http://logback.qos.ch/manual/filters.html#thresholdFilter) to select all the levels above the *error* to log file.

#### Then, put two appenders together

This is the tricky part. We ganna use two ***root logger*** in the same *logback.xml*.

{% highlight xml %}
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>log/process.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- Daily rollover with compression -->
            <fileNamePattern>process-log-%d{yyyy-MM-dd}.gz</fileNamePattern>
            <!-- keep 30 days worth of history -->
            <maxHistory>90</maxHistory>
        </rollingPolicy>
        <append>true</append>
        <!-- encoders are assigned the type
             ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
        <encoder>
            <pattern>%d,%msg%n</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
    <appender name="ERROR_FILE" class = "ch.qos.logback.core.rolling.RollingFileAppender">
        <file>
            log/error.log
        </file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- daily rollover with compression -->
            <fileNamePattern>error-log-%d{yyyy-MM-dd}.gz</fileNamePattern>
            <!-- keep 1 week worth of history -->
            <maxHistory>100</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%date{yyyy-MM-dd HH:mm:ss ZZZZ} %message%n</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
    </appender>
    <root level="ERROR">
        <appender-ref ref="ERROR_FILE" />
    </root>
    <root level="INFO">
        <appender-ref ref="FILE" />
    </root>
</configuration>
{% endhighlight %}

***

#### Finally

Now, in your SomeActor.scala:

{% highlight scala %}
class SomeActor extends Actor with ActorLogging {

  implicit val ec = context.dispatcher

  def receive = {
    case x: SomeMsg =>
      log.info("will be logged into info.log")
      log.error("will be logged into error.log")
  }
}
{% endhighlight %}

Hope this will help you.
