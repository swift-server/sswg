# MongoKitten

* Proposal: [SSWG-0027](0027-mongokitten.md)
* Authors: [Joannis Orlandos](https://github.com/joannis)
* Review Manager: [Sven A. Schmidt](https://github.com/finestructure)
* Status: **Implemented**
* Implementation: [MongoKitten](https://github.com/orlandos-nl/MongoKitten)
* Forum Threads: None

## Package Description
A MongoDB client library for Swift that embraces Codable and Swift's modern concurrency standards.

|  |  |
|--|--|
| **Package Name** | `MongoKitten` |
| **Module Name** | `MongoKitten` |
| **Proposed Maturity Level** | [Sandbox or Incubating](https://www.swift.org/sswg/incubation-process.html#process-diagram) |
| **License** | [MIT](https://github.com/orlandos-nl/MongoKitten/blob/main/LICENSE.md) |
| **Dependencies** | [BSON 8.x](https://github.com/orlandos-nl/BSON), [swift-nio 2.43+](https://github.com/apple/swift-nio), [swift-log 1.0](https://github.com/apple/swift-log), [swift-metrics 1.0 or 2.0](https://github.com/apple/swift-metrics), [swift-atomics 1.0](https://github.com/apple/swift-atomics), [DNSClient 2.2](https://github.com/orlandos-nl/DNSClient), [swift-nio-ssl 2.x](https://github.com/apple/swift-nio-ssl) |

## Introduction

MongoDB is a popular NoSQL database management system that uses a document-oriented data model.
MongoKitten is a MongoDB client library that supports all non-enterprise features. It embraces Codable through [BSON](https://github.com/orlandos-nl/BSON)'s encoders and decoders, and vendors exclusively async/await based APIs.

## Motivation

MongoKitten is a longstanding library for the Swift ecosystem, and has been in development since 2015. MongoDB has also created another database driver, which vendors a Swift API wrapping their internal C implementation. However, this driver is a relative newcomer to the ecosystem and has been discontinued since.

Providing a solution for MongoDB is essential for the Swift ecosystem as it serves a significant portion of the database market.

## Proposed solution

MongoKitten is layered in multiple modules.

**MongoCore** contains a basic implementation of the MongoDB wire protocol.

**MongoClient** contains a MongoDB client based on this protocol. It contains all the tools a user needs to communicate with MongoDB. It has the ability to send & receive messages and authenticate. In addition, MongoClient also has helpers for sending/reading messages sending and receiving Codable conforming messages on such a connection. Finally, MongoClient has a MongoDB Cluster client implementation, functioning as a connection pool, while also observing changes in cluster topology.

**MongoKittenCore** is a module containing the most commonly used messages implemented as Codable types.

The **MongoKitten** module itself imports all of the above, and provides a higher level API for interacting with MongoDB. This is the library that most users end up interacting with.

Finally, the **Meow** module vendors ORM-like helpers for quickly storing and reading entities from MongoDB, by conforming your Codable type to the `Model` protocol.


## Detailed design

The MongoKitten client can be initialized through either the `MongoDatabase` or `MongoCluster` objects.

MongoCluster is the object that is able to observe changes in cluster topology, and is MongoKitten's driver. MongoDB allows a single connection to query multiple databases on the same database server. Since most users will often only need access to one database on the server at a time, the `MongoDatabase` object provides helper APIs to initialize a connection to a server and _then_ access a specific database.

```swift
let mongodb = try await MongoDatabase.connect(to: connectionString)
```

From here, users can subscript the database to access a collection. MongoDB Collections do not need to be create before being accessed, nor does MongoDB require users to specify a schema.
The operation of 'accessing' a collection is purely a form of API design.

```swift
let users = mongodb["users"]
```

Users can insert entities through a MongoDB BSON `Document` type, a binary equivalent to JSON that supports more data types.

```swift
try await users.insert([
  "_id": ObjectId(),
  "username": "Joannis",
  "likesSwift": true
])
```

APIs that support Document also support Codable variants:

```swift
let user = User(
  username: "Joanis",
  likesSwift: true
)

try await users.insertEncoded(user)
```

### Reading Results

When creating a **find** query using the `find()` function, users can provide MongoKitten with a query formed as a Document. Omitting this argument queries _all_ results, and is equivalent to SQL's `SELECT * FROM users`.

```swift
let results = users.find()
```

Note that `find()` is not marked with `try await`. This is because `find()` uses a _builder pattern_, allowing users to chain modifiers to it such as `limit` and `skip`.

```swift
let usersThirdPage = users.find()
  .skip(100)
  .limit(50)
```

Such builder objects exist for all cursor-based results, and will send the query as results are requested. Cursors conform to `AsyncSequence`, allowing users to iterate results in a for-loop.

```swift
for try await user in users.find() {
  // Use the `user` Document
}
```

Like other APIs, cursors also have a helper for decoding results:

```swift
for try await user in users.find().decode(User.self) {
  // `user` is now an instance of `User`
}
```

Should custom behaviour be preferred, users can implement their own lazy mutations of this sequence using the `cursor.map` helper.

### Aggregates

MongoDB provides a powerful system for processing data using more complex queries through **aggregates**. This includes GeoJSON queries, Text Search, Graph Queries, Left Joins and many more features uniquely present in aggregates.

Aggregates are designes a pipeline, whose input is a document from the collection queries. Each stage is triggered per inputted entity, which can then transform, add or omit results. For example, a **Match** stage will filter included entities, omitting results that do not meet the requested conditions.

MongoKitten uses a result builder for designing these aggregate pipelines, making it easy for users to review their pipelines.

```swift
// Find kids to invite to my party
let partygoers = mongodb["kids"].buildAggregate {
  Match([
    "age": [
      "$lte": 10
    ]
  ])
  
  // only 15 friends are invited to the party
  Limit(15)
  
  // Siblings of friends are also invited
  Lookup(
    from: "kids",
    localField: "siblings",
    foreignField: "_id",
    pipelinke: {
      // Siblings may not be too old either
      Match([
        "age": [
          "$lte": 10
        ]
      ])
    },
    as: "invitedSiblings"
  )
}
```

## Maturity Justification

Sandbox or Incubating. MongoKitten adheres to the minimal requirements of the [SSWG Incubation Process](https://github.com/swift-server/sswg/blob/master/process/incubation.md#minimal-requirements). It has a [second contributor](https://github.com/obbut) historically and currently involved with the library, however, the second contributor is not _publicly_ involved through contributing commits. Therefore it's unclear which is more suitable at this time.

## Alternatives considered

The only alternative to MongoKitten during development was [SwiftMongoDB](https://github.com/Danappelxx/SwiftMongoDB), which was in very early stages and eventually discontinued as MongoKitten was further established.

At the time of proposing, MongoDB has released an [official alternative](https://github.com/mongodb/mongo-swift-driver) which has since been [discontinued](https://www.mongodb.com/community/forums/t/updates-on-mongodb-swift-driver/240047).
