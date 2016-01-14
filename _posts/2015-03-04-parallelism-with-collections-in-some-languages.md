---
layout: post
title: "Parallelism with Collections in Popular Languages"
description: "Better way to organize your sbt project"
modified: 2014-09-11
tags: C# Scala Python Parallelism
category: Snippets
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

# Motivation

No matter what programming language you're using, sometimes you get itch to map a whole collection to another. Usually, you do things like:

{% highlight scala %}
val list = (1 to 6) toList
list.map(_ * 2)
{% endhighlight %}

or in *C#*:

{% highlight C# %}
List<int> list = new List<int>{ 1, 2, 3, 4, 5, 6 };
List<int> mappedList = list.Select(x => x * 2);
{% endhighlight %}

or like in *Python*:

{% highlight python %}
l = [1, 2, 3, 4, 5, 6]
mapped_list = map(lambda x: x * 2, l)
{% endhighlight %}

The main problem with above codes is that when the size of the collection grows, or the mapping itself becomes more complex, this operation will be very time-take. And that is because those codes are running in **one thread**.

Since all the items in the collection are mapped in the same way, why don't we just put them onto threads and let them to be done seperatly?

# Go Parallel

Parallel operations on collections is very easy in popular languages. The operation like this in most language will allow system to distribute thread resource according to the current status of CPU, and how much the task is, which could be extremely overwhelming if we write it our own.

***usecase***: To make it more practical, let's say we want to download all the images based on a list, which contains 10000 urls of those images.

**C#**

To download these images, it could be a little complex in C#. C# has a `foreach` key word used to make list traversal like this:

{% highlight C# %}


foreach (String item in strList)
{
    // ...    
}

{% endhighlight %}

to do that in parallel way, we use the parallel version of `foreach`, like this:

{% highlight C# %}

using System;
using System.Net;
using System.Drawing; // requires system.Drawing.dll 
using System.IO;
using System.Threading;
using System.Threading.Tasks;
using System.Collections.Concurrent;

class SimpleForEach
{
    static void Main()
    {
        // A simple source for demonstration purposes. Modify this path as necessary. 
        List<string> strList = LoadStrings();
        ConcurrentBag<string> destList = new ConcurrentBag<string>();
        //  Method signature: Parallel.ForEach(IEnumerable<TSource> source, Action<TSource> body)
        Parallel.ForEach(strList, str =>
        {
            // do some modification
            string changedString = ChangeString(str);
            destList.add(changedString);
        });
    }

    static List<String> LoadStrings()
    {
        // get a batch of strings here

    }

    static string ChangeString(string org)
    {
        // change the string as dest
        return dest;
    }


}
{% endhighlight %}

> One thing you should be careful about is that in C# you can not map collections directly. You will have to put all the items in a parallel version of list, which is `ConcurrentBag` here, and it is thread safe. You can refer to another way by using [ParallelLoopResult](http://msdn.microsoft.com/en-us/library/system.threading.tasks.parallelloopresult%28v=vs.110%29.aspx).



**Python**

Things could be even more easier in Python:


{% highlight python %}
from multiprocessing import Queue
from multiprocessing import Pool

def downloadImage(url):
  # download image with the url
  return img

strs = [...]

pool = Pool(20) # define how much parallelism you want
results = pool.map(mapString, strs)
pool.close()
pool.join()
{% endhighlight %}


> Please make sure you have imported the right library of pool.

**scala**

In scala, it is super easy to do the same thing, and you literally only one line to do that:

{% highlight scala %}
strList.par.map(downloadUrl(_))
{% endhighlight %}

And done!

Of course you can define your own function to modify the string, but be sure this function is thread safe.


# References

- [1] - [Parallelism in one line
](https://medium.com/@thechriskiehl/parallelism-in-one-line-40e9b2b36148)

- [2] - [MSDN - How to: Write a Simple Parallel.ForEach Loop
](http://msdn.microsoft.com/en-us/library/dd460720(v=vs.110).aspx?cs-save-lang=1&cs-lang=csharp#code-snippet-1)
- [3] - [PARALLEL COLLECTIONS](http://docs.scala-lang.org/overviews/parallel-collections/overview.html)
