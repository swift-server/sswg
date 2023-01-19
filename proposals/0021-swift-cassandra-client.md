# Swift Cassandra Client

* Proposal: [SSWG-0021](0021-swift-cassandra-client.md)
* Authors: [Yim Lee](https://github.com/yim-lee)
* Review Manager: [Konrad 'ktoso' Malawski](https://github.com/ktoso)
* Status: **Implemented**
* Implementation: [swift-cassandra-client](https://github.com/apple/swift-cassandra-client)
* Forum Threads: 
    * [Pitch thread](https://forums.swift.org/t/pitch-swift-cassandra-client/62509) 

## Package Description

The Swift Cassandra Client package is a Cassandra database client based on [DataStax Cassandra C++ Driver](https://github.com/datastax/cpp-driver), wrapping it with Swift-friendly APIs and data structures.

It offers API based on [SwiftNIO](https://github.com/apple/swift-nio) futures, as well as Swift Concurrency based API in Swift 5.5 and newer.

|  |  |
|--|--|
| **Package Name** | `swift-cassandra-client` |
| **Module Name** | `CassandraClient` |
| **Proposed Maturity Level** | [Incubating](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [Apache 2.0](https://github.com/swift-server/swift-cassandra-client/blob/master/LICENSE.txt) |
| **Dependencies** | [swift-nio](https://github.com/apple/swift-nio), [swift-log](https://github.com/apple/swift-log) |



## Motivation

Apple is an active contributor to the Apache Cassandra open source project and server teams at Apple use Cassandra to support various use-cases. The Swift Cassandra Client has been used in that capacity for several years and we believe the broader Swift community would find it useful as well.

## Usage

### Swift concurrency based API

#### Creating a client instance

```swift
var configuration = CassandraClient.Configuration(...)
let cassandraClient = CassandraClient(configuration: configuration)
```

The client has a default session established (lazily) so that it can be used directly to perform
queries on the configured keyspace:

```swift
let result = try await cassandraClient.query(...)
```

The client must be explicitly shut down when no longer needed:

```swift
try cassandraClient.shutdown()
```

#### Creating a session for a different keyspace

```swift
let session = cassandraClient.makeSession(keyspace: <KEYSPACE>)
let result = try await session.query(...)
```

The session must be explicitly shut down when no longer needed:

```swift
try session.shutdown()
```

You can also create a session and pass in a closure, which will automatically release the resource when the closure exits:

```swift
try await cassandraClient.withSession(keyspace: <KEYSPACE>) { session in
  ...
}
```

#### Running result-less commands (e.g. insert, update, delete or DDL)

```swift
try await cassandraClient.run("create table ...")
```

Or at session level:

```swift
try await session.run("create table ...")
```

#### Running queries returning small datasets that fit in memory

Returning a model object, having `Model: Codable`:

```swift
let result: [Model] = try await cassandraClient.query("select * from table ...")
```

```swift
let result: [Model] = try await session.query("select * from table ...")
```

Or using free-form transformations on the row:

```swift
let values = try await cassandraClient.query("select * from table ...") { row in
  row.column(<COLUMN_NAME>).int32
}
```

```swift
let values = try await session.query("select * from table ...") { row in
  row.column(<COLUMN_NAME>).int32
}
```

#### Running queries returning large datasets that do not fit in memory

```swift
// `rows` is a sequence that one needs to iterate on
let rows: Rows = try await cassandraClient.query("select * from table ...")
```

```swift
// `rows` is a sequence that one needs to iterate on
let rows: Rows = try await session.query("select * from table ...")
```

### SwiftNIO future based API

#### Creating a client instance

```swift
var configuration = CassandraClient.Configuration(...)
let cassandraClient = CassandraClient(configuration: configuration)
```

The client has a default session established (lazily) so that it can be used directly to perform
queries on the configured keyspace:

```swift
let resultFuture = cassandraClient.query(...)
```

The client must be explicitly shut down when no longer needed:

```swift
try cassandraClient.shutdown()
```

#### Creating a session for a different keyspace

```swift
let session = cassandraClient.makeSession(keyspace: <KEYSPACE>)
let resultFuture = session.query(...)
```

The session must be explicitly shut down when no longer needed:

```swift
try session.shutdown()
```

You can also create a session and pass in a closure, which will automatically release the resource when the closure exits:

```swift
try cassandraClient.withSession(keyspace: <KEYSPACE>) { session in
  ...
}
```

#### Running result-less commands (e.g. insert, update, delete or DDL)

```swift
let voidFuture = cassandraClient.run("create table ...")
```

Or at session level:

```swift
let voidFuture = session.run("create table ...")
```

#### Running queries returning small datasets that fit in memory

Returning a model object, having `Model: Codable`:

```swift
cassandraClient.query("select * from table ...").map { (result: [Model]) in
  ...
}
```

```swift
session.query("select * from table ...").map { (result: [Model]) in
  ...
}
```

Or using free-form transformations on the row:

```swift
cassandraClient.query("select * from table ...") { row in
  row.column(<COLUMN_NAME>).int32
}.map { value in
  ...
}
```

```swift
session.query("select * from table ...") { row in
  row.column(<COLUMN_NAME>).int32
}.map { value in
  ...
}
```

#### Running queries returning large datasets that do not fit in memory

```swift
cassandraClient.query("select * from table ...").map { (rows: Rows) in
  // `rows` is a sequence that one needs to iterate on
  rows.map { row in
    ...
  }
}
```

```swift
session.query("select * from table ...").map { (rows: Rows) in
  // `rows` is a sequence that one needs to iterate on
  rows.map { row in
    ...
  }
}
```

## Maturity justification

As the library has been in use at Apple for several years, and we would like more time and exposure to collect user feedback to move towards its 1.0 release, we propose to include Swift Cassandra Client in **Incubating** level.
