---
layout: post
title: "PlayFramework(Java) - Use Filter to Record Process Time in API"
description: "Hack the scala filter and use it in Java"
modified: 2015-02-08
tags: Java Scala PlayFramework
category: PlayFramework
image:
  feature: abstract-4.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---


[Filter](https://www.playframework.com/documentation/2.4.x/ScalaHttpFilters) is a very
useful concept in scala version of Play Framework.
It can allow you to do some general process to the requests before and after they are passed to controller,
which also allows you to get **access to the context of requests before and after processed** by controller.

However, the Filter for Java version is not, well, so useful.

> You can compare Filter of these two kinds in the [Doc](https://www.playframework.com/documentation/2.4.x/Home).

Luckily, PlayFramework for Java is actually a wrapper on the scala's PlayFramework.
So there is still a way to use scala version's `Filter` in Java's PlayFramework.

I will show how to achieve this by trying use `Filter` to record the process time of requests.


To use scala's filter in Java's PlayFramework, we need first to wrap it:

{% highlight java %}
package filters.JavaFilter;

import play.api.mvc.*;

public abstract class JavaFilter implements Filter {
    public void $init$() {
        Filter$class.$init$(this);
    }

    @Override
    public EssentialAction apply(EssentialAction next) {
        return Filter$class.apply(this, next);
    }
}
{% endhighlight %}


Note that interacting with scala library in Java is not really clean.


Then we could use this wrapped filter in our code:

{% highlight java %}
package filters.JavaFilter;

import play.api.Logger;
import play.api.libs.concurrent.Execution$;
import play.api.mvc.RequestHeader;
import play.api.mvc.ResponseHeader;
import play.api.mvc.Result;
import scala.Function1;
import scala.Option;
import scala.concurrent.Future;
import scala.runtime.AbstractFunction0;
import scala.runtime.AbstractFunction1;

public class RecordProcessTimeFilter extends JavaFilter {

    private final Logger logger = Logger.apply("process_time");

    @Override
    public Future<Result> apply(
                                  Function1<RequestHeader, Future<Result>> next,
                                  RequestHeader rh) {

        // you can access the context before the request is processed
        final long start = System.currentTimeMillis();

        return next.apply(rh).map(new AbstractFunction1<Result, Result>() {
            @Override
            public Result apply(Result result) {
                // you can access the context after request is processed here
                final long end = System.currentTimeMillis();
                final int status = result.header().status();

                // record in the log
                logger.info(new AbstractFunction0<String>() {
                    @Override
                    public String apply() {
                        // you can only access the start here
                        return "%d - %d".format(status, end - start)
                    }
                });

                // pass the result to the next filter
                return new Result(
                    new ResponseHeader(status, result.header().headers()),
                    result.body(),
                    result.connection());
            }
        }, Execution$.MODULE$.defaultContext());
    }
}

{% endhighlight %}


As you can see, you can access to the context before request is processed in `RecordProcessTimeFilter.apply()`.
You can get the time before request is processed.

And you can access to the context after request is processed within the block under `return next.apply(rh).map(new AbstractFunction1<Result, Result>(){}`.
Note you can only access to the variable `start` you stored via an `AbstractFunction0` in the context when request is processed.

There are many other use cases for `Filter`:

- Modify the http headers in the response after request is processed for all request to solve Cors problems.
- Check and modify `Content-Type` in the request before the request is processed.
