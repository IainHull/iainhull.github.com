---
layout: post
title: "A Simple REST DSL Part 2"
description: ""
category: 
tags: [scala, dsl, builder, testing, rest]
---
{% include JB/setup %}

Previously we have seen how the builder pattern can improve code readability and composibility of REST requests.  Now lets see if the builder can be used as the bases for a simple DSL.

First we need to decide what words or phrases to use bootstrap the request DSL.  Do this by describing some sample requests in simple English and looking for patterns.

* Get from url http://api.rest.org/person/
* Post personJson to url http://api.rest.org/person/
* Get from url http://api.rest.org/person/personId
* Delete from url http://api.rest.org/person/personId

All of these start with the http method so this is the ideal place to start the DSL.  We can define an implicit function to convert a `Method` object into a `RequestBuilder`.  Now when a `RequestBuilder` method is called on a `Method` object, the implicit function is called to convert it to a `RequestBuilder` first.

{% highlight scala %}
implicit def methodToRequestBuilder(method: Method): RequestBuilder = RequestBuilder().withMethod(method)
{% endhighlight %}

Now the following expression ...

{% highlight scala %}
driver.execute(RequestBuilder().withMethod(GET).withUrl("http://api.rest.org/person/"))
{% endhighlight %}

... can be expressed with a simple DSL.

{% highlight scala %}
driver.execute(GET withUrl "http://api.rest.org/person/")
{% endhighlight %}

This along with Scala's infix notation is used to improve the readability of the use case.

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

Now lets try to simplify the `driver.execute` part of the code.  Again implicits conversions enable us to automatically add an `execute` method to the `RequestBuilder` when required.  

{% highlight scala %}
implicit class RichRequestBuilder(builder: RequestBuilder) {
  def execute()(implicit driver: Driver): Response = {
    driver.execute(builder)
  }
}
{% endhighlight %}

When this class is in scope any `RequestBuilder` objects will be automatically be converted to `RichRequestBuilder` objects when the `execute` method is called.  This technique is the [Pimp My Library Pattern](https://wiki.scala-lang.org/display/SYGN/Pimp-my-library) and is heavily use in Scala to safely extend existing classes.  [Implicit classes](http://docs.scala-lang.org/overviews/core/implicit-classes.html) are a new feature in Scala 2.10, they are shorthand for a class definition and implicit function to convert the class' parameters to an instance of the class.  

Now the use case reads much more like English.  The `driver` is passed to the `execute` method implicitly removing some more boiler plate.  However since the method now takes no parameters empty parentheses are required with infix notation to enable the Scala compiler can find the end of the expression.  This also matches Scala's style of using empty parentheses to highlight side effects.

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

Wouldn't it be nice to not have to duplicate the `withUrl` on every line, maybe we can put this outside a code block and pass the url in implicitly.

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

To support this the Dsl must be changed, so that each new expression will use the closest implicit `RequestBuilder` object is scope.  This will enable partial `RequestBuilder` objects be used as the starting point for later expressions.  

First add implicit support to the `RequestBuilder` helper object.  The `emptyBuilder` is defined as implicit, now if there are no implicit `RequestBuilder` objects in scope the empty one will be used.  Next an implicit second argument list is added to the `apply` method, this now simply returns the implicit builder.  The end result is that the expression `RequestBuilder()` will always return the correct object using implicit resolution.

{% highlight scala %}
object RequestBuilder {
  implicit val emptyBuilder = RequestBuilder(None, None, Seq(), Seq(), None)

  def apply()(implicit builder: RequestBuilder): RequestBuilder = {
    builder
  }
}
{% endhighlight %}

The same technique can be used with the `methodToRequestBuilder` function.  Now instead of always using the `emptyBuilder` it adds the method the latest one using implicit resolution.

{% highlight scala %}
implicit def methodToRequestBuilder(method: Method)(implicit builder: RequestBuilder): RequestBuilder = builder.withMethod(method)
{% endhighlight %}

### TO BE CONTINUED

The `execute` method returns a `Response` object, this provides access to the `statusCode`, `body` and `headers` returned by the server.  With these we can extract the returned data and verify their value.  However it would be nicer if the DSL supported the operations to access and verify this information directly.



{% highlight scala %}
type Return[T] = Response => T

implicit class RichRequestBuilder(builder: RequestBuilder)(implicit driver: Driver) {
  def returning[T](r: Return[T]): T = {
    val resp = execute()
    r(resp)
  }
}
{% endhighlight %}
