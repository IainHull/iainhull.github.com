---
layout: post
title: "A Simple REST DSL Part 2"
description: ""
category: 
tags: [scala, dsl, builder, testing, rest, resttest]
---
{% include JB/setup %}

In part 1, I showed the builder pattern can improve code readability and composibility of REST requests.  Now, let's discover how the builder can be used as the basis for a simple DSL.

First you need to decide what words or phrases to use to bootstrap the request DSL.  Do this by describing some sample requests in simple English and looking for patterns, for example:

* Get from url http://api.rest.org/person/
* Post personJson to url http://api.rest.org/person/
* Get from url http://api.rest.org/person/personId
* Delete from url http://api.rest.org/person/personId

All of these start with the http method so this is the ideal place to start the DSL.  You can define an implicit function to convert a `Method` object into a `RequestBuilder`.  Now when a `RequestBuilder` method is called on a `Method` object, the implicit function is called to convert it to a `RequestBuilder` first.

{% highlight scala %}
implicit def methodToRequestBuilder(method: Method): RequestBuilder = RequestBuilder().withMethod(method)
{% endhighlight %}

Now you can rewrite the following expression

{% highlight scala %}
driver.execute(RequestBuilder().withMethod(GET).withUrl("http://api.rest.org/person/"))
{% endhighlight %}

with a simple DSL as follows

{% highlight scala %}
driver.execute(GET withUrl "http://api.rest.org/person/")
{% endhighlight %}

You can improve he readability of the use case using this simple DSL and Scala's infix notation, as follows.

{% highlight scala %}
val personJson = """{ "name": "Jason" }"""
val r1 = driver.execute(GET withUrl "http://api.rest.org/person/")
val r2 = driver.execute(POST withUrl "http://api.rest.org/person/" withBody personJson)
val id = r2.headers.get("X-Person-Id").get.head
val r3 = driver.execute(GET withUrl "http://api.rest.org/person/" addPath id)
val r4 = driver.execute(GET withUrl "http://api.rest.org/person/")
val r5 = driver.execute(DELETE withUrl "http://api.rest.org/person/" addPath id)
val r6 = driver.execute(GET withUrl "http://api.rest.org/person/")
{% endhighlight %}

Now, let's try to simplify the `driver.execute` part of the code.  Again implicit conversions enable you to automatically add an `execute` method to the `RequestBuilder` when required.  

{% highlight scala %}
implicit class RichRequestBuilder(builder: RequestBuilder) {
  def execute()(implicit driver: Driver): Response = {
    driver.execute(builder)
  }
}
{% endhighlight %}

When this class is in scope any `RequestBuilder` objects will be automatically be converted to `RichRequestBuilder` objects when the `execute` method is called.  This technique is the [Pimp My Library Pattern](https://wiki.scala-lang.org/display/SYGN/Pimp-my-library) and is heavily use in Scala to safely extend existing classes.  [Implicit classes](http://docs.scala-lang.org/overviews/core/implicit-classes.html) are a new feature in Scala 2.10.  They are shorthand for a class definition and an implicit function to convert the class' parameters to an instance of the class.  

Now the use case reads much more like English.  The `driver` is passed to the `execute` method implicitly removing some more boiler-plate code.  However, because the method now takes no parameters empty parentheses are required with infix notation to enable the Scala compiler to find the end of the expression.  This also matches Scala's style of using empty parentheses to highlight side effects.

{% highlight scala %}
implicit val driver = ...
val personJson = """{ "name": "Jason" }"""
val r1 = GET withUrl "http://api.rest.org/person/" execute ()
val r2 = POST withUrl "http://api.rest.org/person/" withBody personJson execute ()
val id = r2.headers.get("X-Person-Id").get.head
val r3 = GET withUrl "http://api.rest.org/person/" addPath id execute ()
val r4 = GET withUrl "http://api.rest.org/person/" execute ()
val r5 = DELETE withUrl "http://api.rest.org/person/" addPath id execute ()
val r6 = GET withUrl "http://api.rest.org/person/" execute ()
{% endhighlight %}

Wouldn't it be nice not to have to duplicate the `withUrl` on every line?  Maybe you can put this outside a code block and pass the url in implicitly:

{% highlight scala %}
val personJson = """{ "name": "Jason" }"""
RequestBuilder() withUrl "http://api.rest.org/person/" apply { implicit rb =>
  val r1 = GET execute ()
  val r2 = POST withBody personJson execute ()
  val id = r2.headers("X-Person-Id").head
  val r3 = GET addPath id execute ()
  val r4 = GET execute ()
  val r5 = DELETE addPath id execute ()
  val r6 = GET execute ()
}
{% endhighlight %}

To support this you must change the DSL, so that each new expression will use the closest implicit `RequestBuilder` object in scope.  This will enable partial `RequestBuilder` objects be used as the starting point for later expressions.  

First add implicit support to the `RequestBuilder` helper object.  The `emptyBuilder` is defined as implicit, now if there are no implicit `RequestBuilder` objects in scope, the empty one will be used.  Next, add an implicit second argument list to the `apply` method, this now simply returns the implicit builder.  The end result is that the expression `RequestBuilder()` will always return the correct object using implicit resolution.

{% highlight scala %}
object RequestBuilder {
  implicit val emptyBuilder = RequestBuilder(None, None, Seq(), Seq(), None)

  def apply()(implicit builder: RequestBuilder): RequestBuilder = {
    builder
  }
}
{% endhighlight %}

The same technique can be used with the `methodToRequestBuilder` function.  Now, instead of always using the `emptyBuilder`, it adds the method to the latest one using implicit resolution.

{% highlight scala %}
implicit def methodToRequestBuilder(method: Method)(implicit builder: RequestBuilder): RequestBuilder = builder.withMethod(method)
{% endhighlight %}


Now that you can use codeblocks to reuse common request parameters it would be nice to simplify the DSL by adding nicer aliases for some of the common `RequestBuilder` methods.

{% highlight scala %}
implicit class RichRequestBuilder(builder: RequestBuilder) {
  def url(u: String) = builder.withUrl(u)
  def body(b: String) = builder.withBody(b)
  def /(p: Symbol) = builder.addPath(p.name)
  def /(p: Any) = builder.addPath(p.toString)
  def :?(params: (Symbol, Any)*) = builder.addQuery(params map (p => (p._1.name, p._2.toString)): _*)

  // ...
}
{% endhighlight %}

This enables you to create the more descriptive version or our use case below.  The `/` method is shorthand for `addPath` and the `:?` method is shorthand for `addQuery`.  It is unfortunate that the ":" is required to preserve operator precedence rules in Scala.  Methods starting with `?` have a higher precedence to other methods, adding the ":" reduces the precedence so the expressions works as expected.

{% highlight scala %}
val personJson = """{ "name": "Jason" }"""
RequestBuilder() url "http://api.rest.org/" apply { implicit rb =>
  val r1 = GET    / 'person execute ()
  val r2 = POST   / 'person body personJson execute ()
  val id = r2.headers("X-Person-Id").head
  val r3 = GET    / 'person / id execute ()
  val r4 = GET    / 'person execute ()
  val r5 = DELETE / 'person / id execute ()
  val r6 = GET    / 'person :? ('page -> 2, 'per_page -> 100) execute ()
}
{% endhighlight %}


This shows how easy it is to create a simple DSL based the [Pimp My Library Pattern](https://wiki.scala-lang.org/display/SYGN/Pimp-my-library) and a few implicit conversions.  The intent of each expression is made more explicit by removing the boiler-plate code and simplifying the syntax so only information that affects the meaning of the expression is used.

The source code for this post is available on [github](https://github.com/IainHull/resttest/tree/e2d9b5e3640980a38aafeaa69c986d8d6da1ee6c)

Next I will examine extending the DSL to extract specific values and verifying parts of the response.

If you would like to give me any feedback regarding this post I am happy to discuss anything and everything on Twitter and email. If there is enough interest I will also consider enabling comments on my blog.