---
layout: post
title: "Python - Quick Start of Logging"
description: "Simple samples about logging"
modified: 2014-10-16
tags: Logging Python
category: Snippets
image:
  feature: abstract-13.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

#### A very simple file logger:


{% highlight python %}
import logging

logger = logging.getLogger(__name__) # this will show current module in the log line
logger.setLevel(logging.INFO)

# create a file handler

handler = logging.FileHandler('hello.log')
handler.setLevel(logging.INFO)

# create a logging format

formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
handler.setFormatter(formatter)

# add the handlers to the logger

logger.addHandler(handler)

logger.info('Happy logging!')
{% endhighlight %}

#### Logging in a catch

{% highlight python %}
try:
    open('/path/to/does/not/exist', 'rb')
except (SystemExit, KeyboardInterrupt):
    raise
except Exception, e:
    logger.error('Failed to open file', exc_info=True)
{% endhighlight %}

Ref:

[1] - [Good logging practice in Python](http://victorlin.me/posts/2012/08/26/good-logging-practice-in-python)
