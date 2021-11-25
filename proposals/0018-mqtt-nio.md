# Solution name

* Proposal: [SSWG-0018](0018-mqtt-nio.md)
* Authors: [Adam Fowler](https://github.com/adam-fowler)
* Review Manager: TBD
* Status: **Implemented**
* Implementation: [mqtt-nio](https://github.com/adam-fowler/mqtt-nio)
* Forum Threads: [Pitch](https://forums.swift.org/t/mqttnio/53238/)

*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Previous Revision(s): [1](https://github.com/swift-server/sswg/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal(s): [SSWG-XXXX](XXXX-filename.md)

## Package Description
MQTTNIO is a MQTT client built on top of SwiftNIO, that provides full support for both v3.1.1 and v5 of the MQTT protocol.

|  |  |
|--|--|
| **Package Name** | mqtt-nio |
| **Module Name** | MQTTNIO |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [Apache 2.0](https://choosealicense.com/licenses/apache-2.0/) |
| **Dependencies** | [swift-nio](https://github.com/apple/swift-nio), [swift-nio-ssl](https://github.com/apple/swift-nio-ssl), [swift-nio-transport-services](https://github.com/apple/swift-nio-transport-services), [swift-log](https://github.com/apple/swift-log) |

## Introduction

MQTT is a messaging protocol commonly used for communicating with IoT (Internet of Things) devices. It is a lightweight publish/subscribe message transport designed to have a small code footprint and network bandwidth.

MQTTNIO is a MQTT client built on top of SwiftNIO, that provides full support for both v3.1.1 and v5 of the MQTT protocol. It runs on macOS, iOS and Linux. It supports WebSocket connections and TLS through both NIOSSL and NIOTransportServices.

## Motivation

There are a number of Swift MQTT libraries out there but many are not built on top of SwiftNIO. And many only support one version of the protocol or don’t provide WebSocket or TLS connections. MQTTNIO provides all of these. The library has also recently gained new Swift concurrency APIs.

## Detailed design

Most functionality MQTTNIO has is accessed through one object `MQTTClient`. This allows you to send MQTT messages to a MQTT server and add listeners to read messages received from the server. `MQTTClient` has `EventLoopFuture` and `async` APIs. For the purposes of this proposal I shall use the `async` APIs. 

### Connection

First you need to create your client and connect to your MQTT server.

```swift
// connect to mosquitto test server
let client = MQTTClient(
    host: "test.mosquitto.org", 
    port: 1883,
    identifier: "My unique identifier",
    eventLoopGroupProvider: .createNew
)
// if a session was previously present for your identifier `sessionPresent`
// will be `true`
let sessionPresent = try await client.connect(cleanSession: false)
```

As with many Swift NIO based packages, the client will need to be shutdown before it can be deleted.
```swift
try await client.shutdown()
```

### Sending packets

`MQTTClient` has functions to send all the standard MQTT packets, except the ACK packets which are dealt with internally. Where it is expected the server will respond with an ACK the function will wait for the ACK before returning.

### Publish

You can send a `PUBLISH` message to a topic as follows
```swift
try await client.publish(
    to: "TopicName",
    payload: ByteBuffer(string: "Message Payload"),
    qos: .atLeastOnce
)
```
Publish packets can be sent with three different levels of QoS: `.atMostOnce`, `.atLeastOnce` and `.exactlyOnce`. The `publish` function will deal with all three QoS levels and only return when all relevant ACK messages have been dealt with.

### Subscribe and Listen

You can send a `SUBSCRIBE` message to the server and listen for `PUBLISH` messages from the server. 

```swift
let subscription = MQTTSubscribeInfo(topicFilter: "MyTopic", qos: .exactlyOnce)
// subscribe returns a SUBACK packet which contains the QoS to be expected for each
// topic subscription
let suback = try await client.subscribe(to: [subscription])
// By adding a PUBLISH listener you can process all the PUBLISH messages being sent from
// the server to the client
client.addPublishListener("My Listener") { result in
    switch result {
    case .success(let publish):
        var buffer = publish.payload
        let string = buffer.readString(length: buffer.readableBytes)
        print(string)
    case .failure(let error):
        print("Error while receiving PUBLISH event")
    }
}
```

## Maturity Justification

Sandbox. The package has one author. It is fairly new, v1.0.0 was released a year ago.  

## Alternatives considered

There are two other major Swift MQTT client implementations [CocoaMQTT](https://github.com/emqx/CocoaMQTT) and  [SwiftMQTT](https://github.com/aciidb0mb3r/SwiftMQTT). Neither of these are built on top of Swift NIO or support v5 of the protocol.