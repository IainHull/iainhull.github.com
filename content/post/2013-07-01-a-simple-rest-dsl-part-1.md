---
layout: post
title: "A Simple REST DSL Part 1"
subtitle: "Creating a DSL for testing REST based Web services."
description: ""
date:    2013-07-01T12:00:00
author:     "Iain Hull"
categories: [ ]
tags: [scala, dsl, builder, testing, rest, resttest]
---

I spend a lot of time working on REST based Web services.  Writing system tests for these can lead to a lot of boiler-plate code which is tedious to read and write and obscures the intention of the tests.  When it comes to removing boiler plate, Scala is your friend, so lets see what you can do to improve your tests.  I want to explore internal [Domain Specific Languages](http://en.wikipedia.org/wiki/Domain-specific_language) (DSLs), an area that Scala excels at, but first I want to look at the Builder pattern.

Here is an abstract interface for making simple Web service requests and receiving their responses.  This could be any API like [Dispatch](http://dispatch.databinder.net/Dispatch.html) or [Jersey](https://jersey.java.net/) but I have created my own to keep things simple and API specifics out of my examples.  We will use this to see the boiler-plate code required to use such an API for system testing REST Web services and explore how this boiler-plate can be removed.


{% highlight scala %}
sealed abstract class Method(name: String)
case object GET extends Method("GET")
case object POST extends Method("POST")
case object PUT extends Method("PUT")
case object DELETE extends Method("DELETE")

case class Request(method: Method, url: URI, headers: Map[String, List[String]], body: Option[String])

case class Response(statusCode: Int, headers: Map[String, List[String]], body: Option[String])

trait Driver {
  def execute(request: Request): Response
}
{% endhighlight %}

Lets see how to write a simple system test with this API.

{% highlight scala %}
val request = Request(GET, new URI("http://api.rest.org/person", Map(), None))
val response = driver.execute(request)
response.statusCode should be(Status.OK)
response.body match {
  Some(body) => objectMapper.readValue[List[Person]](body) should have length(0)
  None => fail("Expected a body"))
}
{% endhighlight %}

This makes a single Web service call and verifies the result, it is obvious that this pattern doesn't scale to multiple requests required for most system testing use cases.  Imagine the following simple use case:

* Retrieve the list of `Person` objects, verify it's empty
* Create a new `Person` object, verify it succeeds and returns an id
* Retrieve the `Person` by id, verify it is the same as the `Person` we create
* Retrieve the list of `Person` objects, verify the list contains the created `Person`
* Delete the `Person` by id,
* Retrieve the list of `Person` objects, verify its empty

The five lines verifying the response are reasonably straight forward.  You can easily abstract them with a `verifyPersonList` function.  However, the first line is more difficult to read and abstract.  Let's examine the request expressions for the more complicated system testing use case above:

{% highlight scala %}
val personJson = """{ "name": "Jason" }"""
val r1 = driver.execute(Request(GET, new URI("http://api.rest.org/person/"), Map(), None))
val r2 = driver.execute(Request(POST, new URI("http://api.rest.org/person/"), Map(), Some(personJson)))
val id = r2.headers("X-Person-Id").head
val r3 = driver.execute(Request(GET, new URI("http://api.rest.org/person/" + id), Map(), None))
val r4 = driver.execute(Request(GET, new URI("http://api.rest.org/person/"), Map(), None))
val r5 = driver.execute(Request(DELETE, new URI("http://api.rest.org/person/" + id), Map(), None))
val r6 = driver.execute(Request(GET, new URI("http://api.rest.org/person/"), Map(), None))
{% endhighlight %}

There is a lot of duplication here and the code's intent is lost in the boiler-plate code.  The last two parameters to `Request` are also problematic, because they are not always set and this will only get worse as more parameters are added.  Again, you could extract the `execute` method calls with a simple function.  This will remove some of the boiler-plate code but doesn't improve the code's intent.  The standard way to abstract this kind of code in Java is to use the Builder pattern.  So lets examine how this pattern can help in Scala.

{% highlight scala %}
case class RequestBuilder(
  method: Option[Method],
  url: Option[URI],
  query: Seq[(String, String)],
  headers: Seq[(String, String)],
  body: Option[String]) {

  def withMethod(method: Method): RequestBuilder = 
    copy(method = Some(method))

  def withUrl(url: URI): RequestBuilder = 
    copy(url = Some(url))

  def withBody(body: String): RequestBuilder = 
    copy(body = Some(body))

  def addPath(path: String): RequestBuilder =  {
      val s = url.get.toString
      val slash = if (s.endsWith("/")) "" else "/"
      copy(url = Some(new URI(s + slash + path)))
    }

  def addHeaders(hs: Seq[(String, String)]) = 
    copy(headers = headers ++ hs)

  def toRequest: Request = {
    Request(method.get, url.get, toHeaders(headers: _*), body)
  }
}

object RequestBuilder {
  val emptyBuilder = RequestBuilder(None, None, Seq(), Seq(), None)
  
  def apply(): RequestBuilder = {
    emptyBuilder
  }
}

implicit def toRequest(builder: RequestBuilder): Request = builder.toRequest
{% endhighlight %}

The Scala implementation of the builder differs from its Java cousin in that, it is immutable so each method returns a new instance.  It also uses the power of Scala case classes and named parameters, the `copy` method is generated automatically for case classes.  This implementation is not perfect.  Some of the `Option` fields are accessed directly with `get` calls in the `toRequest` method.  This is acceptable because this builder is designed to simplify tests so if a misconfigured builder results in a failure, it will be caught by the test.  This can be resolved by the [Type Safe Builder Pattern](http://blog.rafaelferreira.net/2008/07/type-safe-builder-pattern-in-scala.html), which I will explore at a later date.

Lets see how the builder can be used to simplify our requests.

{% highlight scala %}
val personJson = """{ "name": "Jason" }"""
val rb = RequestBuilder().withUrl("http://api.rest.org/person/")
val r1 = driver.execute(rb.withMethod(GET))
val r2 = driver.execute(rb.withMethod(POST).withBody(personJson))
val id = r2.headers("X-Person-Id").head
val r3 = driver.execute(rb.withMethod(GET).addPath(id))
val r4 = driver.execute(rb.withMethod(GET))
val r5 = driver.execute(rb.withMethod(DELETE).addPath(id))
val r6 = driver.execute(rb.withMethod(GET))
{% endhighlight %}

This is much simpler, the `driver.execute` is a little distracting, but apart from this the builder brings the intent of the code to the foreground.  There is no need to mention the header and body, when they are not required.  The first line creates the default builder `rb`, which specifies the root url that is shared by the subsequent requests.  This is the power of an immutable builder, components can be composed to build the final request. The function `toRequest` is an implicit conversion that converts `RequestBuilder` instances to `Request` instances when required, this helps remove the clutter in each request.

Whenever I consider developing a DSL, I examine a Fluent Builder first. It can greatly improve code readability with very little effort.  You can implement builders similar to this in Java, Scala or any other object oriented language.  Maybe after using the Builder, you will decide that you don't need a custom DSL.  If you do discover that a custom DSL would be beneficial the builder is a useful starting point for implement that DSL.  Next I will examine using this builder to implement a DSL and how it improves the readability and intent of the system tests.

The source code for this post is available on [github](https://github.com/IainHull/resttest/tree/1e7fe664b3369657ef5ebf190a1470b0838f2102)

If you would like to give me any feedback regarding this post I am happy to discuss anything and everything on Twitter and email.  If there is enough interest I will also consider enabling comments on my blog.
