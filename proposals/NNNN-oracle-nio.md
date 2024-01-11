# Solution name

* Proposal: [SSWG-NNNN](NNNN-oracle-nio.md)
* Authors: [Timo Zacherl](https://github.com/lovetodream)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [oracle-nio](https://github.com/lovetodream/oracle-nio)
* Forum Threads: [Pitch](https://forums.swift.org/t/pitch-oraclenio-oracle-db-driver-built-on-swiftnio/69088)

## Package Description
A client library for Oracle databases based on swift-nio. It does neither require OCI nor ODPI.

|  |  |
|--|--|
| **Package Name** | `oracle-nio` |
| **Module Name** | `OracleNIO` |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [Apache 2.0](https://github.com/lovetodream/oracle-nio/blob/main/LICENSE) |
| **Dependencies** | [swift-log 1.4.4+](https://github.com/apple/swift-log), [swift-nio 2.58.0+](https://github.com/apple/swift-nio), [swift-nio-transport-services](https://github.com/apple/swift-nio-transport-services), [swift-nio-ssl 2.25.0+](https://github.com/apple/swift-nio-ssl), [swift-crypto 3.2.0+](https://github.com/apple/swift-crypto), [postgres-nio 1.20.0](https://github.com/vapor/postgres-nio) |

## Introduction

Oracle DB is a popular relational database management system by the Oracle Corporation. 
OracleNIO is a client library that allows users to connect, authorize and query these Oracle database servers, by using Swift's latest concurrency features. 
It uses Oracle's proprietary **T**ransparent **N**etwork **S**ubstrate (TNS) protocol to communicate with the database peer.

## Motivation

Previously it was only possible to integrate with Oracle databases by wrapping C libraries, which required additional system dependencies such as the Oracle Call Interface (OCI) and/or others. 
The setup and usage was not ideal and error prone.

Providing a driver for Oracle databases is a great addition to the Swift on the Server ecosystem as it is a very popular database.

## Proposed solution

OracleNIO is a standalone, first-class Swift driver for Oracle databases. 
It supports all modern Oracle databases, starting with 12.1.
The library allows user's to use Oracle databases in their Swift projects without requiring additional setup.

## Detailed design

There are two ways of connecting to an Oracle database, namely [`OracleConnection`](https://swiftpackageindex.com/lovetodream/oracle-nio/main/documentation/oraclenio/oracleconnection) or [`OracleClient`](https://swiftpackageindex.com/lovetodream/oracle-nio/main/documentation/oraclenio/oracleclient). 
`OracleConnection` instanciates a single connection to the database, whereas `OracleClient` creates a pool of connections.
Both take a [`OracleConnection.Configuration`](https://swiftpackageindex.com/lovetodream/oracle-nio/main/documentation/oraclenio/oracleconnection/configuration) parameter to configure the individual connections.

```swift
let logger = Logger(label: "my-oracle-logger")

let config = OracleConnection.Configuration(
    host: "127.0.0.1", 
    port: 1521,
    service: .serviceName("my_service"),
    username: "my_username",
    password: "my_password"
)

let connection = try await OracleConnection.connect(
  configuration: config,
  id: 1,
  logger: logger
)
```

With the now successfully established connection, one can start to query the database using the [`query(_:)`](https://swiftpackageindex.com/lovetodream/oracle-nio/main/documentation/oraclenio/oracleconnection/query(_:options:logger:file:line:)) method. 
This method allows the user to run a Query, DML, DDL or [PL/SQL](https://www.oracle.com/database/technologies/appdev/plsql.html).

```swift
let rows = try await connection.query("SELECT id, username, birthday FROM users")
```

A query returns an [`OracleRowSequence`](https://swiftpackageindex.com/lovetodream/oracle-nio/main/documentation/oraclenio/oraclerowsequence), which is an AsyncSequence of [`OracleRow`](https://swiftpackageindex.com/lovetodream/oracle-nio/main/documentation/oraclenio/oraclerow)s. The rows can either be collected altogether or iterated one-by-one.
It's possible to decode rows to a set of Swift types directly. 

```swift
for try await (id, username, birthday) in rows.decode((Int, String, Date).self) {
  // do something with the datatypes.
}
```

A type must implement the [`OracleDecodable`](https://swiftpackageindex.com/lovetodream/oracle-nio/main/documentation/oraclenio/oracledecodable) protocol in order to be decoded from a row. 
OracleNIO provides default implementations for most of Swift's builtin types, as well as some types provided by Foundation. 
Users of the library can implement their own `OracleDecodable` types, too.

Once done, the database has to be closed.

```swift
try await connection.close()
```

An up-to-date documentation of the public API can be found on the [Swift Package Index](https://swiftpackageindex.com/lovetodream/oracle-nio/main/documentation/oraclenio).

It should be noted that the public API and many of the internals are very similar to [PostgresNIO](https://github.com/vapor/postgres-nio), which was the main inspiration in the process of creating OracleNIO. This should make adoption of OracleNIO easier to those already familiar with PostgresNIO.

### Dependency changes since the Pitch

I stated in the [pitch](https://forums.swift.org/t/pitch-oraclenio-oracle-db-driver-built-on-swiftnio/69088), that I'd like to drop the dependency on CryptoSwift. 
I've already done so by using the newly added no-padding variant of CBC in swift-crypto. 
And implementing an internal `_PKBDF2` library as part of OracleNIO. 
This internal library is planned to be superseded by swift-crypto's implementation, if it will be added in the future.

## Maturity Justification

OracleNIO adheres to the minimal requirements of the SSWG Incubation Process, therefor the Sandbox maturity level should be sufficient.

## Alternatives considered

I don't know about any actively maintained alternatives to OracleNIO. 

There are various seemingly unmaintained forks of `SwiftOracle`, (the most promising one seemed to be [this once](https://github.com/iliasaz/SwiftOracle)) in existance, but they aren't as confinient to use as OracleNIO and they all require C dependencies.
