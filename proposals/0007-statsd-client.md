# Statsd Client

* Proposal: [SSWG-0007](https://github.com/swift-server/sswg/blob/master/proposals/SSWG-0007.md)
* Authors: [Tom Doron](https://github.com/tomerd)
* Sponsor(s): TBD
* Review Manager: TBD
* Status: **Implemented**
* Implementation: [tomerd/swift-statsd-client](https://github.com/tomerd/swift-statsd-client)
* Forum Threads: [Pitch](https://forums.swift.org/t/statsd-client-in-swift/25134),  [Discussion](https://forums.swift.org/t/discussion-swift-statsd-client-implementation/26109)

## Package Description

metrics backend for swift-metrics that uses the statsd protocol

|||
| --- | --- |
|**Package Name**|`swift-statsd-client`|
|**Module Name**|`StatsdClient `|
|**Proposed Maturity Level**|[Sandbox ](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram)|
|**License**|[Apache 2.0](https://github.com/MrLotU/SwiftPrometheus/blob/master/LICENSE)|
|**Dependencies**|swift-nio > 1.0.0, swift-metrics > 1.0.0|

## Introduction

for a background on metrics in swift applications, see the [metrics proposal](https://forums.swift.org/t/feedback-server-metrics-api)

the [statsd](https://github.com/b/statsd_spec) protocol is simpler than most custom metrics solutions (eg prometheus), yet extremely popular since it is language agnostic and easy to implement both client and server side. as such, it can be used to integrate applications with many observability solutions such as:

* [aws ](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-custom-metrics-statsd.html)
* [azure ](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/data-platform)
* [google cloud ](https://cloud.google.com/monitoring/agent/plugins/statsd)
* [ibm cloud ](https://cloud.ibm.com/catalog/services/ibm-cloud-monitoring-with-sysdig)
* [grafana ](https://grafana.com)
* [graphite ](https://graphiteapp.org) 

## Motivation

a `statsd` client will allow server applications written in swift to easily integrate with many popular observability solutions. the client, like the protocol, is designed to be lightweight and delegate most computation to the observability server

## Proposed solution

the proposed solution is a client library that implements the [swift metric api](https://github.com/apple/swift-metrics) and uses [swift nio](https://github.com/apple/swift-nio) to establish a `UDP` connection to the statsd server.

metrics types are mapped as following:

| Metrics API | Statsd |
| --- | --- |
| Counter | Counter |
| Gauge | Gauge |
| Timer | Timer |
| Recorder | Histogram |

## Detailed design

### api

the client is primarily designed to be used within the context of the swift metrics api. as such, the application needs to create an instance of the `StatsdClient` and bootstrap the `MertricsSystem` as part of the application's main:

```swift
let statsdClient = try StatsdClient(host: host, port: port)
MetricsSystem.bootstrap(statsdClient)
```

to cleanly release the networking resources occupied by the client, the application should also shutdown the client before it terminates:

```swift
statsdClient.shutdown()
```
as mentioned above, usage of the client is designed to be done primarily via the swift metrics api and the client does not provide additional functionality beyond the metrics api surface area. in other words, users are not expected to interact with the client directly beyond bootstrap and shutdown. emitting metrics information is done via the standard metrics api, as described in https://github.com/apple/swift-metrics#metric-types

### protocol

the client implements the `statsd` protocol as following:

|||
| --- | --- |
| counter | `<name>:<value>|c` |
| timer | `<name>:<value>|ms` |
| gauge | `<name>:<value>|g` |
| histogram | `<name>:<value>|h` | 

**name** is constructed from the metric object label and dimensions concatenated with a dot between them

**value** is computed from the data provided to the metrics API. no in-memory aggregation is done at the client level, and data is mostly pass-through. note that some `statsd` metrics types accept positive values only in the UInt64 range, while the metrics API generally accepts Int64. in cases of such mismatch, the client truncates negative values to zero

### networking 

thew client uses [swift-nio](https://github.com/apple/swift-nio) to establish a `UDP` connection to the statsd server. in later phases, the client will also support `TCP` connections.

## Maturity Justification

this is a fairly new library, not enough production usage to justify anything beyond sandbox

## Alternatives considered

there are no clear alternatives
