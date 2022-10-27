# Distributed Actors Cluster

* Proposal: [SSWG-0020](0020-distributed-actors-cluster.md)
* Authors: [Konrad ‘ktoso’ Malawski](https://github.com/ktoso)
* Review Manager: TBD
* Status: **Implemented**
* Implementation: [swift-distributed-actors](https://github.com/apple/swift-distributed-actors)
* Forum Threads: **TODO**

## Package Description

The swift-distributed-actors package provides a fully functional server-side focused cluster implementation for the `distributed actor` language feature, introduced in Swift 5.7. 

This package contains a feature rich implementation of an actor system cluster, and general framework for building distributed systems in Swift. It integrates with various observability libraries the SSWG endorses and is based on top of swift-nio for its networking stack.

|  |  |
|--|--|
| **Package Name** | `swift-distributed-actors` |
| **Module Name** | `DistributedCluster` |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [Apache 2.0](https://github.com/swift-server/swift-backtrace/blob/master/LICENSE.txt) |
| **Dependencies** | [swift-nio](https://github.com/apple/swift-nio), [swift-log](https://github.com/apple/swift-log), [swift-metrics](https://github.com/apple/swift-metrics), [swift-service-discovery](https://github.com/apple/swift-service-discovery), [swift-cluster-membership](https://github.com/apple/swift-cluster-membership) |


## Introduction

Distributed actors are an extension of the "local only" actor model offered by Swift with its `actor` keyword.

Distributed actors used the `distributed actor` keywords (after importing the `Distributed` module, provided by Swift), and enable the declaring of `distributed func` methods inside such actor. Such methods may then be invoked remotely, from other peers in a distributed actor system.

The distributed actor *language feature* does not include any specific *runtime*, and only defines the language and semantic rules surrounding distributed actors. This library provides a feature-rich clustering server-side focused implementation of such runtime (i.e. a `DistributedActorSystem` implementation) for distributed actors.

Clusters are a fundamental building block of server-side distributed systems, such as lobby systems for game backends, IoT service backends, orchestration systems and general control plane systems of distributed databases or other distributed systems.

## Motivation

Distributed actors are introduced as nominal types in Swift 5.7. Similar to actors, they can be declared using the `distributed actor` pair of keywords. By themselves, they cannot really do anything – all the distributed actions such actor performs are actually handled by an `ActorSystem` associated with given actor type. 

Specifically, an actor must declare what type of actor system it is going to be used with, like this:

```swift
import Distributed
import DistributedCluster

distributed actor Greeter {
    typealias ActorSystem = ClusterSystem

    distributed func hello(name: String) -> String {
        return "Hello \(name)!"
    }
}
```

Such `Greeter` declaration can then be used in the context of the clustered distributed actor system. It is also possible to declare a module-wide default distributed actor system type, for more information refer to [Swift Distributed Actor Runtime](https://github.com/apple/swift-evolution/blob/main/proposals/0344-distributed-actor-runtime.md) and [Swift Distributed Actor Isolation](https://github.com/apple/swift-evolution/blob/main/proposals/0336-distributed-actor-isolation.md). It is also possible to declare a module-wide default distributed actor system, which is how most users of distributed actors are likely to use this feature, like this:

```swift
typealias DefaultDistributedActorSystem = ClusterSystem
```

Which avoids having to repeat the `ActorSystem` typealias in every distributed actor in a module.

The package being proposed here provides the implementation of the `ClusterSystem`.

## Proposed solution

The `DistributedCluster` package includes the `ClusterSystem` type which is the central piece of the library. Once created, it binds to a host/port pair and begins listening for incoming connections:

```swift
@main
struct Main {
    static func main() async throws {
        let system = await ClusterSystem("FirstSystem") { settings in
            settings.endpoint.host = "127.0.0.1"
            settings.endpoint.port = 7337
        }
        
        try await system.terminated
    }
}
```

Clusters are formed by “joining” other nodes. So the above created node has bound on port 7337, and if we had another node running on the same host but on port 8228 we could join in programmatically like this:

```swift
let other = Cluster.Endpoint(host: "127.0.0.1", port: 8228)
system.cluster.join(endpoint: other)
```

The `join(endpoint:)` method returns immediately, however it is possible to use `try await joined(endpoint:within:)` to suspend until the other node has been joined (or we failed doing so within the timeout).

Alternatively we can configure an actor system to automatically discover and join nodes by configuring a [`swift-service-discovery` node-based discovery mechanism](https://apple.github.io/swift-distributed-actors/1.0.0-beta.3/documentation/distributedcluster/clustering#Automatic-Node-Discovery), like this:

```swift
import ServiceDiscovery
import K8sServiceDiscovery // See: [tuplestream/swift-k8s-service-discovery](https://github.com/tuplestream/swift-k8s-service-discovery)
import DistributedCluster

ClusterSystem("DiscoverNodes") { settings in
    let discovery = K8sServiceDiscovery() 
    let target = K8sObject(labelSelector: ["name": "actor-cluster"], namespace: "actor-cluster")
    
    settings.discovery = ServiceDiscoverySettings(discovery, service: target)
}
```

Once a cluster has been formed, distributed actors use the [Receptionist](https://apple.github.io/swift-distributed-actors/1.0.0-beta.3/documentation/distributedcluster/receptionist) to discover check-in and discover distributed actors from other nodes. For example, we can write a distributed worker actor like this:

```swift
distributed actor Worker {
    typealias ActorSystem = ClusterSystem
    
    init(actorSystem: ActorSystem) async {
        self.actorSystem = actorSystem
        await actorSystem.receptionist.checkIn(worker, with: .workers) 
    }
    
    distributed func work() -> String { "Completed some work!" }
}

extension DistributedReception.Key {
    static var workers: DistributedReception.Key<Worker> {
        "workers"
    }
}
```

Such distributed actors are then created using normal initialization on nodes within the cluster. Each such node can then use the [receptionist listing](https://apple.github.io/swift-distributed-actors/1.0.0-beta.3/documentation/distributedcluster/receptionist#Receptionist-Listings) API to obtain all workers from all other nodes, like this:

```swift
for await worker in await actorSystem.receptionist.listing(of: .workers) {
    log.info("Found worker \(worker) on node \(worker.id.node)")
    let reply = try await worker.work()
    log.info("Worker \(worker.id) returned: \(reply)")
}
```

While this allows nodes to discover distributed actors on other processes, it does not deal with their termination. For example, if a remote node crashes, remote calls to any actor located on this node will fail. In order to notice such crash, or just plain actor termination (i.e. the remote `worker` having deinitialized) one can use the [`LifecycleWatch`](https://apple.github.io/swift-distributed-actors/1.0.0-beta.2/documentation/distributedcluster/lifecycle) functionality of the cluster:

```swift
distributed actor Boss: LifecycleWatch { 
    func findWorkers() {
        for await worker in await actorSystem.receptionist.listing(of: .workers) {
            watchTermination(of: worker)
        }
    }
    
    // Invoked whenever a watched actor terminates
    func terminated(actor id: ActorID) async {
        print("Oh no! Worker \(id) has terminated!")
    }
}
```

The mechanisms underlying lifecycle watch are powered by the advanced distributed failure detection mechanisms implemented in [swift-cluster-membership](https://github.com/apple/swift-cluster-membership), and allow for reliable failure detection in a distributed system. The same mechanism also works if the workers were local actors, and just happened to deinitialize. A remote actor that deinitializes also signals to all of its watchers that it has terminated. This way, applications can program against actor “lifecycle” rather than having to worry about where and how a distributed actor was running.

This concludes a quick overview of the primary features of the cluster. The cluster also offers low-level [cluster events](https://apple.github.io/swift-distributed-actors/1.0.0-beta.3/documentation/distributedcluster/cluster/event), as well as high level abstractions such as “[Cluster Singletons](https://apple.github.io/swift-distributed-actors/1.0.0-beta.3/documentation/distributedcluster/clustersingleton)“ which are explored in depth in the library’s [reference documentation](https://apple.github.io/swift-distributed-actors/1.0.0-beta.3/documentation/distributedcluster/introduction).

## Maturity Justification

As the package is currently still on its way towards its 1.0 release, we propose to include it in **Sandbox** level. At this point in time the package does not guarantee source stability, but we are very close to doing so along with the 1.0 release of the package.


## Alternatives considered

None.

There is no alternative feature complete clustering solution available in Swift today. The inclusion of this library does not preclude future inclusion of other distributed actor system implementations, if and when they come up.


## Discussion

We are considering the naming of the package still. The focus of this package is to serve as a reference implementation of an actor system, but it also is a server-side cluster focused solution, thus we may still consider renaming the package to be more aligned with the primary purpose of this package rather than a more generic name which leaves more room for alternative actor system implementations to be offered from within the same SwiftPM package.
