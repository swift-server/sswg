# SSWG HTTP Client Library

> **Note**: Since this proposal was published, the module `NIOHTTPClient` has been renamed to `AsyncHTTPClient`. The renamed GitHub repo is found at https://github.com/swift-server/async-http-client/
> 
> For more context behind this change, see the following threads: [[1]](https://forums.swift.org/t/namespacing-of-packages-modules-especially-regarding-swiftnio/24726), [[2]](https://forums.swift.org/t/sswg-minimum-requirements-to-require-no-existing-clashes/24932), [[3]](https://forums.swift.org/t/feedback-nio-based-http-client/26149/27)

* Proposal: SSWG-0005
* Authors: [Artem Redkin](https://github.com/artemredkin), [Tomer Doron](https://github.com/tomerd), [Tanner Nelson](https://github.com/tanner0101), [Ian Partridge](https://github.com/ianpartridge/)
* Sponsors: Swift Server Working Group
* Status: **Accepted as Sandbox Maturity**
* Implementation: [swift-server/async-http-client](https://github.com/swift-server/async-http-client/)
* Forum Threads: [Pitch](https://forums.swift.org/t/generic-http-client-library-pitch/23341), [Discussion](https://forums.swift.org/t/discussion-nio-based-http-client/24195/), [Review](https://forums.swift.org/t/feedback-nio-based-http-client/26149)
* Decision Notes: [Rationale](https://forums.swift.org/t/june-27th-2019/26580)

## Introduction
Number of projects implemented their own HTTP client libraries, like:
 - [IBM-Swift](https://github.com/IBM-Swift/Kitura-NIO/blob/master/Sources/KituraNet/HTTP/HTTPServer.swift)
- [vapor](https://github.com/vapor/http/blob/master/Sources/HTTP/Responder/HTTPClient.swift)
- [smoke-framework](https://github.com/amzn/smoke-http/blob/master/Sources/SmokeHTTPClient/HTTPClient.swift)

This shows that there is a need for generic, multi-purpose, non-blocking, asynchronous HTTP client library built on top of `SwiftNIO`. SSWG aims to provide a number of packages that could be shared between different projects, and I think proposed HTTP client library would be a good fit for those projects and other use-cases.

## Motivation
Having one, community-driver project that can be shared between different projects will hopefully solve the need to implement this functionality in every project from scratch.

## Proposed solution
Proposed solution is to have a `HTTPClient` class, that support a number of often-used methods, as well as an ability to pass a delegate for more precise control over the HTTP data transmission:
```swift
class HTTPClient {

    public func get(url: String, timeout: Timeout? = nil) -> EventLoopFuture<Response> {
    }

    public func post(url: String, body: Body? = nil, timeout: Timeout? = nil) -> EventLoopFuture<Response> {
    }

    public func put(url: String, body: Body? = nil, timeout: Timeout? = nil) -> EventLoopFuture<Response> {
    }

    public func delete(url: String, timeout: Timeout? = nil) -> EventLoopFuture<Response> {
    }

    public func execute(request: Request, timeout: Timeout? = nil) -> EventLoopFuture<Response> {
    }

    public func execute<T: HTTPClientResponseDelegate>(request: Request, delegate: T, timeout: Timeout? = nil) -> Task<T.Response> {
    }
}
```
For the first release we have the following features implemented:
1. Simple follow-redirects (cookie headers are dropped)
2. Streaming body download
3. TLS support
4. Cookie parsing (but not storage)

## Detailed design

### Lifecycle
Creating a client is strait-forward:
```swift
let client = HTTPClient(eventLoopGroupProvider: .createNew)
```
This initializer will create new `EventLoopGroup`, so every client instance will have it's own separate `EventLoopGroup`. Alternatively, users can supply shared `EventLoopGroup`:
```swift
let group = MultiThreadedEventLoopGroup(numberOfThreads: 1)
let client = HTTPClient(eventLoopGroupProvider: .shared(group))
```
It is important to shutdown your client instance after it is no longer needed, in order to shutdown internal `SwiftNIO` machinery:
```swift
try client.syncShutdown()
```
In case `EventLoopGroup` was provided to the client instance, there is no need to shutdown the client, we expect that lifecycle of that group will be controlled by its owner.

### Request
In case helper methods do not provide required functionality (for example, if user needs to set headers, or use specific HTTP method), clients of the library can use `HTTPClient.Request`:
```swift
extension HTTPClient {
    typealias ChunkProvider = (@escaping (IOData) -> EventLoopFuture<Void>) -> EventLoopFuture<Void>

    struct Body {
        var length: Int?
        var provider: HTTPClient.ChunkProvider

        static func byteBuffer(_ buffer: ByteBuffer) -> Body
        static func stream(length: Int? = nil, _ provider: @escaping HTTPClient.ChunkProvider) -> Body
        static func data(_ data: Data) -> Body
        static func string(_ string: String) -> Body
    }

    struct Request {
        public var version: HTTPVersion
        public var method: HTTPMethod
        public var url: URL
        public var headers: HTTPHeaders
        public var body: Body?

        public init(url: String,
                    version: HTTPVersion = HTTPVersion(major: 1, minor: 1),
                    method: HTTPMethod = .GET,
                    headers: HTTPHeaders = HTTPHeaders(),
                    body: Body? = nil) throws {}
    }
}
```
Example:
```swift
var request = try HTTPClient.Request(url: "http://swift.org")
request.headers.add(name: "User-Agent", value: "nio-http-client")

let future = client.execute(request: request)
```

### Response
`HTTPClient`'s methods return an `EventLoopFuture<HTTPClient.Reponse>`. This struct is defined as follows:
```swift
extension HTTPClient {
    struct Response {
        public var host: String
        public var status: HTTPResponseStatus
        public var headers: HTTPHeaders
        public var body: ByteBuffer?
    }
}
```
where `HTTPResponseStatus` is an enum that describes HTTP codes that could be returned by the server:
```swift
client.get(url: "http://swift.org").whenSuccess { response in
    switch response.status {
        case .ok: print("server return 200 OK")
        case .notFound: print("server return 404 Not Found")
        case .internalServerError: print("server return 500 Internal Server Error")
        ...
    }
}
```

### HTTPClientResponseDelegate
In addition to helper/request methods, library also provides the following delegate, which provides greater control over how HTTP response is processed:
```swift
public protocol HTTPClientResponseDelegate: class {
    associatedtype Response

    // this method will be called when request body is sent  
    func didTransmitRequestBody(task: HTTPClient.Task<Response>)

    // this method will be called when we receive Head response, with headers and status code
    func didReceiveHead(task: HTTPClient.Task<Response>, _ head: HTTPResponseHead)

    // this method will be called multiple times with chunks of the HTTP response body (if there is a body)
    // returning event EventLoopFuture could be used for backpressure handling, all reads will be stopped untill this future is resolved
    func didReceivePart(task: HTTPClient.Task<Response>, _ buffer: ByteBuffer) -> EventLoopFuture<Void>?

    // this method will be called if an error occurs during request processing
    func didReceiveError(task: HTTPClient.Task<Response>, _ error: Error)

    // this will be called when HTTP response is read fully
    func didFinishRequest(task: HTTPClient.Task<Response>) throws -> Response
}
```
This delegate will be especially helpful when you need to process HTTP response body in a streaming fashion, for example:
```swift
class CountingDelegate: HTTPClientResponseDelegate {
    typealias Response = Int

    var count = 0
    
    func didTransmitRequestBody() {
    }

    func didReceiveHead(_ head: HTTPResponseHead) {
    }

    func didReceivePart(_ buffer: ByteBuffer) -> EventLoopFuture<Void>? {
        count += buffer.readableBytes
        return nil
    }

    func didFinishRequest() throws -> Int {
        return count
    }
    
    func didReceiveError(_ error: Error) {
    }
}

let request = try HTTPRequest(url: "https://swift.org")
let delegate = CountingDelegate()

try client.execute(request: request, delegate: delegate).future.whenSuccess { count in
    print(count) // count is of type Int
}
```

## Seeking feedback
Feedback that would really be great is:
 - Streaming API: what do you like, what don't you like?
 - What feature set would be acceptable for `1.0.0`?
