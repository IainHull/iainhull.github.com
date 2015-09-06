---
layout: post
title: Akka and the Cake Pattern
description: ''
category: 
tags: ! '[]'
---
{% include JB/setup %}

The best advice I found when trying to apply the [Cake Pattern](http://jonasboner.com/2008/10/06/real-world-scala-dependency-injection-di/) to an [Akka](http://akka.io) application is this answer on [Stack Overflow](http://stackoverflow.com/questions/15996098/akka-and-cake-pattern). After applying this to our own Akka application I wanted to share our lessons.

Akka actors can you three major types of dependencies:

 * A single actor reference used to access a singleton actor. This could represent an external service or a facade for an actor subsystem.
 * The Props used to construct an child actor. This child actor might require other dependencies or information to access to the external service.
 * Other Non-actor services.

 Most of the Akka documentation and examples on the Internet use constructor dependency injection.  Here all dependencies are passed as constructor arguments, normally as actor references or Props.  This technique supports unit and integration testing by enabling them to inject test actors and [Probes](http://doc.akka.io/docs/akka/snapshot/scala/testing.html#Using_Multiple_Probe_Actors) for actors with external dependencies or actors not important for the test. 

 However there are some drawbacks:

  * This technique does not easily scale to large actor systems. For example when the actor hierarchy has multiple layers like  when using the [Error Kernel Pattern](http://doc.akka.io/docs/akka/snapshot/general/actor-systems.html#Hierarchical_Structure).
  * The more parameters that are added to an actor's constructor the harder it is to construct. This is especially a problem in tests when some of the parameters are dependencies that are required by the actors children or grand-children.
  * A simple Props object might not be enough to construct an actor it requires some additional parameters.  A function is required to construct the Props object.  While this could be passed as a constructor argument the redirection muddies the codes intent.

  Any dependency injection technique must address these issues but it must also support idiomatic Akka code and work well with [Akka best practices](http://doc.akka.io/docs/akka/snapshot/scala/actors.html#Recommended_Practices).

  problems with construtor/setter dep injection:
  * ensure that tests set the same value to all instances of the same singleton
  