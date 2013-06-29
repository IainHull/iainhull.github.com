---
layout: post
title: "A Simple REST DSL Part 1"
description: ""
category: 
tags: [scala, dsl, builder, testing, rest]
---
{% include JB/setup %}

I spend a lof of time working on REST based web services.  Writing system tests for these can lead to a lot of boiler plate code which is tedious to read and write and obscures the intention of the tests.  When it comes to removing boiler plate, Scala is your friend, so lets see what we can do to improve our tests.  I want to explore internal DSLs, an area that Scala excels at, but first I want to look at the Builder pattern.

Here is an abstract interface for interacting with simple REST web services, this could be any API like [Dispatch](http://dispatch.databinder.net/Dispatch.html) or [Jersey](https://jersey.java.net/).  We will use this to explore how boiler-plate code can be removed.


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

Lets see what this simple API is like to use

{% highlight scala %}
val request = Request(GET, new URI("http://api.rest.org/person", Map(), None))
val response = driver.execute(request)
response.statusCode should be(Status.OK)
response.body match {
  Some(body) => objectMapper.readValue[List[Person]](body) should have length(0)
  None => fail("Expected a body"))
}
{% endhighlight %}

This is six lines of code just to execute a simple rest request and verify the result.  It is obvious that this code doesn't scale to multiple requests per use case.  Imagine the following simple use case:

* Retrieve the list of `Person` objects, verify its empty
* Create a new `Person` object, verify it succeeds and returns an id
* Retrieve the `Person` by id, verify it is the same as the `Person` we create
* Retrieve the list of `Person` objects, verify the list contains the created `Person`
* Delete the `Person` by id,
* Retrieve the list of `Person` objects, verify its empty

The five lines verifying the response are reasonably straight forward and can easily be abstracted with a `verifyPersonList` function.  However the first line is more difficult to read and abstract.  Lets examine the request expressions for this use case.

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

There is a lot of duplication here and the code's intent is lost in the boiler-plate.  The last two parameters to `Request` are also mysterious since they are not always set and this will only get worse as more parameters are added.  Again the execute calls could be abstracted with a simple function this will remove some of the boiler-plate but doesn't improve the code's intent.  The standard way to abstract this kind of code in Java is to use the Builder pattern.  So lets look at how this can help in Scala.

{% highlight scala %}
case class RequestBuilder(
  method: Option[Method],
  url: Option[URI],
  query: Seq[(String, String)],
  headers: Seq[(String, String)],
  body: Option[String]) {

  def withMethod(method: Method): RequestBuilder = 
    copy(method = Some(method))

  def withUri(url: URI): RequestBuilder = 
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

The Scala implementation of the builder differs from its Java cousin, it is immutable so each method returns a new instance.  It also uses the power of Scala `case class`es and named parameters, the `copy` method is generated automatically for `case class`es.  This implementation is not perfect, some of the `Option` fields are accessed directly in with calls to `get` in `toRequest`.  This is Ok in this case because this builder is designed to simplify tests so a misconfigured builder results is a failure, this will be caught by the test.  This can be resolved by the [Type Safe Builder Pattern](http://blog.rafaelferreira.net/2008/07/type-safe-builder-pattern-in-scala.html), which I will explore at a later date.

Lets see how this can be used to simplify our requests.

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

This is much simpler, the `driver.execute` is a little distracting, but a part from this the builder brings the intent of the code to the foreground.  There is no need to mention the header and body, when they are not required.  The first line creates the default builder `rb`, which specifies the root url that is shared by the subsequent requests.  This is the power of an immutable builder, components can be composed to the build the final request. The function `toRequest` is an implicit conversion that converts `RequestBuilder` instances to `Request` instances when required, this helps remove the clutter in each request.

When ever I consider developing a DSL I examine a Fluent Builder first. It can greatly improve code readability for very little effort.  This builder can be implemented in Java, Scala or any other object oriented language.  Maybe after using the Builder, I will decide that I don't need a DSL.  If I do discover that a DSL would be beneficial the Builder is a useful building block to implement the DSL.

The source code for this post is available on [github](https://github.com/IainHull/resttest/tree/1e7fe664b3369657ef5ebf190a1470b0838f2102)

Next I will look at what happens when this Builder is wrapped with a DSL.
