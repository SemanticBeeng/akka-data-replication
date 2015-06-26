Akka Distributed Data
=====================

This library (akka-data-replication) has been included in Akka, in the module
[Distributed Data](http://doc.akka.io/docs/akka/snapshot/scala/distributed-data.html).

**It will not be maintained in patriknw/akka-data-replication.** All bug fixes and new features
will be done in [akka/akka](https://github.com/akka/akka/).

Migration Guide
---------------

The functionality of akka-distributed-data-experimental 2.4-M2 is very similar to akka-data-replication 0.11.
Here is a list of the most important changes:

* Dependency 
  `"com.typesafe.akka" % "akka-distributed-data-experimental_2.11" % 2.4-M2`
  (or later)
* The package name changed to `akka.cluster.ddata`
* The extension was renamed to `DistributedData`
* The keys changed from strings to classes with unique identifiers and type information of the data values,
  e.g. `ORSetKey[Int]("set2")`
* The data value was removed from unapply extractor in `GetSuccess` and `Changed` messages. Instead it
  is accessed with the `get` method. E.g. `case c @ Changed(DataKey) => val e = c.get(DataKey).elements`.
  The reason is to utilize the type information from the typed keys.
* The optional read consistency parameter was removed from the `Update` message. If you need to read from
  other replicas before performing the update you have to first send a `Get` message and then continue with
  the ``Update`` when the ``GetSuccess`` is received.
* `BigInt` is used in `GCounter` and `PNCounter` instead of `Long`
* Improvements of java api

Akka Data Replication
=====================

This was (see above) an **EARLY PREVIEW** of a library for replication of data in an Akka cluster.
It is a replicated in-memory data store supporting low latency and high availability
requirements. The data must be so called **Conflict Free Replicated Data Types** (CRDTs), 
i.e. they provide a monotonic merge function and the state changes always converge.

For good introduction to CRDTs you should watch the 
[Eventually Consistent Data Structures](http://www.google.com/url?q=http%3A%2F%2Fvimeo.com%2F43903960&sa=D&sntz=1&usg=AFQjCNF0yKi4WGCi3bhhdtLvBc33uVia6w)
talk by Sean Cribbs.

CRDTs can't be used for all types of problems, but when they can they have very nice properties:

- low latency of both read and writes
- high availability (partition tolerance)
- scalable (no central coordinator)
- strong eventual consistency (eventual consistency without conflicts)

Built in data types:

- Counters: `GCounter`, `PNCounter`
- Registers: `LWWRegister`, `Flag`
- Sets: `GSet`, `ORSet`
- Maps: `ORMap`, `LWWMap`, `PNCounterMap`, `ORMultiMap`

You can use your own custom data types by implementing the `merge` function of the `ReplicatedData`
trait. Note that CRDTs typically compose nicely, i.e. you can use the provided data types to build richer
data structures.

The `Replicator` actor implements the infrastructure for replication of the data. It uses
direct replication and gossip based dissemination. The `Replicator` actor is started on each node
in the cluster, or group of nodes tagged with a specific role. It communicates with other 
`Replicator` instances with the same path (without address) that are running on other nodes. 
For convenience it is typically used with the `DataReplication` Akka extension.

A short example of how to use it:

``` scala
class DataBot extends Actor with ActorLogging {
  import DataBot._
  import Replicator._

  val replicator = DataReplication(context.system).replicator
  implicit val cluster = Cluster(context.system)

  import context.dispatcher
  val tickTask = context.system.scheduler.schedule(5.seconds, 5.seconds, self, Tick)

  replicator ! Subscribe("key", self)

  def receive = {
    case Tick =>
      val s = ThreadLocalRandom.current().nextInt(97, 123).toChar.toString
      if (ThreadLocalRandom.current().nextBoolean()) {
        // add
        log.info("Adding: {}", s)
        replicator ! Update("key", ORSet(), WriteLocal)(_ + s)
      } else {
        // remove
        log.info("Removing: {}", s)
        replicator ! Update("key", ORSet(), WriteLocal)(_ - s)
      }

    case _: UpdateResponse => // ignore

    case Changed("key", ORSet(elements) =>
      log.info("Current elements: {}", elements)
  }

  override def postStop(): Unit = tickTask.cancel()

}
```
    
The full source code for this sample is in 
[DataBot.scala](https://github.com/patriknw/akka-data-replication/blob/v0.11/src/test/scala/akka/contrib/datareplication/sample/DataBot.scala).   

More detailed documentation can be found in the
[ScalaDoc](http://dl.bintray.com/patriknw/maven/com/github/patriknw/akka-data-replication_2.11/0.11/akka-data-replication_2.11-0.11-javadoc.jar)
of `Replicator` and linked classes.

Other examples:

- [Replicated Cache](https://github.com/patriknw/akka-data-replication/blob/v0.11/src/multi-jvm/scala/sample/datareplication/ReplicatedCacheSpec.scala#L30)
- [Replicated Metrics](https://github.com/patriknw/akka-data-replication/blob/v0.11/src/multi-jvm/scala/sample/datareplication/ReplicatedMetricsSpec.scala#L30)
- [Replicated Service Registry](https://github.com/patriknw/akka-data-replication/blob/v0.11/src/multi-jvm/scala/sample/datareplication/ReplicatedServiceRegistrySpec.scala#L46)
- [VotingService](https://github.com/patriknw/akka-data-replication/blob/v0.11/src/multi-jvm/scala/sample/datareplication/VotingContestSpec.scala#L30)
- [ShoppingCart](https://github.com/patriknw/akka-data-replication/blob/v0.11/src/multi-jvm/scala/sample/datareplication/ReplicatedShoppingCartSpec.scala#L31)

Dependency
----------

Latest version of `akka-data-replication` is `0.11`. This version depends on Akka 2.3.9 and is
cross-built against Scala 2.10.5 and 2.11.6.

Add the following lines to your `build.sbt` file:

    resolvers += "patriknw at bintray" at "http://dl.bintray.com/patriknw/maven"

    libraryDependencies += "com.github.patriknw" %% "akka-data-replication" % "0.11"

More Resources
--------------

* [Eventually Consistent Data Structures](http://www.google.com/url?q=http%3A%2F%2Fvimeo.com%2F43903960&sa=D&sntz=1&usg=AFQjCNF0yKi4WGCi3bhhdtLvBc33uVia6w)
  talk by Sean Cribbs
* [Strong Eventual Consistency and Conflict-free Replicated Data Types](http://www.google.com/url?q=http%3A%2F%2Fresearch.microsoft.com%2Fapps%2Fvideo%2Fdl.aspx%3Fid%3D153540&sa=D&sntz=1&usg=AFQjCNFiwLpLjF-AQXPUm1Nmoy8hNIfrSQ)
  talk by Mark Shapiro
* [A comprehensive study of Convergent and Commutative Replicated Data Types](http://www.google.com/url?q=http%3A%2F%2Fhal.upmc.fr%2Fdocs%2F00%2F55%2F55%2F88%2FPDF%2Ftechreport.pdf&sa=D&sntz=1&usg=AFQjCNEGvFJ9I5m7yKpcAs8hcMP9Y5vy6A)
  paper by Mark Shapiro et. al. 
