# NIO-based Apple Push Notification Service

* Proposal: SSWG-0006
* Authors: [Kyle Browning](https://github.com/kylebrowning)
* Sponsors: Vapor
* Review Manager: TBD
* Status: **Implemented**
* Pitch: [Server/Pitches/APNS Implementation](https://forums.swift.org/t/apple-push-notification-service-implementation-pitch/20193)
* Implementation: [kylebrowning/swift-nio-http2-apns](https://github.com/kylebrowning/swift-nio-http2-apns)

## Package Description
Apple push notification service implementation built on Swift NIO.

|  |  |
|--|--|
| **Package name** | `nio-apns` |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram) |
| **License** | [Apache 2](https://www.apache.org/licenses/LICENSE-2.0.html) |
| **Dependencies** | [SwiftNIO](https://github.com/apple/swift-nio) 2.x, [SwiftNIOSSL](https://github.com/apple/swift-nio-ssl.git) 2.x, [SwiftNIOHTTP2](https://github.com/apple/swift-nio-http2.git) 1.x|

## Introduction

`NIOAPNS` is a module thats gives server side swift applications the ability to use the [Apple push notification service.](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server)


## Motivation

APNS is used to push billions of pushes a day, (7 billion per day in 2012). Many of us using Swift on the backend are using it to power our iOS applications. Having a community supported APNS implementation would go a long way to making it the fastest, free-ist, and simplest solution that exists.

All of the currently maintained libraries either have framework specific dependencies, are not built with NIO, or do not provide enough extensibility while providing "out of the box" capabilities.

### Existing Solutions

- [NIO-APNS](https://github.com/moritzsternemann/nio-apns)
- [PerfectNotifications](https://github.com/PerfectlySoft/Perfect-Notifications)
- [VaporNotifications](https://github.com/hjuraev/VaporNotifications)

## Proposed Solution

`NIOApns` provides the essential types for interacting with APNS Server (both production and sandbox).

### What it does do

- Provides an API for handling connection to Apples HTTP2 APNS server.
- Provides proper error messages that APNS might respond with.
- Uses custom/non dependency implementations of JSON Web Token specific to APNS (using [rfc7519](https://tools.ietf.org/html/rfc7519).
- Imports OpenSSL for SHA256 and ES256.
- Provides an interface for signing your Push Notifications.
- Signs your token request.
- Sends push notifications to a specific device.
- [Adheres to guidelines Apple Provides.](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/establishing_a_token-based_connection_to_apns)

### What it won't do.
- Store/register device tokens.
- Build an HTTP2 generic client.
- Google Cloud Message.
- Refresh your token no more than once every 20 minutes and no less than once every 60 minutes. (up to connection handler)

### Future considerations and dependencies
- BoringSSL
- Common connection handler options
- swift-log
- swift-metrics
- swift-jwt?
- swift-http2-client?

### APNSConfiguration

[`APNSConfiguration`](https://github.com/kylebrowning/swift-nio-http2-apns/blob/master/Sources/NIOAPNS/APNSConfiguration.swift) is a structure that provides the system with common configuration.

```swift
public struct APNSConfiguration {
    public var keyIdentifier: String
    public var teamIdentifier: String
    public var signingMode: SigningMode
    public var topic: String
    public var environment: APNSEnvironment
    public var tlsConfiguration: TLSConfiguration

    public var url: URL {
        switch environment {
        case .production:
            return URL(string: "https://api.push.apple.com")!
        case .sandbox:
            return URL(string: "https://api.development.push.apple.com")!
        }
    }
}
```
#### Example `APNSConfiguration`
```swift
let apnsConfig = try APNSConfiguration(keyIdentifier: "9UC9ZLQ8YW",
                                   teamIdentifier: "ABBM6U9RM5",
                                   signingMode: .file(path: "/Users/kylebrowning/Downloads/AuthKey_9UC9ZLQ8YW.p8"),
                                   topic: "com.grasscove.Fern",
                                   environment: .sandbox)
```

### SigningMode

[`SigningMode`](https://github.com/kylebrowning/swift-nio-http2-apns/blob/master/Sources/NIOAPNSJWT/SigningMode.swift) provides a method by which engineers can choose how their certificates are signed. Since security is important keeping we extracted this logic into three options. `file`, `data`, or `custom`.

```swift
public struct SigningMode {
    public var signer: APNSSigner
    init(signer: APNSSigner) {
        self.signer = signer
    }
}

extension SigningMode {
    public static func file(path: String) throws -> SigningMode {
        return .init(signer: try FileSigner(url: URL(fileURLWithPath: path)))
    }
    public static func data(data: Data) throws -> SigningMode {
        return .init(signer: try DataSigner(data: data))
    }
    public static func custom(signer: APNSSigner) -> SigningMode {
        return .init(signer: signer)
    }
}
```
### APNSSigner
[`APNSSigner`](https://github.com/kylebrowning/swift-nio-http2-apns/blob/master/Sources/NIOAPNSJWT/Signer.swift) provides a protocol to be used with the signing mode `custom`. This gives developers the ability to sign their digests with whatever package they see fit.
```swift
public protocol APNSSigner {
    func sign(digest: Data) throws -> Data
}
```
#### Example Custom SigningMode that uses AWS for private keystorage
```swift
public class CustomSigner: APNSSigner {
   public func sign(digest: Data) throws -> Data {
     return try AWSKeyStore.sign(digest: digest)
   }
   public func verify(digest: Data, signature: Data) -> Bool {
      // verification
   }
}
let customSigner = CustomSigner()
let apnsConfig = APNSConfig(keyId: "9UC9ZLQ8YW",
                      teamId: "ABBM6U9RM5",
                      signingMode: .custom(signer: customSigner),
                      topic: "com.grasscove.Fern",
                      environment: .sandbox)
```
### APNSConnection

[`APNSConnection`](https://github.com/kylebrowning/swift-nio-http2-apns/blob/master/Sources/NIOAPNS/APNSConnection.swift) is a class with methods thats provides a wrapper to NIO's ClientBootstrap. The `swift-nio-http2` dependency is utilized here. It also provides a function to send a notification to a specific device token string.


#### Example `APNSConnection`
```swift
let apnsConfig = ...
let apns = try APNSConnection.connect(configuration: apnsConfig, on: group.next()).wait()
```

### Alert

[`Alert`](https://github.com/kylebrowning/swift-nio-http2-apns/blob/master/Sources/NIOAPNS/APNSRequest.swift) is the actual meta data of the push notification alert someone wishes to send. More details on the specifics of each property are provided [here](https://developer.apple.com/library/archive/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/PayloadKeyReference.html). They follow a 1-1 naming scheme listed in Apple's documentation


#### Example `Alert`
```swift
let alert = Alert(title: "Hey There", subtitle: "Full moon sighting", body: "There was a full moon last night did you see it")
```

### APSPayload

[`APSPayload`](https://github.com/kylebrowning/swift-nio-http2-apns/blob/master/Sources/NIOAPNS/APNSRequest.swift) is the meta data of the push notification. Things like the alert, badge count. More details on the specifics of each property are provided [here](https://developer.apple.com/library/archive/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/PayloadKeyReference.html). They follow a 1-1 naming scheme listed in Apple's documentation


#### Example `APSPayload`
```swift
let alert = ...
let aps = APSPayload(alert: alert, badge: 1, sound: "cow.wav")
```

## Putting it all together

```swift
let group = MultiThreadedEventLoopGroup(numberOfThreads: 1)
let apnsConfig = try APNSConfiguration(keyIdentifier: "9UC9ZLQ8YW",
                                   teamIdentifier: "ABBM6U9RM5",
                                   signingMode: .file(path: "/Users/kylebrowning/Downloads/AuthKey_9UC9ZLQ8YW.p8"),
                                   topic: "com.grasscove.Fern",
                                   environment: .sandbox)

let apns = try APNSConnection.connect(configuration: apnsConfig, on: group.next()).wait()
let alert = Alert(title: "Hey There", subtitle: "Full moon sighting", body: "There was a full moon last night did you see it")
let aps = APSPayload(alert: alert, badge: 1, sound: "cow.wav")
let notification = BasicNotification(aps: aps)
let res = try apns.send(notification, to: "de1d666223de85db0186f654852cc960551125ee841ca044fdf5ef6a4756a77e").wait()
try apns.close().wait()
try group.syncShutdownGracefully()
```

### Custom Notification Data

Apple provides engineers with the ability to add custom payload data to each notification. In order to facilitate this we have the `APNSNotification`.

#### Example
```swift
struct AcmeNotification: APNSNotification {
    let acme2: [String]
    let aps: APSPayload

    init(acme2: [String], aps: APSPayload) {
        self.acme2 = acme2
        self.aps = aps
    }
}

let apns: APNSConnection: = ...
let aps: APSPayload = ...
let notification = AcmeNotification(acme2: ["bang", "whiz"], aps: aps)
let res = try apns.send(notification, to: "de1d666223de85db0186f654852cc960551125ee841ca044fdf5ef6a4756a77e").wait()
```


## Maturity

This package meets the following criteria according to the [SSWG Incubation Process](https://github.com/swift-server/sswg/blob/master/process/incubation.md):
- Ecosystem (SwiftNIO)
- Code Style is up to date
- Errors implemented
- Tests being used
- documentation
- [Apache 2 license](https://github.com/kylebrowning/swift-nio-apns/blob/master/LICENSE)
- [Swift Code of Conduct](https://github.com/kylebrowning/swift-nio-apns/blob/master/CODE_OF_CONDUCT.md)
- [Contributing Guide](https://github.com/kylebrowning/swift-nio-apns/blob/master/CONTRIBUTING.md)
- [CircleCI builds](https://circleci.com/gh/kylebrowning/swift-nio-apns/tree/master)

Vapor, and I, are in the processes of providing a (framework agnostic) higher-level library: [`APNSkit`](https://github.com/vapor/apns-kit). This is the initial work on connection pooling and things we thought would be hard to get right in the initial incubation periods before submitting..

## Alternatives Considered

N/A

## Special Thanks
|  |  |
|--|--|
| **[Tanner](https://github.com/tanner0101)** | Everything really |
| **[fumoboy007](https://forums.swift.org/u/fumoboy007)** | APNSSigner idea |
| **[David Hart](https://forums.swift.org/u/hartbit)** | Custom Notifications, best practices and Apples naming conventions |
| **[IanPartridge](https://forums.swift.org/u/IanPartridge)** | JWT, crypto usages, overall feedback |
| **[Nathan Harris](https://forums.swift.org/u/mordil/summary)** | General questions on Incubation process and templates |
| **[Laurent](https://github.com/lgaches)** | All the tests |
|**[Everyone who participated in the pitch](https://forums.swift.org/t/apple-push-notification-service-implementation-pitch/20193)**| The feedback thread was so lively, thank you!|
