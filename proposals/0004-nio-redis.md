# RedisNIO: A NIO-based Redis Driver

> **Note**: Since this proposal was published, the module `RedisNIO` has been renamed to `RediStack`.
>
> The renamed GitLab repo is found at https://gitlab.com/mordil/swift-redi-stack. The GitHub Mirror is found at https://github.com/mordil/swift-redi-stack
> 
> For more context behind this change, see the following threads: [[1]](https://forums.swift.org/t/namespacing-of-packages-modules-especially-regarding-swiftnio/24726), [[2]](https://forums.swift.org/t/sswg-minimum-requirements-to-require-no-existing-clashes/24932), [[3]](https://forums.swift.org/t/redisnio-name-brainstorm/26521/)

* Proposal: [SSWG-0004](https://github.com/swift-server/sswg/blob/master/proposals/SSWG-0004.md)
* Authors: [Nathan Harris](https://github.com/Mordil)
* Sponsor(s): Vapor
* Status: **Accepted as Sandbox Maturity**
* Implementation: https://gitlab.com/mordil/swift-redis-nio-client ([GitHub Mirror](https://github.com/mordil/swift-redis-nio-client))
* Forum Threads: [Pitch](https://forums.swift.org/t/swiftnio-redis-client), [Discussion](https://forums.swift.org/t/discussion-nioredis-nio-based-redis-driver/22455/)
* Revisions: [[1]](https://github.com/swift-server/sswg/blob/f3bdea501b7c32d4cee64c989525dcafed247971/proposals/0004-nio-redis.md)
* Decision Notes: [Rationale](https://forums.swift.org/t/june-27th-2019/26580)

## Package Description
Non-blocking Swift driver for Redis built on SwiftNIO.

|  |  |
|--|--|
| **Package Name** | `swift-redis-nio-client` |
| **Module Name** | `RedisNIO` |
| **Proposed Maturity Level** | [Incubating](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram) |
| **License** | [Apache 2](https://www.apache.org/licenses/LICENSE-2.0.html) |
| **Dependencies** | [SwiftNIO](https://github.com/apple/swift-nio) 2.x, [SwiftLog](https://github.com/apple/swift-log) 1.x, [SwiftMetrics](https://github.com/apple/swift-metrics) 1.x |

## Introduction

**RedisNIO** is a module providing general implementations for connecting to a Redis instance and executing commands against it using Redis' proprietary [**Re**dis **S**eralization **P**rotocol (RESP)](https://redis.io/topics/protocol).

These types are designed to work in a request / response loop, representing individual connections to Redis.

The goal of this library is to provide individual building blocks for working with Redis, while still providing enough mechanisms "out of the box" so users can get started immediately with Redis.

## Motivation

Implementations of Swift Redis clients have been around for as long as Swift has, but most have been abandoned, rely on Objective-C runtimes, or use C libraries.

All of the currently maintained libraries either have framework specific dependencies, are not built with NIO, or do not provide enough extensibility while providing "out of the box" capabilities.

### Existing Solutions

- [Kitura](https://github.com/IBM-Swift/Kitura-redis)
- [Vapor](https://github.com/vapor/redis)
- Noze.io
    - [nio-redis](https://github.com/SwiftNIOExtras/swift-nio-redis) - RESP encoding/decoding + pipeline setup
    - [nio-redis-client](https://github.com/NozeIO/swift-nio-redis-client) - API on top of the RESP encoding/decoding library
    - [redi/s](https://github.com/NozeIO/redi-s) - Redi/S server
- [RedisClient](https://github.com/reswifq/redis-client)

## Proposed Solution

**RedisNIO** provides the essential types for interacting with **RESP** and building NIO `Channel` pipelines for communicating with Redis, with default implementations designed to cover most use cases.

### RESP Specification

As a low level library, this package implements the **RESP** specification by providing:
  - an intermediate representation (`RESPValue`)
  - a protocol for conforming user-defined types to the intermediate (`RESPValueConvertible`)
  - a `RESPTranslator` that handles the conversion to/from the intermediate representation

These types should be able to be used independently of any NIO pipeline or some sort of connection to Redis.

#### RESPTranslator

This is a namespace enum for the static methods that follow the **RESP** specification.

```swift
public enum RESPTranslator {
    /// Representation of the result of a parse attempt on a byte stream.
    /// - incomplete: The stream contains an incomplete RESP message from the position provided.
    /// - parsed: The parsed `RESPValue`
    public enum ParsingResult {
        case incomplete
        case parsed(RESPValue)
    }

    /// Attempts to parse the `ByteBuffer`, starting at the specified position,
    /// following the RESP specification.
    ///
    /// See [https://redis.io/topics/protocol](https://redis.io/topics/protocol)
    /// - Parameters:
    ///     - buffer: The buffer that contains the bytes that need to be parsed.
    ///     - position: The index of the buffer that should contain the first byte of the message.
    public static func parseBytes(_ buffer: inout ByteBuffer, fromIndex position: inout Int) throws -> ParsingResult

    /// Writes the `RESPValue` into the provided `ByteBuffer` following the RESP specification.
    ///
    /// See [https://redis.io/topics/protocol](https://redis.io/topics/protocol)
    /// - Parameters:
    ///     - value: The value to write to the buffer.
    ///     - out: The buffer being written to.
    public static func writeValue(_ value: RESPValue, into out: inout ByteBuffer)
}
```

#### RESPValue

[`RESPValue`](https://gitlab.com/Mordil/swift-redis-nio-client/blob/master/Sources/RedisNIO/RESP/RESPValue.swift) represents the different types outlined in Redis' protocol as an **enum**.

Even though Redis defines two storage types as "String" (`simpleString` and `bulkString`) they are fundamentally byte blobs - which are represented as NIO's `ByteBuffer` in Swift.

```swift
public enum RESPValue {
    case null
    case simpleString(ByteBuffer)
    case bulkString(ByteBuffer?)
    case error(RedisError)
    case integer(Int)
    case array(ContiguousArray<RESPValue>)
}
```

#### RESPValueConvertible

Native and User Defined Swift types need to be translatable to `RESPValue` for communicating in the **RESP** format - which is provided by the [`RESPValueConvertible`](https://gitlab.com/Mordil/swift-redis-nio-client/blob/master/Sources/RedisNIO/RESP/RESPValueConvertible.swift) protocol.

```swift
public protocol RESPValueConvertible {
    init?(_ value: RESPValue)

    func convertedToRESPValue() -> RESPValue
}
```

Default conformance is provided for:
 - `Optional where Wrapped: RESPValueConvertible`
 - `Array where Element: RESPValueConvertible`
 - `ContiguousArray where Element: RESPValueConvertible`
 - `RedisError`
 - `RESPValue`
 - `String`
 - `FixedWidthInteger` (`Int`, `Int8`, `Int16`, ...)
 - `Double`
 - `Float`

### NIO Integration

As this is a NIO-based driver, integration with **SwiftNIO** is provided with 3 `ChannelHandler`s out of the box: `RedisByteDecoder`, `RedisMessageEncoder`, and `RedisCommandHandler`.

While both `RedisByteDecoder` and `RedisMessageEncoder` are both sealed types, `RedisCommandHandler` is _open_ as to provide a customization point for Redis command traffic.

However, the NIO pipeline is not set in stone, and each of these could be replaced wholesale by end users.

A factory method is provided for easily making a "default" `ClientBootstrap` instance with these `ChannelHandler`s added to the pipeline:

```swift
extension Redis {
    /// Makes a new `ClientBootstrap` instance with a default Redis `Channel` pipeline
    /// for sending and receiving messages in Redis Serialization Protocol (RESP) format.
    ///
    /// See `RedisMessageEncoder`, `RedisByteDecoder`, and `RedisCommandHandler`.
    /// - Parameter using: The `EventLoopGroup` to build the `ClientBootstrap` on.
    /// - Returns: A `ClientBootstrap` with the default configuration of a `Channel` pipeline for RESP messages.
    public static func makeDefaultClientBootstrap(using group: EventLoopGroup) -> ClientBootstrap
}
```

#### RedisByteDecoder

In a NIO pipeline, `RedisByteDecoder` plays the role of a `ByteToMessageDecoder` to translate incoming **RESP** byte streams into `RESPValue`s.

As an implementation detail, it uses the `RESPTranslator` for parsing the byte stream.

```swift
public final class RedisByteDecoder: ByteToMessageDecoder {
    public typealias InboundOut = RESPValue

    public func decode(context: ChannelHandlerContext, buffer: inout ByteBuffer) throws -> DecodingState {
        // RESPTranslator.parseBytes(_:fromIndex:)
    }
}
```

#### RedisMessageEncoder

In the outgoing direction of the pipeline, `RedisMessageEncoder` translates `RESPValue`s into bytes to be sent to Redis.

This also uses `RESPTranslator` as an implementation.

As a means of debugging errors that might occur in the serialization of complex structures, this type has a SwiftLog `Logger` instance to print the output.

This `Logger` instance can be overridden in the `init`.

```swift
public final class RedisMessageEncoder: MessageToByteEncoder {
    public typealias OutboundIn = RESPValue

    public init(logger: Logger = Logger(label: "RedisNIO.RedisMessageEncoder"))

    public func encode(data: RESPValue, out: inout ByteBuffer) throws {
        // RESPTranslator.writeValue(_:into:)
    }
}
```

#### RedisCommandHandler

Executing "commands" against a Redis instance on the same connection follows an synchronous request / response cycle so `RedisCommandHandler` serves the role of coordinating incoming responses and outgoing commands in a queue.

Commands, their arguments, and the response callback are written in the `RedisCommandContext` struct.

As this is a core type, with plenty of traffic, `RedisCommandHandler` uses a SwiftLog `Logger` to track the lifecycle of a command.

Just like all parts of this library, this default initialized instance can be overridden.

```swift
public struct RedisCommandContext {
    public let command: RESPValue
    public let responsePromise: EventLoopPromise<RESPValue>

    public init(/* member-wise intializer */)
}

open class RedisCommandHandler: ChannelDuplexHandler {
    public typealias InboundIn = RESPValue
    public typealias OutboundIn = RedisCommandContext
    public typealias OutboundOut = RESPValue

    public init(logger: Logger = Logger(label: "RedisNIO.CommandHandler"))

    public func errorCaught(context: ChannelHandlerContext, error: Error)
    public func channelRead(context: ChannelHandlerContext, data: NIOAny)
    public func write(context: ChannelHandlerContext, data: NIOAny, promise: EventLoopPromise<Void>?)
}
```

### Connections

Regardless of which way a user might choose to source their Redis `ClientBootstrap`, they can create a `RedisConnection` to pass around as they see fit for sending commands to Redis.

#### RedisConnection

As the primary connection type, `RedisConnection` is designed to be long-lived and intended to be thread-safe.

In an effort to give users some finer degree of performance control over how frequently commands are written and flushed over the socket, `RedisConnection` has the property `sendCommandsImmediately`.

Much of the reasoning behind this can be found in the [discussion thread](https://forums.swift.org/t/discussion-nioredis-nio-based-redis-driver/22455/18).

```swift
public final class RedisConnection {
    public var eventLoop: EventLoop
    /// Is the client still connected to Redis?
    public var isConnected: Bool
    /// Controls the timing behavior of sending commands over this connection. The default is `true`.
    ///
    /// When set to `false`, the host will "queue" commands and determine when to send all at once,
    /// while `true` will force each command to be sent as soon as they are "queued".
    /// - Note: Setting this to `true` will trigger all "queued" commands to be sent.
    public var sendCommandsImmediately: Bool

    /// - Important: Call `close()` before deinitializing to properly cleanup resources.
    /// - Note: This connection will take ownership of the channel.
    public init(channel: Channel, logger: Logger = Logger(label: "RedisNIO.RedisConnection")

    /// Sends a `QUIT` command, then closes the `Channel` this instance was initialized with.
    ///
    /// See [https://redis.io/commands/quit](https://redis.io/commands/quit)
    /// - Returns: An `EventLoopFuture` that resolves when the connection has been closed.
    @discardableResult
    public func close() -> EventLoopFuture<Void>

    /// Sends commands to the Redis instance this connection is tied to.
    ///
    /// See `RedisClient.send(command:with:)`
    ///
    /// - Note: The timing of when commands are actually sent to Redis are controlled by
    ///     the `sendCommandsImmediately` property.
    public func send(command: String, with arguments: [RESPValueConvertible]) -> EventLoopFuture<RESPValue>
}
```

As a convenience, a factory method is provided under the `Redis` namespace enum.

```swift
extension Redis {
    /// Makes a new connection to a Redis instance.
    ///
    /// As soon as the connection has been opened on the host, an "AUTH" command will be sent to
    /// Redis to authorize use of additional commands on this new connection.
    ///
    /// See [https://redis.io/commands/auth](https://redis.io/commands/auth)
    ///
    /// Example:
    ///
    ///     let elg = MultiThreadedEventLoopGroup(numberOfThreads: 3)
    ///     let connection = Redis.makeConnection(
    ///         to: .init(ipAddress: "127.0.0.1", port: 6379),
    ///         using: elg,
    ///         password: "my_pass"
    ///     )
    ///
    /// - Parameters:
    ///     - socket: The `SocketAddress` information of the Redis instance to connect to.
    ///     - group: The `EventLoopGroup` to build the connection on. Default is a single threaded `EventLoopGroup`.
    ///     - password: The optional password to authorize the client with.
    ///     - logger: The `Logger` instance to log with.
    /// - Returns: A `RedisConnection` instance representing this new connection.
    public static func makeConnection(
        to socket: SocketAddress,
        using group: EventLoopGroup = MultiThreadedEventLoopGroup(numberOfThreads: 1),
        password: String? = nil,
        logger: Logger = Logger(label: "RedisNIO.RedisConnection")
    ) -> EventLoopFuture<RedisConnection>
}
```

#### RedisClient

While `RedisConnection` is the designed _concrete_ common currency - the goal is to not enforce it as the sole implementation of Redis connections.

To this effect, the `RedisClient` protocol defines the base-level implementation requirements for any Redis connection, which `RedisConnection` conforms to.

Additionally, since Redis (as of writing this proposal) has over 200 separate commands (with some sub-commands), there is a practical desire to have a type-safe and ergonomic API around the `send(command:with:)` method.

To serve this need, `RedisClient` has Swift-y convenience extension methods that map to specific Redis commands.

When conforming to `RedisClient`, these implementations will be gained for free.

```swift
public protocol RedisClient {
    /// The `EventLoop` that this client operates on.
    var eventLoop: EventLoop { get }

    /// Sends the desired command with the specified arguments.
    /// - Parameters:
    ///     - command: The command to execute.
    ///     - arguments: The arguments, if any, to be sent with the command.
    /// - Returns: An `EventLoopFuture` that will resolve with the Redis command response.
    func send(command: String, with arguments: [RESPValueConvertible]) -> EventLoopFuture<RESPValue>
}
```

### Module Usage Example

```swift
import RedisNIO

// create a new event loop group
let elg = MultiThreadedEventLoopGroup(numberOfThreads: 3)
defer { try! elg.syncShutdownGracefully() }

// create a new connection
let connection = Redis.makeConnection(
    to: try .init(ipAddress: "127.0.0.1", port: 6379),
    using: elg,
    password: "my_password"
).wait()
defer { try! connection.close.wait() }

let value = connection.set("my_key", to: 3)
    .flatMap { return connection.increment("my_key") }
    .flatMap { return connection.increment("my_key", by: 10)}
    .flatMap { return connection.get("my_key") }
    .wait()
// value == Optional("14")
```

## Metrics

**RedisNIO** comes with basic `SwiftMetrics` integration for the following data points:

- (Counter) # of failed requests
- (Counter) # of successful requests
- (Counter) # of connections made
- (Gauge) # of active connections
- (Timer) Round-trip time in `ms` to complete a request

They are defined under a `RedisMetrics` struct, and are labeled with the `RedisMetrics.Label` enum for referencing within metric library integrations.

## Maturity Justification

Until now, packages through the SSWG process have been accepted as _Sandbox_ maturity - so it's appropriate to justify why **RedisNIO** might be considered mature enough for _Incubating_.

This package supports:
- Logging through **SwiftLog**
- Metrics through **SwiftMetrics**
- ~100 (and counting!) of Redis' commands as convenient extension methods
- 130+ unit tests, including RESP encoding / decoding and all command extensions

In addition, it meets the following criteria according to the [SSWG Incubation Process](https://github.com/swift-server/sswg/blob/master/process/incubation.md):
- [Apache 2 license](https://gitlab.com/Mordil/swift-redis-nio-client/blob/master/LICENSE.txt)
- [Swift Code of Conduct](https://gitlab.com/Mordil/swift-redis-nio-client/blob/master/CODE_OF_CONDUCT.md)
- [Contributing Guide](https://gitlab.com/Mordil/swift-redis-nio-client/blob/master/CONTRIBUTING.md)
- [SSWG Member Access](https://gitlab.com/Mordil/swift-redis-nio-client/project_members) (including GitHub mirror)
- [CI builds](https://gitlab.com/Mordil/swift-redis-nio-client/pipelines)
- [Generated API Docs](https://mordil.gitlab.io/swift-redis-nio-client/)

Vapor has also written a higher-level, framework agnostic, library [`RedisKit`](https://github.com/vapor/redis-kit) built from **RedisNIO** that will be the foundation for their Redis solution in Vapor 4.

This is in part because **RedisNIO** is almost a drop-in replacement for their implementation, with the current version being used by several dozens of developers daily.
