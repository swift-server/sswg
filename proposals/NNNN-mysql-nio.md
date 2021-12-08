# MySQLNIO: An NIO-based MySQL Driver

* Proposal: [SSWG-NNNN](https://github.com/swift-server/sswg/tree/mysql-nio-proposal/proposals/NNNN-mysql-nio.md)
* Author(s): [Gwynne Raskind](https://github.com/gwynne)
* Review Manager: TBD
* Status: **Implemented**
* Implementation: [vapor/mysql-nio](https://github.com/vapor/mysql-nio)
* Forum Threads: Pitch (Pending), Discussion (Pending)

## Introduction

`MySQLNIO` is a client package for connecting to, authorizing, and querying a MySQL server. At the heart of this module are channel handlers for parsing and serializing messages in MySQL's proprietary wire protocol. These channel handlers are combined in a request / response style connection type that provides a convenient, client-like interface for performing queries. Support for both simple (text) and parameterized (binary) querying is provided out of the box alongside a `MySQLData` type that handles conversion between MySQL's wire format and native Swift types. 

## Motiviation

Most Swift implementations of MySQL clients are based on the [libmysqlclient](https://dev.mysql.com/doc/c-api/8.0/en/) C library which handles transport internally. Building a library directly on top of MySQL's wire protocol using SwiftNIO should yield a more reliable, maintainable, and performant interface for MySQL databases. MySQLNIO also supports both MySQL 5.7 and MySQL 8.x - as well as several versions of MariaDB and Percona Server - out of the box. Conversely, `libmysqlclient` requires you to install the correct version of the client to match your version of the server to be compatible. For example, the 5.7 client cannot connect to a default-configured 8.0 server, and the 8.0 version is unreliable when used with a 5.7 server.

## Dependencies

This package has four dependencies:

- `swift-nio` from `2.0.0`
- `swift-nio-ssl` from `2.14.0`
- `swift-log` from `1.0.0`
- `swift-crypto` from `1.0.0 ..< 3.0.0`

This package has no additional system dependencies.

## Proposed Solution

This section goes into detail on a few distinct types from this module to give an idea of how they work together and what using the package looks like.

### MySQLConnection

The base connection type, `MySQLConnection`, is a wrapper around NIO's `ClientBootstrap` that initializes the pipeline to communicate via MySQL packets using a request / response pattern. 

```swift
import MySQLNIO

// create a new event loop group
let elg = MultiThreadedEventLoopGroup(numberOfThreads: 1)
defer { try! elg.syncShutdownGracefully() }

// create a new connection and authenticate using credentials
let address = try SocketAddress(ipAddress: "127.0.0.1", port: 5432)
let conn = try await MySQLConnection.connect(
    to: address, 
    username: "username",
    password: "password",
    // optionally configure TLS
    tlsConfiguration: .makeClientConfiguration(), 
    serverHostname: "127.0.0.1",
    on: elg.next()
)
defer { try! conn.close().wait() } // `await` is not available in `defer`

// ready to query
print(conn) // MySQLConnection
```

#### Closing

A connection _must_ be closed before it deinitializes. `MySQLConnection` ensures this by asserting that it has been closed in its `deinit` handler. This is meant to help developers implement proper graceful shutdown early and avoid leaking memory or sockets. 

### Simple Query

Assuming we have an active, authenticated `MySQLConnection`, we can query the connected server using MySQL's simply query command:

```swift
import MySQLNIO

let conn = try await MySQLConnection.connect(...)
defer { try! conn.close().wait() }

// select the current version
let rows = try await conn.simpleQuery("SELECT VERSION() AS version")
print(rows) // [MySQLRow]

// fetch the version column from the first row by attempting to
// read it as a Swift string
let version = rows.first?.column("version")?.string
print(version) // String?
```

This format does not support parameterizing input and returns all data in string format. To provide input values, they must be directly inserted into the query:

```swift
try await conn.simpleQuery("SELECT * FROM planets WHERE name = 'Earth'")
```

### Query

We can also perform parameterized queries with an active `MySQLConnection`. These queries support binding input parameters and return data in a more compact binary format.

Input parameters are passed as an array of `MySQLData` following the SQL string. In the query string, input parameters are referenced by the `?` placeholder marker:

```swift
import MySQLNIO

let conn = try await MySQLConnection.connect(...)
defer { try! conn.close().wait() }

// selects all planets where name is equal to the first bound parameter
let rows = try await conn.query("SELECT * FROM planets WHERE name=?", ["Earth"])

// fetch the "name" column from the first row as a string
let foo = rows.first?.column("name")?.string
print(foo) // Optional("Earth")
```

### MySQLData

`MySQLData` represents data both going to and coming from MySQL.

#### Input

An array of `MySQLData` is supplied alongside parameterized queries, one for each parameter. There are many initializers for creating `MySQLData` instances from Swift's standard types. For example:

```swift
import MySQLNIO

let string = MySQLData(string: "Hello")
let double = MySQLData(double: 3.14)
let date = MySQLData(date: Date(timeIntervalSince1970: 42))
```

`MySQLData` also conforms to Swift's `ExpressibleBy...` protocols, allowing various kinds of literals to be used directly:

```swift
import MySQLNIO

let inputs: [MySQLData] = ["hello", 3]
```

#### Output

Likewise, `MySQLData` can be converted back to Swift types. This is useful for converting data returned by MySQL queries into meaningful types. There are many methods for Swift's standard types; for example:

```swift
import MySQLNIO

let data: MySQLData
print(data.string) // String?
```

The full list of supported types at the time of this writing is:

- `Swift.String`
- `Swift.Int`
- `Swift.Int64`
- `Swift.Int32`
- `Swift.Int16`
- `Swift.Int8`
- `Swift.UInt`
- `Swift.UInt64`
- `Swift.UInt32`
- `Swift.UInt16`
- `Swift.UInt8`
- `Swift.Float`
- `Swift.Double`
- `Swift.Bool`
- `Foundation.Decimal`
- `Foundation.Date`
- `Foundation.Data`
- `Foundation.UUID`
- `MySQLNIO.MySQLTime`
- Any `Swift.Encodable` value (automatically encoded as JSON)

### MySQLRow

Both `simpleQuery()` and `query()` return an array of `MySQLRow`. Each row can be thought of as a dictionary with column names as the key and data as the value. While the actual storage implementation is private, `MySQLRow` provides the `column(_:)` method for accessing column data:

```swift
struct MySQLRow {
	func column(_ name: String, table: String? = nil) -> MySQLData?
}
```

If no column with the given name exists in the row, `nil` is returned. If a table name is provided, the column must have been selected from the table with that name to be returned. If no table name is provided (the default), the first matching column in the row from _any_ table is returned.

### MySQLError

The `MySQLError` type represents errors thrown from both the MySQL package itself (during encoding, for example) and errors returned by the server:

```swift
public enum MySQLError: Error {
    case secureConnectionRequired
    case unsupportedAuthPlugin(name: String)
    case authPluginDataError(name: String)
    case missingOrInvalidAuthMoreDataStatusTag
    case missingOrInvalidAuthPluginInlineCommand(command: UInt8?)
    case missingAuthPluginInlineData
    case unsupportedServer(message: String)
    case protocolError
    case server(MySQLProtocol.ERR_Packet)
    case closed
    case duplicateEntry(String)
    case invalidSyntax(String)
}
```

### MySQLDatabase

While `MySQLConnection` is the primary type used for connecting, authorization, and TLS negotation, the `MySQLDatabase` protocol provides an abstract interface for issuing commands to the MySQL server and explicitly requesting a dedicated connection (intended for use cases where connection pooling must be avoided, such as leveraging transactions). `MySQLConnection` conforms to this protocol:

```swift
protocol MySQLDatabase {
    var eventLoop: EventLoop { get }
    var logger: Logger { get }
    func send(_ command: MySQLCommand, logger: Logger) -> EventLoopFuture<Void>
    func withConnection<T>(_ closure: @escaping (MySQLConnection) -> EventLoopFuture<T>) -> EventLoopFuture<T>
}
```

It is expected that consumers of this package will adopt the `MySQLDatabase` protocol to provide, for example, automatic connection pooling. (See Vapor's [MySQLKit](https://github.com/vapor/mysql-kit) for an example of such an implementation.)

The `send(_:logger:)` method communicates with the server to perform the operation implemented by the provided `MySQLCommand` instance. `MySQLNIO` users should not call this method directly; it is an unfortunately exposed implementation detail which will be removed in a future revision of the API.

The `withConnection(_:)` method calls the provided closure with an instance of `MySQLConnection`. This instance is guaranteed, by the fact that is specifically a "connection" and not a generic "database", to represent a single, active database connection. This method exists so that users can require any `MySQLDatabase` to provide a singular connection, guaranteed to be the same connection (i.e., not newly opened or from a pool) for the duration of the closure's scope. Normally, a connection pool implementation of `MySQLDatabase` would be free to, for example, send every query on a different underlying connection; this behavior is incompatible with database-level transactions and other uses of per-connection state (such as MySQL's local variables).

The following additional methods are also available on `MySQLDatabase`:

```swift
extension MySQLDatabase {
    public func query(
        _ sql: String,
        _ binds: [MySQLData] = [],
        onMetadata: @escaping (MySQLQueryMetadata) throws -> () = { _ in }
    ) -> EventLoopFuture<[MySQLRow]>
    
    public func query(
        _ sql: String,
        _ binds: [MySQLData] = [],
        onRow: @escaping (MySQLRow) throws -> (),
        onMetadata: @escaping (MySQLQueryMetadata) throws -> () = { _ in }
    ) -> EventLoopFuture<Void>

    public func simpleQuery(_ sql: String) -> EventLoopFuture<[MySQLRow]>

    public func simpleQuery(_ sql: String, onRow: @escaping (MySQLRow) -> ()) -> EventLoopFuture<Void>

    public func logging(to logger: Logger) -> MySQLDatabase
}
```

The `query()` and `simpleQuery()` methods call the base protocol's `send(_:logger:)` method; these are the only `MySQLCommand` types currently implemented at the time of this writing, and it should never be necessary to call `send(_:logger:)` directly.

The `logging(to:)` method returns a new "database" which uses the specified logger for all operations. The returned instance is a simple wrapper around the original, and the original remains usable, though calling methods on the original will not use the new logger. The wrapper works with any `MySQLDatabase`, regardless of type.

Finally, Concurrency-based convenience methods are also available:

```swift
extension MySQLDatabase {
    public func send(
        _ command: MySQLCommand,
        logger: Logger
    ) async throws -> Void
    
    public func withConnection<T>(
        _ closure: @escaping @Sendable (MySQLConnection) async throws -> T
    ) async throws -> T

    public func query(
        _ sql: String,
        _ binds: [MySQLData] = [],
        onMetadata: @escaping (MySQLQueryMetadata) throws -> Void = { _ in }
    ) async throws -> [MySQLRow]
    
    public func simpleQuery(_ sql: String) async throws -> [MySQLRow]
}
```

#### Note on usage

By design, most of `MySQLNIO`'s useful methods are made available on `MySQLDatabase`, rather than only on `MySQLConnection`. It is intended that the `MySQLDatabase` type be used for all operations not directly related to opening and closing connections. For example, in a theoretical controller:

```swift
final class UserController: Controller {
    let db: MySQLDatabase
    
    init(db: MySQLDatabase) { 
        self.db = db
    }

    func names(_ req: HTTPRequest) async throws -> [String] {
        return try await self.db.query("SELECT name FROM users").compactMap { 
            $0.column("name")?.string
        }
    }
}
```

Because this controller uses `MySQLDatabase`, any of the following could be supplied to it:

- A connected `MySQLConnection`
- A pool of `MySQLConnection`s
- A mock database for testing purposes

#### `MySQLCommand`

MySQL's wire protocol follows a stateful command/reply pattern, where a single command can involve anything from a simple request and reply, to an entire "secondary" protocol specific to the command in question. `MySQLCommand` provides an abstraction over the common behaviors defined by the underlying wire protocol, allowing a command to be defined in terms of a packet handler and a "state" encapsulating the various kinds of possible responses:

```swift
public protocol MySQLCommand {
    func handle(
        packet: inout MySQLPacket,
        capabilities: MySQLProtocol.CapabilityFlags
    ) throws -> MySQLCommandState
    
    func activate(capabilities: MySQLProtocol.CapabilityFlags) throws -> MySQLCommandState
}

public struct MySQLCommandState {
    static var noResponse: MySQLCommandState
    static var done: MySQLCommandState
    static func response(_ packets: [MySQLPacket]) -> MySQLCommandState

    public init(
        response: [MySQLPacket] = [],
        done: Bool = false,
        resetSequence: Bool = false,
        error: Error? = nil
    )
}
```

A `MySQLCommand` is responsible for sending the initial command message and handling the server's responses. The command returns the `.done` command state when processing is completed, causing the client's send future to be fulfilled.

### Cryptographic requirements

The `mysql_native_password` authentication mechanism (the default up through MySQL 5.7, available by configuration in MySQL 8.0+) requires the use of SHA-1 hashing. The SHA-1 digest implementation is provided by the SwiftCrypto package.

The `caching_sha2_password` mechanism (the default starting in MySQL 8.0) requires SHA-256 hashing in all cases. SwiftCrypto also provides this implementation. If the connection to the server is unencrypted (e.g. not over TLS), an RSA encryption operation must also be performed to avoid sending credentials in plaintext. No version of MySQL offers an alternative encryption for this purpose. Once again, the implementation in SwiftCrypto is relied upon.

### Todo / Discussion

Here are some things that are still a work in progress:

- **Reuse of prepared statements**: In addition to providing binary data formats and safe parameterization for bound values in queries, MySQL's prepared statements are intended to speed up repetitions of the same query. This should ideally be leveraged.

### How to use

To try this package out at home, add the following dependency to your `Package.swift` file:

```swift
.package(url: "https://github.com/vapor/mysql-nio.git", from: "1.0.0"),
```

Then add `.product(name: "MySQLNIO", package: "mysql-nio")` to your module target's dependencies array.

### Seeking Feedback

* If anything, what does this proposal *not cover* that you will definitely need?
* If anything, what could we remove from this and still be happy?
* API-wise: what do you like, what don't you like?

Feel free to post feedback as response to this post and/or GitHub issues on [vapor/mysql-nio](https://github.com/vapor/mysql-nio).
