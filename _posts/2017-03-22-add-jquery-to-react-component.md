---
layout: post
title: "Add jQuery to your React Components"
description: "
React comes with a lot of handy features, but there are people still wanting to add more."
modified: 2017-01-26
tags: [react, jquery]
category: Snippets
image:
  feature: abstract-11.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---


To use plugins supported by jQuery, you will have to add jQuery to your component.

***Step 1***:
First you need to add the jQuery into your project.
To achieve that you could either **add the jQuery link to your html wrapper**, or simply run

```
npm i -S jquery
```

then import it in your component.

***Step 2***:
After you import jQuery into your project, you are already able to use it in your components.

But if you are using Server Side Rendering, this may not work for you, because jQuery is supposed to run on a rendered DOM document. So in SSR, you can only expect jQuery work on client side.

With that being said, the solution for usage jQuery in SSR is simple. Add your jQuery code to the

```
componentDidMount()
```

or

```
componentDidUpdate()
```
of your components.
