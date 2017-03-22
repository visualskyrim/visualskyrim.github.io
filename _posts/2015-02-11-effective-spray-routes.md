---
layout: post
title: Effective Spray - Best way to arrange your routes
description: "Best practice for routes"
modified: 2015-02-11
tags: [python,locust,load test]
category: Snippets
image:
  feature: abstract-10.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---
[Spray](http://spray.io/) is an elegant Framework in many ways. Here is my opinion on how to arrange your controllers(actions) in your projects, so that you can easily maintain them and test them.

***Project structure***

![UserService_scala_-__badboy-api__-_badboy-api_-____Documents_projects_badboy-api_.png](https://qiita-image-store.s3.amazonaws.com/0/39824/109076a7-1c37-5c0c-0ee9-c40d034c86ef.png)

<figure>
  <figcaption><a href="https://qiita-image-store.s3.amazonaws.com/0/39824/109076a7-1c37-5c0c-0ee9-c40d034c86ef.png" title="My Spray project structure">My Spray project structure</a>.</figcaption>
</figure>



***Controllers***

You will find that I didn't use `class` to extend `Actor`. I only use `trait` in these files.

{% highlight scala %}
package com.yours.services

import spray.routing._
import spray.http._

// this trait defines our service behavior independently from the service actor
trait PreferenceService extends HttpService {
  val preferenceRoute =
    path("prefer") {
      get {
        complete("ok")
      }
    }
}
{% endhighlight %}

{% highlight scala %}
package com.yours.services

import spray.routing._
import spray.http._

// this trait defines our service behavior independently from the service actor
trait PreferenceService extends HttpService {
  val preferenceRoute =
    path("prefer") {
      get {
        complete("ok")
      }
    }
}
{% endhighlight %}

***Routes***

Here is your **route actor**, which will combine all the controller traits.

{% highlight scala %}
package com.yours

import akka.actor.{ActorRefFactory, Actor}
import com.yours.services.{UserService, FeedService, PreferenceService}

/**
 * Created by visualskyrim on 12/25/14.
 */
class RoutesActor extends Actor with Routes {
  override val actorRefFactory: ActorRefFactory = context
  //def receive = runRoute(boyRoute ~ feedRoute ~ preferenceRoute)
  def receive = runRoute(routes)
}

trait Routes extends UserService with FeedService with PreferenceService {
  val routes = {
    userRoute ~
    feedRoute ~
    preferenceRoute
  }
}
{% endhighlight %}

***Boot***

This part is like `Global` in Play framework, used to start your application. We launch our route actor here.

{% highlight scala %}
import akka.actor.{ActorSystem, Props}
import akka.io.IO
import com.yours.utils.db.DbSupport
import com.yours.RoutesActor
import spray.can.Http
import akka.pattern.ask
import akka.util.Timeout
import scala.concurrent.duration._

import com.yours.utils.DynamodbSupport

object Boot extends App with DbSupport with DynamodbSupport {
  implicit val system = ActorSystem("your-api")

  val service = system.actorOf(Props[RoutesActor], "your-service")

  implicit val timeout = Timeout(5.seconds)
  IO(Http) ? Http.Bind(service, interface = "localhost", port = 8080)
  // initialize the helpers
  // initialize the Db
  initTables()
  initDynamoDbTables()
}
{% endhighlight %}

# Benefits

I believe this arrangement fits most cases.

- Most people would like request handler to be separated in several files, grouping the controllers according to the related features. If people want to change anything or add anything, they can easily locate the place.
- It's clean. There will be no association with the Akka system in the middle of your real logic.
- You might have noticed that since we use trait to build controllers, we can test our controllers without getting hands dirty with Akka.
