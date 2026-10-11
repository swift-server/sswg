# Solution name

* Proposal: [SSWG-0035](0035-valkey-swift.md)
* Authors: [Adam Fowler](https://github.com/adam-fowler), [Nilanshu Sharma](https://github.com/nilanshu-sharma), [Fabian Fett](https://github.com/fabianfett)
* Review Manager: TBD
* Status: **Implemented**
* Implementation: [valkey-swift](https://github.com/valkey-io/valkey-swift)
* Forum Threads: [Pitch](https://forums.swift.org/)

<!--
*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Previous Revision(s): [1](https://github.com/swift-server/sswg/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal(s): [SSWG-XXXX](XXXX-filename.md)
-->

## Package Description

Valkey-swift is a client for the Valkey memory datastore.

|  |  |
|--|--|
| **Package Name** | `valkey-swift` |
| **Module Name** | `Valkey` |
| **Proposed Maturity Level** | [Incubated](https://www.swift.org/sswg/incubation-process.html#process-diagram) |
| **License** | [Apache 2.0](https://choosealicense.com/licenses/apache-2.0/) |
| **Dependencies** | [swift-atomics](https://github.com/apple/swift-atomics), [swift-collections](https://github.com/apple/swift-collections), [swift-configuration](https://github.com/apple/swift-configuration), [swift-distributed-tracing](https://github.com/apple/swift-distributed-tracing), [swift-log](https://github.com/apple/swift-log), [swift-nio](https://github.com/apple/swift-nio), [swift-nio-ssl](https://github.com/apple/swift-nio-ssl), [swift-nio-transport-services](https://github.com/apple/swift-nio-transport-services), [swift-service-lifecycle](https://github.com/apple/swift-service-lifecycle) |

## Introduction

Valkey is a high performance key/value datastore. It supports a variety of workloads. It is an open source fork of the Redis project. Valkey-swift is a client for the Valkey datastore, it is written using modern Swift features. It provides strongly typed APIs for all the Valkey commands. The client API uses structured concurrency to define lifetimes of tasks and their associated resources.

As Valkey is a fork of Redis and uses the same protocol (RESP), valkey-swift will work with Redis databases as well.

## Motivation

The current Redis client listed as an SSWG Sandbox project, RediStack, has been deprecated. The SSWG projects should include a RESP client that is actively being maintained.

## Proposed solution

The valkey-swift project contains two clients. `ValkeyClient` for working with standalone Valkey deployments (which can include replicas) and `ValkeyClusterClient` for working with cluster deployments. Each client manages multiple connection pools to multiple database.

### Running the client

Both clients requires a background process primarily to manage its connection pools. This can be run just using swift concurrency

```swift
let valkeyClient = ValkeyClient(
    .hostname("127.0.0.1", port: 6379),
    logger: logger
)
async let _ = valkeyClient.run()
```

Or alternatively you can use swift-service-lifecycle to manage the lifecyle of the client.

```swift
let services: [Service] = [myApp, valkeyClient]
let serviceGroup = ServiceGroup(
    services: services,
    gracefulShutdownSignals: [.sigint, .sigterm],
    logger: logger
)
try await serviceGroup.run()
```

### Command execution

Execute commands directly from the client (connection is managed automatically):

```swift
try await valkeyClient.set(key: "user:123", value: "alice")
let value = try await valkeyClient.get(key: "user:123")
```

### Pipelining

Send multiple commands at once without waiting for each response:

```swift
let (setResponse, getResponse) = await valkeyClient.execute(
    SET(key: "MyKey", value: "TestString"),
    GET(key: "MyKey")
)
let value = try getResponse.get()
```

### Connection Management

Get a dedicated connection when multiple commands need to run on the same connection:

```swift
try await valkeyClient.withConnection { connection in
    try await connection.set(key: "foo1", value: "bar")
    try await connection.set(key: "foo2", value: "baz")
    let value = try await connection.get(key: "foo1")
}
```

### Subscriptions (Pub/Sub)

Subscribe to one or more channels and process messages as an AsyncSequence. When the closure exits, the client automatically unsubscribes.

```swift
try await valkeyClient.subscribe(to: "notifications") { stream in
    for try await message in stream {
        print("Received: \(message.message)")
    }
}
```

When you subscribe from a `ValkeyClient`, the subscription runs on a dedicated connection, kept separate from the connection pool used for every other command. Because of this, you can publish and run other commands on the *same* client while a subscription is active, the client manages the two connections for you.

### Cluster deployments

Valkey scales horizontally with a cluster deployment. The key namespaces is partitioned into 16384 hash slots and then this is shared across multiple shards. Each shard will consist of a primary node plus an array of replica nodes. The valkey-swift `ValkeyClusterClient` will automatically choose the correct node to communicate with based on the key being accessed. It will dynamically update its representation of the cluster topology in the background and handle redirect messages returned from the cluster.

```swift
let clusterClient = ValkeyClusterClient(
    nodeDiscovery: ValkeyStaticNodeDiscovery([.init(endpoint: "node1.example.com", port: 6379)]),
    configuration: configuration,
    logger: logger
)
```

With the cluster client you shouldn't need to know you are running a cluster, so the API is the same as if you are running the standalone client.

```swift
try await clusterClient.set(key: "user:123", value: "alice")
let value = try await clusterClient.get(key: "user:123")
```

The cluster client also supports pipelining, transactions and subscriptions and works in the same way as the standalone client. The only difference is that each command sent to the server can only access keys from the same hash slot. 

### Redis compatibility

As Valkey is a fork of Redis v7.2.4, valkey-swift is compatible with Redis databases up to then. The v7.2.4 features should be compatible in later versions of Redis, but these are now considered two different projects and they may diverge. Valkey-swift uses the RESP3 protocol.

## Maturity Justification

I am proposing valkey-swift as `incubated`, as it has multiple maintainers and is actively being used.

## Alternatives considered

The main alternative RediStack has been deprecated and will not receive anymore updates.
