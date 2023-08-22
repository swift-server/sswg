# DiscordBM

- Proposal: [SSWG-NNNN](NNNN-discordbm.md)
- Authors: [Mahdi Bahrami](https://github.com/MahdiBM)
- Review Manager: [Joannis Orlandos](https://github.com/joannis)
- Status: Implemented and Released
- Implementation: [DiscordBM](https://github.com/DiscordBM/DiscordBM)
- Forum Threads: [Pitch](https://forums.swift.org/t/pitch-discordbm-for-sswg-incubation-process/64833)

## Package Description

A New Multi-platform Swift Discord Library For Making Bots.

|                             |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Package Name**            | DiscordBM                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Module Name**             | DiscordBM                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Proposed Maturity Level** | [Sandbox / Incubating](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| **License**                 | Apache-2.0                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| **Dependencies**            | From Apple/SSWG: [swift-nio](https://github.com/apple/swift-nio.git), [swift-log](https://github.com/apple/swift-log.git), [swift-atomics](https://github.com/apple/swift-atomics.git), [swift-nio-ssl](https://github.com/apple/swift-nio-ssl.git), [swift-nio-transport-services](https://github.com/apple/swift-nio-transport-services.git), [async-http-client](https://github.com/swift-server/async-http-client.git), From Vapor: [multipart-kit](https://github.com/vapor/multipart-kit.git) (MIT License), Others: [Yams](https://github.com/jpsim/Yams.git) (Only for code generation. MIT License), [zlib](https://github.com/madler/zlib) (See License on Github) |

## Introduction

DiscordBM is a new multi-platform Server-Side focused Swift library for interacting with the Discord API, for making bots.

It's already being used in Vapor's [Penny](https://github.com/vapor/penny-bot) bot for the past 11+ months, and it's been serving the community very nicely.

## Motivation

The Server-Side Swift community lacks a maintained and robust package to communicate with Discord.
This is important because Discord is gaining more and more popularity amongst communities and has already replaced Slack as the go-to place for Open Source communities. A lot of successful projects and languages such as `Rust`, `Python`, `Vue` have chosen Discord as the place for their community.

## Proposed solution

DiscordBM provides an elegant, robust, type-safe, swifty and [insert positive word here] way to communicate with Discord's bot APIs. This can be useful to bridge the gap between the Swift community and Discord bots.

## Design

DiscordBM not only comes with the core and necessary means to communicate with Discord, but also has a bunch of convenience or useful stuff that users might otherwise need to implement themselves. I'll try to explain the core implementation in detail, and briefly mention some of the rest.

### Core Of DiscordBM

The 2 most important concepts in DiscordBM are the `GatewayManager` and the `DiscordClient` protocols.
Everything else in the library revolves around these two. Both of the protocols are just abstractions to make DiscordBM easier to test for users. The main types that conform to those protocols are `BotGatewayManager` and `DefaultDiscordClient`.

A `GatewayManager` is where most apps start using DiscordBM:

```swift
/// First, add these 2 as dependencies
import DiscordBM
import AsyncHTTPClient

@main
struct EntryPoint {
    static func main() async throws {
        let httpClient = HTTPClient(eventLoopGroupProvider: .createNew)

        let bot: any GatewayManager = await BotGatewayManager(
            eventLoopGroup: httpClient.eventLoopGroup,
            httpClient: httpClient,
            token: YOUR_BOT_TOKEN,
            presence: .init(
                /// Will show up as "Playing Swift" in Discord
                activities: [.init(name: "Swift", type: .game)],
                status: .online,
                afk: false
            ),
            /// Add all the intents you want
            /// You can also use `Gateway.Intent.unprivileged` or `Gateway.Intent.allCases`
            intents: [.guildMessages, .messageContent]
        )

        await bot.connect()
    }
}
```

The `GatewayManager` will start making a connection to the Discord Gateway through Websockets. For Websockets, DiscordBM uses a customized version of Vapor's `websocket-kit` which supports zlib decompression.

In case Discord doesn't accept the identification, DiscordBM will try to resolve the problem or log a message at `critical` level with explanations on how to solve the problem:

```
Will not reconnect because Discord does not allow it. Something is wrong. Your close code is '<CODE>', check Discord docs at https://discord.com/developers/docs/topics/opcodes-and-status-codes#gateway-gateway-close-event-codes and see what it means. Report at https://github.com/DiscordBM/DiscordBM/issues if you think this is a library issue
```

In the other case where Discord accepts the identification, Discord will start sending the Discord events to you. For more context, there are 70+ types of events and they include a created/updated/deleted message, a create/updated/deleted channel, and many more. DiscordBM allows user access to these events using an `AsyncStream<Gateway.Event>` which users can acquire and use like so:

```swift
...

/// Get an `AsyncStream` of `Gateway.Event`s
let stream = await bot.makeEventsStream()

/// Handle each event in the stream
/// When used in the @main entry point, this stream will never end,
/// therefore preventing your executable from exiting
for await event in stream {
    switch event.data {
    case let .messageCreate(message):
        print("NEW MESSAGE!", message)

        /// You can also use the `GatewayEventHandler` protocol for more convenience
    default: break
    }
}
```

This is also in line with the latest structured concurrency advancements, and users won't need to block a thread or call functions such as `RunLoop.main.run()`. Generally speaking, DiscordBM has gone full-in on structured concurrency and makes extensive use of actors. The whole library is already in full compliance with `-strict-concurrency=complete`.

`GatewayManager` also needs to keep the Discord connection alive. This happens in two ways.

1. The usual websocket ping-pongs.
2. By reconnecting when Discord closes a Websocket connection.

Connection closures are normal and initiated by Discord when they see fit. The closures could happen 10+ times a day, but are mostly not distruptive. Upon a conneciton closure, DiscordBM will correctly resume the last connection, and Discord will send the events lost when there was no connection.

After receiving an event, users can use the `DiscordClient` to communicate with Discord through the REST API. For example you can send a message back to the Discord channel that a message came from, like so:

```swift
/// Use `bot.client` to send requests to Discord
case let .messageCreate(message):
    let response = try await bot.client.createMessage(
        channelId: message.channel_id,
        payload: .init(content: "Got a message: '\(message.content)'")
    )
```

And you can safely and effortlessly decode Discord's response to the correct type:

```swift
/// `message` will be of type `DiscordChannel.Message`
let message = try response.decode()
```

The REST API endpoints are all declared as extensions to the `DiscordClient` protocol like so:

```swift
/// https://discord.com/developers/docs/resources/channel#create-message
@inlinable
func createMessage(
    channelId: ChannelSnowflake,
    payload: Payloads.CreateMessage
) async throws -> DiscordClientResponse<DiscordChannel.Message> {
    let endpoint = APIEndpoint.createMessage(channelId: channelId)
    return try await self.sendMultipart(
        request: .init(to: endpoint),
        payload: payload
    )
}
```

This function returns a `DiscordClientResponse`. The generic `C` type that `DiscordClientResponse` takes makes it possible to decode Discord's responses to a library-defined type as you saw in `let message = try response.decode()`:

```swift
/// Represents a Discord HTTP response for endpoints that return some data in the body.
public struct DiscordClientResponse<C>: Sendable where C: Codable {
    /// The raw http response.
    public let httpResponse: DiscordHTTPResponse

    ...

    /// Decodes the response to `C`.
    @inlinable
    public func decode() throws -> C {
        try httpResponse.decode(as: C.self)
    }
}
```

The `DiscordHTTPResponse` type contains a HTTP response:

```swift
/// Represents a raw Discord HTTP response.
public struct DiscordHTTPResponse: Sendable, CustomStringConvertible {

    ...

    public var host: String
    public var status: HTTPResponseStatus
    public var version: HTTPVersion
    public var headers: HTTPHeaders
    public var body: ByteBuffer?

    ...
}
```

This is the basics of `DiscordClient`, but `DefaultDiscordClient` comes with quite a few useful configurations and behaviors as I'll explain next:

```swift
public struct DefaultDiscordClient: Sendable, DiscordClient {
    ...

    let client: HTTPClient
    let configuration: ClientConfiguration
    let cache: ClientCache

    ...
}
```

- The `cache` is a storage for the cached responses.
  The `configuration` has options to customize how caching is done.
  `ClientCache` is an actor, and it's shared across all `DefaultDiscordClient`s that have the same authorization token.

The configuration options:

```swift
public struct ClientConfiguration: Sendable {
    ...

    public let cachingBehavior: CachingBehavior
    public var retryPolicy: RetryPolicy?
    public var performValidations: Bool

    ...
}
```

- The `cachingBehavior` can be customized for each endpoint with a different time-to-live.
  Whenever there is a successful response, the discord-client will cache the response if allowed by this.
- `performValidations` specifies whether or not to perform client-side validations.
  This is what `ValidatablePayload` protocol is useful for, which adds a `validate()` func to payloads.
  The validations only cover what is mentioned in Discord docs. They don't guarantee a valid payload.
- And there is the `retryPolicy`, which specifies how to retry requests.
  DiscordBM has some magically-nice behavior built around `retryPolicy`, as I'll explain next.

`DefaultDiscordClient` takes help from a `HTTPRateLimiter` actor to handle rate-limits.
`HTTPRateLimiter` keeps track of bucket-infos that are available in request headers. Before each request there is a call to `HTTPRateLimiter` that asks "can I do a request now?" and the rate-limiter, considering Discord docs' notes about how exactly Discord rate-limits work, decides to allow a request or not.   
For example if a bucket is exhausted, the `HTTPRateLimiter` will have no choice but to reject the request, Right?   
That's not exactly what happens in DiscordBM. If there is a request that is disallowed by the bucket info, `HTTPRateLimiter` instead tries to return the time amount the `DiscordClient` needs to wait before making the request. Then `DiscordClient` takes a look at the `retryPolicy`, and if the `retryPolicy` specifies that requests failed with the `429 Too Many Requests` header can be retried based on headers, `DiscordClient` will just wait the time, and make the request after! This is best of the both worlds:

- Even if you make a lot of requests in a loop, `DiscordClient` not only won't fail, but also won't even let users notice anything. To users it will look like as if they don't even have a rate limit to worry about!
- On the other hand, `DiscordClient` didn't make a request that would have gotten a `429` back from Discord, so that's another win.

For all public API, DiscordBM comes with a lot of unit tests as well as integration tests.
This means:

- `BotGatewayManager` has [extensive tests](https://github.com/DiscordBM/DiscordBM/blob/main/Tests/IntegrationTests/GatwayConnection.swift) to make sure it connects and stays connected.
- Most of ~200 `DiscordClient` REST API functions [have a test](https://github.com/DiscordBM/DiscordBM/blob/main/Tests/IntegrationTests/DiscordClient.swift) that sends requests to Discord to make sure the functions do actually work in practice.

If you want to more about the DiscordBM's public APIs, please refer to the GitHub [README](https://github.com/DiscordBM/DiscordBM#readme).

## Missing Features

DiscordBM doesn't yet support Discord's Voice feature (e.g. joining a Discord server's voice channel and playing music). Discord's Voice seems to follows their own "standard" and DiscordBM will need to manually handle the UDP traffic etc... This feature can be added in a minor release without causing trouble.

## Versioning

DiscordBM will try to follow Semantic Versioning 2.0.0, with exceptions.
To keep DiscordBM up to date with Discord API's frequent changes, DiscordBM will _add_ any new properties to any types in _minor_ versions, even if it's technically a breaking change.
This includes adding new cases to enums. If you want to try to avoid breaking changes, make sure you have a `default` case in your `switch` statements or use `if case let`/`if case`.

### Initial committers (how long working on project)

The project has been public on Github for more than 11 months. Even before that it was being used as a private library in one of my projects, but ~80+ % of the development has happened after I made it public.

I am aware of how a project is supposed to make changes, and think it's a good thing to have a second contributor review your PRs before a merge and possibly catch some errors. That's why **I want to also ask for another maintainer for DiscordBM.** Anyone who's interested, please let me know here or anywhere else you can find me.
Optimally you should be a frequent user of Discord. If you have a bot built using DiscordBM, that would be even better.

## Maturity

I think DiscordBM can accept either of 'Sandbox' or 'Incubating' maturity levels. Please let me know what you think.
