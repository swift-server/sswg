# Hummingbird

* Proposal: [SSWG-0032](0032-hummingbird.md)
* Authors: [Adam Fowler](https://github.com/adam-fowler), [Joannis Orlandos](https://github.com/Joannis)
* Review Manager: [Tim Condon](https://github.com/0xTim)
* Status: **Implemented**
* Implementation: [Hummingbird](https://github.com/hummingbird-project/hummingbird)
* Forum Threads:
  * [Pitch](https://forums.swift.org/t/pitch-hummingbird/70700)

<!-- *During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Previous Revision(s): [1](https://github.com/swift-server/sswg/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal(s): [SSWG-XXXX](XXXX-filename.md)
-->

## Package Description
Lightweight, modular, modern, extensible HTTP server framework written in Swift.

|  |  |
|--|--|
| **Package Name** | `hummingbird` |
| **Module Name** | `Hummingbird` |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [Apache 2.0](https://choosealicense.com/licenses/apache-2.0/) |
| **Dependencies** | [swift-async-algorithms](https://github.com/apple/swift-async-algorithms), [swift-atomics](https://github.com/apple/swift-atomics), [swift-distributed-tracing](https://github.com/apple/swift-distributed-tracing), [swift-http-types](https://github.com/apple/swift-http-types), [swift-log](https://github.com/apple/swift-log), [swift-metrics](https://github.com/apple/swift-metrics), [swift-nio](https://github.com/apple/swift-nio), [swift-nio-http2](https://github.com/apple/swift-nio-http2), [swift-nio-extras](https://github.com/apple/swift-nio-extras), [swift-nio-ssl](https://github.com/apple/swift-nio-ssl), [swift-nio-transport-services](https://github.com/apple/swift-nio-transport-services), [swift-service-lifecycle](https://github.com/swift-server/swift-service-lifecycle) |

## Introduction

Hummingbird is a HTTP server framework. It consists of a core HTTP server, web application framework with a router and middleware and various modules providing additional functionality such as authentication, websockets, and integration with Redis and Fluent.

## Motivation

HTTP servers come in many forms, they can be simple HTTP1 servers with a few routes, HTTP proxies, file servers to complex servers supporting a number of protocols and talking to many other services.

Providing a solution that covers all of these cases is desirable. But when you build your simple HTTP1 server do you want to be including code for a bunch of protocols and services you are never going to use? At the same time you want these available should you ever need them.

## Proposed solution

Hummingbird is a lightweight, modular, flexible HTTP server framework. It provides a server framework to build lightweight servers but can be extended to support as complex a server as you require. Don't want TLS, HTTP2, WebSockets don't include them. But they are available if you require them.

## Detailed design

Hummingbird v2.0 is currently under development and this is what we are proposing. Version 2.0 brings a complete Swift concurrency based solution based off SwiftNIO's `NIOAsyncChannel` bringing all the benefits of structured concurrency including task cancellation, task locals and local reasoning. It also uses the [new HTTP types](https://github.com/apple/swift-http-types) that Apple recently published and integrates with the recently released structured concurrency based version of [ServiceLifecycle](https://github.com/swift-server/swift-service-lifecycle).

The Hummingbird web application framework is broken into three main components. The router `Router`, the server `Server` and the application framework `Application` which provides the glue between the server and the router. A simple Hummingbird application involves setting up a router and building an application from that router. 

```swift
import Hummingbird

// construct router with one route that returns the string "Hello" in
// its response
let router = Router()
router.get{"hello"} { request, context in
    "Hello"
}
// construct application which 
let application = Application(
    router: router,
    configuration: .init(address: .hostname("127.0.0.1", port: 8080))
)
try await application.runService()
```

### Service Lifecycle

`Application` conforms to the Swift Service Lifecycle protocol `Service` so can be included in the list of services controlled by a `ServiceGroup`. In actual fact `Application.runService` is a shortcut to setting up a `ServiceGroup` with just `Hummingbird` and running it.

### Router

The router has functions to setup routes using the most common HTTP verbs

```swift
router.put("todo") { _,_ in }
router.post("login") { _,_ in }
router.on("test", .connect) {_,_ in }
...
```

Parameters can be extracted from the URI path when matching routes. 

```swift
router.get("todo/{id}") { request, context in
    let id = try context.parameters.require("id", as: Int.self)
    return try await getTodo(id)
}
```

By default the request body is a stream of NIO `ByteBuffers`. 
```swift
router.post("upload") { request,_ -> HTTPResponse.Status in 
    for try await buffer in request.body {
        processBuffer(buffer)
    }
    return .ok
}
```
If you need your request body to be collated into one ByteBuffer you can use `request.body.collect(upTo:)`.

### Request decoding, response encoding

Any type that conforms to `ResponseGenerator` can be returned from a route. These include `String`, `ByteBuffer` and `HTTPResponse.Status`. Types that conform to `ResponseEncodable` automatically conform to `ResponseGenerator` and will use Codable to generate a response.

```swift
struct User: ResponseEncodable {
    let name: String
}
router.get("user/{id}") { request, context in
    let id = try context.parameters.require("id", as: Int.self)
    let user = try await getUser(id)
    return User(name: user.name)
}
```

Similarly any type that conforms to `Decodable` can be extracted from the request body. 

```swift
struct SignUpRequest: Decodable {
    let name: String
}
router.put("user") { request, context -> HTTPResponse.Status in
    let user = try await request.decode(as: SignUpRequest.self, using: context)
    try await createUser(user)
    return .created
}
```

### Middleware

You can add middleware to the router that is run on all routes. The framework has a number of middleware already implemented 
- `LogRequestMiddleware`: Logging of requests
- `MetricsMiddleware`: Provide metrics for each request
- `TracingMiddleware`: Create distributed tracing spans for each request. 
- `FileMiddleware`: Serve static files
- `CORSMiddleware`: Implementing Cross-Origin Resource Sharing (CORS) headers

```swift
router.middlewares.add(LogRequestMiddleware(.debug))
```

Middleware can also be applied to a group of routes instead of all routes

```swift
router.group("todo")
    .add(middleware: MyMiddleware())
    .get() { _,_ in
        return getTodo()
    }
    .put() { request,_ in
        return putTodo(request)
    }
```

### Request context

The route handler and middleware functions all include a second parameter. This provides contextual information that is passed along with the request type. This is a generic type and the user can set it to be any type they want as long as if conforms to the protocol `RequestContext`. The router defaults to using `BasicRequestContext` which is provided out of the box.

If you create your own context type you can add additional information to it, or change its default values. For example the Hummingbird authentication framework requires that you use a custom request context that includes login information.

Below is a request context that includes an additional `string`` value which can be edited in a middleware and passed on to the eventual route handler.

```swift
public struct MyRequestContext: RequestContext {
    /// Initializer required by `RequestContext`
    public init(channel: Channel, logger: Logger) {
        self.coreContext = .init(allocator: channel.allocator, logger: logger)
        self.string = ""
    }

    /// member variable required by `RequestContext`
    public var coreContext: CoreRequestContext

    /// my additional data
    public var string: String
}
```

To use this context you need to provide the type in the router initializer.

```swift
let router = Router(context: MyRequestContext.self)
```

### TLS/HTTP2

When initializing an `Application` you can include a parameter to indicate the type of server you want. This defaults to `.http1()`. But servers with TLS, HTTP2 support are available and WebSocket support will be available soon.

These are all added in similar ways. Import library and include `server` parameter in `Application.init` 

```swift
import HummingbirdHTTP2

let app = Application(
    router: router
    server: .http2(tlsConfiguration: myTLSConfig)
)
app.runService()
```

The TLS server type can be used to wrap any other server type.

```swift
import HummingbirdTLS

let app = Application(
    router: router
    server: .tls(.http1(), tlsConfiguration: myTLSConfig)
)
```

### Additional support

Hummingbird also comes along with a series of other packages. Providing an authentication framework, integration with [RediStack](https://github.com/swift-server/RediStack), WebSocket support (in development), integration with Vapor's [FluentKit](https://github.com/Vapor/fluent-kit), Mustache templating and a AWS Lambda framework.

## Maturity Justification

Sandbox. While Hummingbird adheres to the minimal requirements of the [SSWG Incubation Process](https://github.com/swift-server/sswg/blob/master/process/incubation.md#minimal-requirements) the package is going through a major re-write and should be considered completely new. 

## Alternatives considered

Vapor: You cannot discuss server frameworks and Swift without mentioning Vapor. It is the defacto solution that most people will turn to. Vapor is a great solution and provides everything that most people will ever need but it does that in one package. Hummingbird was written as a response to this. I thought a more modular solution should also be available.
