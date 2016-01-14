---
layout: post
title: "Python - Check value type"
description: ""
modified: 2014-09-11
tags: Python
category: Snippets
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---


Ref:
[1] - [Stackoverflow - What's the canonical way to check for type in python?](http://stackoverflow.com/questions/152580/whats-the-canonical-way-to-check-for-type-in-python)


#### Exact type

{% highlight python %}
if type(o) is str:
  print "this is a string"
{% endhighlight %}


#### Instance of type or instance of the subclass

{% highlight python %}
if isinstance(o, str):
  print "this is a string or a instance of subclass of string"
{% endhighlight %}


There are more patterns in the reference to deal with more complex usecases.
