# NIOPostgres: A NIO-based PostgreSQL Driver

> **Note**: Since this proposal was published, `NIOPostgres` has been renamed to `PostgresNIO`.
> 
> For more context behind this change, see the following threads: [[1]](https://forums.swift.org/t/namespacing-of-packages-modules-especially-regarding-swiftnio/24726), [[2]](https://forums.swift.org/t/sswg-minimum-requirements-to-require-no-existing-clashes/24932)

* Proposal: [SSWG-0003](https://github.com/swift-server/sswg/tree/master/proposals/SSWG-0003.md)
* Author(s): [Tanner Nelson](https://github.com/tanner0101)
* Review Manager: TBD
* Status: **Accepted as Sandbox Maturity**
* Implementation: [vapor/nio-postgres](https://github.com/vapor/nio-postgres)
* Forum Threads: [Pitch](https://forums.swift.org/t/pitch-swiftnio-based-postgresql-client/18020), [Discussion](https://forums.swift.org/t/discussion-niopostgres-a-nio-based-postgresql-driver/22178), [Review](https://forums.swift.org/t/feedback-niopostgres-a-nio-based-postgresql-driver/24398)
* Decision Notes: [Rationale](https://forums.swift.org/t/may-16th-2019/25036)

## Introduction

`NIOPostgres` is a client package for connecting to, authorizing, and querying a PostgreSQL server. At the heart of this module are channel handlers for parsing and serializing messages in PostgreSQL's proprietary wire protocol. These channel handlers are combined in a request / response style connection type that provides a convenient, client-like interface for performing queries. Support for both simple (text) and parameterized (binary) querying is provided out of the box alongside a `PostgresData` type that handles conversion between PostgreSQL's wire format and native Swift types. 

## Motiviation

Most Swift implementations of Postgres clients are based on the [libpq](https://www.postgresql.org/docs/11/libpq.html) C library which handles transport internally. Building a library directly on top of Postgres' wire protocol using SwiftNIO should yield a more reliable, maintainable, and performant interface for PostgreSQL databases.

## Dependencies

This package has four dependencies:

- `swift-nio` from `2.0.0`
- `swift-nio-ssl` from `2.0.0`
- `swift-log` from `1.0.0`
- `swift-metrics` from `1.0.0`

This package has no additional system dependencies.

## Proposed Solution

This section goes into detail on a few distinct types from this module to give an idea of how they work together and what using the package looks like.

### PostgresConnection

The base connection type, `PostgresConnection`, is a wrapper around NIO's `ClientBootstrap` that initializes the pipeline to communicate via Postgres messages using a request / response pattern. 

```swift
import NIOPostgres

// create a new event loop group
let elg = MultiThreadedEventLoopGroup(numberOfThreads: 1)
defer { try! elg.syncShutdownGracefully() }

// create a new connection
let address = try SocketAddress(ipAddress: "127.0.0.1", port: 5432)
let conn = try PostgresConnection.connect(
    to: address, 
    // optionally configure TLS
    tlsConfiguration: .forClient(certificateVerification: .none), 
    serverHostname: "127.0.0.1"
    on: elg.eventLoop
).wait()
defer { try! conn.close().wait() }

// authenticate the connection using credentials
try conn.authenticate(username: "username", database: "database", password: "password").wait()

// ready to query
print(conn) // PostgresConnection
```

#### Closing

A connection _must_ be closed before it deinitializes. `PostgresConnection` ensures this by asserting that it has been closed in its `deinit` handler. This is meant to help developers implement proper graceful shutdown early and avoid leaking memory or sockets. 

### Simple Query

Assuming we have an active, authenticated `PostgresConnection`, we can query the connected server using PostgreSQL's simple, text format. 

```swift
import NIOPostgres

let conn: PostgresConnection ...

// select the current version
let rows = try conn.simpleQuery("SELECT version()").wait()
print(rows) // [PostgresRow]

// fetch the version column from the first row casting it to a Swift string
let version = rows[0].column("version")?.string
print(version) // String?
```

This format does not support parameterizing input and returns all data in string format. To bind values, insert them into the string:

```swift
try conn.simpleQuery("SELECT * FROM planets WHERE name = 'Earth'")
```

### Query

We can also perform parameterized queries with an active `PostgresConnection`. These queries support binding input parameters and return data in a more compact binary format.

Input parameters are passed as an array of `PostgresData` following the SQL string. In the query string, input parameters are referenced by incrementing placeholders, starting with `$1`.

```swift
import NIOPostgres

let conn: PostgresConnection ...

// selects all planets where name is equal to the first bound parameter
let rows = try conn.query("SELECT * FROM planets WHERE name = $1", ["Earth"]).wait()

// fetch the "name" column from the first row, casting it to a string
let foo = rows[0].column("name")?.string
print(foo) // "Earth"
```

### PostgresData

`PostgresData` represents data both going to and coming from Postgres. 

#### Input

An array of `PostgresData` is supplied alongside parameterized queries, one for each parameter. There are many initializers for creating `PostgresData` from Swift's standard types. For example:

```swift
import NIOPostgres

let string = PostgresData(string: "Hello")
let double = PostgresData(double: 3.14)
let date = PostgresData(date: Date(timeIntervalSince1970: 42))
```

`PostgresData` also conforms to Swift's `Expressible...` protocols, allowing for conversion between Swift literals.

```swift
import NIOPostgres

let inputs: [PostgresData] = ["hello", 3.14]
```

#### Output

Likewise, `PostgresData` can be converted back to Swift types. This is useful for converting data returned by Postgres queries into meaningful types. There are many methods for Swift's standard types, for example:

```swift
import NIOPostgres

let data: PostgresData
print(data.string) // String?
```

Here is a full list of types supported currently:

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
- `Foundation.Date`
- `Foundation.Data`
- `Foundation.UUID`

### PostgresRow

Both `simpleQuery` and `query` return an array of `PostgresRow`. Each row can be thought of as a dictionary with column names as the key and data as the value. While the actual storage implementation is private, `PostgresRow` gives the following method for accessing column data:

```swift
struct PostgresRow {
	func column(_ column: String) -> PostgresData?
}
```

If no column with that name is contained by the row, `nil` is returned. Matching columns from _any_ table will be returned on a first match basis. 

### PostgresError

The `PostgresError` type represents errors thrown from both the Postgres package itself (during parsing, for example) and errors returned by the server:

```swift
public enum PostgresError: Error {
    case proto(String)
    case server(PostgresMessage.Error)
    case connectionClosed

    var code: Code { ... }
}
```

The `PostgresError.Code` type is a large enum-like struct containing all recognized Postgres error codes. This is useful for error handling:

```swift
let conn: PostgresConnection ...

do {
    _ = try conn.simpleQuery("SELECT &").wait()
} catch let error as PostgresError {
	switch error.code {
	case .syntaxError: ...
	default: ...
	}
}
```

### PostgresClient

While `PostgresConnection` is the main type to use for connecting, authorizing, and requesting TLS, the `PostgresClient` protocol is sufficient for performing both text and parameterized queries.

```swift
protocol PostgresClient {
    var eventLoop: EventLoop { get }
    func send(_ request: PostgresRequest) -> EventLoopFuture<Void>
}
```

`PostgresConnection` is the only conformer that `NIOPostgres` provides, but it is expected that dependencies will add additional conformers. For example, a consumer of this package might add conformance to a _pool_ of connections, allowing for automatic recycling as needed, a crucial feature for long-running applications. 

#### Note on usage

Since most of `NIOPostgres`'s convenience methods are added to `PostgresClient` instead of `PostgresConnection` directly, any additional conformers should feel exactly the same to use. Because of this, it is expected that `PostgresClient` should be used any place where you need to make queries. For example, in a theoretical controller:

```swift
final class UserController: Controller {
    let db: PostgresClient
    init(db: PostgresClient) { 
        self.db = db
    }

    func names(_ req: HTTPRequest) -> EventLoopFuture<[String]> {
        return self.db.query("SELECT name FROM users").map { rows in
            return rows.map { $0.column("name")?.string! }
        }
    }
}
```

Because this controller relies on `PostgresClient`, any of the following could be supplied to it:

- Connected `PostgresConnection`
- Pool of `PostgresConnection`s
- Dummy conformer for testing

#### PostgresRequest

Postgres' wire protocol uses a request / response pattern, but unlike HTTP or Redis, one request can yield one _or more_ responses. `PostgresRequest` conformers handle this with the following protocol.

```swift
protocol PostgresRequest {
    func respond(to message: PostgresMessage) throws -> [PostgresMessage]?
    func start() throws -> [PostgresMessage]
}
```

`PostgresRequest` is responsible for sending zero or more initial messages and handling the server's responses. When the request is complete, `nil` is returned by `respond`, causing the client's send future to complete.

### CMD5

MD5 hashing is required for PostgreSQL's authentication flow. This module follows NIO's approach and embeds a private C-based implementation rather than relying on external Swift crypto libraries.

### Todo / Discussion

Here are some things that are still a work in progress:

- **Prepared Statement API**: Postgres allows for parameterized queries to be re-used multiple times with different inputs. An API for doing this in NIO Postgres would be useful.
- **PostgresRequest edge cases**: Finer grain input / output from this protocol would be useful in assisting with protocol edge cases. For example, sometimes a Postgres error message can signal request completion depending on state. 

### How to use

To try this package out at home, add the following dependency to your `Package.swift` file:

```swift
.package(url: "https://github.com/vapor/nio-postgres.git", .branch("master")),
```

Then add `"NIOPostgres"` to your module target's dependencies array.

### Seeking Feedback

* If anything, what does this proposal *not cover* that you will definitely need?
* If anything, what could we remove from this and still be happy?
* API-wise: what do you like, what don't you like?

Feel free to post feedback as response to this post and/or GitHub issues on [vapor/nio-postgres](https://github.com/vapor/nio-postgres).
