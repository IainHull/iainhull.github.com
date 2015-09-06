---
layout: post
title: "A Simple REST DSL Part 7"
description: ""
category:
tags: [scala, dsl, builder, testing, rest, resttest]
---
{% include JB/setup %}


```scala
trait TestDriver {
  import Api._

  implicit def httpClient: HttpClient

  implicit def defBuilder: RequestBuilder
}
```

```scala
trait SprayUnitTestDriver extends TestDriver with ScalatestRouteTest {
  this: Suite =>

  import Api._

  override implicit val httpClient: HttpClient = { req => ??? }

  override implicit def defBuilder: Api.RequestBuilder = Api.RequestBuilder.emptyBuilder withUrl ("")

  def actorRefFactory = system
  def myRoute: Route
}
```

```scala
trait JerseySystemTestDriver extends TestDriver {
  override implicit val httpClient: HttpClient = { req => ??? }

  override implicit def defBuilder = RequestBuilder.emptyBuilder withUrl baseUrl

  def baseUrl: String
}
```

```scala
class DslSpec extends FlatSpec with Matchers {

  "Sample use-case" should "support 'using' function to abstract common parameters in a readable way" in {

    using(_ url "http://api.rest.org/person") { implicit rb =>
      GET asserting (StatusCode === Status.OK, jsonBodyAsList[Person] === EmptyList)
      val id = POST body personJson asserting (StatusCode === Status.Created) returning (Header("X-Person-Id"))
      GET / id asserting (StatusCode === Status.OK, jsonBodyAs[Person] === Jason)
      GET asserting (StatusCode === Status.OK, jsonBodyAsList[Person] === Seq(Jason))
      DELETE / id asserting (StatusCode === Status.OK)
      GET / id asserting (StatusCode === Status.NotFound)
      GET asserting (StatusCode === Status.OK, jsonBodyAsList[Person] === EmptyList)
    }
  }

  "The DSL" should "support 'using' function to abstract common parameters in a readable way" in {
    import JsonExtractors._

    using(_ url "http://api.rest.org/person/") { implicit rb =>
      GET asserting (StatusCode === Status.OK)
      driver.lastRequest should have('method(GET), 'url(new URI("http://api.rest.org/person/")))

      val e = the[AssertionError] thrownBy { GET asserting (StatusCode === Status.Created) }
      e should have('message("StatusCode: 200 did not equal 201"))
      driver.lastRequest should have('method(GET), 'url(new URI("http://api.rest.org/person/")))
    }
  }
}
```

```scala
/**
 * Extract the response body as a json document
 */
val JsonBody = BodyText andThen Json.parse as "JsonBody"

/**
 * Extract the response body as an object.
 */
def jsonBodyAs[T: Reads : reflect.ClassTag]: Extractor[T] = jsonBodyAs(JsPath)

/**
 * Extract a portion of the response body as an object.
 *
 * @param path the path for the portion of the response to use
 */
def jsonBodyAs[T: Reads : reflect.ClassTag](path: JsPath) = {
  val tag = implicitly[reflect.ClassTag[T]]
  JsonBody andThen (jsonToValue(_, path)) as (s"jsonBodyAs[${tag.runtimeClass.getSimpleName}]")
}

/**
 * Extract the response body as a List of objects.
 */
def jsonBodyAsList[T: Reads](implicit tag : reflect.ClassTag[T] ): Extractor[Seq[T]] = jsonBodyAsList(JsPath)

/**
 * Extract a portion of the response body as a List of objects.
 *
 * @param path the path for the portion of the response to use
 */
def jsonBodyAsList[T: Reads : reflect.ClassTag](path: JsPath) = {
  val tag = implicitly[reflect.ClassTag[T]]
  JsonBody andThen (jsonToList(_, path)) as (s"jsonBodyAsList[${tag.runtimeClass.getSimpleName}]")
}

val BodyAsPerson = jsonBodyAs[Person]
val BodyAsName = jsonBodyAs[String](__ \ "name")
```


```scala
class PersonSpec extends FlatSpec with Dsl {

  "/person api" should "support basic crud operations" in {

    GET / "person" asserting (StatusCode === Status.OK,
      BodyAsPersonList === EmptyList)
  }
}


class PersonSpec extends FlatSpec {
  import Dsl._

  "/person api" should "support basic crud operations" in {

    GET / "person" asserting (StatusCode === Status.OK,
      BodyAsPersonList === EmptyList)
  }
}
```

```scala
trait Dsl extends Api with Extractors {
  implicit def toRequest ...

  implicit class RichRequestBuilder ...
  implicit class RichResponse ...
}

object Dsl extends Dsl
```

```scala
type Assertion = Request => Option[String]
```
```scala
implicit class RichExtractor[A](ext: ExtractorLike[A]) {
  def ===[B >: A](expected: B): Assertion = Assertion = { res =>
    val maybeValue = ext.value(res)
    maybeValue match {
      case Success(value) if (value == expected) =>
        None
      case Success(_) =>
        Some(s"${ext.name}: $value did not equal $expected")
      case Failure(e) =>
        Some(e.getMessage)
    }
}
```



Here

```scala
object Extractors {
  trait ExtractorLike[+A] {
    def name: String
    def value(implicit res: Response): Try[A]
    def unapply(res: Response): Option[A] = value(res).toOption
  }

  case class Extractor[+A](name: String, op: Response => A) extends ExtractorLike[A] {
    override def value(implicit res: Response): Try[A] = {
      Try { op(res) } recoverWith {
        case e =>
          Failure[A](new ExtractorFailedException(s"Cannot extract $name from Response: $e",e))
      }
    }
    def andThen[B](nextOp: A => B): Extractor[B] = copy(name = name + ".andThen ?", op = op andThen nextOp)
    def as(newName: String) = copy(name = newName)
  }

  val StatusCode = Extractor[Int]("StatusCode", r => r.statusCode)
  val BodyText = Extractor[String]("BodyText", r => r.body.get)
  def header(name: String) = {
    Extractor[String](s"header($name)", r => r.headers(name).mkString(", "))
  }

  val JsonBody = BodyText andThen Json.parse as "JsonBody"
  def jsonBodyAs[T : Reads : ClassTag](path: JsPath): Extractor[T] = {
    val tag = implicitly[ClassTag[T]]
    JsonBody andThen (jsonToValue(_, path)) as (s"jsonBodyAs[${tag.runtimeClass.getSimpleName}]")
  }
  def jsonBodyAs[T : Reads : ClassTag]: Extractor[T] = jsonBodyAs(JsPath)
  val BodyAsPerson = jsonBodyAs[Person]
}

trait Dsl extends Api with Extractors {
  implicit def toRequest(b: RequestBuilder): Request = b.toRequest

  implicit def methodToRequestBuilder(m: Method)(implicit builder: RequestBuilder): RequestBuilder = builder.withMethod(m)

  def using(config: RequestBuilder => RequestBuilder)(process: RequestBuilder => Unit)(implicit builder: RequestBuilder): Unit = {
    process(config(builder))
  }

  type Assertion = Response => Option[String]

  implicit class RichRequestBuilder(builder: RequestBuilder) {
    def url(u: String) = b.withUrl(u)
    def body(b: String) = b.withBody(b)  
    def /(p: Any) = b.addPath(p.toString)
    def :?(params: (Symbol, Any)*) = b.addQuery(params map (p => (p._1.name, p._2.toString)): _*)

    def execute()(implicit httpClient: HttpClient): Response = {
      httpClient(builder)
    }

    def returning[T1](ext1: Extractor[T1])(implicit httpClient: HttpClient): T1 = {
      val res = execute()
      ext1.value(res)
    }

    def returning[T1, T2](ext1: Extractor[T1], ext1: Extractor[T2])(implicit httpClient: HttpClient): (T1, T2) = {
      val res = execute()
      (ext1.value(res), ext2.value(res))
    }

    def asserting(assertions: Assertion*)(implicit client: HttpClient): Response = {
      val response = execute()
      val assertionFailures = for {
        assertion <- assertions
        failureMessage <- assertion(response)
      } yield failureMessage
      if (assertionFailures.nonEmpty) {
        throw assertionFailed(assertionFailures)
      }
      response
    }
  }

  implicit class RichExtractor[A](ext: ExtractorLike[A]) {
    def ===[B >: A](expected: B): Assertion = { res =>
      val maybeValue = ext.value(res)
      maybeValue match {
        case Success(value) if (value == expected) =>
          None
        case Success(value) =>
          Some(s"${ext.name}: $value != $expected")
        case Failure(e) =>
          Some(e.getMessage)
      }
    }

    def !==[B >: A](expected: B): Assertion = ???
    def < [B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = ???
    def <=[B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = ???
    def > [B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = ???
    def >=[B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = ???
  }
}
```
