# Solution name

* Proposal: [SSWG-NNNN](NNNN-hummingbird.md)
* Authors: [Adam Fowler](https://github.com/adam-fowler)
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
Lightweight, modular, extensible HTTP server framework written in Swift.

|  |  |
|--|--|
| **Package Name** | `hummingbird` |
| **Module Name** | `Hummingbird` |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [Apache 2.0](https://choosealicense.com/licenses/apache-2.0/) |
| **Dependencies** | [swift-atomics](https://github.com/apple/swift-atomics), [swift-distributed-tracing](https://github.com/apple/swift-distributed-tracing), [swift-log](https://github.com/apple/swift-log), [swift-metrics](https://github.com/apple/swift-metrics), [swift-nio](https://github.com/apple/swift-nio), [swift-nio-http2](https://github.com/apple/swift-nio-http2), [swift-nio-extras](https://github.com/apple/swift-nio-extras), [swift-nio-ssl](https://github.com/apple/swift-nio-ssl), [swift-nio-transport-services](https://github.com/apple/swift-nio-transport-services), [swift-service-lifecycle](https://github.com/swift-server/swift-service-lifecycle) |

## Introduction

Hummingbird is a HTTP server framework. It consists of a core HTTP server, web application framework with a router and middleware and various modules providing additional functionality such as authentication, websockets, and integration with Redis and Fluent.

## Motivation

HTTP servers come in many forms, they can be simple HTTP1 servers with a few routes, HTTP proxies, file servers to complex servers supporting a number of protocols and talking to many other services.

Providing a solution that covers all of these cases is desirable. But when you build your simple HTTP1 server do you want to be including code for a bunch of protocols and services you are never going to use? At the same time you want these available should you ever need them.

## Proposed solution

Hummingbird is designed in a modular format. It provides a server framework to build lightweight servers but can be extended to support as complex a server as you require. Don't want TLS, HTTP2, WebSockets don't include them. But they are available if you require them.

## Detailed design

The Hummingbird web application framework is managed by the `HBApplication` type. This gives you access to the server, router, middleware chain and can be extended with your own types as well. A simple hummingbird application could be setup as follows

```swift
import Hummingbird

let app = HBApplication(configuration: .init(address: .hostname("127.0.0.1", port: 8080)))
// add route that returns the string "Hello"
app.router.get("hello") { request -> String in
    return "Hello"
}
try app.start()
app.wait()
```

### Router

The router has functions to setup routes using all the standard HTTP verbs

```swift
app.router.put("todo") { _ in }
app.router.get("todo/:id") { _ in }
app.router.post("login") { _ in }
...
```

Routes can be flagged as using streamed request bodies
```swift
app.router.post("upload", options: .streamBody) { request -> HTTPResponseStatus in 
    guard let stream = request.body.stream else { return .ok }
    for try await buffer in stream.sequence {
        processBuffer(buffer)
    }
    return .ok
}
```

Any type that conforms to `HBResponseGenerator` can be returned from a route. These include `String`, `ByteBuffer` and `HTTPResponseStatus`. Types that conform to `HBResponseEncodable` automatically conform to `HBResponseGenerator` and will use the Codable encoder attached to `HBApplication` to generate a response.

```swift
struct User: HBResponseEncodable {
    let name: String
}
app.router.get("user/:id") { request in
    let id = try request.parameters.require("id", as: Int.self)
    let user = try await getUser(id)
    return User(name: user.name)
}
```

### Middleware

From `HBApplication` you can also add middleware that is run on all routes. The framework has a number of middleware already implemented 
- `HBLogRequestMiddleware`: Logging of requests
- `HBMetricsMiddleware`: Provide metrics for each request
- `HBTracingMiddleware`: Create distributed tracing spans for each request. 
- `HBCORSMiddleware`: Implementing Cross-Origin Resource Sharing (CORS) headers

```swift
app.middleware.add(HBLogRequestMiddleware(.debug))
```

Middleware can also be applied to a group of routes instead of all routes

```swift
app.router.group("todo")
    .add(middleware: MyMiddleware())
    .get() { _ in
        return getTodo()
    }
    .put() { request in
        return putTodo(request)
    }
```

### HummingbirdFoundation

The Hummingbird library makes no use of Foundation to further reduce the size of applications built with Hummingbird, but there are many features that cannot be built without the functionality provided by Foundation. These include JSON and URLEncodedForm encoding and decoding, file serving and Cookie parsing. You can find support for these in the `HummingbirdFoundation` library. 

### TLS/HTTP2/WebSockets

These are all added in similar ways. Import library, call `add` function before starting the server

```swift
import HummingbirdHTTP2

let app = HBApplication()
try app.server.addHTTP2Upgrade(tlsConfiguration: myTLSConfig)
try app.start()
app.wait()
```

WebSockets have their own repository [hummingbird-websockets](https://github.com/hummingbird-project/hummingbird-websockets) and are added in a similar manner to above except you also need to add a route for websocket setup

```swift
// add HTTP to WebSocket upgrade
app.ws.addUpgrade(maxFrameSize: 1<<14)
// on WebSocket connect.
self.ws.on(
    "/ws",
    shouldUpgrade: { _ in return nil },
    onUpgrade: { request, ws -> HTTPResponseStatus in
        // send ping and wait for pong and repeat every 60 seconds
        ws.initiateAutoPing(interval: .seconds(60))
        ws.onRead { frame, ws in
            // echo frame read back to client
            ws.write(frame, promise: nil)
        }
        return .ok
    }
)
```
### Documentation

Above only covers a small amount of what is possible. You can find documentation covering most of the Hummingbird libraries [here](https://hummingbird-project.github.io/hummingbird-docs/documentation/hummingbird/).

## Maturity Justification

Sandbox. While Hummingbird adheres to the minimal requirements of the [SSWG Incubation Process](https://github.com/swift-server/sswg/blob/master/process/incubation.md#minimal-requirements) the package up until very recently has had only one maintainer. 

## Alternatives considered

Vapor: You cannot discuss server frameworks and Swift without mentioning Vapor. It is the defacto solution that most people will turn to. Vapor is a great solution and provides everything that most people will ever need but it does that in one package. Hummingbird was written as a response to this. I thought a more modular solution should also be available.
