# Solution name

* Proposal: SSWG-00??
* Authors: [tomerd](https://github.com/tomerd)
* Review Manager: TBD
* Status: TBD
* Implementation: [Swift Backtrace](https://github.com/swift-server/swift-backtrace)

## Package Description

Swift Backtrace provides support for automatically printing crash backtraces of Swift programs on Linux.

|  |  |
|--|--|
| **Package Name** | `swift-backtrace` |
| **Module Name** | `Backtrace` |
| **Proposed Maturity Level** | [Incubating](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram) |
| **License** | [Apache License 2.0](https://github.com/swift-server/swift-backtrace/blob/master/LICENSE.txt) |
| **Dependencies** | [libbacktrace](https://github.com/ianlancetaylor/libbacktrace) (vendored) |

## Introduction & Motivation

Printing backtraces when applications crash is a critical component in diagnostics of
real-world production issues on the Server.

When applications crash on the Server, it is desired for the crash information to be
captured and printed along side the application logs so that log aggregation tools 
(e.g. Splunk) could include the crash information and used for alerting and further 
analysis offline.

At this point of time, Swift does not print crash backtraces when compiled in 
release mode on Linux. 
This means that Swift server applications can crash silently making the operation of 
such applications difficult.


## Proposed solution

Introduce a Linux centric library that captures crashes and prints backtraces 
before the program exits.

Applications (or application frameworks on their behalf) will need to add the 
following as the first thing on the application's entrypoint (`main.swift`):

```swift
import Backtrace      // <-- Import the module

Backtrace.install()   // <--- Install the crash handler
```

## Detailed design

### Capturing backtraces

Crashes are captured by installing a `SIGILL` signal listener.
Swift will uses `SIGILL` to signal a crash, which allows the library to begin 
collecting the backtraces. `libbacktrace` which is vendored as part of this library
is then used to produce symbolic backtraces. 

Note: Capturing crash signal and halting the execution of the program is generally not a good idea.
For example, attempting to allocate memory while crashing under certain conditions 
may lead to security vulnerabilities and deadlocks.
In this case, this is acceptable since the backtrace collection will continue to 
eventually crash the program.

### Demangaling symbols

The library parses the symbolic backtraces from `libbacktrace` and demangles 
them using the internal `swift_demangle` function from the stdlib.

Note: Using internal functions like `swift_demangle` is generally not a good idea.
In this case, this is acceptable since we plan to deprecate this package as soon
as the Swift runtime will offer this functionality.

### Printing

The library currently prints directly to `srderr`.
This works in most cases, but we should consider an extension with 
[swift-log](https://github.com/apple/swift-log) to hook the printing into the 
Application's logging system.

## Maturity Justification

This package is already being used across the Swift server ecosystem in many projects.

## Alternatives considered

The obvious and correct solution to backtraces on Linux is to add support 
for the in the Swift runtime. 
Since the changes required to support that are non-trivial and may take a awhile to ship, 
this package provides a stop-gap solution.

Another aspect considered was the in-process vs. out-of-process approach. 
While macOS/iOS have strong support for collecting crash reports out-of-process
such support does not exits universally on the server ecosystem which is made of 
a wide variety of operating systems. Therefore, we found the in-process approach 
to be more practical.
