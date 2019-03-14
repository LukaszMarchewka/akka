# Coordination

@@@ warning

This module is currently marked as @ref:[may change](common/may-change.md). It is ready to be used
in production but the API may change without warning or a deprecation period.

@@@

Akka Coordination is a set of tools for distributed coordination.

## Dependency

@@dependency[sbt,Gradle,Maven] {
  group="com.typesafe.akka"
  artifact="akka-coordination_$scala.binary_version$"
  version="$akka.version$"
}

## Lease

The lease is a pluggable API for a distributed lock. 

## Using a lease

Leases are loaded with:

* Lease name
* Config location to indicate which implementation should be loaded
* Owner name 

Any lease implementation should provide the following guarantees:

* A lease with the same name loaded multiple times, even on different nodes, is the same lease 
* Only one owner can acquire the lease at a time

To acquire a lease:

Scala
:  @@snip [LeaseDocSpec.scala](/akka-coordination/src/test/scala/docs/akka/coordination/LeaseDocSpec.scala) { #lease-usage }

Java
:  @@snip [LeaseDocTest.scala](/akka-coordination/src/test/java/jdocs/akka/coordination/lease/LeaseDocTest.java) { #lease-usage }

Acquiring a lease returns a @scala[Future]@java[CompletionStage] as lease implementations typically are implemented 
via a third party system such as the Kubernetes API server or Zookeeper.

Once a lease is acquired `checkLease` can be called to ensure that the lease is still acquired. As lease implementations
are based on other distributed systems a lease can be lost due to a timeout with the third party system. This operation is 
not asynchronous so can be called before performing any action for which having the lease is important.

A lease has an owner. If the same owner tries to acquire the lease multiple times it will succeed i.e. leases are reentrant. 

It is important
to pick a lease name that will be unique for your use case. If a lease needs to be unique for each node in a Cluster the cluster host port can be use:

Scala
:  @@snip [LeaseDocSpec.scala](/akka-coordination/src/test/scala/docs/akka/coordination/LeaseDocSpec.scala) { #cluster-owner }

Java
:  @@snip [LeaseDocTest.scala](/akka-coordination/src/test/java/jdocs/akka/coordination/lease/LeaseDocTest.java) { #cluster-owner }

For use cases where multiple different leases on the same node some then something unique to your use cases must be used. For example
a lease can be used with Cluster Sharding and in this case the shard Id is included in the lease name used for each shard.

## Usages in other Akka modules

Leases can be used for @ref[Cluster Singletons](cluster-singleton.md#lease) and @ref[Cluster Sharding](cluster-sharding.md#lease). 

## Lease implementations

* [Kubernetes API](https://developer.lightbend.com/docs/akka-commercial-addons/current/kubernetes-lease.html)

## Implementing a lease

Implementations should extend
the @scala[`akka.coordination.lease.scaladsl.Lease`]@java[`akka.coordination.lease.javadsl.Lease`] 

Scala
:  @@snip [LeaseDocSpec.scala](/akka-coordination/src/test/scala/docs/akka/coordination/LeaseDocSpec.scala) { #lease-example }

Java
:  @@snip [LeaseDocTest.scala](/akka-coordination/src/test/java/jdocs/akka/coordination/lease/LeaseDocTest.java) { #lease-example }

The methods should provide the following guarantees:

* `acquire` should complete with: `true` if the lease has been acquired, `false` if the lease is taken by another owner, or fail if it can't communicate with the third party system implementing the lease.
* `release` should complete with: `true` if the lease has definitely been released, `false` if the lease has definitely not been released, or fail if it is unknown if the lease has been released.
* `checkLease` should return false until an `acquire` Future has completed and should return `false` if the lease is lost due to an error communicating with the third party. Check lease should also not block.
* The `acquire` lease lost callback should only be called after an `aquire` future has completed and should be called if the lease is lose e.g. due to losing communication with the third party system.

In addition it is expected that if a node that holds a lease that a lease implementation will include a time to live mechanism meaning that it won't be held for ever.
If a user prefers to have outside intervention in this case for maximum safety then the time to live can be set to infinite.

Create a configuration location with the following configuration:

* `lease-class` property for the FQN of the lease implementation
* `hearthbeat-timeout` if the node that acquired the leases crashes, how long should the lease be held before another owner can get it 
* `heartbeat-interval` interval for communicating with the third party to confirm the lease is still held
* `lease-operation-timeout` lease implementations are expected to time out the @scala[`Future`s]@java[`CompletionStages`s] returned by `acquire` and `release` this is how long that timeout can be. Lease implementations can choose to not implement this and ignore this timeout

This configuration location is passed into `getLease`.

Scala
:  @@snip [LeaseDocSpec.scala](/akka-coordination/src/test/scala/docs/akka/coordination/LeaseDocSpec.scala) { #lease-config }

Java
:  @@snip [LeaseDocSpec.scala](/akka-coordination/src/test/scala/docs/akka/coordination/LeaseDocSpec.scala) { #lease-config }




