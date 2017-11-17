---
layout: post
title: "Akka Typed Test Probes"
subtitle: "Missing Akka TestKit with Akka Typed."
tags: [akka akka-typed test-probe tdd types]
---

It is no secret that I love types and have been using Akka for a number of years, so I could not wait to play with Akka Typed. Akka Typed is quite different from its untyped cousin. Every time I reach for my favorite class or idiom I find it works differently or maybe doesn't even exist at all. One of the big differences is that there is not TestKit for typed actors. People recommend you test the units without actors then use simple `tell` (`!`) and `askPattern` (`?`) for integration tests. I have been following this advice by and large, however recently discovered I require test probes to test an actor that simply orchestrates other actors.

Currently, I am implementing a multi-tenant queue for a scheduling system I am working on. The overall queue is represented by an actor, each tenant has its own subqueue which is also an actor. Splitting the tenant queues simplifies the logic of each queue, improves parallelism and helps me ensure jobs are allocated fairly across tenants. Later I will implement tenant lifecycle management where tenants will enter and leave the system dynamically based on external events.

For now I will simply support jobs entering and leaving the queue. The QueueActor and the TenantQueueActor support the same protocol. The QueueActor simply forwards the messages to the TenantQueueActor based on the tenant associated with the messages job, if a tenant actor does not exists one is created.

The logic of the QueueActor is very straight forward it creates actors on demand and routes messages to them. The protocol and QueueActor is implemented below, the `TenantQueueActor` is not implemented yet:


```scala
object QueueActor {

  // Protocol
  type JobId = String
  type Tenant = String
  case class Job(id: JobId, tenant: Tenant)


  sealed trait JobRequest {
    def job: Job
  }

  sealed trait JobResponse

  final case class EnqueueJob(job: Job, sender: ActorRef[EnqueueJobResponse]) extends JobRequest

  sealed trait EnqueueJobResponse extends JobResponse
  final case class JobEnqueued(job: Job) extends EnqueueJobResponse
  final case class JobNotEnqueued(job: Job, reason: String) extends   EnqueueJobResponse

  final case class CompleteJob(job: Job, sender: ActorRef[CompleteJobResponse]) extends JobRequest

  sealed trait CompleteJobResponse extends JobResponse
  final case class JobCompleted(job: Job) extends CompleteJobResponse
  final case class JobNotCompleted(job: Job, reason: String) extends   CompleteJobResponse

  // Actor implementation

  type MyContext = ActorContext[JobRequest]
  type MyActorRef = ActorRef[JobRequest]
  type MyBehavior = Behavior[JobRequest]

  def apply(tenantActors: Map[Tenant, MyActorRef] = Map(),
            childCounter: Int = 0,
            childFactory: Tenant => MyBehavior = TenantQueueActor): MyBehavior = {

    def getOrCreateTenantActor(ctx: MyContext, tenant: Tenant): (MyActorRef, MyBehavior) = {
      if (tenantActors.contains(tenant)) {
        tenantActors(tenant) -> Actor.same
      } else {
        val next = childCounter + 1
        val child = ctx.spawn(childFactory(tenant), s"${tenant}-${next}")
        val newBehaviour = apply(tenantActors + (tenant -> child), next, childFactory)

        child -> newBehaviour
      }
    }

    Actor.immutable { (ctx, message) =>
      val (child, newBehavior) = getOrCreateTenantActor(ctx, message.job.tenant)
      child ! message
      newBehavior
    }
  }
}

object TenantQueueActor extends (QueueActor.Tenant => QueueActor.MyBehavior) {
  def apply(tenant: QueueActor.Tenant): QueueActor.MyBehavior = ???
}

```

This is a perfect example of where I would use `TestProbe`s in untyped Akka. However these beasts do not exist in typed Akka, so I have created my own with an asynchronous twist. These test probes are designed to use [ScalaTest's Asynchronous](http://www.scalatest.org/user_guide/async_testing) testing framework.

At their simplest a test probe lets you construct `ActorRef`s and pass them to actors. Then you can intercept messages sent to the probe and optionally reply to these messages. The explicit sender in Akka Typed makes it very easy to support reply.

Here is a sample test.

```scala
class QueueActorSpec extends fixture.AsyncFlatSpec with CompleteLastly with Matchers {

  import QueueActor._


  // A fixture contains the actor as an ActorSystem and a single test probe
  case class Fixture(actor: ActorSystem[JobRequest], probe: AsyncTestProbe[JobRequest])
  type FixtureParam = Fixture

  // Construct the ActorSystem run the test lastly terminate the ActorSystem
  override def withFixture(test: OneArgAsyncTest): FutureOutcome = {
    val testProbe = new AsyncTestProbe[JobRequest]

    val actorSystem = ActorSystem(QueueActor(Map(), 0, _ => testProbe.behavior), "QueueActor")

    complete {
      withFixture(test.toNoArgAsyncTest(Fixture(actorSystem, testProbe)))
    } lastly {
      Await.result(actorSystem.terminate(), 2.second)
      println("Terminated")
    }
  }

  behavior of "QueueActor"

  it should "create child actors on demand" in { param =>
    val Fixture(actor, testProbe) = param
    implicit val timeout = Timeout(2.seconds)
    implicit val sched = actor.scheduler
    val tenant = "sometenant"
    val job = Job("12345678", tenant)

    // Now test

    val futureMessage = testProbe.nextMessage()  // 1
    val futureResponse: Future[EnqueueJobResponse] = actor ? (EnqueueJob(job, _)) // 2
    for {
      EnqueueJob(job, sender) <- futureMessage // 3
      _ = sender ! JobEnqueued(job) // 4
      JobEnqueued(j) <- futureResponse // 5
    } yield {
      j shouldBe job // 6
    }
  }
}
```

This test constructs an `AsyncTestProbe` and a `QueueActor` with a simple function for `childFactory` that always returns the `testProbe`'s behavior.

Now lets look at this test:

1. First read a future for the next message sent to the test probe.  Even though the test hasn't sent it a message (this works because we are using futures now).
2. Send the `EnqueueJob` to the `QueueActor` using the ask pattern, which returns a future for the response (the important part here is that we create both futures before the for comprehension).
3. Next read the message sent in step 2 from the test probe's future created in step 1. Deconstruct the `EnqueueJob` to read the `job` and the `sender`.
4. Use the `sender` to reply that the `JobEnqueued`. This completes the future returned in step 2.
5. Then read the message from the response future created in step 1, and sent in step 4.
6. Finally we assert that the two jobs are the same.

This test verifies the basic flow of the `QueueActor` using an `AsyncTestProbe` to mock out the `TenantQueueActor`. Keeping the messages returned by the test probe in futures means we can leverage the full power of ScalaTest's asynchronous tests to verify the value of messages and do not need a lot of the `expect*` methods found on untyped Akka's [TestProbe](https://doc.akka.io/api/akka/current/akka/testkit/TestProbe.html). This orthogonal design simplifies the implementation and reduces the learning curve. Of course much more sophisticated tests are possible and indeed required to verify the `QueueActor` but this example shows how useful test probes still are in Akka typed. Test probes can use used to build more sophisticated testing primitives.

Next we explore the implementation the the `AsyncTestProbe` for below. Even though the test probe works with futures, it still must use synchronization, the `nextMessage()` method is linking an `Actor` to the outside world (which should normally only be done with async message passing).

It uses two `Queue`s of `Promise`s:

* The `queueIn` holds promises for messages that the test probe receives. When a message is received the next promise is read from this queue and completed with the massage. The promises in this queue represent uncompleted futures returned from `nextMessage()`.
* The `queueOut` holds promises for futures returned from `nextMessage()`.  When this is called the next promise is read from this queue and its future returned.  The promises in this queue are completed and represent messages that have already be received for which `nextMessage()` has not yet been called.
* One of these queues is always empty. If more messages have been received than calls to `nextMessage()` then `queueIn` is empty. Likewise if there are more calls to `nextMessage()` than messages received then `queueOut` is empty. Both queues are empty when the calls are balanced.
* `nextPromiseFrom` returns the next promise from the specified queue, if the queue is empty a new promise is added to both queues before the next promise is read. This is the only method that modifies the queues and is the point of synchronization between the actor and the outside world.


```scala
/**
  * An actor test probe, that returns its results asynchronously. Each message the actor receives completes a promise
  *
  * @tparam A
  */
class AsyncTestProbe[A] {
  /**
    * Queue of promises that are completed when a message is received.
    * New promises are added to queueIn and queueOut at the same time
    */
  private val queueIn = mutable.Queue[Promise[A]]()

  /**
    * Queue of promises that are returned by `nextMessage`.
    */
  private val queueOut = mutable.Queue[Promise[A]]()

  /**
    * Read the next promise from the queue or create a new promise in both the in and out queues.
    *
    * NOTE: This method is synchronized as the requests for the queueOut come from the callers thread,
    * while the requests for the queueIn come from the actor's thread.
    *
    * @param queue The queue to read from
    *
    * @return the next promise.
    */
  private def nextPromiseFrom(queue: mutable.Queue[Promise[A]]): Promise[A] = this.synchronized {
    if (queue.isEmpty) {
      val promise = Promise[A]()
      queueIn.enqueue(promise)
      queueOut.enqueue(promise)
    }
    queue.dequeue()
  }

  /**
    * @return a future for the next message this test probe will receive
    */
  def nextMessage(): Future[A] = nextPromiseFrom(queueOut).future

  /**
    * @return a future for the next n messages this probe will receive
    */
  def nextMessages(n: Int)(implicit executor: ExecutionContext): Future[Seq[A]] = this.synchronized {
    require(n >= 1, s"Can only ask for one or more messages, not $n")

    val futures =  (0 to n) map { _ => nextPromiseFrom(queueOut).future }
    Future.sequence(futures)
  }

  private val mutableBehavior = new Actor.MutableBehavior[A] {
    override def onMessage(msg: A) = {
      nextPromiseFrom(queueIn).success(msg)
      this
    }
  }

  /**
    * @return the behavior of this test probe
    */
  def behavior: Behavior[A] = Actor.mutable { _ =>
    mutableBehavior
  }
}
```

I am still getting up to speed with Akka Typed, so there might be cleaner ways to test typed actor interactions without test probes. I hope this blog shows their utility or helps me discover the alternatives :-).

The source code for this blog is available on [github]( https://github.com/IainHull/akka-typed-testprobe).
