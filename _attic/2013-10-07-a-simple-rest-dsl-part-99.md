---
layout: post
title: "A Simple REST DSL Part 5"
description: ""
category: 
tags: [scala, dsl, scalatest, testing, rest, resttest]
---

```scala
val Jason: Person = ???
val personJson = Json.stringify(Jason)
val EmptyList = List[Person]()

using(_ url "http://api.rest.org/person") { implicit rb =>
  GET asserting (statusCode is Status.OK, jsonBodyAsList[Person] is EmptyList)
  val id = POST body personJson asserting (statusCode is Status.Created) returning (header("X-Person-Id"))
  GET / id asserting (statusCode is Status.OK, jsonBodyAs[Person] is Jason)
  GET asserting (statusCode is Status.OK, jsonBodyAsList[Person] is Seq(Jason))
  DELETE / id asserting (statusCode is Status.OK)
  GET / id asserting (statusCode is Status.NotFound)
  GET asserting (statusCode is Status.OK, jsonBodyAsList[Person] is EmptyList)
}
```


```scala
val Jason: Person = ???
val personJson = Json.stringify(Jason)
val EmptyList = List[Person]()

using(_ url "http://api.rest.org/person") { implicit rb =>
  GET should have(StatusCode(Status.OK), jsonBodyAsList[Person](EmptyList)
  val (status, id) = POST body personJson returning (StatusCode, headerText("X-Person-Id"))
  status should be(Status.Created)
  val foo = GET / id should have(StatusCode(Status.OK), jsonBodyAs[Person](Jason))
  GET should have(StatusCode(Status.OK), jsonBodyAsList[Person](Seq(Jason)))
  DELETE / id should have(StatusCode(Status.OK))
  GET / id should have(StatusCode(Status.NotFound))
  GET should have(StatusCode(Status.OK), jsonBodyAsList[Person](EmptyList))
}
```


```scala
val Jason: Person = ???
val personJson = Json.stringify(Jason)
val EmptyList = List[Person]()

val CustomHeaderId = headerText("X-Person-Id")
val BodyAsPerson = jsonBodyAs[Person]
val BodyAsListPerson = jsonBodyAsList[Person]

using(_ url "http://api.rest.org/person") { implicit rb =>
  GET expecting {
    case StatusCode(Status.OK) ~ BodyAsListPerson(list) => list should be(EmptyList)
  }

  val id = POST body personJson expecting {
    case StatusCode(Status.Created) & CustomHeaderId(id) => id
    case StatusCode(Status.Created) & LocationHeader(loc) => extractId(loc)
  }

  GET / id expecting {
    case StatusCode(Status.OK) ~ BodyAsPerson(person) => person should be(EmptyList)
  }

  GET expecting { 
    case StatusCode(Status.OK) ~ BodyAsListPerson(list) => list should be(Seq(Jason))
  }

  DELETE / id expecting {
    case StatusCode(code) => code should be(Status.OK)
  }

  GET / id expecting {
    case StatusCode(code) => code should be(Status.NotFound)
  }

  GET expecting {
    case StatusCode(Status.OK) ~ BodyAsListPerson(list) => list should be(EmptyList)
  }
}
```


```scala
val Jason: Person = ???
val personJson = Json.stringify(Jason)
val EmptyList = List[Person]()

using(_ url "http://api.rest.org/person") { implicit rb =>
  GET expecting { implicit res =>
    StatusCode should be(Status.OK)
    jsonBodyAsList[Person] should be(EmptyList)
  }

  val id = POST body personJson expecting { implicit res =>
    StatusCode should be(Status.Created)
    headerText("X-Person-Id").value
  }

  GET / id expecting { implicit res =>
    StatusCode should be(Status.OK)
    jsonBodyAs[Person] should be(Jason)
  }

  GET expecting { implicit res =>
    StatusCode should be(Status.OK)
    jsonBodyAsList[Person] should be(Seq(Jason))
  }

  DELETE / id expecting { implicit res =>
    StatusCode should be(Status.OK)
  }

  GET / id expecting { implicit res =>
    StatusCode should be(Status.NotFound)
  }

  GET expecting { implicit res =>
    StatusCode should be(Status.OK)
    jsonBodyAsList[Person] should be(EmptyList)
  }
}
```

```scala
val Jason: Person = ???
val personJson = Json.stringify(Jason)
val EmptyList = List[Person]()

using(_ url "http://api.rest.org/") { implicit rb =>
  GET expecting { implicit res =>
    StatusCode should be(Status.OK)
    jsonBodyAsList[Person] should be(EmptyList)
  }

  val id = POST / "person" body personJson expecting { implicit res =>
    StatusCode should be(Status.Created)
    headerText("X-Person-Id").value
  }

  GET  / "person" / id expecting { implicit res =>
    StatusCode should be(Status.OK)
    jsonBodyAs[Person] should be(Jason)
  }

  GET / "person" expecting { implicit res =>
    StatusCode should be(Status.OK)
    jsonBodyAsList[Person] should be(Seq(Jason))
  }

  DELETE / "person" / id expecting { implicit res =>
    StatusCode should be(Status.OK)
  }

  GET / "person" / id expecting { implicit res =>
    StatusCode should be(Status.NotFound)
  }

  GET / "person" expecting { implicit res =>
    StatusCode should be(Status.OK)
    jsonBodyAsList[Person] should be(EmptyList)
  }
}
```

```scala
val Jason: Person = ???
val personJson = Json.stringify(Jason)
val EmptyList = List[Person]()

using(_ url "http://api.rest.org/") { implicit rb =>
  GET expecting { implicit res =>
    StatusCode should be(Status.OK)
    jsonBodyAsList[Person] should be(EmptyList)
  }

  val request1 = Request(GET, new URI("http://api.rest.org/person", Map(), None))
  val response1 = driver.execute(request1)
  response1.statusCode should be(Status.OK)
  response1.body match {
    Some(body) => objectMapper.readValue[List[Person]](body) should be(EmptyList)
    None => fail("Expected a body"))
  }

  val request2 = Request(POST, new URI("http://api.rest.org/person", Map(), None))
  val response2 = driver.execute(request2)
  response2.statusCode should be(Status.Created)
  val id = response2.headers("X-Person-Id")

  val request3 = Request(GET, new URI("http://api.rest.org/person/" + id, Map(), None))
  val response3 = driver.execute(request3)
  response3.statusCode should be(Status.OK)
  response3.body match {
    Some(body) => objectMapper.readValue[Person](body) should be(Jason)
    None => fail("Expected a body"))
  }

  val request4 = Request(GET, new URI("http://api.rest.org/person", Map(), None))
  val response4 = driver.execute(request4)
  response4.statusCode should be(Status.OK)
  response4.body match {
    Some(body) => objectMapper.readValue[List[Person]](body) should be(Seq(Jason))
    None => fail("Expected a body"))
  }

  val request5 = Request(GET, new URI("http://api.rest.org/person", Map(), None))
  val response5 = driver.execute(request5)
  response5.statusCode should be(Status.OK)

  val request6 = Request(GET, new URI("http://api.rest.org/person", Map(), None))
  val response6 = driver.execute(request6)
  response6.statusCode should be(Status.OK)
  response6.body match {
    Some(body) => objectMapper.readValue[List[Person]](body) should be(EmptyList)
    None => fail("Expected a body"))
  }
}
```

```scala
val StatusCode = Extractor[Int]("statusCode", r => r.statusCode)

val BodyOption = Extractor[Option[String]]("bodyOption", r => r.body)

val BodyText = Extractor[String]("body", r => r.body.get)

def header(name: String) = ExtractorString](s"headerList($name)", r.headers(name).mkString(","))

def header(name) = headerList(name) andThen (l => l.mkString(",")) as (s"header($name)")

val JsonBody: Extractor[JsValue] = BodyText andThen Json.parse

def jsonToValue[T: Reads](json: JsValue, path: JsPath): T = {
  path.asSingleJson(json).as[T]
}

def jsonBodyAs[T: Reads](path: JsPath): Extractor[T] = { 
  jsonBody andThen (jsonToValue(_, path))
}

def jsonBodyAs[T: Reads]: Extractor[T] = jsonBodyAs(JsPath)

val BodyAsPerson = jsonBodyAs[Person](JsPath)

val BodyAsName = jsonBodyAs[String](JsPath \ "name")
```

```scala

class PersonSpec extends FlatSpec with Dsl {
  this: HasDriver =>

  "/person" should "support a basic rest use case" in {
    GET / "person" asserting (StatusCode === Status.OK, 
      BodyAsPersonList === EmptyList)
    val id = POST / "person" body personJson 
      asserting (StatusCode === Status.Created) 
      returning (header("X-Person-Id"))
    GET / "person" / id asserting (StatusCode === Status.OK, 
      BodyAsPersonList === Jason)
    GET asserting (StatusCode === Status.OK, 
      BodyAsPersonList === Seq(Jason))
    DELETE / "person" / id asserting (StatusCode === Status.OK)
    GET / id asserting (StatusCode === Status.NotFound)
    GET / "person" asserting (StatusCode === Status.OK, 
      BodyAsPersonList === EmptyList)
  }
}

class PersonUnitSpec extends PersonSpec with UnitTestDriver

class PersonSystemSpec extends PersonSpec with SystemTestDriver {
  override val baseUrl = "http://api.rest.org/person"
}
```

```scala

val personJson = """{ "name": "Jason" }"""

val r1 = driver.execute(Request(
  GET, new URI("http://api.rest.org/person/"), 
  Map(), None))
assert(r1.statusCode === Status.OK)
r1.body match {
  Some(body) => 
    assert(jsonAsList[Person](body) === EmptyList)
  None => fail("Expected a body"))
}

val r2 = driver.execute(Request(
  POST, new URI("http://api.rest.org/person/"), 
  Map(), Some(personJson)))
assert(r2.statusCode === Status.Created)
val id = r2.headers("X-Person-Id").head

val r3 = driver.execute(Request(
  GET, new URI("http://api.rest.org/person/" + id), 
  Map(), None))
assert(r3.statusCode === Status.OK)
r3.body match {
  Some(body) => 
    assert(jsonAs[Person](body) === Jason)
  None => fail("Expected a body"))
}

val r4 = driver.execute(Request(
  GET, new URI("http://api.rest.org/person/"), 
  Map(), None))
assert(r4.statusCode === Status.OK)
r4.body match {
  Some(body) => 
    assert(jsonAsList[Person](body) === Seq(Jason))
  None => fail("Expected a body"))
}

val r5 = driver.execute(Request(
  DELETE, new URI("http://api.rest.org/person/" + id),
  Map(), None))
assert(r5.statusCode === Status.OK)

val r6 = driver.execute(Request(
  GET, new URI("http://api.rest.org/person/"), 
  Map(), None))
assert(r6.statusCode === Status.OK)
r6.body match {
  Some(body) => 
    assert(jsonAsList[Person](body) === EmptyList)
  None => fail("Expected a body"))
}
```


```scala
  trait Assertion {
    def result(res: Response): Option[String]
  }


def assert(assertions: Assertion*)(implicit driver: Driver): Response = {
  val res = execute()
  val assertionResults = for {
    a <- assertions
    r <- a.result(res)
  } yield r
  if(assertionResults.nonEmpty) {
    throw assertionFailed(assertionResults)
  }
  res
}

implicit class RichExtractor[A](ext: ExtractorLike[A]) {
    def ===[B >: A](expected: B): Assertion = ???
    def !==[B >: A](expected: B): Assertion = ???

    def < [B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = ???
    def <=[B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = ???
    def > [B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = ???
    def >=[B >: A](expected: B)(implicit ord: math.Ordering[B]): Assertion = ???   
}

case class Extractor[+A](name: String, op: Response => A) extends ExtractorLike[A] {
  def value(res: Response): A = op(res)
}
```


```scala
trait Workday {
  def apply[U : Smart](you: U): Future[Career]
}

workday(me)
```