---
layout: post
title: Play Framework - Connect to Remote Akka Actor
description: "Make your API more powerful"
modified: 2014-10-05
tags: [Ebean]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---



REF:

- [1] - [play - Integrating with Akka](https://www.playframework.com/documentation/2.3.x/JavaAkka)
- [2] - [akka - Remoting](http://doc.akka.io/docs/akka/2.3.6/scala/remoting.html)
- [3] - [google group - Actor system does not listen on public IP, just on localhost](https://groups.google.com/forum/#!topic/akka-user/J4IvabpuV6k)
- [4] - [alvin alexander - A simple Akka (actors) remote example](http://alvinalexander.com/scala/simple-akka-actors-remote-example)


***


# Add depencency

In `build.sbt` for both your Play API and akka actor program, add akka's remoting lib, because this is not included in akka's main lib.

{% highlight scala %}
libraryDependencies ++= Seq(
  // ... other libs
  "com.typesafe.akka" %% "akka-remote" % "2.3.4"
)
{% endhighlight %}
```

# (Re)Config akka

## For API

For Play API, change the default akka settings in your `conf/application.conf` like following:

{% highlight py %}
akka {
  actor {
    provider = "akka.remote.RemoteActorRefProvider" # offer the provider
  }

  remote {
    enabled-transports = ["akka.remote.netty.tcp"] # enable protocol
    netty.tcp {
      hostname = "127.0.0.1" # your host
      port = 2553 # port
    }
  }
}
{% endhighlight %}

If there is no akka object in your config file, you could just add above to the config file.

## For Actor

Almost do the same thing as you did in API in your `src/main/resources/application.conf`.


{% highlight py %}
yourSystem { # the name your actor system is going to use
  akka { 
    # other thing is just the same as that in API
    loglevel = "DEBUG"
    loggers = ["akka.event.slf4j.Slf4jLogger"]

    actor {
      provider = "akka.remote.RemoteActorRefProvider"

      default-dispatcher {
      }
    }

    remote {
      enabled-transports = ["akka.remote.netty.tcp"]
      netty.tcp {
        hostname = "127.0.0.1"
        port = 2552 # Note if you are running API and Actor program in localhost, make sure they are not using the same port
      }
    }
  }
}
{% endhighlight %}

Then apply the settings in your code:

{% highlight scala %}

// ... some codes for launch the program
val system = ActorSystem("CoolSystem",config.getConfig("yourSystem")) // you will see this below

{% endhighlight %}


> **Note** The *provider* shown above changes over versions of akka, check your version carefully and choose the right provider.


# Program

## API 

I use Play in Java for this example. For Scala, see [this](https://www.playframework.com/documentation/2.3.x/ScalaAkka).

In your **controller(action)**, connect your remote actor using:

{% highlight java %}
public static F.Promise<Result> askYourActorSomething(final String info) {
    String actorPath = actorHelper.getPath(); // get akka path of your worker, this will not show in my example
    ActorSelection actor = Akka.system().actorSelection(actorPath); // get actor ref
    return play.libs.F.Promise.wrap(ask(actor, new MessageToActor(info), 5000)).map( // use ask pattern so that we can get sync response from actor; wrap into Promise
        new F.Function<Object, Result>() { // the callback when actor sends back response
            public Result apply(Object resp) {
                return ok((String) resp);
            }
        }
    );
}
{% endhighlight %}

The main point here is that if you use **ask pattern** in your code, you have to wrap your result into Promise.


## Actor

Actor code:

{% highlight scala %}
class Worker extends Actor {
  override def receive = {
    case MessageToActor(info) => // get message from API
      sender ! "worked!" //  response to API
  }
}
{% endhighlight %}

Launch code:

{% highlight scala %}
// fetch configs
val remoteConfig = ConfigFactory.load.config.getConfig("yourSystem").getConfig("akka").getConfig("remote").getConfig("netty.tcp")
val actorHost = remoteConfig.getString("hostname")
val actorPort = remoteConfig.getInt("port")
val workerName = "worker"

val actorPath = "akka.tcp://" + "yourSystem" + "@" + actorHost + ":" + actorPort + "/user/" + workerName
println(actorPath) // here you know what your actor's path is, well, just for show, don't do this sort of thing your code.
val system = ActorSystem("CoolSystem",config.getConfig("yourSystem"))
val actor = system.actorOf((new Worker()), name = workerName)
{% endhighlight %}

Now, if the controller ***askYourActorSomething*** is called, it will send a message to your actor, which is specified by your path. Then the actor receives this message and send a String back to the API controller, which consequently cause API return "worked!".

# There is a one more thing

If you are gonna use remote actor in Play application in Production, especially in distributed environment, things are going to be a little bit tough. 

## Firewall

This will cause it impossible to API and Actor program access to each other. 

If you are using EC2, this could be solved by setting **security groups**. You must make sure the API and Actor program is in each other's group's inbound.

## Pass path to the API

It seems very easy in the first sight, you can just put the actor path in the a database table by
overriding actor's ***preStart*** method. Programatically, the API will never know the remote actor it is asking for is still working or already dead. Even if you change the record in your table when the actor is not accessible any more by overriding the ***postStop*** of Actor, this method can hardly be called in real situation.

***
