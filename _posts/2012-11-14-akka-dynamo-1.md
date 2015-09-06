---
layout: post
title: "Akka Dynamo 1"
subtitle: "First experiments with implementing Dynamo in Akka."
description: ""
category: 
tags: [akka, scala, dynamo, sbt]
---


http://lethain.com/hands-on-review-of-the-dynamo-paper/

## Setup ##

1. Install sbt

        brew install sbt

1. Create an akka project

        mkdir akka-dynamo
        cd akka-dynamo
        mkdir -p src/main/scala
        mkdir -p src/test/scala

1. Create build.sbt

        name := "Akka Dynamo"

        scalaVersion := "2.9.2"

        resolvers += "Typesafe Repository" at "http://repo.typesafe.com/typesafe/releases/"

        retrieveManaged := true

        libraryDependencies ++= Seq("com.typesafe.akka" % "akka-actor" % "2.0.3",
                                    "com.typesafe.akka" % "akka-remote" % "2.0.3",
                                    "com.typesafe.akka" % "akka-testkit" % "2.0.3" % "test",
                                    "org.specs2" %% "specs2" % "1.9" % "test",
                                    "junit" % "junit" % "4.5" % "test")

## The messages

Scala is a strongly typed language so the message types have to be defined. [Case classes] (http://www.scala-lang.org/node/107) are ideal for defining value types like akka messages and enable pattern matching in the actors.  Note `Ok`, `Fail` and `Stop` do not require any parameters so are defined as singltons with `case object` instead.

Erlang is weakly typed and most data structures are a combination of list, tuples and litterals.  In scala tuples are replaced cases classes, these are like named tuple types.  Litterals are either case objects or java primatives.

__messages.scala__

{% highlight scala %}
package org.iainhull.akka.dynamo

trait Response
case object Ok extends Response
case object Fail extends Response

trait ServerMessage
case class Start(n: Int) extends ServerMessage
case object Stop extends ServerMessage
case class GetRequest(id: String) extends ServerMessage
case class SetRequest(id: String, value: String) extends ServerMessage
case class GetResponse(response: Response, id: String, value: Option[String]) extends ServerMessage
case class SetResponse(response: Response, id: String) extends ServerMessage
{% endhighlight %}

## ServerActor

Akka actors are classes that extend `Actor` the method recieve is called for each message.  Unlike Erlang the this method does not have to recursivelly call itself.

The import statements are standard scala imports and are required to use the Akka api.  However the `akka.pattern` imports enable the `ask` and `pipe` patterns.  These enable actors to recieve responses to messages they send with the `?` operator and `pipe` the responses back as their own reply.

The `Start` message starts the server and spawns the database `NodeActor`s and replies with `Ok`.

The `Stop` message stops the nodes and itself by sending a `PoisonPill` to terminate the actor after processing the remaining messages.

The `GetRequest` simply passes the message to the first `NodeActor` and pipes the reply back to the sender.  This makes the GetRequest a simple proxy for the first node.

The `SetRequest` forwards the message to all `NodeActor`s and reduces the replies to a single success if they all succeed or a single failure if any of them fail

__ServerActor.scala__

{% highlight scala %}
package org.iainhull.akka.dynamo

import akka.actor.{ Actor, ActorRef, PoisonPill, Props }
import akka.dispatch.Future._
import akka.pattern.{ ask, pipe }
import akka.util.{ Duration, Timeout }

class ServerActor extends Actor {
  import context.dispatcher

  implicit val timeout: Timeout = Duration(2, "seconds")

  var nodes: Seq[ActorRef] = Seq()

  def receive = {
    case Start(n) => 
      nodes = 1 to n map { _ => 
        context.actorOf(Props[NodeActor]) 
      } 
      sender ! Ok
    case Stop => 
      nodes foreach { _ ! PoisonPill }
      self ! PoisonPill
      sender ! Ok
    case GetRequest(id) =>
      (nodes(0) ? GetRequest(id)) pipeTo sender
    case SetRequest(id, value) =>
      val results = nodes map { 
        _ ? SetRequest(id, value) 
      }
      reduce(results) {
        case (SetResponse(Ok, _), SetResponse(Ok, _)) => SetResponse(Ok, id)
        case (SetResponse(Fail, _), _) => SetResponse(Fail, id)
        case (_, SetResponse(Fail, _)) => SetResponse(Fail, id)
      } pipeTo sender
  }
}
{% endhighlight %}


1. Create specs2 unit test
1. Create server actor

Notes:

* have to define message classes (scala is strongly typed)
* initial version only uses server actor (no nodes)