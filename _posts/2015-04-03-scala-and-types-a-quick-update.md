---
layout: post
title: "Scala and Types a Quick Update"
description: ""
category:
tags: [scala, scaladays, wrappertypes, types]
---
{% include JB/setup %}

I haven't had any time to blog recently, work has been too busy, which is just the way I like it :-).  I have spent the past year or so thinking about types in general and Scala's type system in particular, so here is a quick update on what I have been doing.

# Wrapping simple values to add semantics and typesafety

I managed to find some time to write a [blog post on the Workday developer blog](http://workday.github.io/scala/2015/02/05/scala-typesafe-wrappers/)

> # Why Wrap?

> Primitives like Strings, Ints, and Booleans are excellent at holding values, but in most use cases they are weakly typed. Primitives do not carry any semantics or invariants. For example, a string can store an email address but a string _is not_ an email address. If you store an email address in a simple String, any invariants such as 'is it valid?', have to be assumed or checked every time the value is used. This encourages bugs at worst or bloated and overly defensive code at best.

> So why are primitives so over used?

> * They already exist and are easy to use.
> * The cost of using primitives is not paid by original developer who knows where is safe to assume a value is valid and when it must be checked.  However people who maintain or use this code are not so lucky, they have to ensure all preconditions are met before calling the code and don't know when to assume or check any postconditions it guarantees.  

> For a more complete explanation on the ills of primitive typing see this excellent blog [The abject failure of weak typing](http://techblog.realestate.com.au/the-abject-failure-of-weak-typing/).

# Scala Days 2015 - Improving Correctness with Types

I also had an opportunity to speak at [Scala Days in San Francisco](http://event.scaladays.org/scaladays-sanfran-2015#!#schedulePopupExtras-6553).

## Abstract

This talk is aimed at Scala developers with a background in object oriented programming who want to learn new ways to use types to improve the correctness of their code. It introduces the topic in a practical fashion, concentrating on the “easy wins” developers can apply to their code today.

![Slides](/assets/scala-days-improving-correctness-with-types/thumbnail.png)

[View slides online](http://www.slideshare.net/IainHull/improving-correctness-with-types)

More details and a full set of references are up on the [Workday developer blog](http://workday.github.io/2015/03/17/scala-days-improving-correctness-with-types/).

I will update this page once the video is online.