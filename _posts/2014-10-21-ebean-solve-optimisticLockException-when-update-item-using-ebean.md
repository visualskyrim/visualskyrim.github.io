---
layout: post
title: Ebean - Solve OptimisticLockException when update item using Ebean
description: "Or maybe use transaction"
modified: 2014-10-05
tags: [Ebean]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---


Ref:

1 - [Ebean and the OptimisticLockException](http://blog.matthieuguillermin.fr/2012/11/ebean-and-the-optimisticlockexception/)
2 - [http://www.avaje.org/occ.html](http://www.avaje.org/occ.html)

# The problem

Recently, I have suffered a wired problem when using *Ebean* to update item. When I update it by doing following:

{% highlight java %}
SomeEntity entity = SomeEntity.find.byId(siteId);
if (entity == null) {
    // throw exception or send signal
}
entity.setTextAttr("A big text here");
entity.save();
{% endhighlight %}

Then when I run this code, the `save()` statement give me a exception:

{% highlight java %}
javax.persistence.OptimisticLockException: Data has changed
{% endhighlight %}

The update code is pretty simple, and so does the definition of *SomeEntity*.

{% highlight java %}
create table SomeEntity (
  id                        varchar(255) not null,
  textAttr                  TEXT)
;
{% endhighlight %}

# OptimisticLock

(If you just want a quick solution, refer to the next section)

According to [avaje's document](http://www.avaje.org/occ.html), **OptimisticLock** is a mechanism to avoid lost of updates when these updates occur in the same time on the some data.

This mechanism is very simple: check if old data has been changed. If the attributes before updated have been changed by other threads, the update will be abort by throwing **OptimisticLockException**.

There are two ways for Ebean to decide whether the old data has been changed:

- check all the attributes.(default way)
- check one special attribute defined by user named as **Version Attribute**.

The first one is how problem came out. When I performed `save()`, the Ebean actually mapped it into following sql:

{% highlight sql %}
UPDATE some_entity SET text_attr = 'new value' WHERE id = 'item id' AND text_attr = 'old value';
{% endhighlight %}

This sql seems innocent, and it works well in the most time. However sometimes it can fail even if on one has changed the data. That is because sometimes database saves data as different value(to optimise the storing I guess). This could happen to ***double*** or ***text*** type of data. This will cause the where clause above fails, and make this update exceptional.


As an example mentioned by [this post](http://blog.matthieuguillermin.fr/2012/11/ebean-and-the-optimisticlockexception/):

> It can happen that you retrieve in your Java code something like 0.46712538003907 but in fact, in the database, the data stored is something like 0.467125380039068. When updating, the Java value will be included in the WHERE clause and, as it doesn’t match the value stored in the database, the update will fail.


# Solution: Version Attribute

By adding ***Version Attribute*** to the model, you can specify Ebean only use this column to decide whether old value has been changed.

{% highlight java %}
public class SomeEntity extends Model {
    @Id
    public String Id;

    @Column(columnDefinition = "MEDIUMTEXT")
    @Constraints.Required
    public String TextAttr;

    @Version
    @Column(columnDefinition = "timestamp default '2014-10-06 21:17:06'")
    public Timestamp lastUpdate; // here
}
{% endhighlight %}

You don't need to change anything in the updating code. Now if you perform an update, the sql statement will be:


{% highlight sql %}
UPDATE some_entity SET text_attr = 'new value' WHERE last_update = '2014-10-06 21:17:06';
{% endhighlight %}

And problem should be solved.

***

Note that OptimisticLock only happen in READ_COMMITTED Isolation level. So if you use transaction when update somehow, you might just walk around this problem.
