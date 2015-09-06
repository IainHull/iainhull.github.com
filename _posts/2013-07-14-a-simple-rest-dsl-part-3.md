---
layout: post
title: "A Simple REST DSL Part 3"
subtitle: "Processing http responses."
description: ""
category: 
tags: [scala, dsl, builder, testing, rest, resttest]
---

In part 2, I showed how to describe and execute REST requests with a simple descriptive syntax, but it completely ignored processing the response.  This should be as simple as making the request.  First let's simplify access to the components of the response.  

You can create a `returning` method to execute the `Request` and extract one or more values from the `Response`.  For example:

{% highlight scala %}
val personJson = """{ "name": "Jason" }"""
RequestBuilder() url "http://api.rest.org/" apply { implicit rb =>
  val c1 = GET / 'person returning statusCode
  val (c2, id) = POST / 'person body personJson returning (statusCode, header("X-Person-Id"))
  val (c3, b3) = GET / 'person / id returning (statusCode, body)
  val (c4, b4) = GET / 'person returning (statusCode, body)
  val c5 = DELETE / 'person / id returning statusCode
  val (c6, b6) = GET / 'person returning (statusCode, body)
}
{% endhighlight %}

To implement this, define the generic type `Extractor`, this is any function that takes a `Response` and returns a value of the generic type `T`.  Now you can add a `returning` method to `RichRequestBuilder`.  This takes an `Extractor` that maps the `Response` into the desired value.  It also takes an implicit `driver` which is required to call the `execute` method.  This method executes the `Request` and passes the `Response` to  the `Extractor`.  You can also add overloaded `returning` methods taking multiple `Extractor`s and returning their results in tuples.  This enable multiple values to be returned from a single response.  There are versions of `returning` that take up to four parameters and return a `Tuple4`, more could be added but they should not be required.

{% highlight scala %}
type Extractor[T] = Response => T

implicit class RichRequestBuilder(builder: RequestBuilder) {

  // ...

  def returning[T1](func1: Extractor[T1])(implicit driver: Driver): T1 = {
    val res = execute()
    func1(res)
  }

  def returning[T1, T2](func1: Extractor[T1], func2: Extractor[T2])(implicit driver: Driver): (T1, T2) = {
    val res = execute()
    (func1(res), func2(res))
  }
}
{% endhighlight %}

Here are some of the `Extractor`s used above, `statusCode` and `body` are function objects, `header` is a method that returns a function object for the specified header name.

{% highlight scala %}
val statusCode: Extractor[Int] = _.statusCode

val body: Extractor[String] = _.body.get

def header(name: String): Extractor[String] = _.headers(name).mkString(",")
{% endhighlight %}

These `Extractor`s can be reuse to implement more complicated `Extractor`s.  For example you can create a simple json `Extractor` using the [Play json library](http://www.playframework.com/documentation/2.1.1/ScalaJson).

{% highlight scala %}
def jsonToValue[T: Reads](json: JsValue, path: JsPath = JsPath): T = {
  path.asSingleJson(json).as[T]
}

val jsonBody: Extractor[JsValue] = body andThen Json.parse

def jsonBodyAs[T: Reads](path: JsPath = JsPath): Extractor[T] = jsonBody andThen (jsonToValue(_, path))

def jsonBodyAs[T: Reads]: Extractor[T] = jsonBodyAs(JsPath)
{% endhighlight %}

Most of the time the only reason you require a part of the `Response` is to assert on its value, for example using [ScalaTest](http://www.scalatest.org/):

{% highlight scala %}
val personJson = """{ "name": "Jason" }"""
RequestBuilder() withUrl "http://api.rest.org/person/" apply { implicit rb =>
  val (c1, b1) = GET returning (statusCode, body)
  c1 should be (Status.OK)
  jsonAsList[Person](b1) should be ('empty)
}
{% endhighlight %}

The DSL should support this requirement natively.  You can add an `asserting` method, this will take a sequence of assertion objects, and will verify each one in turn. The `asserting` method uses Scala's [varargs](http://daily-scala.blogspot.ie/2009/11/varargs.html) syntax to support multiple arguments.  The `returning` methods could not use this as varargs must all be of the same type.  However `asserting` just takes a sequence of `Assertion`s and returns a `Response` so generics are not required.

{% highlight scala %}
trait Assertion {
  def verify(res: Response): Unit
}
  
implicit class RichRequestBuilder(builder: RequestBuilder) {

  // ...

  def asserting(assertions: Assertion *)(implicit driver: Driver): Response = {
    val res = execute()
    assertions foreach (_.verify(res))
    res
  }
}
{% endhighlight %}

The assertions should be easy to write and should support reusing the logic in the `Extractor`s.  For example reusing the `statusCode` `Extractor`:

{% highlight scala %}
val personJson = """{ "name": "Jason" }"""
RequestBuilder() withUrl "http://api.rest.org/person/" apply { implicit rb =>
  val (c1, b1) = GET asserting (statusCode is Status.OK, jsonBodyAsList is EmptyList)
}
{% endhighlight %}

To achieve this you can creating a `RichExtractor` class that implements the `is` method and returns an `Assertion` object which, extracts the value on the left-hand-side and compares it to the value on the right-hand-side.

{% highlight scala %}
implicit class RichExtractor[T](func: Extractor[T]) {
  def is(expected: T): Assertion = new Assertion {
    override def verify(res: Response): Unit = {
      val actual = func(res)
      if (actual != expected) throw new AssertionError(actual + " != " + expected)
    }
  }
}
{% endhighlight %}

Now the DSL can be used to express the complete testing use case.

{% highlight scala %}
val Jason: Person = ???
val personJson = Json.stringify(Jason)
val EmptyList = List[Person]()

RequestBuilder() url "http://api.rest.org/person" apply { implicit rb =>
  GET asserting (statusCode is Status.OK, jsonBodyAsList[Person] is EmptyList)
  val id = POST body personJson asserting (statusCode is Status.Created) returning (header("X-Person-Id"))
  GET / id asserting (statusCode is Status.OK, jsonBodyAs[Person] is Jason)
  GET asserting (statusCode is Status.OK, jsonBodyAsList[Person] is Seq(Jason))
  DELETE / id asserting (statusCode is Status.OK)
  GET / id asserting (statusCode is Status.NotFound)
  GET asserting (statusCode is Status.OK, jsonBodyAsList[Person] is EmptyList)
}
{% endhighlight %}


This shows how a few simple techniques can be applied to create a reasonably complete DSL for testing REST Web services.  The amount of code to implement this DSL is quite small, so the cost-reward ratio is very high.  There is one problem, how does a new user of the DSL discover the available syntax?  The DSL makes comprehending existing tests quite easy.  However modifying them or writing new tests is not so straight forward.  While the DSL code is small the interface and ScalaDoc alone are not sufficient to understanding how to apply it (that is, unless you know how the DSL in implemented). I will cover how to documenting the DSL in the next blog post.

The source code for this post is available on [github](https://github.com/IainHull/resttest/tree/22eca9eb7eac05956e589277cd9b2d727d9cf861)

If you would like to give me any feedback regarding this post I am happy to discuss anything and everything on Twitter and email. If there is enough interest I will also consider enabling comments on my blog.
