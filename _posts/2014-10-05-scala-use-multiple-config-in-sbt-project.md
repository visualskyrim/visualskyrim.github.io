---
layout: post
title: "Scala - Use Multiple Configs in Sbt Project"
description: "Better way to organize your sbt project"
modified: 2016-01-11
tags: Scala sbt
category: FrameworkSetting
image:
  feature: abstract-5.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---


Ref:

[1] - [play test 実行時に指定したconfファイルを読み込む](http://qiita.com/TomoyaIgarashi/items/4106feb940fbd2be0b4c#3-1)

[2] - [github - typesafehub/config](https://github.com/typesafehub/config)

***

Using config file absolutely make your life easier to deploy, debug, and test your applications.
Sometimes people want to use different config file to run dev or run prod, Sometimes people just want to use different config to test.

So, I'm going to show how to use multiple config files in *sbt project*.
In this post, I create minimal project by using [typesafe's activator](https://typesafe.com/activator), and I strongly command this tool to create small and clean project with least dependency to train skills.

# Normal way to use config in sbt project

Firstly, create simple sbt Scala project. I created it by using *activator*:

```
./activator new
```

And then follow the command wizard to choose `minimal-scala` as your project type, and fill up the project name.

To load config file, you will need one extra dependency: **typesafe's config**.

And following to the `build.sbt` file under the root directory of the project.

{% highlight scala %}
resolvers += "Typesafe Repo" at "http://repo.typesafe.com/typesafe/releases/"

libraryDependencies += "com.typesafe" % "config" % "1.2.1"
{% endhighlight %}

Now your `build.sbt` file should looks like:

{% highlight scala %}
name := """project-name"""

version := "1.0"

scalaVersion := "2.11.1"

resolvers += "Typesafe Repo" at "http://repo.typesafe.com/typesafe/releases/"

// Change this to another test framework if you prefer
libraryDependencies += "org.scalatest" %% "scalatest" % "2.1.6" % "test"

libraryDependencies += "com.typesafe" % "config" % "1.2.1"
{% endhighlight %}

Ok, by now, you could use a module named ***ConfigFactory*** in your project to load your config files.


It's time to add your own config file.

Add `application.config` under your `src/main/resources`, and write some random configs into it.
Here is mine:

```
test.text1="1"
test.text2="2"
test.text3="3"
```

To check whether this config file could be properly loaded, write a simple program to check it out:

{% highlight scala %}
import com.typesafe.config.ConfigFactory

object Hello {
  def main(args: Array[String]): Unit = {
    println("Hello, config!")
    val configText1 = ConfigFactory.load().getConfig("test").getString("text1")
    val configText2 = ConfigFactory.load().getConfig("test").getString("text2")
    val configText3 = ConfigFactory.load().getConfig("test").getString("text3")
    println(configText1)
    println(configText2)
    println(configText3)
  }
}
{% endhighlight %}

When run this script, it will give:

{% highlight scala %}
Hello, config!
1
2
3
{% endhighlight %}

# Use config in your test

In most cases, people use a set of different configs in their tests.
It would be great if one can just replace some of the configs in the `src/main/resources/application.conf` and keep the others.

Actually, when run `sbt test`, sbt will first look for `src/main/resources/application.config` and then use `src/test/resources/application.confg` to overwrite configs in the first one.

> Note: sbt will do the overwrite even if you not use `include "application.config"` in the `src/test/resources/application.config`.

To check that, add new test case under `src/test/scala` in your project:

{% highlight scala %}
import com.typesafe.config.ConfigFactory
import org.scalatest._

class HelloSpec extends FlatSpec with Matchers {
  "Hello" should "have tests" in {
    val configText1 = ConfigFactory.load().getConfig("test").getString("text1")
    val configText2 = ConfigFactory.load().getConfig("test").getString("text2")
    val configText3 = ConfigFactory.load().getConfig("test").getString("text3")
    println(configText1)
    println(configText2)
    println(configText3)

    true should be === true
  }
}
{% endhighlight %}

Then add config file for the test as `src/test/resources/application.conf`.

{% highlight scala %}
test.text2="22"
{% endhighlight %}

When you run `sbt test` you will get this as expected:

```
1
22
3
```

# Use specific config file

Normally, you can specify the config file by use sbt parameter `-Dconfig.file` or `-Dconfig.resource`.(for detailed information please refer to [this doc](https://github.com/typesafehub/config#standard-behavior))

But in team work, you probably want this to be static, and use the file you have specified whenever and whoever run `sbt test`.
In that case, you need to put a extra in your `build.sbt`:

{% highlight scala %}
fork in Test := true // allow to apply extra setting to Test

javaOptions in Test += "-Dconfig.resource=test.conf" // apply extra setting here
{% endhighlight %}

And then, put your test.conf to the `src/test/resources/test.conf`:

{% highlight scala %}
include "application.conf"

test.text3="333"
{% endhighlight %}

> Note that in this time you will have to use `include "application.conf"`.

And this time when you run `sbt test`, you will get:

```
1
22
333
```

That is because the sequence of config files overwrite each others is:

```
src/test/resources/test.conf
src/test/resources/application.conf
src/main/resources/application.conf
```

***Note*** that no matter you like or not, once you include the "application.conf",
the sbt will first try to find `src/test/resources/application.conf`.
If that does not exist, then it will find `src/test/resources/application.conf`.

Even if you delete the `src/test/resources/application.conf`, the `src/main/resources/application.conf` is still in the system.

Here is the result if there is no `src/test/resources/application.conf` in the project:

```
1
2
333
```
