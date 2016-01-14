---
layout: post
title: "Ebean - Lazy load to check entity's existence"
description: "Without actually fetch the data."
modified: 2016-01-14
tags: Ebean Database
category: ServerDeveloping
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---


Ref:
[1]: [Stack Overflow - Ebean lazy load to see if resource exists](http://stackoverflow.com/questions/16936563/ebean-lazy-load-to-see-if-resource-exists)

When you delete, check count or check existence of one resource, fetch the "resource body" may cause an extra cost. This cost may potentially influence the performance.

In case of that, when you do some query that does not actually need load all data of resource, you might need concept of **lazy load**.

In ***Ebean***, you could do:

{% highlight java %}
boolean itemExists
        = (YourModel.find.where().eq("id", id).findRowCount() == 1) ? true : false;
{% endhighlight %}
