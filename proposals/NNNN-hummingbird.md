# Hummingbird

* Proposal: [SSWG-NNNN](NNNN-hummingbird.md)
* Authors: [Adam Fowler](https://github.com/adam-fowler), [Joannis Orlandos](https://github.com/Joannis)
* Review Manager: TBD
* Status: **Implemented**
* Implementation: [Hummingbird](https://github.com/hummingbird-project/hummingbird)
* Forum Threads:

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

Hummingbird is designed in a modular format. It provides a server framework to build lightweight servers but can be extended to support as complex a server as you require. Don't want TLS, HTTP2, WebSockets don't include them. But they are available if you require them.

## Detailed design

Hummingbird v2.0 is curently being developed and this is what I am proposing. Version 2.0 brings a complete Swift concurrency based solution based off SwiftNIO's `NIOAsyncChannel` bringing all the benefits of structured concurrency including task cancellation, task locals and local reasoning. It also uses the [new HTTP types](https://github.com/apple/swift-http-types) that Apple recently published.

The Hummingbird web application framework is broken into three main components. The router `HBRouter`, the server `HBServer` and the application framework `HBApplication` which provides the glue between the server and the router. A simple Hummingbird application could be setup as follows

```swift
import Hummingbird

// construct router with one route that returns the string "Hello" in
// its response
let router = HBRouter()
router.get{"hello"} { request, context in
    "Hello"
}
// construct application which 
let application = HBApplication(
    router: router,
    configuration: .init(address: .hostname("127.0.0.1", port: 8080))
)
try await application.runService()
```

### Service Lifecycle

`HBApplication` conforms to the Swift Service Lifecycle protocol `Service` so can be included in the list of services controlled by a `ServiceGroup`. In actual fact `HBApplication.runService` is a shortcut to setting up a `ServiceGroup` with just `Hummingbird` and running it.

### Router

The router has functions to setup routes using all the common HTTP verbs

```swift
router.put("todo") { _,_ in }
router.get("todo/:id") { _,_ in }
router.post("login") { _,_ in }
...
```

By default the request body is a stream of NIO `ByteBuffers`. 
```swift
router.post("upload") { request,_ -> HTTPResponseStatus in 
    for try await buffer in request.body {
        processBuffer(buffer)
    }
    return .ok
}
```
If you need your request body to be collated into one ByteBuffer you can use `request.body.collect(upTo:)`.

### Request decoding, response encoding

Any type that conforms to `HBResponseGenerator` can be returned from a route. These include `String`, `ByteBuffer` and `HTTPResponse.Status`. Types that conform to `HBResponseEncodable` automatically conform to `HBResponseGenerator` and will use Codable to generate a response.

```swift
struct User: HBResponseEncodable {
    let name: String
}
router.get("user/:id") { request, context in
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
- `HBLogRequestMiddleware`: Logging of requests
- `HBMetricsMiddleware`: Provide metrics for each request
- `HBTracingMiddleware`: Create distributed tracing spans for each request. 
- `HBCORSMiddleware`: Implementing Cross-Origin Resource Sharing (CORS) headers

```swift
router.middlewares.add(HBLogRequestMiddleware(.debug))
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

You will notice in all the examples above the route handler includes a second parameter. This provides contextual information that is passed along with the request type, for example a Logger, ByteBufferAllocator, Codable Encoder and Decoder instances. The router defaults to using `HBBasicRequestContext` for its context type but you can provide your own context type as long as it conforms to the `HBRequestContext` protocol.

If you create your own context type you can add additional information to it, or change its default values. For example the Hummingbird authentication framework requires that you use a custom request context that includes login information.

Below is a request context that includes an additional `string`` value which can be edited in a middleware and passed on to the eventual route handler.

```swift
public struct MyRequestContext: HBRequestContext {
    /// Initializer required by `HBRequestContext`
    public init(allocator: ByteBufferAllocator, logger: Logger) {
        self.coreContext = .init(allocator: allocator, logger: logger)
        self.string = ""
    }

    /// parameter required by `HBRequestContext`
    public var coreContext: HBCoreRequestContext

    /// my additional data
    public var string: String
}
```

### HummingbirdFoundation

Currently the Hummingbird library makes no use of Foundation to further reduce the size of applications built with Hummingbird, but there are many features that cannot be built without the functionality provided by Foundation. These include JSON and URLEncodedForm encoding and decoding, file serving and Cookie parsing. You can find support for these in the `HummingbirdFoundation` library.

I am currently considering whether to allow the core Hummingbird module to include `FoundationEssentials`. Depending on what is eventually included in `FoundationEssentials` I might be able to merge the whole of `HummingbirdFoundation` back into `Hummingbird`. 

### TLS/HTTP2

The initializer for `HBApplication` includes a few parameters not included in the example at the top. One of these is a `server` parameter which indicates what kind of server you want to run. This defaults to `.http1()`. But servers with TLS, HTTP2 and WebSocket support are also available.

These are all added in similar ways. Import library and include `server` parameter in `HBApplication.init` 

```swift
import HummingbirdHTTP2

let app = HBApplication(
    router: router
    server: .http2(tlsConfiguration: myTLSConfig)
)
app.runService()
```

The TLS server type can be used to wrap any other server type.
```swift
import HummingbirdTLS

let app = HBApplication(
    router: router
    server: .tls(.http1(), tlsConfiguration: myTLSConfig)
)
```

### Additional support

Hummingbird also comes along with a series of other packages. Providing an authentication middleware, integration with [RediStack](https://github.com/swift-server/RediStack), WebSocket support, integration with Vapor's [FluentKit](https://github.com/Vapor/fluent-kit), Mustache templating and a AWS Lambda framework.

## Maturity Justification

Sandbox. While Hummingbird adheres to the minimal requirements of the [SSWG Incubation Process](https://github.com/swift-server/sswg/blob/master/process/incubation.md#minimal-requirements) the package is going through a major re-write and should be considered completely new. 

## Alternatives considered

Vapor: You cannot discuss server frameworks and Swift without mentioning Vapor. It is the defacto solution that most people will turn to. Vapor is a great solution and provides everything that most people will ever need but it does that in one package. Hummingbird was written as a response to this. I thought a more modular solution should also be available.