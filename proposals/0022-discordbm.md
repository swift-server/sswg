# DiscordBM

* Proposal: [SSWG-0022](0022-discordbm.md)
* Authors: [Mahdi Bahrami](https://github.com/MahdiBM)
* Review Manager: TBD
* Status: Implemented; Pre-release stage
* Implementation: [DiscordBM](https://github.com/MahdiBM/DiscordBM)
* Forum Threads: [Pitch](https://forums.swift.org/t/pitch-discordbm-for-sswg-incubation-process/64833)

## Package Description
A New Multi-platform Swift Discord Library For Making Bots.

|  |  |
|--|--|
| **Package Name** | DiscordBM |
| **Module Name** | DiscordBM |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram) |
| **License** | Apache-2.0 |
| **Dependencies** | From Apple/SSWG: [swift-nio](https://github.com/apple/swift-nio.git), [swift-log](https://github.com/apple/swift-log.git), [swift-atomics](https://github.com/apple/swift-atomics.git), [swift-nio-ssl](https://github.com/apple/swift-nio-ssl.git), [swift-nio-transport-services](https://github.com/apple/swift-nio-transport-services.git), [async-http-client](https://github.com/swift-server/async-http-client.git), From Vapor: [multipart-kit](https://github.com/vapor/multipart-kit.git) (MIT License), Others: [Yams](https://github.com/jpsim/Yams.git) (Only for code generation. MIT License), [zlib](https://github.com/madler/zlib) (See License on Github)

## Introduction
DiscordBM is a new multi-platform Server-Side focused Swift library for interacting with the Discord API, for making bots.

We've been using it in Vapor's [Penny](https://github.com/vapor/penny-bot) bot for the past 8+ months, and it's been serving the community very nicely.

## Motivation

The Server-Side Swift community lacks a maintained and robust package to communicate with Discord.   
This is important because Discord is gaining more and more popularity amongst communities and has already replaced Slack as the go-to place for Open Source communities. A lot of successful projects and languages such as `Rust`, `Python`, `Vue` have chosen Discord as the place for their community.

## Proposed solution

DiscordBM provides an elegant, robust, type-safe, swifty and [insert positive word here] way to communicate with Discord's bot APIs. This can be useful to bridge the gap between the Swift community and Discord bots.

## Detailed design

DiscordBM is a big library (38K+ lines based on Github contribution stats) so it would be very difficult for me as a writer and you as readers, if I were to explain the library design in full. Regardless I'll try my best to go into implementation details without making the proposal way too long and boring.

DiscordBM not only comes with the core and necessary means to communicate with Discord, but also has a bunch of convenience or useful stuff that users might otherwise need to implement themselves. I'll try to explain the core implementation in detail, and briefly mention the rest.

### Core Of DiscordBM

The 2 most important concepts in DiscordBM are the `GatewayManager` and the `DiscordClient` protocols.   
Everything else in the library revolves around these two. Both of the protocols are just abstractions to make DiscordBM easier to test for users. The main types that conform to those protocols are `BotGatewayManager` and `DefaultDiscordClient`. The protocols are defined as below:

```swift
public protocol GatewayManager: AnyActor {
    /// The client to send requests to Discord with.
    nonisolated var client: any DiscordClient { get }
    /// This gateway manager's identifier.
    nonisolated var id: UInt { get }
    /// The current state of the gateway manager.
    nonisolated var state: GatewayState { get }
    /// The identification payload that is sent to Discord.
    nonisolated var identifyPayload: Gateway.Identify { get }
    
    /// Starts connecting to Discord.
    func connect() async
    
    /// Proposal Note: These 3 functions below are the only requests that users can send through Gateway.  
    /// The main route of sending requests to Discord is by using a DiscordClient and
    /// through one of the different ~200 REST API endpoints.

    /// https://discord.com/developers/docs/topics/gateway-events#request-guild-members
    func requestGuildMembersChunk(payload: Gateway.RequestGuildMembers) async
    /// https://discord.com/developers/docs/topics/gateway-events#update-presence
    func updatePresence(payload: Gateway.Identify.Presence) async
    /// https://discord.com/developers/docs/topics/gateway-events#update-voice-state
    func updateVoiceState(payload: VoiceStateUpdate) async

    /// Proposal Note: These two functions are to observe the communicated info between the GatewayManager and Discord.

    /// Makes an stream of Gateway events.
    func makeEventsStream() async -> AsyncStream<Gateway.Event>
    /// Makes an stream of Gateway event parse failures.
    func makeEventsParseFailureStream() async -> AsyncStream<(Error, ByteBuffer)>

    /// Disconnects from Discord.
    func disconnect() async
}

public protocol DiscordClient: Sendable {
    /// Your app's id.
    /// If you don't provide it here, DiscordBM will try to extract it from your token.
    /// If there is no `appId` you will need to provide it at all call-sites of
    /// `DiscordClient` functions that accept an `appId`.
    var appId: ApplicationSnowflake? { get }

    /// Send a request to Discord with no body.
    func send(request: DiscordHTTPRequest) async throws -> DiscordHTTPResponse

    /// Send a request to payload with a JSON body.
    func send<E: Sendable & Encodable & ValidatablePayload>(
        request: DiscordHTTPRequest,
        payload: E
    ) async throws -> DiscordHTTPResponse

    /// Proposal Note: DiscordBM uses multipart-kit to encode multipart payloads.
    /// Send a request to Discord with a Multipart body.
    func sendMultipart<E: Sendable & MultipartEncodable & ValidatablePayload>(
        request: DiscordHTTPRequest,
        payload: E
    ) async throws -> DiscordHTTPResponse
}
```

A `GatewayManager` is were apps start using DiscordBM. To initialize the `BotGatewayManager`, you need to have an `EventLoopGroup` and a `HTTPClient`: 
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
            presence: .init( /// Set up bot's initial presence
                /// Will show up as "Playing Swift"
                activities: [.init(name: "Swift", type: .game)], 
                status: .online,
                afk: false
            ),
            /// Add all the intents you want
            intents: [.guildMessages, .messageContent]
        )
        ...
    }
}
```
`token` is a "secret" by which `GatewayManager` will authenticate with Discord. `presence` and `intents` are 2 more Discord-related configuration options, and are not critical for understanding the library, so I'll skip explaining them to focus on what DiscordBM actually does, rather than explaining Discord's behaviors which are already well-documented on their documentation.   

Then, you would want to call `await bot.connect()`:

```swift
    ...
        /// Add all the intents you want
        intents: [.guildMessages, .messageContent]
    )

await bot.connect()
```

What happens is, the `GatewayManager` will start making a connection to the Discord Gateway using Websockets. For Websockets, DiscordBM uses a customized version of Vapor's `websocket-kit` which supports zlib decompression. After the websocket connection is established, Discord will send a 'hello' message to DiscordBM, and then DiscordBM sends an 'identify' payload to Discord which contains stuff like the `token`, `presence` and the `intents`.

In case Discord doesn't accept the identification, it will close the websocket connection with a proper close code. DiscordBM will catch that close code, and translate it to one of Discord's websocket close codes. If the code is a code that allows reconnection, DiscordBM will attempt that. Otherwise it will send a log message with `critical` level to users with a user-friendly description of the code, and instructions on how to try to solve the problem:
```
Will not reconnect because Discord does not allow it. Something is wrong. Your close code is '<CODE>', check Discord docs at https://discord.com/developers/docs/topics/opcodes-and-status-codes#gateway-gateway-close-event-codes and see what it means. Report at https://github.com/MahdiBM/DiscordBM/issues if you think this is a library issue
```

In the other case where Discord accepts the identification, it will send a 'ready' payload to the `GatewayManager`. After this the normal communication starts and Discord will send all the events related to the `intents` you passed to it. For more context, there are 70+ types of events and they include a created/updated/deleted/bulk-deleted message, a create/updated/deleted channel, and many more. DiscordBM allows user access to these events using an `AsyncStream<Gateway.Event>` which users can acquire and use like so:

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
* 1- The usual websocket ping-pongs.
* 2- By reconnecting when Discord closes a Websocket connection.   

Connection closures are normal and initiated by Discord when they see fit. The closures, however, aren't noticeable to most users at all. What happens is that Discord sends a 'reconnect' event to `GatewayManager`, and immediately closes the websocket connection. When this happens, `GatewayManager` reads the close code, and based on what the close code means, will likely attempt a 'resume' (or if not allowed, will just attempt a totally new connection as if you did `await bot.connect()`). The resume happens in a similar way that the identification happens for new connections. The 'resume' payload that is sent contains some info of the past connection which `GatewayManager` tracks, such as a `session-id` and a `sequence-number`.


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

`bot.client` is a `DiscordClient`. The REST API endpoints are all declared as extensions to the `DiscordClient` protocol like so:
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
These functions are for convenience of users and all of them declare an `endpoint` and pass all the info to the actual functions that make the requests, such as the `DiscordClient.sendMultipart(request:payload:)` function. The `sendMultipart` function here is the same as the one that a `DiscordClient` requires types to implement, but in the end it converts the response to a `DiscordClientResponse<TypeOfResponse>` type:

```swift
public extension DiscordClient {
    @inlinable
    func sendMultipart<E: Sendable & MultipartEncodable & ValidatablePayload, C: Codable>(
        request: DiscordHTTPRequest,
        payload: E
    ) async throws -> DiscordClientResponse<C> {
        let response = try await self.sendMultipart(request: request, payload: payload)
        return DiscordClientResponse(httpResponse: response)
    }
}
```

The whole point of `DiscordClientResponse` is the generic type that it takes. This makes it possible to decode Discord's responses to a library-defined type as you saw in `let message = try response.decode()`:

```swift
/// Represents a Discord HTTP response for endpoints that return some data in the body.
public struct DiscordClientResponse<C>: Sendable where C: Codable {
    /// The raw http response.
    public let httpResponse: DiscordHTTPResponse
    
    ...
    
    /// Decodes the response.
    @inlinable
    public func decode() throws -> C {
        try httpResponse.decode(as: C.self)
    }
}
```
The `DiscordHTTPResponse` type contains an HTTP response:

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
    
    let client: HTTPClient
    public let token: Secret
    public let appId: ApplicationSnowflake?
    let configuration: ClientConfiguration
    let cache: ClientCache

    ...
}
```
* The `client` is how DiscordBM sends the HTTP requests.
* The `token` is used in the authorization header only if required by the endpoint.   
  The `Secret` type is to prevent the full `token` ending up in the logs when using string-interpolation. 
* The `appId` is the app's identifier (also a bot's user-id that is accessible in Discord client-apps too).   
  It can be passed to the type just so some REST API functions don't require it at call-sites.
* And the `cache` is a storage for the cached responses.    
  The `configuration` has options to customize how caching is done.   
  `ClientCache` is an actor, and it's shared across all `DefaultDiscordClient`s that have the same `token`.

The configuration options:
```swift
public struct ClientConfiguration: Sendable {
    /// The behavior used for caching requests.
    public let cachingBehavior: CachingBehavior
    /// How much for the `HTTPClient` to wait for a connection before failing.
    public var requestTimeout: Duration
    /// Ask `HTTPClient` to log when needed. Defaults to no logging.
    public var enableLoggingForRequests: Bool
    /// Retries failed requests based on this policy.
    /// Defaults to `.default`.
    public var retryPolicy: RetryPolicy?
    /// Whether or not to perform validations for payloads, before sending.
    /// The point is to catch invalid payload without actually sending them to Discord.
    /// The library will throw a ``ValidationError`` if it finds anything invalid in the payload.
    /// This all works based on Discord docs' validation notes.
    public var performValidations: Bool

    ...
}
```

* The `cachingBehavior` can be customized for each endpoint with a different time-to-live.    
  Whenever there is a successful response, the discord-client will cache the response if allowed by this.
* `requestTimeout` and `enableLoggingForRequests` are `HTTPClient`-related and are passed to each request.
* `performValidations` specifies whether or not to perform client-side validations.     
  This is what `ValidatablePayload` protocol is useful for, which adds a `validate()` func to payloads.    
  The validations only cover what is mentioned in Discord docs. They don't guarantee a valid payload.
* And there is the `retryPolicy`, which specifies how to retry requests.    
  DiscordBM has some magically-nice behavior built around `retryPolicy`, as I'll explain next.

`DefaultDiscordClient` takes help from a `HTTPRateLimiter` actor to handle rate-limits.    
`HTTPRateLimiter` keeps track of bucket-infos that are available in request headers. Before each request there is a call to `HTTPRateLimiter` that asks "can I do a request now?" and the rate-limiter, considering Discord docs' notes about how exactly Discord rate-limits work, decides to allow a request or not. For example if a bucket is exhausted, the `HTTPRateLimiter` will have no choice but to reject the request. 
Well, that's true, but it's not exactly what happens in DiscordBM. If there is a request that is disallowed by the bucket info, `HTTPRateLimiter` instead tries to return the time amount the `DiscordClient` needs to wait before making the request. Then `DiscordClient` takes a look at the `retryPolicy`, and if the `retryPolicy` specifies that requests failed with the `429 Too Many Requests` header can be retried based on headers, `DiscordClient` will just wait the time, and make the request after! This is best of the both worlds:  
* Even if you make a lot of requests in a loop, `DiscordClient` not only won't fail, but also won't even let users notice anything suspicious. To users it will look like as if they don't even have a rate limit to worry about!    
* On the other hand, `DiscordClient` didn't make a request that would have gotten a `429` back from Discord, so that's another win.

For all these core functionalities, DiscordBM comes with a lot of integration tests.
This means:
* `BotGatewayManager` has [extensive tests](https://github.com/MahdiBM/DiscordBM/blob/main/Tests/IntegrationTests/GatwayConnection.swift) to make sure it connects and stays connected.
* Each of ~200 `DiscordClient` REST API functions [have a test](https://github.com/MahdiBM/DiscordBM/blob/main/Tests/IntegrationTests/DiscordClient.swift) that sends requests to Discord to make sure the functions do actually work. 

### Utilities Built Around The Core
DiscordBM contains a bunch of other types that are very useful to users:
* `DiscordCache`
  * Caches the Discord Gateway events and keeps them in sync with Discord. 
  * Why is it needed at all? Because Gateway events are sometimes compliments of each other, and don't repeat the old info that has already been sent. For example, assuming you have the `guilds` and `guildEmojisAndStickers` intents enabled, after the connection to the Gateway, Discord will send you "Guild Create" events containing the info of the guilds that you are joined in. Later, when for example a new emoji is added to the guild, Discord will only send you the new emoji info in a "Guild Emojis Update" event instead of re-sending the whole guild again.
  * Caching of Gateway events is tricky to implement correctly because not only there are a lot of events, but also there are a lot of edge cases to be handled. 
  * Having this functionality built-in allows DiscordBM for integrations of it with other functionalities of DiscordBM. For example you can pass a `DiscordCache` to "React-To-Role" handlers and the handler will use the cache instead of doing API calls. DiscordBM can also do some best-effort permission checking of server members thanks to the `DiscordCache` which allows for real-time up-to-date access to a guild's info.
* `DiscordLogger`
  * Sends your logs to Discord, carefully handling edge cases such as Discord rate limits (thanks to the `HTTPRateLimiter`), Discord message length limits, escaping all messages, etc...
  * The logs utilize Discord's APIs, and look nice in chats. They are not just raw text.   
  * `DiscordLogger` logs both to stdout and Discord, since logging over the wire is much more flaky than logging to disk, and you can't send lengthy messages to Discord in full.
  * It comes with a lot of useful configuration options, indluding to customize what is sent to Discord, and what only ends up in the on-disk logs.
  * This can possibly end up being its own package.  
* `ReactToRoleHandler`
  * Assigns a role to members when they react to a message.
  * This too can possibly end up being its own package. It can very helpful to a few people, but just useless for others.

## Missing Features
DiscordBM doesn't yet support Discord's Voice feature (e.g. joining a Discord server's voice channel and playing music). Discord's Voice seems follows their own "standard" and DiscordBM will need to manually handle the UDP traffic etc... This feature can be added in a minor release without causing trouble. 

## Versioning
DiscordBM will try to follow Semantic Versioning 2.0.0, with exceptions.     
To keep DiscordBM up to date with Discord API's frequent changes, DiscordBM will _add_ any new properties to any types in minor versions, even if it's technically a breaking change.   
This includes adding new cases to enums. If you want to try to avoid breaking changes, make sure you have a `default` case in your `switch` statements or use `if case let`/`if case`.

### Initial committers (how long working on project)
The project has been public on Github for more than 8 months. Even before that it was being used as a private library in one of my projects, but ~80+ % of the development has happened after I made it public.      
99.9+ % of the project is just written by me, but there have been 2 other contributors, one of them being Gwynne Raskind.   

I am aware of how a project is supposed to make changes, and think it's a good thing to have a second contributor review your PRs before a merge and possibly catch some errors. That's why **I want to also ask for another maintainer for DiscordBM.** Anyone who's interested, please let me know in the Swift forums / Vapor's Discord server / Email / Twitter / Mastodon. The related links are available on my Github profile.

## Maturity

Since the library is still in beta, I'd like the 'Sandbox' maturity level.   
After it is out of beta, I can ask SSWG to move it to the 'Incubating' level if appropriate.   
To clarify, there are no major changes planned for the library (only 1 minor planned). I see this beta stage as an opportunity to have little refinements/tweaks to the package if suggested/requested, and can have a release if needed.
