# MongoDB Swift Driver

* Proposal: SSWG-0010
* Authors: [Kaitlin Mahar](https://github.com/kmahar), [Patrick Freed](https://github.com/patrickfreed), [Matt Broadstone](https://github.com/mbroadst)
* Sponsor(s): TBD
* Review Manager: [Tanner Nelson](https://github.com/tanner0101)
* Status: **Implemented**
* Implementation: [mongo-swift-driver](https://github.com/mongodb/mongo-swift-driver)
* Forum Threads: [Pitch](https://forums.swift.org/t/officially-supported-mongodb-driver/30324), [Discussion](https://forums.swift.org/t/discussion-mongodb-swift-driver/34472)

## Package Description
MongoDB driver for Swift.

|  |  |
|--|--|
| **Package Name** | `mongo-swift-driver` |
| **Module Name** | `MongoSwift` |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram) |
| **License** | [Apache 2.0]([https://choosealicense.com/licenses/apache-2.0/](https://choosealicense.com/licenses/apache-2.0/)) |
| **Dependencies** | [libmongoc 1.15.3]([https://github.com/mongodb/mongo-c-driver](https://github.com/mongodb/mongo-c-driver)) (vendored), [SwiftNIO 2]([https://github.com/apple/swift-nio](https://github.com/apple/swift-nio)), [Nimble 8]([https://github.com/quick/nimble](https://github.com/quick/nimble)) (for tests) |

## Introduction

`mongo-swift-driver` is a client library for using MongoDB from Swift. Its module `MongoSwift` provides a SwiftNIO-based asynchronous API for interacting with the database. The core type, `MongoClient`, maintains a pool of connections to servers in a MongoDB deployment and provides an interface for querying, inserting, and updating data in the deployment. In addition, the client handles authentication, TLS, topology monitoring, and automatically retrying failed commands. The driver also contains a [BSON](https://docs.mongodb.com/manual/reference/bson-types/) implementation allowing users to create and manipulate MongoDB documents and to convert between documents and native Swift types.

## Motivation
This driver represents an official effort by MongoDB itself to provide first-class support for using the database from server-side Swift applications. It is developed by the team that writes the official MongoDB drivers for several programming languages and is written in accordance with the official driver [specifications](https://github.com/mongodb/specifications), meaning it provides a user experience consistent with what developers who have worked with other MongoDB in other languages expect.

## Proposed solution

The package `mongo-swift-driver`  contains two modules, `MongoSwift` and `MongoSwiftSync`. `MongoSwift` contains an asynchronous, SwiftNIO-based API, and `MongoSwiftSync` contains a synchronous wrapper of the asynchronous API.

The driver depends on a vendored copy of the MongoDB C driver, libmongoc. As libmongoc is a synchronous driver, the asynchronous API is implemented by running all blocking code within a `SwiftNIO` `NIOThreadPool`.

The main entry point to the API is a `MongoClient`, which handles interactions with a single MongoDB deployment. Clients are thread-safe and automatically pool connections to the deployment's server(s) under the hood. The client also automatically discovers the full topology of the deployment and dynamically tracks its state over time. For most use cases, sharing a single `MongoClient` for an entire application should be sufficient.

Child `MongoDatabase`s and `MongoCollection`s are retrieved from `MongoClient`s, and provide APIs to perform CRUD and various other operations on corresponding databases and collections in the MongoDB deployment.

MongoDB stores data in a format called [BSON](https://bsonspec.org), i.e. binary JSON. To support working with this format, the driver also contains a BSON library. This includes a `Document` type corresponding to a MongoDB document, as well as  a `BSONEncoder` and `BSONDecoder` to support conversion back and forth between `Codable` Swift types and `Document`s.

The driver works with MongoDB 3.6+ and Swift 5.0+. It is [tested](https://evergreen.mongodb.com/waterfall/mongo-swift-driver) against all supported MongoDB and Swift versions on both macOS 10.14 and Ubuntu 18.04, as well as a variety of MongoDB topologies (standalone server, replica set, sharded cluster, TLS and authentication on/off).

This library has been primarily developed by the three authors of this proposal. Kaitlin and Matt began developing it about two years ago, and Patrick has been working on the project for about a year. The library was initially developed with a synchronous API to support use of the mobile/embedded version of MongoDB on iOS, but over time its focus and primary intended use case has shifted toward server-side Swift usage.

We've recently tagged a release candidate for an upcoming 1.0 release, which we plan to release once we've received and incorporated feedback from the community. The long term-plan following 1.0 is to:
1) Catch up on MongoDB features introduced in recent server versions. While the driver works with all versions of MongoDB 3.6+, it lacks APIs for some newer features such as transactions.
2) Decouple the BSON library from the driver, pulling it out into a separate package and repository.
2) Rewrite the BSON library and driver internals in pure Swift, culminating in a 2.0 release.

## Detailed design

Each `MongoClient` is backed by:
* A libmongoc `mongoc_client_pool_t`, which can be thought of as a pool of connections to the MongoDB deployment. This is wrapped in an internal type called `ConnectionPool` which supports checking `Connection`s in and out.
* A SwiftNIO `NIOThreadPool`, whose size may be customized by the user at the time of client creation.
* A SwiftNIO `EventLoopGroup`, provided by the user at the time of client creation.

Each I/O-performing method on `MongoClient` as well as its child objects (`MongoDatabase`, `MongoCollection`) has an internal `Operation` type which encapsulates all of the blocking code and has an `execute(using conn: Connection)` method.

Whenever an I/O-performing API method is called on `MongoClient` or one of its child `MongoDatabase`/`MongoCollection`s, we call the `NIOThreadPool`'s `runIfActive` method to generate the future we return (this is a simplification, but demonstrates the general idea):
```swift
return threadPool.runIfActive(eventLoop: elg.next()) {
    let connection = connectionPool.checkOut()
    defer { connectionPool.checkIn(connection) }
    return try operation.execute(using: connection)
}
```

The driver has a very large API surface, which we have not included in full for the sake of brevity. Instead, we provide a representative sample of the API. The full API may be viewed on our [documentation](https://mongodb.github.io/mongo-swift-driver/) website.

A large portion of our API is Swift translations of APIs defined in MongoDB specifications such as [CRUD](https://github.com/mongodb/specifications/blob/master/source/crud/crud.rst), [Change Streams](https://github.com/mongodb/specifications/blob/master/source/change-streams/change-streams.rst), [Driver Sessions](https://github.com/mongodb/specifications/blob/master/source/sessions/driver-sessions.rst), [Index Management](https://github.com/mongodb/specifications/blob/master/source/index-management.rst), [Enumerate Indexes](https://github.com/mongodb/specifications/blob/master/source/enumerate-indexes.rst), [Enumerate Databases](https://github.com/mongodb/specifications/blob/master/source/enumerate-databases.rst), [Enumerate Collections](https://github.com/mongodb/specifications/blob/master/source/enumerate-collections.rst), etc.

### MongoClient
```swift
/// A MongoDB Client providing an asynchronous, SwiftNIO-based API.
public class MongoClient {
    /**
     * Create a new client for a MongoDB deployment. For options that included in both the connection string URI
     * and the ClientOptions struct, the final value is set in descending order of priority: the value specified in
     * ClientOptions (if non-nil), the value specified in the URI, or the default value if both are unset. This client
     * will lazily establish connections to the MongoDB deployment the first time it is used for I/O.
     *
     * - Parameters:
     *   - connectionString: the connection string to connect to.
     *   - eventLoopGroup: A SwiftNIO `EventLoopGroup` which the client will use for executing operations. It is the
     *                     user's responsibility to ensure the group remains active for as long as the client does, and
     *                     to ensure the group is properly shut down when it is no longer in use.
     *   - options: optional `ClientOptions` to use for this client
     *
     * - SeeAlso: https://docs.mongodb.com/manual/reference/connection-string/
     *
     * - Throws:
     *   - A `InvalidArgumentError` if the connection string passed in is improperly formatted.
     */
    public init(
        _ connectionString: String = "mongodb://localhost:27017",
        using eventLoopGroup: EventLoopGroup,
        options: ClientOptions? = nil
    ) throws

    /**
     * Shuts this `MongoClient` down, closing all connections to the server and cleaning up internal state.
     *
     * Call this method exactly once when you are finished using the client. You must ensure that all operations
     * using the client have completed before calling this.
     * 
     * The returned future must be fulfilled before the `EventLoopGroup` provided to this client's constructor
     * is shut down.
     */
    public func shutdown() -> EventLoopFuture<Void>

    /**
     * Shuts this `MongoClient` down in a blocking fashion, closing all connections to the server and cleaning up
     * internal state.
     *
     * Call this method exactly once when you are finished using the client. You must ensure that all operations
     * using the client have completed before calling this.
     *
     * This method must complete before the `EventLoopGroup` provided to this client's constructor is shut down.
     */
    public func syncShutdown()

    /**
     * Gets a `MongoDatabase` instance for the given database name. If an option is not specified in the optional
     * `DatabaseOptions` param, the database will inherit the value from the parent client or the default if
     * the client’s option is not set. To override an option inherited from the client (e.g. a read concern) with the
     * default value, it must be explicitly specified in the options param (e.g. ReadConcern(), not nil).
     *
     * - Parameters:
     *   - name: the name of the database to retrieve
     *   - options: Optional `DatabaseOptions` to use for the retrieved database
     *
     * - Returns: a `MongoDatabase` corresponding to the provided database name.
     */
    public func db(_ name: String, options: DatabaseOptions? = nil) -> MongoDatabase
}
```

### MongoDatabase
`MongoDatabase`s are instantiated via the `MongoClient.db` method.
```swift
/// A MongoDB Database.
public struct MongoDatabase {
    /**
     * Access a collection within this database. If an option is not specified in the `CollectionOptions` param, the
     * collection will inherit the value from the parent database or the default if the db's option is not set.
     * To override an option inherited from the db (e.g. a read concern) with the default value, it must be explicitly
     * specified in the options param (e.g. ReadConcern(), not nil).
     *
     * - Parameters:
     *   - name: the name of the collection to get
     *   - options: options to set on the returned collection
     *
     * - Returns: the requested `MongoCollection<Document>`
     */
    public func collection(_ name: String, options: CollectionOptions? = nil) -> MongoCollection<Document>

    /**
     * Access a collection within this database, and associates the specified `Codable` type `T` with the
     * returned `MongoCollection`. This association only exists in the context of this particular
     * `MongoCollection` instance. If an option is not specified in the `CollectionOptions` param, the
     * collection will inherit the value from the parent database or the default if the db's option is not set.
     * To override an option inherited from the db (e.g. a read concern) with the default value, it must be explicitly
     * specified in the options param (e.g. ReadConcern(), not nil).
     *
     * - Parameters:
     *   - name: the name of the collection to get
     *   - options: options to set on the returned collection
     *
     * - Returns: the requested `MongoCollection<T>`
     */
    public func collection<T: Codable>(
        _ name: String,
        withType _: T.Type,
        options: CollectionOptions? = nil
    ) -> MongoCollection<T>
    
    /**
     * Issues a MongoDB command against this database.
     *
     * - Parameters:
     *   - command: a `Document` containing the command to issue against the database
     *   - options: Optional `RunCommandOptions` to use when executing this command
     *   - session: Optional `ClientSession` to use when executing this command
     *
     * - Returns:
     *    An `EventLoopFuture<Document>`. On success, contains the server response to the command.
     *
     *    If the future fails, the error is likely one of the following:
     *    - `InvalidArgumentError` if `requests` is empty.
     *    - `LogicError` if the provided session is inactive.
     *    - `LogicError` if this databases's parent client has already been closed.
     *    - `WriteError` if any error occurs while the command was performing a write.
     *    - `CommandError` if an error occurs that prevents the command from being performed.
     *    - `EncodingError` if an error occurs while encoding the options to BSON.
     */
    public func runCommand(
        _ command: Document,
        options: RunCommandOptions? = nil,
        session: ClientSession? = nil
    ) -> EventLoopFuture<Document>
}
```

### MongoCollection
`MongoCollection`s are initialized by calling one of the `collection` or `createCollection` methods on `MongoDatabase`. They are generic over a `Codable` type `T`. This type can be a BSON `Document` or a `Codable` type that matches up with the document structure in the server's corresponding collection.
```swift
/// A MongoDB collection.
public struct MongoCollection<T: Codable> {
    /**
     * Finds the documents in this collection which match the provided filter.
     *
     * - Parameters:
     *   - filter: A `Document` that should match the query
     *   - options: Optional `FindOptions` to use when executing the command
     *   - session: Optional `ClientSession` to use when executing this command
     *
     * - Returns:
     *    An `EventLoopFuture<MongoCursor<CollectionType>`. On success, contains a cursor over the resulting documents.
     *
     *    If the future fails, the error is likely one of the following:
     *    - `InvalidArgumentError` if the options passed are an invalid combination.
     *    - `LogicError` if the provided session is inactive.
     *    - `LogicError` if this collection's parent client has already been closed.
     *    - `EncodingError` if an error occurs while encoding the options to BSON.
     */
    public func find(
        _ filter: Document = [:],
        options: FindOptions? = nil,
        session: ClientSession? = nil
    ) -> EventLoopFuture<MongoCursor<CollectionType>>

    /**
     * Counts the number of documents in this collection matching the provided filter. Note that an empty filter will
     * force a scan of the entire collection. For a fast count of the total documents in a collection see
     * `estimatedDocumentCount`.
     *
     * - Parameters:
     *   - filter: a `Document`, the filter that documents must match in order to be counted
     *   - options: Optional `CountDocumentsOptions` to use when executing the command
     *   - session: Optional `ClientSession` to use when executing this command
     *
     * - Returns:
     *    An `EventLoopFuture<Int>`. On success, contains the count of the documents that matched the filter.
     *
     *    If the future fails, the error is likely one of the following:
     *    - `CommandError` if an error occurs that prevents the command from executing.
     *    - `InvalidArgumentError` if the options passed in form an invalid combination.
     *    - `LogicError` if the provided session is inactive.
     *    - `LogicError` if this collection's parent client has already been closed.
     *    - `EncodingError` if an error occurs while encoding the options to BSON.
     */
    public func countDocuments(
        _ filter: Document = [:],
        options: CountDocumentsOptions? = nil,
        session: ClientSession? = nil
    ) -> EventLoopFuture<Int>

    /**
     * Encodes the provided value to BSON and inserts it. If the value is missing an identifier, one will be
     * generated for it.
     *
     * - Parameters:
     *   - value: A `CollectionType` value to encode and insert
     *   - options: Optional `InsertOneOptions` to use when executing the command
     *   - session: Optional `ClientSession` to use when executing this command
     *
     * - Returns:
     *    An `EventLoopFuture<InsertOneResult?>`. On success, contains the result of performing the insert, or contains
     *    `nil` if the write concern is unacknowledged.
     *
     *    If the future fails, the error is likely one of the following:
     *    - `WriteError` if an error occurs while performing the command.
     *    - `CommandError` if an error occurs that prevents the command from executing.
     *    - `InvalidArgumentError` if the options passed in form an invalid combination.
     *    - `LogicError` if the provided session is inactive.
     *    - `LogicError` if this collection's parent client has already been closed.
     *    - `EncodingError` if an error occurs while encoding the `CollectionType` to BSON.
     */
    public func insertOne(
        _ value: CollectionType,
        options: InsertOneOptions? = nil,
        session: ClientSession? = nil
    ) -> EventLoopFuture<InsertOneResult?>
}
```

### MongoCursor
`MongoCursor`s are initialized via API methods that return them - for example, `MongoCollection.find`.

`MongoCursor` is generic over a `Codable` type `T` , which varies depending on the particular API method that returned the cursor. For example, `MongoCollection<CollectionType>.find` returns a `MongoCursor<CollectionType>`, while `MongoDatabase.listCollections` returns a `MongoCursor<CollectionSpecification>`.

```swift
public class MongoCursor<T: Codable> {
    /**
     * Indicates whether this cursor has the potential to return more data.
     *
     * This method is mainly useful if this cursor is tailable, since in that case `tryNext` may return more results
     * even after returning `nil`.
     *
     * If this cursor is non-tailable, it will always be dead as soon as either `tryNext` returns `nil` or an error.
     *
     * This cursor will be dead as soon as `next` returns `nil` or an error, regardless of the `CursorType`.
     */
    public func isAlive() -> EventLoopFuture<Bool>

    /**
     * Attempt to get the next `T` from the cursor, returning `nil` if there are no results.
     *
     * If this cursor is tailable, this method may be called repeatedly while `isAlive` is true to retrieve new data.
     *
     * If this cursor is a tailable await cursor, it will wait for results server side for a maximum of `maxAwaitTimeMS`
     * before evaluating to `nil`. This option can be configured via options passed to the method that created this
     * cursor (e.g. the `maxAwaitTimeMS` option on the `FindOptions` passed to `find`).
     *
     * Note: You *must not* call any cursor methods besides `kill` and `isAlive` while the future returned from this
     * method is unresolved. Doing so will result in undefined behavior.
     *
     * - Returns:
     *    An `EventLoopFuture<T?>` containing the next `T` in this cursor, an error if one occurred, or `nil` if
     *    there was no data.
     *
     *    If the future evaluates to an error, it is likely one of the following:
     *      - `CommandError` if an error occurs while fetching more results from the server.
     *      - `LogicError` if this function is called after the cursor has died.
     *      - `LogicError` if this function is called and the session associated with this cursor is inactive.
     *      - `LogicError` if this cursor's parent client has already been closed.
     *      - `DecodingError` if an error occurs decoding the server's response.
     */
    public func tryNext() -> EventLoopFuture<T?>

    /**
     * Get the next `T` from the cursor.
     *
     * If this cursor is tailable, this method will continue polling until a non-empty batch is returned from the server
     * or the cursor is closed.
     *
     * A thread from the driver's internal thread pool will be occupied until the returned future is completed, so
     * performance degradation is possible if the number of polling cursors is too close to the total number of threads
     * in the thread pool. To configure the total number of threads in the pool, set the `ClientOptions.threadPoolSize`
     * option during client creation.
     *
     * Note: You *must not* call any cursor methods besides `kill` and `isAlive` while the future returned from this
     * method is unresolved. Doing so will result in undefined behavior.
     *
     * - Returns:
     *   An `EventLoopFuture<T?>` evaluating to the next `T` in this cursor, or `nil` if the cursor is exhausted. If
     *   the underlying cursor is tailable, the future will not resolve until data is returned (potentially after
     *   multiple requests to the server), the cursor is closed, or an error occurs.
     *
     *   If the future fails, the error is likely one of the following:
     *     - `CommandError` if an error occurs while fetching more results from the server.
     *     - `LogicError` if this function is called after the cursor has died.
     *     - `LogicError` if this function is called and the session associated with this cursor is inactive.
     *     - `DecodingError` if an error occurs decoding the server's response.
     */
    public func next() -> EventLoopFuture<T?>

    /**
     * Consolidate the currently available results of the cursor into an array of type `T`.
     *
     * If this cursor is not tailable, this method will exhaust it.
     *
     * If this cursor is tailable, `toArray` will only fetch the currently available results, and it
     * may return more data if it is called again while the cursor is still alive.
     *
     * Note: You *must not* call any cursor methods besides `kill` and `isAlive` while the future returned from this
     * method is unresolved. Doing so will result in undefined behavior.
     *
     * - Returns:
     *    An `EventLoopFuture<[T]>` evaluating to the results currently available in this cursor, or an error.
     *
     *    If the future evaluates to an error, that error is likely one of the following:
     *      - `CommandError` if an error occurs while fetching more results from the server.
     *      - `LogicError` if this function is called after the cursor has died.
     *      - `LogicError` if this function is called and the session associated with this cursor is inactive.
     *      - `DecodingError` if an error occurs decoding the server's responses.
     */
    public func toArray() -> EventLoopFuture<[T]>

    /**
     * Calls the provided closure with each element in the cursor.
     *
     * If the cursor is not tailable, this method will exhaust it, calling the closure with every document.
     *
     * If the cursor is tailable, the method will call the closure with each new document as it arrives.
     *
     * A thread from the driver's internal thread pool will be occupied until the returned future is completed, so
     * performance degradation is possible if the number of polling cursors is too close to the total number of threads
     * in the thread pool. To configure the total number of threads in the pool, set the `ClientOptions.threadPoolSize`
     * option during client creation.
     *
     * Note: You *must not* call any cursor methods besides `kill` and `isAlive` while the future returned from this
     * method is unresolved. Doing so will result in undefined behavior.
     *
     * - Returns:
     *     An `EventLoopFuture<Void>` which will succeed when the end of the cursor is reached, or in the case of a
     *     tailable cursor, when the cursor is killed via `kill`.
     *
     *     If the future evaluates to an error, that error is likely one of the following:
     *     - `CommandError` if an error occurs while fetching more results from the server.
     *     - `LogicError` if this function is called after the cursor has died.
     *     - `LogicError` if this function is called and the session associated with this cursor is inactive.
     *     - `DecodingError` if an error occurs decoding the server's responses.
     */
    public func forEach(_ body: @escaping (T) throws -> Void) -> EventLoopFuture<Void>

    /**
     * Kill this cursor.
     *
     * This method MUST be called before this cursor goes out of scope to prevent leaking resources.
     * This method may be called even if there are unresolved futures created from other `MongoCursor` methods.
     *
     * - Returns:
     *   An `EventLoopFuture` that evaluates when the cursor has completed closing. This future should not fail.
     */
    public func kill() -> EventLoopFuture<Void>
}
```

### Errors
All driver errors are marked by their conformance to an empty protocol `MongoError`. This protocol is conformed to by:
 * `protocol ServerError`, encompassing errors that occur in the database and are returned to the driver in a command response. Conformed to by:
    * `struct CommandError`: occurs when a command encounters an error on the server side that prevents execution (e.g. due to the server being unable to parse a command)
    * `struct WriteError`:  occurs when a single write fails on the server
    * `struct BulkWriteError`: occurs when a bulk write fails on the server
 * `protocol UserError`, encompassing errors caused by incorrect usage of the driver. Conformed to by:
      * `struct LogicError`: occurs when the user makes a logical error, e.g. advancing a cursor that has already been killed
       * `struct InvalidArgumentError`: occurs when the user provides an invalid argument e.g. an empty array is passed to `bulkWrite`.
  * `protocol RuntimeError`, encompassing unexpected errors occurring at runtime that do not fit neatly into either of the above protocols. Conformed to by: 
      * `struct InternalError`: occurs when e.g. something is `nil` that should never be `nil`, or the server sends a response the driver cannot understand. This generally indicates an error in the driver or a system failure (e.g. a memory allocation failure).
      * `struct ConnectionError`: occurs due to connection establishment / socket-related errors.
      * `struct AuthenticationError`: occurs when the driver isn't authorized to perform a command, e.g. due to invalid credentials.
      * `struct ServerSelectionError`: occurs when the driver is unable to select a server for an operation, e.g. due to reaching a timeout or an unsatisfiable read preference.

The driver's BSON library also uses [`EncodingError`](https://developer.apple.com/documentation/swift/encodingerror) and [`DecodingError`](https://developer.apple.com/documentation/swift/decodingerror) from the standard library when appropriate, and will propagate any errors encountered in its use of SwiftNIO to the user.

#### Why not use enums?
It's common to use enumerations to model errors in Swift, and in fact we did this in the driver until not long ago. We switched to the protocol and struct-based approach recently, after realizing enums are a poor fit for error types that evolve over time:
1) Adding an additional associated value to an enum case is a breaking change for anyone who is writing `case let` statements to match that case and its values.
2) Adding a new case to an enum is a breaking change, as a user's previously exhaustive `switch` statement will no longer compile. (If non-frozen enums were available for general use this wouldn't be an issue, but for now we can't use them.)

Over time, the MongoDB server has added more and more information to the errors it returns, and has added various new categories of errors. For example, MongoDB 4.0 introduced the concept of "error labels", containing labels we expose to the user when present. Using protocols and structs gives us the ability to change our errors additively without breaking backwards compatibility.

### Example Usage
```swift
import MongoSwift
import NIO

struct Cat: Codable {
    let name: String
    let color: String
    let _id: ObjectId
}

let elg = MultiThreadedEventLoopGroup(numberOfThreads: 4)
let client = try MongoClient("mongodb://localhost:27017", using: elg)
defer {
    client.syncShutdown()
    try? elg.syncShutdownGracefully()
}

let db = client.db("test")
let cats = db.collection("cats", withType: Cat.self)

let data = [
    Cat(name: "Chester", color: "tan", _id: ObjectId()),
    Cat(name: "Roscoe", color: "orange", _id: ObjectId())
]

// insert values into the collection
let insert = cats.insertMany(data)

insert.whenSuccess { result in
    print(result?.insertedCount) // prints 2
}

insert.whenFailure { error in
    if let err = error as? MongoError {
        switch err {
        case let runtimeError as RuntimeError:
            // handle RuntimeError
        case let serverError as ServerError:
            // handle ServerError
        case let userError as UserError:
            // handle UserError
        default:
            // currently would never get here, but will handle
            // any new types of MongoError added in the future
        }
    } else if let err = error as? DecodingError {
        // handle DecodingError
    } else {
        // handle any other error types
    }
}

// after insert create a cursor over the collection to read back the documents
let result = insert.flatMap { _ in
	cats.find()
}.flatMap { cursor in
	cursor.forEach { cat in
		print(cat) // prints out a `Cat` struct
	}
}
```

## Maturity Justification

To our knowledge there is no significant production usage of the library; therefore, Sandbox is most appropriate.

## Alternatives considered

An alternative solution would be to write a pure Swift driver from the ground up, rather than initially writing a driver on top of libmongoc. However, this would have significantly delayed the development of a stable, feature-complete API which developers can start to depend on while we implement pure Swift driver internals.
