# Vapor

* Proposal: [SSWG-NNNN](NNNN-vapor.md)
* Authors: [Tim Condon](https://github.com/0xTim), [Gwynne Raskind](https://github.com/gwynne)
* Review Manager: Adam Fowler
* Status: **Implemented**
* Implementation: [vapor/vapor](https://github.com/vapor/vapor)
* Forum Threads: [Proposal - TBD](), [Discussion](https://forums.swift.org/), [Review](https://forums.swift.org/)

## Package Description

[Vapor](https://www.vapor.codes) is a framework for building server applications, APIS and backends in Swift.

|  |  |
|--|--|
| **Package Name** | [`vapor`](https://github.com/vapor/vapor) |
| **Module Name** | `Vapor` (Main library), `XCTVapor` (Testing Library) |
| **Proposed Maturity Level** | [Incubating](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [MIT](https://github.com/vapor/vapor/blob/main/LICENSE) |
| **Dependencies** | [AsyncHTTPClient](https://github.com/swift-server/async-http-client.git) (v1), [AsyncKit](https://github.com/vapor/async-kit.git) (v1), [ConsoleKit](https://github.com/vapor/console-kit.git) (v4), [Swift Crypto](https://github.com/apple/swift-crypto.git) (v1, v2, v3, v4), [RoutingKit](https://github.com/vapor/routing-kit.git) (v4), [SwiftNIO](https://github.com/apple/swift-nio.git) (v2), [SwiftNIO SSL](https://github.com/apple/swift-nio-ssl.git) (v2), [SwiftNIO HTTP2](https://github.com/apple/swift-nio-http2.git) (v1), [SwiftNIO Extras](https://github.com/apple/swift-nio-extras.git) (v1), [Swift Log](https://github.com/apple/swift-log.git) (v1), [Swift Metrics](https://github.com/apple/swift-metrics.git) (v2), [Swift Algorithms](https://github.com/apple/swift-algorithms.git) (v1), [WebsocketKit](https://github.com/vapor/websocket-kit.git) (v2), [MultipartKit](https://github.com/vapor/multipart-kit.git) (v4), [Swift Atomics](https://github.com/apple/swift-atomics.git) (v1) |

## Introduction

[Vapor](https://www.vapor.codes) is a full-featured framework for building server applications and API in Swift. It is a mature framework that has been around since 2016 and has build a large engaged community around it. Vapor provides everything you need to build a server-side Swift application. The main framework provides HTTP routing, middleware, sessions, authentication, websockets, and more. Vapor also provides a suite of packages for working with databases, templating, and other common tasks.

## Motivation

HTTP libraries are a core component of any server-side Swift application. Vapor provides a high-level API for working with HTTP requests and responses to make it easy for developers to build full-featured server applications. It has been around since 2016 and built a large community around it. We believe it should be part of the incubation process to further integrate it into the ecosystem.

## Proposed solution

We propose that the existing Vapor library be promoted to an officially supported package as is.

## Detailed design

### Basic Routing

Routes can be defined on anything that conforms to `RoutesBuilder`, nominally the main `Application`:

```swift
app.get("hello") { req in
    "Hello, world!"
}
```

Routing supports all standard HTTP methods, such as a **POST** request:

```swift
app.post("info") { req -> Info in
    let data = try req.content.decode(Info.self)
    return data
}
```

Dynamic path parameters are supported as well:

```swift
app.get("users", ":userID") { req -> User in
    let userID = req.parameters.get("userID")!
    return User(id: userID)
}
```

These examples also demonstrate how to return responses and decode request bodies. Vapor has extensive [documentation](https://docs.vapor.codes) on all of the supported features and [API Documentation](https://api.vapor.codes) for Vapor's API.

## Maturity Justification

We are proposing this package at the "Graduated" level of maturity. Vapor has a stable and mature API and ecosystem, and is widely used by many companies. It also has a large pool of contributors as well as an established Core Team that is responsible for the project's direction and stability.

## Alternatives considered

There are a number of alternative frameworks for writing APIs in Swift and users should choose the framework that best suits their needs.
