---
layout: post
title: "A Simple REST DSL Part 5"
subtitle: "Today I want to talk about how I structure my DSL code base."
description: ""
date:    2014-05-18T12:00:00
author:     "Iain Hull"
categories: [ ]
tags: [scala, dsl, builder, testing, rest, resttest]
---

I haven't had a chance to write about my REST DSL in ages.  So today I want to talk about how I structure my DSL code base.

First all functionality is implemented in a simple API.  This is a collection of classes and methods defined in the `Api` object.  This has a lot of advantages

* The full functionality is available to users who do not want to use the DSL.
* The full functionality can be tested independently of the DSL.
* The DSL tests only have to verify that the DSL calls the API the correct way (it does not have to verify the functionality)

```scala
object Api {
  
  sealed abstract class Method(name: String)
  case object DELETE extends Method("DELETE")
  case object GET extends Method("GET")
  // ...

  case class Request(...)

  case class Response(...)

  case class RequestBuilder(...) {
    // ...
  }

  object RequestBuilder {
    // ...
  }

  type HttpClient = Request => Response
  
  object Status {
    val Continue = 100
    val SwitchingProtocols = 101
    val OK = 200
    // ....
  }

  def toHeaders(hs: (String, String)*): Map[String, List[String]] = {
    //...
  }

  def toQueryString(qs: (String, String)*): String = {
    // ...
  }
}
```

One important change from previous blogs is that I have replaced the single method trait `Driver` with the type `HttpClient` which is a function from `Request` to `Response`. First the name change better explains the role. Second there is no reason to invent a new type for this when a function carries the same amount of information.

Declaring `Api` as an object makes using the API as simple as importing `Api`'s members.

```scala
import org.iainhull.resttest.Api._
```

While using `Api` is simple extending it is not. Ideally the DSL will extend this, so that the full API is available when you using the DSL.  The simplest way to achieve this is to define the `Api`'s members in a trait that the `Api` object extends.

```scala
trait Api {
  // Define members here
}

object Api extends Api
```

However if you do this the classes `Request`, `Response` and `RequestBuiler` become [path dependent types](http://danielwestheide.com/blog/2013/02/13/the-neophytes-guide-to-scala-part-13-path-dependent-types.html).  Path dependent types are very powerful tools but they make mixing code that uses the `Api` object and trait problematic.  Therefore I chose to keep the implementation in the `Api` object, this keeps the member classes global and not path dependent.  

It would still be nice to offer an `Api` trait for extensions to use.  This is possible but it is a little more typing than the _object extending trait_ approach. The trait can declare the classes as types, the objects as vars and the methods as defs, then it can include all of `Api`'s members.

```scala
trait Api {
  type Method = Api.Method
  val GET = Api.GET
  val POST = Api.POST
  // ...

  type Request = Api.Request
  val Request = Api.Request

  type Response = Api.Response
  val Response = Api.Response

  type RequestBuilder = Api.RequestBuilder
  val RequestBuilder = Api.RequestBuilder

  type HttpClient = Api.HttpClient

  val Status = Api.Status

  def toHeaders(hs: (String, String)*): Map[String, List[String]] = Api.toHeaders(hs: _*)
  def toQueryString(qs: (String, String)*): String = Api.toQueryString(qs: _*)
}
```

Next I defined the Extractors in their own object. The `Extractor` type from previous blogs has been promoted from a simple function to a trait.  This is to add extra functionality that will make `Extrator`s easier to use.  

First the basic trait `ExtractLike` is defined.  This specifies an extractor's basic properties a `name` and a `value` method that takes a `Response` to extracts a value. It also defines the `unapply` method which enables `Extractor`s to be used in pattern matching expressions.

`Extractor` is a case class that implements `ExtractorLike` using a `name` and a `op` which is a function to extract the value from the `Response`.  It also defines `andThen` and `as` these are used to compose `Extractor`s and will be covered in another blog.

The basic `Extractor`s for `StatusCode`, `Body` and `BodyText` are also defined.  There are also classes and objects for extracting headers but these will also be explained later.

```scala
object Extractors {
  trait ExtractorLike[+A] {
    def name: String
    def unapply(res: Response): Option[A] = Try { value(res) }.toOption
    def value(implicit res: Response): A
  }

  case class Extractor[+A](name: String, op: Response => A) extends ExtractorLike[A] {
    def value(implicit res: Response): A = op(res)

    def andThen[B](nextOp: A => B): Extractor[B] = copy(name = name + ".andThen ?", op = op andThen nextOp)

    def as(newName: String) = copy(name = newName)
  }

  val StatusCode = Extractors.StatusCode

  val Body = Extractors.Body

  val BodyText = Extractors.BodyText

  val & = Extractors.&

  // ...
}
```

There is also an `Extractors` trait that includes all of these.  

The DSL is defined in a trait as opposed to an object, this is because the types defined in `Dsl` are short lived so are not affected by _path dependent type_ behavior. They only exist for the duration of a single expression and they are not reused by any other objects or traits.

```scala
trait Dsl extends Api with Extractors {
  import language.implicitConversions

  implicit def toRequest(builder: RequestBuilder): Request = builder.toRequest
  implicit def methodToRequestBuilder(method: Method)(implicit builder: RequestBuilder): RequestBuilder = builder.withMethod(method)
  implicit def methodToRichRequestBuilder(method: Method)(implicit builder: RequestBuilder): RichRequestBuilder = new RichRequestBuilder(methodToRequestBuilder(method)(builder))

  trait Assertion {
    def result(res: Response): Option[String]
  }
  
  def assertionFailed(assertionResults: Seq[String]): Throwable = {
    new AssertionError(assertionResults.mkString(","))
  }

  implicit class RichRequestBuilder(builder: RequestBuilder) {
    // ...
  }

  def using(config: RequestBuilder => RequestBuilder)(process: RequestBuilder => Unit)(implicit builder: RequestBuilder): Unit = {
    process(config(builder))
  }

  implicit class RichResponse(response: Response) {
    def returning[T1](ext1: ExtractorLike[T1])(implicit client: HttpClient): T1 = // ...
    def returning[T1, T2](ext1: ExtractorLike[T1], ext2: ExtractorLike[T2]): (T1, T2) = //...

    def returning[T1, T2, T3](ext1: ExtractorLike[T1], ext2: ExtractorLike[T2], ext3: ExtractorLike[T3]): (T1, T2, T3) = // ...

    def returning[T1, T2, T3, T4](ext1: ExtractorLike[T1], ext2: ExtractorLike[T2], ext3: ExtractorLike[T3], ext4: ExtractorLike[T4]): (T1, T2, T3, T4) = // ...
  }

  implicit def requestBuilderToRichResponse(builder: RequestBuilder)(implicit client: HttpClient): RichResponse = new RichResponse(builder.execute())
  implicit def methodToRichResponse(method: Method)(implicit builder: RequestBuilder, client: HttpClient): RichResponse = new RichResponse(builder.withMethod(method).execute())

  implicit class RichExtractor[A](ext: ExtractorLike[A]) {
    def ===[B >: A](expected: B): Assertion = makeAssertion(_ == expected, expected, "did not equal")
    def !==[B >: A](expected: B): Assertion = makeAssertion(_ != expected, expected, "did equal")

    // ...
  }
}

object Dsl extends Dsl
```

Now the `Dsl` can be used by importing the `Dsl`'s members or extending the `Dsl` trait.

```scala
class SomeRestApiSpec extends FlatSpec with Matchers {
  import Dsl._

  "Some REST API" should "support a basic use case" in {
    // ...
  }
}
```

Or

```scala
class SomeRestApiSpec extends FlatSpec with Matchers with Dsl {

  "Some REST API" should "support a basic use case" in {
    // ...
  }
}
```
