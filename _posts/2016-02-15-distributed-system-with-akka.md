---
layout: post
title: "Akka - Build distributed load balanced actor system on docker"
description: "Behold, the power of Akka."
modified: 2016-02-22
tags: [akka,scala,distributed system]
category: Akka
image:
  feature: abstract-14.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

# Objectives

1. Use **[this pattern](http://letitcrash.com/post/29044669086/balancing-workload-across-nodes-with-akka-2)** to build a load balance Akka system.
2. Make each node in this pattern within the docker container, so that we can easily deploy new node.


# Pattern overview

You can refer to this pattern in detail in [this post](http://letitcrash.com/post/29044669086/balancing-workload-across-nodes-with-akka-2).


(Or just look at [this picture in that post](http://33.media.tumblr.com/tumblr_m8h9p0T5zX1r4vwx1.png))

But in general, this pattern has following features:

- The system consists of one *master* and several *worker*s.
- The *master* receives tasks and send them to *worker*s.
- Workers work on and finish these tasks.
- The *master* will track the status of *worker**s, and only send task to *worker*s that is not currently working. (By doing so, we can achieve load balance.)
- After *worker* finishes its task, it will send message back to the *master*, telling *master* that it is not working now.

To achieve our objectives, we are going to:

- Make master into a container.
- Make a group of workers into another container.
- After we deploy above containers, we start another group of workers in another container. (To confirm the hot scale-out)


# Implementation

As the features described above, writing master and worker is pretty straight forward. You can find detailed source code about this pattern at the bottom of [that post](http://letitcrash.com/post/29044669086/balancing-workload-across-nodes-with-akka-2).

Here is the system that I simplified:

***TaskWorker***

{% highlight scala %}
package actors

import actors.TaskMaster._
import actors.TaskWorker.WorkFinished
import akka.actor.{Actor, ActorLogging, ActorPath, ActorRef}


object TaskWorker {
  case class WorkFinished(isFinished: Boolean = true)
}

abstract class TaskWorker(masterPath: ActorPath) extends Actor with ActorLogging {

  val master = context.actorSelection(masterPath)

  override def preStart() = master ! NewWorker(self)

  // TODO: add postRestart function for better robustness

  def working(task: Any, masterRef: ActorRef): Receive = {
    case WorkFinished(isFinished) =>
      log.debug("Worker finished task.")
      masterRef ! TaskResponse(self, isFinished)
      context.become(idle)
  }



  def idle: Receive = {
    case TaskIncoming =>
      master ! TaskRequest(self)
    case TaskTicket(task, taskOwner) =>
      log.debug("Worker get task.")
      context.become(working(task, sender()))
      work(taskOwner, task)
    case TaskConsumedOut =>
      log.debug("Worker did not get any task.")

  }

  def receive = idle

  def finish(): Unit = {
    self ! WorkFinished(isFinished = true)
  }

  def fail(): Unit = {
    self ! WorkFinished(isFinished = false)
  }

  def work(taskOwner: Option[ActorRef], task: Any): Unit

  // define this method if you want to deal with the result of processing
  // def submitWork(workResult: WorkerResult): Unit

}
{% endhighlight %}

***TaskMaster***

{% highlight scala %}
package actors

import actors.TaskMaster._
import akka.actor.{Actor, ActorLogging, ActorRef}

import scala.collection.mutable

// companion object for protocol
object TaskMaster {
  case class NewWorker(worker: ActorRef)
  case class TaskList(taskList: List[Any])
  case class TaskIncoming()
  case class TaskResponse(worker: ActorRef, isFinished: Boolean=true)
  case class TaskRequest(worker: ActorRef)
  case class TaskConsumedOut()
  // Add original sender of task, so that worker can know who send this.
  // In that case, worker can directly send result of the task to the original sender
  // Change the type of `task` to apply more specified task, or you can define you own task.
  case class TaskTicket(task: Any, taskOwner: Option[ActorRef])
}


class TaskMaster extends Actor with ActorLogging {

  val taskQueue = mutable.Queue.empty[Object]
  val workers = mutable.Map.empty[ActorRef, Option[TaskTicket]]


  override def receive: Receive = {

    // when new worker spawns
    case NewWorker(worker) =>
      context.watch(worker)
      workers += (worker -> None)
      notifyFreeWorker()

    // when worker send task result back
    case TaskResponse(worker, isFinished) =>
      if (isFinished)
        log.debug(s"task is finished.")
      else
        log.debug(s"task failed to finish.")
      workers += (worker -> None)
      self ! TaskRequest(worker)

    // when worker want task
    case TaskRequest(worker) =>
      if (workers.contains(worker)) {
        if (taskQueue.isEmpty)
          worker ! TaskConsumedOut()
        else {
          if (workers(worker) == None) {
            val task = taskQueue.dequeue()
            assignTask(task, worker)
          } else {
            // this will never happen
            log.error("Some worker is requiring task while processing the task.")
          }
        }
      }

    // when receive tasks
    case TaskList(taskList: List[Object]) =>
      taskQueue.enqueue(taskList: _*)
      notifyFreeWorker()

  }

  def assignTask(task: Object, worker: ActorRef) = {
    workers += (worker -> Some(TaskTicket(task, Some(self))))
    worker ! TaskTicket(task, Some(self))
  }

  def notifyFreeWorker() = {
    if (taskQueue.nonEmpty)
      workers.foreach {
        case (worker, m) if m.isEmpty => worker ! TaskIncoming()
        case _ => log.error("Something wired in the worker map!")
      }
  }
}
{% endhighlight %}

# Start the system

Our system should have two entries: one for starting the master, and the other is starting the worker to be connected to the master.

{% highlight scala %}

import java.util.UUID
import java.util.concurrent.TimeUnit

import akka.actor.{ActorPath, ActorSystem, Props}
import akka.routing.RoundRobinRouter
import com.typesafe.config.ConfigFactory
import actors._

import scala.concurrent.duration.Duration


object StartNode {

  val systemName = "load-balance"
  val systemConfigKey = "load-balance"
  val systemNodeConfigKey = "node"
  val masterNodeConfigKey = "master"
  val workerNodeConfigKey = "worker"

  val workerNodePopulationConfigKey = "population"

  val configHostKey = "host"
  val configPortKey = "port"


  val masterActorName = "master"

  // pattern used to get actor location
  val masterPathPattern = s"akka.tcp://$systemName@%s:%s/user/$masterActorName"

  def main(args: Array[String]): Unit = {

    if (args.length == 1) {
      args(0) match {
        case "worker" =>
          startAsWorkerGroup()
        case "master" =>
          startAsMaster()
        case _ =>
          println(s"Can not parse start mode: ${args(0)}")
      }
    } else {
      println(s"Please choose start mode.")
    }
  }

  def startAsWorkerGroup(): Unit = {

    println("Start actor system ...")
    val system = ActorSystem(systemName, ConfigFactory.load.getConfig(systemConfigKey))

    val workerNodeConfig = ConfigFactory.load.getConfig(systemNodeConfigKey).getConfig(workerNodeConfigKey)
    val masterNodeConfig = ConfigFactory.load.getConfig(systemNodeConfigKey).getConfig(masterNodeConfigKey)

    println("Parse worker config ...")
    // get worker config
    val workerCounter = workerNodeConfig.getInt(workerNodePopulationConfigKey)
    println("Connect to master ...")
    // connect to master

    val masterHost = masterNodeConfig.getString(configHostKey)
    val masterPort = masterNodeConfig.getInt(configPortKey)
    val masterLocation = masterPathPattern.format(masterHost, masterPort)
    println(s"to location $masterLocation")

    println("Connect to agent ...")

    val masterOpt = try {
      Option(system.actorSelection(masterLocation))
    } catch {
      case x: Throwable =>
        Option.empty
    }

    if (masterOpt.isEmpty) {
      println("Can not connect to master node!")
    } else {
      println("Worker Start!")
      val workerGroup = system.actorOf(Props(new DemoWorker(
        ActorPath.fromString(masterLocation)))
        .withRouter(RoundRobinRouter(workerCounter)))
    }
  }

  def startAsMaster(): Unit = {

    println("Master Start!")
    println("Parse master config ...")

    println("Start actor system ...")
    val system = ActorSystem(systemName, ConfigFactory.load.getConfig(systemConfigKey))

    println("Spawn master ...")
    val master = system.actorOf(Props(new TaskMaster()), name = masterActorName)

    /* Test this system with a scheduled task sending tasks to master */


    val scheduler = system.scheduler
    val task = new Runnable {
      def run() {
        // create random task
        val task = s"Task:${UUID.randomUUID().toString}"
        master ! List(task)
      }
    }

    implicit val executor = system.dispatcher
    scheduler.schedule(
      initialDelay = Duration(2, TimeUnit.SECONDS),
      interval = Duration(1, TimeUnit.SECONDS),
      runnable = task)
  }
}
{% endhighlight %}

# Configuration

There are basically two kinds of things should be set in the config file.

- The binding information of local actor system.
- The remote actor system that is going to be connected.

Since the **master** can get **worker**'s location when **worker** send message to ask for connection,
we can only set the location information of **master**.

A example of master actor:

{% highlight json %}
load-balance-system { // system name
  akka {
    // settings of logging
    loglevel = "DEBUG"
    # change log handler
    loggers = ["akka.event.slf4j.Slf4jLogger"]

    scheduler {
      tick-duration = 50ms
      ticks-per-wheel = 128
    }

    // actor type: remote
    actor {
      provider = "akka.remote.RemoteActorRefProvider"
    }

    remote {
      enabled-transports = ["akka.remote.netty.tcp"]
      log-sent-messages = on
      log-received-messages = on

      // hostname and port of current system
      netty.tcp {
        // if the config is used for worker, then change the hostname and port to worker's host and port
        hostname = 127.0.0.1
        port = 2550
      }
    }
  }
}

node {
  master {
    id = "test"
    activity-chunk-size = 300

    host = "127.0.0.1"
    port = 2550
  }

  worker {
    host = "127.0.0.1"
    port = 2552
    population = 20
  }
}
{% endhighlight %}


# Integrate with Docker

## Make native package

Add `sbt-native` package to your `project/plugins.sbt` file.

```
// SBT - Native Packager Plugin
addSbtPlugin("com.typesafe.sbt" % "sbt-native-packager" % "0.8.0-M2")
```

Add following line to your `build.sbt` file to specify the package type.

```
packageArchetype.java_server
```

Run following command to build your deploy package.

```
sbt clean stage universal:packageZipTarball
```

## Dockerfile

```
## Dockerfile for Sprocket Engine
#
# CAUTION: You must provide env file and /mnt/logs volumes to work properly.
#
# example:
#  docker run --name=sprocket-engine-master -d --net=host --env-file=/path/to/sprocket-engine.env -e AKKA_PORT=2550 -v /path/to/logs:/mnt/logs sprocket-engine:VERSION master
#  docker run --name=sprocket-engine-agent  -d --net=host --env-file=/path/to/sprocket-engine.env -e AKKA_PORT=2551 -v /path/to/logs:/mnt/logs sprocket-engine:VERSION agent
#  docker run --name=sprocket-engine-worker -d --net=host --env-file=/path/to/sprocket-engine.env -e AKKA_PORT=2552 -v /path/to/logs:/mnt/logs sprocket-engine:VERSION worker
#

FROM java:7-jre
MAINTAINER Chris Kong <chris.kong.cn@gmail.com>

RUN mkdir -p /home/kong
ENV USER_HOME /home/kong

# Copy your package to container
COPY target/universal/stage/bin $USER_HOME/bin
COPY target/universal/stage/lib $USER_HOME/lib

# Copy config file
COPY /mnt/application.conf $USER_HOME/conf/application.conf

# symlink log directory
RUN ln -sfn /mnt/logs $USER_HOME/logs
RUN ln -sfn /mnt/config $USER_HOME/conf

ENV LANG C.UTF-8
WORKDIR $USER_HOME

# container entrance
ENTRYPOINT [ "scala-akka-loadbalance-pattern-with-docker" ]

```

## Build image

{% highlight scala %}
sbt clean stage universal:packageZipTarball
docker build -f Dockerfile -t load-balance-pattern
{% endhighlight %}


## Run the container

There is only one thing to remember when start the container, that is using the host's network interface by specifying `--net=host`.

For master:

```
docker run --name=load-balance-master -d \
--net=host \
-v <your-log-folder>:/mnt/logs \
-v <your-master-config-file>:/mnt/config \
load-balance-pattern master
```

For worker:

```
docker run --name=load-balance-worker -d \
--net=host \
-v <your-log-folder>:/mnt/logs \
-v <your-worker-config-file>:/mnt/config \
load-balance-pattern worker
```

When above commands are executed, you will see the message indicating that workers has been connected to the master in the master's log file.

And that is how the use docker and Akka to build a load balanced distributed system.

> Please make sure your master and workers are in the same VPC network when deploy onto AWS's EC2 servers.

# Improvements

- Currently, deployments need different config files, which is annoying. You can use `--env-file` and `-e` to forge a config file inside the container, so that role-based config is no longer needed. Only a config template will be needed.
- It is ok that we add worker node to our master node on runtime. But when we try to disconnect worker node from master node, there will be something wrong. We can add a mechanism to detect dying worker and disconnect them.


> You can find above source code at:
> https://github.com/visualskyrim/scala-akka-loadbalance-pattern-with-docker
