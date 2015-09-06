---
layout: post
title: "A Simple REST DSL Part 6"
subtitle: "A big thank you to everyone who attended my talk at ScalaDays"
description: ""
category:
tags: [scala, dsl, builder, testing, rest, resttest]
---

A big thank you to everyone who attended my talk at
[ScalaDays](http://scaladays.org). I really enjoyed all your excellent
feedback. Here is a blog I was working on before hand.

Before talking about testing I would like to cover some changes I made to
extractors and assertions. Previously extractors were simple functions, this
is an efficient implementation but the test failures result in generic
exceptions and the only clue to the cause is a stack trace. This is not an
efficient use of programmers time. The DSL makes tests easy to read and write,
so it should make failures easy to diagnose as well as.

Here is an example of the DSL using extractors to create assertions and return
values from response.  Notice how the `returning` method can be called after
`asserting`.  This works because `asserting` returns the `Response` when the
assertions succeed (and throws an exception when it fails).

```scala
val Jason: Person = ???
val personJson = Json.stringify(Jason)
val EmptyList = List[Person]()

using(_ url "http://api.rest.org/person") { implicit rb =>
  GET asserting (StatusCode === Status.OK, jsonBodyAsList[Person] === EmptyList)
  val id = POST body personJson asserting (StatusCode === Status.Created) returning (Header("X-Person-Id"))
  GET / id asserting (StatusCode === Status.OK, jsonBodyAs[Person] === Jason)
  GET asserting (StatusCode === Status.OK, jsonBodyAsList[Person] === Seq(Jason))
  DELETE / id asserting (StatusCode === Status.OK)
  GET / id asserting (StatusCode === Status.NotFound)
  GET asserting (StatusCode === Status.OK, jsonBodyAsList[Person] === EmptyList)
}
```

Previously extractors were defined as simple functions.

```scala
type Extractor[T] = Response => T

val statusCode: Extractor[Int] = _.statusCode
val body: Extractor[String] = _.body.get
```

The extractors core functionality is now described by the `ExtractorLike` trait.
Most extractors are implemented with the `Extractor` case class.

```scala
trait ExtractorLike[+A] {
  def name: String
  def unapply(res: Response): Option[A] = value(res).toOption
  def value(implicit res: Response): Try[A]
}

case class Extractor[+A](name: String, op: Response => A) extends ExtractorLike[A] {
  override def value(implicit res: Response): Try[A] = {
    Try { op(res) } recoverWith {
      case e =>
        Failure[A](new ExtractorFailedException(
            s"Cannot extract $name from Response: ${e.getMessage}",
            e))
    }
  }

  def andThen[B](nextOp: A => B): Extractor[B] = copy(name = name + ".andThen ?", op = op andThen nextOp)

  def as(newName: String) = copy(name = newName)
}
```

The `name` method enables extractors to include a detailed error message
when they fail. The `unapply` method enables extractors to be used in pattern
matching and partial functions.  The `value` method extracts the value from the
`Response`, it returns a `Try` which highlights that this method can fail
better than simply throwing an exception.

The `Extractor` case class implements the `value` method by calling the `op`
function and wrapping the result in a `Try` with a detailed exception message.
The `andThen` and `as` methods enable `Extractor`s to be composed and the
result given a new name.  These will be covered further down. The most
important addition is the `name` method, which enables the DSL to provide
meaningful failure messages when the extracting values from the `Response`
fails.

Here are some examples of how `Extractor`s are defined

```scala
val StatusCode = Extractor[Int](
  "StatusCode", r => r.statusCode)

val BodyText = Extractor[String](
  "BodyText", r => r.body.get)

val JsonBody = BodyText andThen Json.parse as "JsonBody"

def jsonBodyAs[T : Reads](path: JsPath): Extractor[T] = {
  JsonBody andThen (json => jsonToValue(json, path)) as "jsonBodyAs"
}

val BodyAsPerson = jsonBodyAs[Person](JsPath)

val BodyAsName = jsonBodyAs[String](JsPath \ "name")
```

The implicit class `RichResponse` adds the `returning` method to the
`RequestBuilder`. This takes one or more extractors and returns one or more
values (using tuples to return multiple values). The `returning` method is
overloaded four times for one to four extractors. This duplicated code could
be reduced and made more generic with the
[Shapeless library](https://github.com/milessabin/shapeless), however in this
case I didn't think the extra dependency and complication was worth the code
reuse. The last thing each method does is call `tryValue.get`, this will throw
a exception for the first extractor that fails. However it is possible for
more than one extractor to fail. In this case it would be nice to gather all
the failures and throw them in one exception. Again I have left this out but
the [Scalactic library](http://scalatest.org/release_notes/2.2.0), which is a
part of ScalaTest, contains an [Or Type](http://www.artima.com/docs-scalatest-2.2.0-RC1/index.html#org.scalactic.Or)
which is like `Try` but accumulates errors. I might consider using this in the
future.

```scala
implicit class RichResponse(response: Response) {

  // ...

  def returning[T1](ext1: ExtractorLike[T1])(implicit client: HttpClient): T1 = {
    ext1.value(response).get
  }

  def returning[T1, T2](ext1: ExtractorLike[T1], ext2: ExtractorLike[T2]): (T1, T2) = {
    val tryValue = for {
      r1 <- ext1.value(response)
      r2 <- ext2.value(response)
    } yield (r1, r2)
    tryValue.get
  }
}
```

The `returning` can also be used without without calling `asserting` or
`execute`. This works because there in an implicit conversion from
`RequestBuilder` to `RichResponse`.

```scala
implicit def requestBuilderToRichResponse(builder: RequestBuilder)(implicit client: HttpClient): RichResponse = new RichResponse(builder.execute())
```

The `asserting` method takes a list of `Assertion`s

```scala
GET / "person" asserting (
  StatusCode === Status.OK,
  BodyAsPersonList === EmptyList)
```

`Assertion`s are simply functions that take a `Response` and return an
`Option[String]`. They return `None` if the assertion passes and a failure
message wrapped in a `Some` if they fail.  `Assertion`s are normally created
from `Extractor`s using one of the operators defined in `RichExtractor`.  The
`===` and `!==` operators are defined for all types `A` and `B` that are
related. Using `B :> A` enables the value returned from an extractor to be
compared with a super class instance as well as subclass instances. The
relation operators `<`, `<=`, `>` and `>=` are defined for any types that have
an implicit `Ordering`.

```scala
type Assertion = Function1[Response, Option[String]]

implicit class RichExtractor[A](ext: ExtractorLike[A]) {
  def ===[B >: A](expected: B): Assertion = makeAssertion(_ == expected, expected, "did not equal")
  def !==[B >: A](expected: B): Assertion = makeAssertion(_ != expected, expected, "did equal")

  def <[B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = makeAssertion(ord.lt(_, expected), expected, "was not less than")
  def <=[B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = makeAssertion(ord.lteq(_, expected), expected, "was not less than or equal")
  def >[B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = makeAssertion(ord.gt(_, expected), expected, "was not greater than")
  def >=[B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = makeAssertion(ord.gteq(_, expected), expected, "was not greater than or equal")

  def in[B >: A](expectedVals: B*): Assertion = makeAssertion(expectedVals.contains(_), expectedVals.mkString("(", ", ", ")"), "was not in")
  def notIn[B >: A](expectedVals: B*): Assertion = makeAssertion(!expectedVals.contains(_), expectedVals.mkString("(", ", ", ")"), "was in")

  private def makeAssertion[B](pred: A => Boolean, expected: B, text: String): Assertion = { res =>
    val actual = ext.value(res)
    actual match {
      case Success(a) if (!pred(a)) =>
        Some(s"${ext.name}: $a $text $expected")
      case Success(_) =>
        None
      case Failure(e) =>
        Some(e.getMessage)
    }
  }
}
```

The `asserting` method first executes the request to retrieve the response.
Then evaluates a sequence of `Assertion`s with this response, if any fail
an `AssertionFailedException` is thrown containing the list of failure
messages. Remember `Asertion`s return `Some[String]` on failure.

```scala
implicit class RichRequestBuilder(builder: RequestBuilder) {

  def asserting(assertions: Assertion*)(implicit client: HttpClient): Response = {
    val res = execute()
    val assertionFailures = for {
      a <- assertions
      r <- a(res)
    } yield r
    if (assertionFailures.nonEmpty) {
      throw assertionFailed(assertionFailures)
    }
    res
  }
}
```


Extractors can also be used with partial functions.  Notice how the post method
contains two cases. Extractors can be used for testing API migration. In this
case the API can be modified to support the location header, the test continues
to pass. When the location header is supported the test uses it, otherwise
it defaults to the old behaviour. Later the old behaviour can be removed along
with the extra case.

```scala
val BodyAsPersonList = jsonBodyAsList[Person]
val BodyAsPerson = jsonBodyAs[Person]
val PersonIdHeader = Header("X-Person-Id")

using(_ url "http://api.rest.org/person") { implicit rb =>
  GET expecting {
    case StatusCode(Status.OK) & BodyAsPersonList(EmptyList) =>
  }
  val id = POST body personJson expecting {
    case StatusCode(Status.Created) & LocationHeader(loc) => extractId(loc)
    case StatusCode(Status.Created) & PersonIdHeader(id) => id
  }
  GET / id expecting {
    case StatusCode(Status.OK) & BodyAsPerson(p) => p should be(Jason)
  }
  GET expecting {
    case StatusCode(Status.OK) & BodyAsPersonList(xp) => xp should be(Seq(Jason))
  }
  DELETE / id expecting {
    case StatusCode(Status.OK) =>
  }
  GET / id expecting {
    case StatusCode(Status.NotFound) =>
  }
  GET expecting {
    case StatusCode(Status.OK) & BodyAsPersonList(EmptyList) =>
  }
}
```

Notice how extractors are anded together. This is enabled by the `&` object
whose unapply method takes a `Response` and returns a
`Some[(Response, Response)]`. This enables both extractors `unapply` methods
to accepts a `Response` each.

```scala
object & {
  def unapply(res: Response): Option[(Response, Response)] = {
    Some((res, res))
  }
}
```


The expecting function that supports this usage is very simple.


```scala
def expecting[T](func: Response => T)(implicit client: HttpClient): T = {
  val res = execute()
  func(res)
}
```

The source code for this post is available on [github](https://github.com/IainHull/resttest)

If you would like to give me any feedback regarding this post I am happy to discuss anything and everything on Twitter and email. If there is enough interest I will also consider enabling comments on my blog.
