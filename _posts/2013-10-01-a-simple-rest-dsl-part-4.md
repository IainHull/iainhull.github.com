---
layout: post
title: "A Simple REST DSL Part 4"
description: ""
category: 
tags: [scala, dsl, builder, testing, rest, resttest]
---
{% include JB/setup %}

At the end of part 3 we had a Domain Specific Language (DLS) that could neatly express a complete REST use case. It fully described the requests and their expected responses.

```scala
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
```

Unfortunately how it specifies common request properties and the statements that depend on it is not as fluid as the rest of the syntax.

```scala
RequestBuilder() url "http://api.rest.org/person" apply { implicit rb =>
  // ...
}
```

There are a number of issues with this expression:

1. The intent, supplying a common url, is hidden between `RequestBuilder()` and `apply { implicit rb =>`.
1. Ideally the current `RequestBuilder` instance would be passed into the expression implicitly instead of requiring the `RequestBuilder()` call.
1. The `apply` method suggests the common properties are applied to the nested expressions.  However in Scala the `apply` method has special meaning and is normally called with bracket syntax. This is confusing to novice and experienced Scala programmers alike. Here `apply` must be expressed explicitly because of the lower precedence of infix expressions. A DSL should not have to explain its self like this. It should be obvious and follow the idioms of its host language.
1. The syntax is not very discoverable. How would a user figure out that pairing a `RequestBuilder()` with an `apply` method call would enable common properties to be reused.

Scala places some constraints on infix expressions:

1. They should be of the form `<subject> <verb> <object>`, this expression can return a new subject resulting in `<subject> <verb1> <object1> <verb2> <object2>` etc. So expressions must have an old number of terms, in our example the code block is the last term.
1. Expressions that do not follow this pattern must be terminated or wrapped with parentheses.
1. The expression must also be bootstrapped with a subject, in our case the current `RequestBuilder` object.  The call to `RequestBuilder()` does this but it is very verbose. Some DSLs introduce a special symbol in this case but that can obscure the intent even more.

Can this expression be revised to highlight its intent, work with Scala norms and be more discoverable? How can we work around the constraints of infix expressions? 

```scala
using(_ url "http://api.rest.org/person") { implicit rb =>
  // ...
}
```

1. The `using` method looks like a standard control structure, which encapsulates the feature in a single expression. It has two clear sections: 
    1. common request configuration in parentheses.
    1. and a code block for expressions that reuse this configuration. 
1. This simpler structure makes the intent clearer to someone reading the code for the first time.
1. It also provides a single place to document the feature making it more discoverable, and easier to write for the first time.
1. The single underscore `_` represents the current `RequestBuilder` this is a little subtle but follows the standard Scala convention for anonymous functions. It is shorter than `RequestBuilder()` and is also more idiomatic than a special DSL specific symbol.
1. The `apply` method does not have to be explicitly specified because the parentheses around the expression enable the Scala compiler to infer it.

So how is the `using` method implemented?

```scala
def using(config: RequestBuilder => RequestBuilder)
         (process: RequestBuilder => Unit)
         (implicit builder: RequestBuilder): Unit = {
  process(config(builder))
}
```

The `using` method takes three parameter lists each with a single parameter.  Breaking the parameters into their own list enables the method to be used like a control structure, where the function in the second parameter is specified as a code block.

The `config` parameter is just a function that takes a current `RequestBuilder`and returns the updated `RequestBuilder` required by the enclosing block.  This function is typically applied between `( ... )` and uses the `_` syntax to generate the function.  The `process` parameter is another function taking a `RequestBuilder` and returning `Unit`.  This is typically applied with `{ implicit rb => ... }`, my making the `rb` parameter implicit the `using` method can be nested.  The final `builder` parameter is the currently configured `RequestBuilder`, this is passed in implicitly.  

The implementation is very simple, generate updated `RequestBuilder` by calling the  `config` function with implicit `builder`. Then pass the result to the `process` function, where the updated `RequestBuilder` is used.

Now the full use case looks like:

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

Next I want to examine how assertions are implemented and whether they can be integrated with [ScalaTest](http://scalatest.org). I also have some opinions on how best to test DSLs. But both of these will have to wait for a subsequent post.