# WebURL


* Proposal: [SSWG-NNNN](NNNN-filename.md)
* Authors: [Karl Wagner](https://github.com/karwa)
* Review Manager: TBD
* Status: **Awaiting Something**
* Implementation: [karwa/swift-url](https://github.com/karwa/swift-url)
* Forum Threads: [Announcement](https://forums.swift.org/t/announcing-weburl-a-new-url-type-for-swift/48000),
  [WebURL Category](https://forums.swift.org/c/related-projects/weburl/73)


## Package Description


WebURL is a new URL type for Swift, implementing the WHATWG URL Standard. It includes a comprehensive set of APIs for parsing and manipulating URLs,
percent-encoding and decoding strings and data, and working with hosts such as IP addresses and domains.

|                             |                                                                 |
|-----------------------------|-----------------------------------------------------------------|
| **Package Name**            | `WebURL`                                                        |
| **Module Name**             | `WebURL`                                                        |
| **Proposed Maturity Level** | Sandbox                                                         |
| **License**                 | Apache 2.0                                                      |
| **Dependencies**            | swift-system (optional integration library)                     |
|                             | [checkit](https://github.com/karwa/swift-checkit)   (test only) |
|                             | [swifter](https://github.com/httpswift/swifter.git) (test only) |


## Introduction


The WHATWG URL Standard is an effort led by the major browser vendors to unify their behavior, and produce a consistent description of how actors
on the web platform should interpret URLs. 

WebURL is an implementation of this standard in Swift. It has a full complement of APIs based on modern best practices, and leverages Swift's
language features and design patterns to achieve a leap forward in usability and performance. The core implementation has no external dependencies
or platform-specific behavior, although the package includes optional compatibility libraries to integrate with swift-system and Foundation.

WebURL brings web-compatible URL processing to Swift, offering a level of consistency and reliability across systems that has not been possible
until now. For the first time, we're able to align with web browsers on a common standard for URLs, and can even share a test suite with them
to ensure there is no divergence.


## Motivation


We'll discuss two issues with Swift's current URL processing abilities: standards compliance, and ergonomics.

### Standards Compliance

Almost all systems rely on URLs - for making requests to remote services, processing requests from remote clients, locating files,
establishing zones of trust, and other vital operations. It may be surprising, then, to learn that **URL strings are ambiguous**.

URL standards have been revised many times over the decades, each time introducing subtle differences. It is now difficult to ensure that
all code which processes a URL string interprets it in exactly the same way and derives the same information from it. Moreover, web browsers
(typically one of the most important clients) have not been able to conform to any of these historical standards due to compatibility constraints.
This all results in a surprising amount of variety in how URL strings can be interpreted, and given how much we rely on them,
any disagreements could lead to unexpected behavior and even exploitable vulnerabilities.

The latest industry standard is the WHATWG URL Standard, a living standard developed in an open process by the WHATWG. 
The WHATWG is a web platform standards body, led by the major browser vendors - Apple, Google, Mozilla, and Microsoft, and they have agreed
to align their browsers with the WHATWG's standards. The URL standard aims to obsolete historical standards, such as RFC-2396 (used by Foundation.URL),
and RFC-3986 (used by Foundation.URLComponents), and replace it with a new model of URLs based on how browsers _actually_ interpret them,
and how they effectively need to work in the model of the web that exists.

The latest version of Safari (WebKit) is compliant with this standard, as is NodeJS's URL class, and Chrome and Firefox are working on it.
There are new WHATWG-aligned URL libraries in many other languages, including a JavaScript polyfill, Rust, Python, Perl, and C++ libraries,
and other libraries (such as cURL) have started to borrow pieces of it. A library implementation of this standard would also be a
valuable addition to the Swift Server ecosystem. Consider the following example:

```swift
// What is the hostname of this URL?:

let urlString = "http://foo@evil.com:80@example.com/" 

WebURL(urlString)!.hostname   // "example.com"
URL(string: urlString)!.host  // "evil.com"
```

Chrome, Safari, Firefox, Go, Python, NodeJS, and Rust all agree that this URL points to `"example.com"`. If you paste it in your browser,
that’s where it will go. But since Foundation’s interpretation is based on an obsolete standard, it would send a request to `"evil.com"` instead.
These sorts of disagreements among URL parsers are being actively exploited, particularly in Server-Side Request Forgery (SSRF).

Or consider an HTTP request, whose response redirects the client to a different location:

```
// [Client -> Server]
GET /foo/test HTTP/1.1

// [Client <- Server]
HTTP/1.1 301 Moved Permanently
Location: /foo/test/../bar
```

The value of the `Location` header is a relative URL reference, to be resolved _by the client_ against the URL it used to make the request.
Since URL processing can vary between clients, there are several options for which request the client might make next:

```
// [Client -> Server] - Foundation
GET /foo/test/../bar HTTP/1.1

// [Client -> Server] - WHATWG, Safari, Chrome
GET /foo/bar HTTP/1.1
```

Swift's current URL type, Foundation.URL, leaves the path exactly as it is - including the `".."` components, whereas the WHATWG standard
automatically simplifies them as part of parsing the relative reference. There are plenty of differences like that: for example, whether back-slashes
are allowed as path separators (sometimes they are! So `"\foo\bar"` can be equivalent to `"/foo/bar"`), how repeated slashes are interpreted,
which whitespace characters get trimmed or filtered, and more. But it is important, because it means that in the web's URL model,
it is literally not possible for HTTP URLs to express the following as distinct request targets:

- `"/foo/test/../bar"`
- `"\foo\bar"`
- `"/foo/bar"`

That's important information for all Swift applications to know - but particularly in the context of servers, it is valuable for web servers
to accept all of the above spellings and interpret them according to the browser/web-compatible interpretation.

### Ergonomics

Besides standards compliance, Foundation.URL suffers from a number of ergonomics and performance issues. Its API is a swiftified version of `NSURL`'s
interface, which itself dates back to macOS 1.0. Its object model is rather unique - it models relative and absolute URLs in a single type,
with separately-stored baseURL and relative string portions, and computes URL components on-demand. It is a rather heavy object,
with surprising behavior around concepts like equality, and it essentially rules out in-place mutation.

```swift
// Equality in Foundation.URL:

let urlA = URL(string: "http://example.com/a/b/c")!
let urlB = URL(string: "/a/b/c", relativeTo: URL(string: "http://example.com")!)!
let urlC = URL(string: "b/c", relativeTo: URL(string: "http://example.com/a/")!)!

// All of these URLs have the same .absoluteString.

urlA.absoluteString == urlB.absoluteString // true
urlB.absoluteString == urlC.absoluteString // true

// But they are not interchangeable.

urlA == urlB // false (!)
urlB == urlC // false (!)

// Let's imagine an application using URLs as keys in a dictionary:

var operations: [URL: TaskHandle] = [:]
operations[urlA] = TaskHandle { ... }
operations[urlA] // TaskHandle
operations[urlB] // nil (!)

// Converting to/from a String (e.g. in a JSON document)
// can return a not-equal URL.

URL(string: urlB.absoluteString) == urlB // false (!)
```

Several other API decisions do not follow modern best practices. For example, Foundation is not consistent about whether URL components
are returned in their raw (percent-encoded) form, or if they are always returned _decoded_: `query` and `fragment` are returned raw,
but `host` and `path` are decoded; `username` is decoded, but `password` is raw! There is no obvious order to it, and developers
need to remember to check the documentation for each component, and there is no alternative if you really want a raw `host` or `path`
(which you very often do -- percent-decoding is lossy, so requests should be made and processed using raw components).

```swift
// Percent-decoding issues 1:

// Percent-encoded slash, meaning "AC%2FDC" should be one path component.
//                                                       VVV
let url = URL(string: "https://example.com/music/bands/AC%2FDC")! 

// Automatic decoding means it looks like two components in the returned string.
//                          V
url.path // "/music/bands/AC/DC"
```

```swift
// Percent-decoding issues 2:

// A path containing percent-encoded binary data, that is not UTF-8.
// This is actually fairly common - especially in file: and data: URLs.

let urlA = URL(string: "scheme://h/somepath/%72")!
print(urlA.path) // "/somepath/r"
                 //            ^ - byte 0x72 = ASCII 'r'

let urlB = URL(string: "scheme://h/somepath/%82")!
print(urlB.path) // "" - byte 0x82 is not valid UTF-8, so it returns an empty string.
```

It's so strange that Foundation's APIs themselves struggle to be consistent about it (SR-15363).

```swift
// Percent-decoding issues 3:

let url = URL(string: "https://example.com/music/bands/AC%2FDC")! 

// Some APIs incorrectly use URL's .path property, so they get a percent-decoded
// string before splitting it. That makes 'AC%2FDC' look like 2 components.

url.lastPathComponent    // "DC"
url.pathComponents.last  // "DC"

// Other APIs correctly look at the raw path, so they know that 'AC%2FDC'
// is a single path component.

url.deletingLastPathComponent() // "https://example.com/music/bands/"
                                //                     ^^^^^^^^^^^^^
```

**Summary**

Swift currently lacks URL processing that works according to modern web standards. Many important platforms already align their URL processing
with the WHATWG URL Standard (including WebKit, NodeJS, and rust-url), and browsers and other libraries are improving their conformance all the time.
It is important for Swift servers to have a consistent understanding of URLs with the web platform.

URLs are also valuable in their own right, as "universal identifiers". They can describe abstract hierarchical and non-hierarchical data,
which means they can be applied to just about everything - people, places, devices, memory locations, physical objects, nodes in a cluster,
drivers in a ride-sharing system, internal resources (e.g. `rdar:` URLs), app configuration settings (`chrome:` URLs),
or UI flows and internal logic. They are incredibly useful infrastructure components for humans and machines, and have broad support 
in existing tooling.

But some aspects of Foundation's URL model and API do not make it easy enough for Swift applications to fully embrace the power of URLs. 💪

Reading and writing URL components should be easy, with convenient, high-level APIs, that ensure correctness and deliver high performance.


## Detailed Design


**WebURL** is a new URL type, for Swift. It means "Web-Platform URL", since it is built to web specifications.
It is **not** limited to http/s URLs, and has excellent support for file URLs, data URLs, and other, application-defined URLs.

The full documentation is available [here][docs-home]. Broadly, WebURL's API can be grouped in a couple of sections:

1. Working with URL strings
2. Getting/Setting component strings
3. Richer component data
4. Views

We'll take a high-level look at each of these, with discussions along the way about how normalization works, and how it can help you write simpler,
better code, and how well all of this performs.

### URL Strings

Construct a WebURL by [initializing][docs-init-str] a value from a URL string:

```swift
// Parsing strings:

WebURL("https://github.com/karwa/swift-url")    // ✅ Typical HTTPS URL.
WebURL("file:///usr/bin/swift")                 // ✅ Typical file URL.
WebURL("my.app:/settings/language?debug=true")  // ✅ Typical custom URL.
```

The string is parsed as defined by the WHATWG URL Standard; it is very forgiving, and interprets strings as a browser would, 
including all of the compatibility behaviors necessary for accurate URL processing on the web.

> Note:
>
> When I say "interprets strings as a browser would", I mean at the HTML, JavaScript, or HTTP level.
>
> The browser's main search box/"omnibox" can use proprietary heuristics to resolve user input; 
> for example, it might consult your bookmarks, history, or cloud services, allowing it to turn the input "apple"
> in to the URL "https://apple.com/", or a recent online grocery receipt for apples. Those algorithms are a competitive feature
> which do not affect web compatibility, and are not standardized.
>
> The point is: even though I talk about parsing differences, don't think WebURL will break everything.
> Does it work in Safari? or NodeJS? or Rust? It'll work like that in WebURL. With guarantees now. Easy.

A WebURL value is always an absolute URL - in other words, it must include a _scheme_ (such as `https:`, `file:`, or `my.app:`).
Once you have a WebURL value to serve as a base, you can parse a relative reference against it by using the [resolve function][docs-resolve].
Resolving a relative reference returns the same location as an HTML `<a>` tag on the base URL's "page", and can be used to process HTTP redirects, 
although it is not limited to `http:` URLs.

```swift
// Resolving relative references:

let base = WebURL("https://github.com/karwa/swift-url/")!

base.resolve("pulls/39")            // ✅ "https://github.com/karwa/swift-url/pulls/39"
base.resolve("/apple/swift/")       // ✅ "https://github.com/apple/swift/"
base.resolve("..?tab=repositories") // ✅ "https://github.com/karwa/?tab=repositories"

let appURL = WebURL("my.app:/settings/payment/preferred")!
appURL.resolve("../current_region") // ✅ "my.app:/settings/current_region"
```

To obtain a WebURL's string representation ("serialization"), use the [serialized function][docs-serialized] or simply construct a String:

```swift
// Serializing a WebURL:

let url: WebURL = ...
url.serialized()
String(url)      // Same as above.
print(url)       // Same as print(url.serialized())
```

### Normalization and Idempotence

Foundation.URL generally tries to keep URL strings as you provide them. Its parser is strict about minor syntax mistakes,
and components are generally treated as opaque strings (except for percent-encoding) and not automatically normalized.

The new URL standard, on the other hand, essentially requires a different model. For one thing, it defines two operations for URL strings:
parsing a string in to a URL record, and serializing a URL record as a string. WebURL doesn't just scan URL strings - it also _interprets them_,
breaking them down in to URL records and rewriting them. It has a more complete understanding of what a URL means, which allows it to offer richer APIs
with stricter guarantees, such as the guarantee that WebURL values are **always normalized**.

```swift
// Normalization when parsing:

// Unicode characters (and even spaces!) are allowed, and automatically percent-encoded.

let url = WebURL("HTTPS://MYAPP.COM/foo/../sendMessage?I saw a 🦆!")!
String(url)  // ✅ "https://myapp.com/sendMessage?I%20saw%20a%20%F0%9F%A6%86!"

// Normalization makes URLs easier to process.
// The scheme and domain are lowercased, path is simplified, etc.

if url.scheme == "https", url.hostname == "myapp.com", url.path == "/sendMessage" {
  url.query?.percentDecoded()  // ✅ "I saw a 🦆!"
}
```

These guarantees permeate the _entire API_. If you set a URL's path string, WebURL automatically parses the new string, simplifies it,
adds percent-encoding, etc. And if you set a hostname which happens to contain an IPv6 address, WebURL will actually parse it,
extract its IP address value, and rewrite the address in canonical notation. The standard requires it.

The good news is that this makes URL processing a lot easier and more robust. You don't need to worry about all of the 
bizarre web compatibility quirks - WebURL cleans it up for you, so straightforward code just works.

```swift
// Normalization when modifying:

// The path is normalized when set.

var url  = WebURL("https://github.com/")!
url.path = "/apple/swift/pulls/../../swift-package-manager"
String(url)  // ✅ "https://github.com/apple/swift-package-manager"

// IP addresses are detected and rewritten.

url.hostname = "[ABC:0:0:0:0::0:192.168.0.1]"
String(url)  // ✅ "https://[abc::c0a8:1]/apple/swift-package-manager"
//                          ^^^^^^^^^^^^
```

Another important feature of the standard is that "the combination of parser, serializer, and API guarantee idempotence". 

In other words, taking a WebURL's string representation and parsing it again is **guaranteed** to return an identical value
to the one you started with (for example, if you encode it and decode it from JSON). That is a much stricter guarantee than Foundation.URL offers,
and is how developers _expect_ URLs to behave. In Swift terms, we can say that WebURL is `LosslessStringConvertible`.

```swift
// Idempotence in WebURL:

if let parsedURL = WebURL("...") {
  // 1. You can always re-parse a WebURL's string.
  let reparsedURL = WebURL(parsedURL.serialized())!
  // 2. And it always returns an identical WebURL.
  assert(reparsedURL == parsedURL)
  // 3. Re-parsing a normalized string doesn't normalize it again.
  assert(reparsedURL.serialized() == parsedURL.serialized())

  // This applies for any number of parse -> serialize -> parse -> serialize -> (etc) operations.
}
```

And this has some interesting implications of its own; it means that all of a WebURL's information must be contained in its URL string,
and hence all WebURLs with the same string must be identical. Again, it makes a lot of intuitive sense, but recall from the previous section
that **none** of those straightforward intuitions can be guaranteed about Foundation.URL values.

```swift
// Equality in WebURL:

let urlA = WebURL("http://example.com/a/b/c")!
let urlB = WebURL("http://example.com")?.resolve("/a/b/c")!
let urlC = WebURL("http://example.com/a/")?.resolve("b/c")!

// All of these URLs have the same string.

urlA.serialized() == urlB.serialized() // true
urlB.serialized() == urlC.serialized() // true

// With Foundation.URL, they were not interchangeable (!).
// With WebURL, they are.

urlA == urlB // true
urlB == urlC // true

// Let's imagine an application using URLs as keys in a dictionary:

var operations: [WebURL: TaskHandle] = [:]
operations[urlA] = TaskHandle { ... }
operations[urlA] // TaskHandle
operations[urlB] // TaskHandle (works now!)

// Converting to/from a String (e.g. in a JSON document) makes an exact copy.

WebURL(urlB.serialized()) == urlB // true (works now!)
```

### Getting and Setting Components

After parsing, URLs are typically processed by inspecting their components. WebURL includes your expected assortment of String-typed properties
for accessing URL components (or an Int, in the case of the port number):

```swift
// Reading String Components:

let url = WebURL("http://127.0.0.1:8888/music/bands/AC%2FDC?client=mobile")!

url.scheme    // "http"
url.hostname  // "127.0.0.1"
url.port      // 8888
url.path      // "/music/bands/AC%2FDC"
url.query     // "client=mobile"
```

There is not too much to say about these. Normalization ensures they have nice, predictable values, although in contrast to Foundation.URL they
are **not** automatically percent-decoded. This is important for reliably making requests, and matches the behavior of URL libraries in JavaScript,
Ruby, Python, Rust, and more.

It also means that `urlA.path = urlB.path` can copy a component between URLs while preserving percent-encoding,
which is important for WebURL, as its components may be directly assigned to:

```swift
// Assigning String Components:

var url = WebURL("https://github.com/")!

// Can set the path. Will automatically be parsed and normalized.

url.path = "/apple/swift/pulls/../../swift-package-manager"
String(url)  // ✅ "https://github.com/apple/swift-package-manager"
//                                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^

// Can set the hostname. Will automatically be parsed and normalized.

url.hostname = "[ABC:0:0:0:0::0:192.168.0.1]"
String(url)  // ✅ "https://[abc::c0a8:1]/apple/swift-package-manager"
//                          ^^^^^^^^^^^^
```

With Foundation.URL, assigning a URL's path or hostname requires constructing a `URLComponents` object 
(remembering to pass `resolvingAgainstBaseURL: true`), assigning a value, and constructing an entirely new URL from those components.
With WebURL, you just directly assign a value, and it will be normalized and written in-place using the existing storage when possible.

At this point, it is worth briefly touching on performance.

### A Note on Performance

Where Foundation.URL can be content with basic questions, such as _"where is the hostname?"_, the URL standard requires that WebURL
ask deeper questions, like _"what kind of host is it? is it percent-encoded? is it an IP address?"_, and to ask those sorts of questions
about every component. Path simplification, in particular, can be complex, full of ad-hoc compatibility behavior for things like
Windows drive letters in `file:` URLs and percent-encoded `".."` components. All of this stuff is great, and it's _especially nice_
that it all gets handled so you never need to care about it, but it costs, and nobody likes when things get slower than before.

After a lot of careful (and ongoing) optimization and validation work, I am confident in claiming that most users will experience
WebURL as faster than Foundation.URL, and often by a significant amount. Here are a couple of comparisons using tests from WebURL's
benchmark suite, running on an Intel MacBook Pro:

| Operation                | Foundation (us) | WebURL (us) | Speedup |
| ------------------------ | --------------- | ----------- | ------- |
| Parsing                  | 37.7            | 26.6        | 1.42x   |
| Set Path                 | 14.6            | 2.2         | 6.6x    |
| Set Host                 | 10.7            | 0.7         | 15.3x   |
| Iterate Path (average)   | 6.3             | 0.6         | 10.5x   |
| Iterate Path (very long) | 16.0            | 2.9         | 5.5x    |
| Append a Path Component  | 4.3             | 1.2         | 3.5x    | 
| Pop a Path Component     | 5.6             | 0.4         | 14.0x   |

The parsing test benchmarks [20 "average" HTTP URL strings][weburl-benchmark-averageurls] (from sites like reddit and wikipedia),
and shows that WebURL can parse, interpret, and rewrite URLs significantly faster than Foundation.URL takes to only parse them.

WebURL values are also much lighter - they are just one allocation, with much less overhead than Swift's current URL type
(no side-tables, caches, or references to other URL objects). Common utilities such as hashing or testing for equality are many times faster,
and they can often be efficiently mutated in-place just like other copy-on-write values.

The result is that a lot of operations that used to take noticeable time have become instant.

On the raspberry pi 4, the benchmarks shown above are enormously in WebURL's favor (Parsing: 1.36 _milliseconds_ vs 45.05 _microseconds_, or 30x faster).
I'm not sure what to make of that, but I think it might be worth checking to rule out a toolchain build defect
(I used the popular swiftlang.xyz aarch64 toolchain, 5.5.2).

The API can scale to meet your performance requirements. For even lower overheads, [the `.utf8` view][docs-utf8] can return non-copying slices
of the WebURL's string representation, and even allows you to modify components using arbitrary collections of bytes rather than strings.
If you are loading a database of URLs (for example, as part of a proxy, crawler or filter), you can parse WebURL values directly
from the bytes of the file, with no copies or extra Unicode processing in-between. Servers can write a (full or partial) WebURL directly to the network,
or perform URL processing directly on bytes from the network.

### Richer APIs

So far, we have mostly been discussing URL components as strings, but some components also have internal structure and meaning. 
WebURL is an API designed for Swift, so it goes beyond just strings - making use of features such as enums and zero-cost wrapper views
to provide a more expressive, powerful, and performant way to work with URLs.

- The [`.host` property][docs-host] returns an enum, giving a more detailed look at which host a URL points to.
  
  Currently, when making a request specified using a URL, the request engine often has to work out for itself what the URL's hostname means,
  typically using POSIX functions such as `inet_pton`. Since WebURL already knows which kinds of hosts are allowed and what they mean,
  it can tell you that directly - this URL's host is the domain `"example.com"`, or the IPv4 address `127.0.0.1`, etc.

  WebURL comes with pure Swift, platform-independent [IPv4Address][docs-ipv4address] and [IPv6Address][docs-ipv6address] types. 
  The parsing and serialization of these types are defined by the URL Standard, and they can be used as replacements for C's `in_addr` types,
  as well as the `inet_pton`, `inet_aton`, `inet_ntop`, and `inet_ntoa` functions.

  This is how the [WebURL port of async-http-client][ahc-connectiontarget] makes connections, integrating with swift-nio.

  ```swift
  // Connecting to a host:

  let url = WebURL("http://127.0.0.1:8888/my_site")!

  guard url.scheme == "http" || url.scheme == "https" else {
    throw InvalidSchemeError("Expected a http/s URL")
  }
  switch url.host {
    case .domain(let name): ... // DNS lookup.
    case .ipv4Address(let address): ... // Connect to known address.
    case .ipv6Address(let address): ... // Connect to known address.
    case .opaque, .empty, .none: fatalError("Not possible for http/s URLs")
  }
  ```

- The [`.pathComponents` property][docs-pathcomponents] returns a mutable `BidirectionalCollection` view of a URL’s path components.

  Looking at Foundation.URL's API, there are a lot more APIs related to the path than to any other component: `path`, `lastPathComponent`,
  `pathComponents` (a read-only array of strings), `pathExtension`, `relativePath`, `standardize()` (x3 variants), `hasDirectoryPath`,
  `resolveSymlinksInPath()`, `appendPathComponent(...)` (x4), `appendPathExtension(...)` (x2), `deleteLastPathComponent()` (x2),
  `deletePathExtension()` (x2), and more. But even though there are a large _number_ of APIs, their overall feature set is actually fairly limited,
  and it is difficult to compose more complex processing using them.

  And it's clear why the path is so important - it is an abstract, hierarchical space, and you can model lots of things with that.
  Protocols such as HTTP will generally only send the path and query as part of a request (and the host, but it isn't easy to change),
  so it is a critical part of how we encode information in URLs. Web frameworks are well-known for creating concise path-matching DSLs for routing,
  sometimes embedding regular expressions or instructions to capture a particular component.
  
  With WebURL, I wanted to provide a familiar API that works for simple path processing, but could also scale to more complex tasks,
  ensure it is done correctly, with optimal performance, and not take over the entire API while doing so. 
  What's the point of a cake if you can't eat it?
  
  The WebURL `.pathComponents` view is a zero-cost wrapper over the URL's existing storage. Thanks to `BidirectionalCollection` conformance,
  it supports both simple processing such as loops, as well as complex processing using indices, slices, and other
  generic pattern-matching algorithms.
  
  ```swift
  // Reading Path Components:

  let url = WebURL("https://github.com/karwa/swift-url/issues/63")!
  for component in url.pathComponents {
    // ✅ component = "karwa", "swift-url", "issues", "63"
  }

  if url.pathComponents.dropLast().last == "issues",
    let issueNumber = url.pathComponents.last.flatMap(Int.init) {
    // ✅ issueNumber = 63
  }
  ```

  What's more - pathComponents automatically handles percent-encoding for you (which can be tricky to do manually),
  and includes a full API for _modifying_ the path as well.

  To append some components, you can simply use the `+=` operator, or for more complex URL rewriting tasks, the API scales all the way
  up to arbitrary replacements with [`replaceSubrange`][docs-pathcomponents-replacesubrange]. These are novel additions for Swift
  that go beyond what is defined in the standard, but they uphold the same guarantees about idempotence, such as always leaving the URL
  in a normalized state and behaving the same way every time.

  ```swift
  // Modifying Path Components:

  var url = WebURL("https://info.myapp.com")!

  // += to batch-append:
  url.pathComponents += ["music", "bands", "AC/DC"]
  // ✅ "https://info.myapp.com/music/bands/AC%2FDC"
  //                           ^^^^^^^^^^^^^^^^^^^^
  url.pathComponents.last! 
  // ✅ "AC/DC" - Automatically decoded.

  // Other APIs:
  url.pathComponents.removeLast()
  url.pathComponents.append("The Rolling Stones")
  // ✅ "https://info.myapp.com/music/bands/The%20%Rolling%20Stones"
  //                                        ^^^^^^^^^^^^^^^^^^^^^^^
  url.pathComponents.last!
  // ✅ "The Rolling Stones"
  ```

- The [`.formParams` property][docs-formparams] is a mutable view of a URL’s query parameters.

  The path and query are especially important URL components - but while the path can be a list of components,
  there is no standard interpretation of what a URL's query means. One convention that started with HTML forms is to use it for
  a list of key-value pairs.

  The formParams view lets you read and write values as though they were properties on the view (using Swift's dynamic-member feature),
  so `url.formParams.client` lets you get and set the value of the `"client"` key in the query.

  ```swift
  // Form Parameters:

  var url = WebURL("https://example.com/search?category=food&client=mobile")!
  url.formParams.category  // "food"
  url.formParams.client    // "mobile"
  
  url.formParams.format = "json"
  // ✅ "https://example.com/search?category[...]&format=json"
  //                                             ^^^^^^^^^^^^

  // '+=' appends a Dictionary of key-value pairs:

  var url = WebURL("https://example.com/search")!
  url.formParams += [
    "category" : "sports",
      "client" : "mobile"
  ]
  // ✅ "https://example.com/search?category=sports&client=mobile"
  //                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  ```

  The formParams view has a deliberately small API because of its support for dynamic members, but it includes `get` and `set` functions for keys
  that aren't valid Swift identifiers, `getAll` for retrieving duplicates, an `allKeyValuePairs` sequence of key-value tuples, and more.

  This API is currently being considered for revision before the package is stabilized. The current implementation corresponds
  to JavaScript's `URLSearchParams` class (also defined by the URL standard) and while it works, it has some subtleties that we
  might not want to inherit.

  I'll also be looking to include query parameter sorting (for more effective caching), and allowing the use of percent-encoding rather
  than strict form-encoding.

### Foundation Interop

The latest release of WebURL, 0.3.1, includes bidirectional interoperability with Foundation.URL. For a new URL type to really be usable in Swift,
it needs to interoperate with the type we already have - libraries cannot realistically use it if the cost is to break all of their existing clients.

Now you can construct a WebURL from a Foundation.URL or vice versa, so existing interfaces can still be supported, and there are a number of extensions
to support using WebURLs with Foundation APIs such as `URLSession`. It is not _quite_ as excellent as using a native WebURL request engine
(like our [port of async-http-client][ahc-port]), because it doesn't use WebURL for internal processing such as redirects.
Even so, it can help to resolve some instances where Foundation.URL is not web-compatible, and allows applications to manipulate URLs
using WebURL's API.

```swift
// Parsing differences with Foundation:

// Using WebURL: Sends a request to "example.com". 
// Chrome, Safari, Firefox, Go, Python, NodeJS, Rust agree. ✅
print( try String(contentsOf: WebURL("http://foo@evil.com:80@example.com/")!) )

// Using Foundation.URL: Sends a request to "evil.com"! 😵
print( try String(contentsOf: URL(string: "http://foo@evil.com:80@example.com/")!) )
```

That said, mixing URL standards can be a delicate business. Unfortunately, as the previous example shows, the status quo is that there
are ambiguous URL strings - and there can be serious consequences to that. It's a defect with URLs in general rather than a problem
caused by WebURL, but we should provide guidance and best practices, which is why WebURL's documentation includes the **highly recommended**
guide: [Using WebURL with Foundation][docs-foundationinterop].
  
### Would you believe there's more?

We've already covered a lot, and there are still great APIs that don't _quite_ make this discussion but are worth checking out anyway.

- The JavaScript `URL` API.

  The JavaScript `URL` class API is also defined by the URL standard, meaning WebURL is able to offer a truly compatible interface
  via its [`.jsModel` view][docs-jsmodel]. The string-typed URL components are already mostly the same (we trim some delimiters, that's it),
  but JavaScript has some quirks that aren't desirable in WebURL:

  ```javascript
  // JavaScript assignment allows trailing garbage:

  var url      = new URL("https://example.com/");
  url.protocol = "ftp://foobar.org/";
  url.href;
  // "ftp://example.com"
  //  ^^^ - Set the scheme and just ignored the rest. Not even an error.
  ```

  WebURL's native API is designed to be a better choice for Swift developers, but if you're creating content that will primarily
  be consumed by JavaScript, it _can be helpful_ to have the exact same API available in Swift (especially if it includes the delimiters and other quirks).
  With WebURL, you can have that.

- Swift-system integration

  WebURL includes conversions between swift-system's `FilePath` and `file:` URLs for both POSIX and Windows-style paths.
  This conversion is not standardized anywhere, there are several issues with Foundation's implementation,
  and a lot of compatibility quirks to consider. For WebURL, I devised a new implementation and [extensive test-suite][weburl-filepath-tests],
  based on an analysis of Chromium, IE8, Firefox, WebKit, Rust, and the Windows shell APIs.

  It took a long time to get it right!

### IDNA

WebURL is currently lacking one feature before it has full compatibility with the WHATWG URL Standard: Internationalized Domain Names (IDNA).

The reason for this is that IDNA requires Unicode normalization, and the required data has not been made available by the standard library yet.
Being "pure Swift" and not having any dependencies beyond the standard library are important goals for WebURL, so a dependency on ICU
to fill the void is not acceptable. Any inputs that would require IDNA to process, are treated as invalid.

I consider it a very important feature to add, although it will likely arrive some time after v1.0 and overall API stability.
Our current URL type also has poor support for IDNA (`URL(string: "https://👁️👄👁️.fm")` also returns `nil`), so it technically isn't a regression.

### Summary

WebURL is a new URL type, designed to improve URL processing in Swift on all fronts:

- It has modern **standards support**, bringing web-compatible URL processing to Swift for the first time,
  and matching browsers and more libraries in other languages.

- Its API is truly **designed for Swift**, and leverages the language to great effect: the `Host` enum gives you greater
  confidence that you're handling URLs exactly as the parser does, and the `pathComponents` and `formParams` properties
  provide a range of powerful, composable tools as zero-cost views. And there is broad support for generics throughout,
  so you can get the lowest possible overheads, cutting out copies that only exist to convert data types.

- It makes **strict guarantees**, which better match how developers expect URLs to work and make code more robust.

- It is **fast**. WebURL is [regularly benchmarked][weburl-benchmarks], and when compared to equivalent implementations
  using Swift's current URL type, it is faster across the board, often by an order of magnitude.

- It is **compatible with Foundation**. This was tricky to pull off. It is lenient where it can be (for maximum compatibility),
  while also comprehensively blocking any ambiguous URLs. So you can actually adopt it, today, incrementally if you like.

## Maturity Justification

Requirements for maturity level "Sandbox":

**General**
- ✅ Has relevance to Swift on Server specifically
- ✅ Publicly accessible source managed by an SCM such as github.com or similar
- ✅ Prefer to use main as the default branch name, in line with Swift's guidelines
- 🔳 Adopt the Swift Code of Conduct (TODO: Email address for reports)

**Ecosystem**
- ✅ Uses SwiftPM
- ✅ Integrated with critical SSWG ecosystem building blocks, e.g., Logging and Metrics APIs, SwiftNIO for IO

**Longevity**
- ✅ Must be from a team that has more than one public repository (or similar indication of experience)
- 🔳 SSWG should have access / authorization to graduated repositories in case of emergency
- 🔳 Adopt the SSWG Security Best Practices (TODO: Email address for reports)

**Testing, CI and Release**
- ✅ Have unit tests for Linux
- ✅ CI setup, including testing PRs and the main branch
- ✅ Follow semantic versioning, with at least one published pre-release (e.g. 0.1.0, 1.0.0-beta.1) or release (e.g. 1.0.0)

**Licensing**
- ✅ Apache 2, MIT, or BSD (Apache 2 recommended)

**Conventions and Style**
- ✅ Adopt Swift API Design Guidelines
- ✅ Follow SSWG Technical Best Practices when applicable.
- ✅ Prefer to adopt code formatting tools and integrate them into the CI

There are still a couple of minor clerical gaps, but they can be sorted out quickly if the project is accepted.

I'm not sure that it meets the requirements for maturity level "Incubating" quite yet:

> Document that it is being used successfully in production by at least two independent end users which, in the SSWG judgement,
> are of adequate quality and scope.

It is being used in production, but being part of the SSWG would help attract more users of greater scope.

> Must have 2+ maintainers and/or committers. In this context, a committer is an individual who was given write access to the codebase
> and actively writes code to add new features and fix any bugs and security issues. A maintainer is an individual who has write access
> to the codebase and actively reviews and manages contributions from the rest of the project's community. 
> In all cases, code should be reviewed by at least one other individual before being released.

Currently, the project has one maintainer. Hopefully, being part of the SSWG would help attract more users of greater scope.

I would actually really love code reviews. I think they work, and I invite them, but - who? any volunteers?

> Demonstrate an ongoing flow of commits and merged contributions, or issues addressed in timely manner, or similar indication of activity.

I've been building and maintaining the project for a couple of years, so there certainly has been activity.

[docs-home]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl
[docs-init-str]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/weburl/init(_:)
[docs-resolve]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/weburl/resolve(_:)
[docs-serialized]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/weburl/serialized(excludingfragment:)
[docs-host]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/weburl/host-swift.property
[docs-ipv4address]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/ipv4address
[docs-ipv6address]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/ipv6address
[docs-pathcomponents]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/weburl/pathcomponents-swift.property
[docs-pathcomponents-replacesubrange]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/weburl/pathcomponents-swift.struct/replacesubrange(_:with:)
[docs-formparams]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/weburl/formparams
[docs-jsmodel]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/weburl/jsmodel-swift.property
[docs-utf8]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/weburl/utf8

[ahc-port]: https://github.com/karwa/async-http-client/
[ahc-connectiontarget]: https://github.com/karwa/async-http-client/blob/4b695461bbd67630e9c549774fa1372284e283c8/Sources/AsyncHTTPClient/ConnectionTarget.swift#L29

[docs-foundationinterop]: https://karwa.github.io/swift-url/0.3.1/documentation/weburl/foundationinterop
[weburl-filepath-tests]: https://github.com/karwa/swift-url/blob/186d0b3895a3601132a6fe2279359ccf40618a0c/Sources/WebURLTestSupport/Resources/file_url_path_tests.json
[weburl-benchmarks]: https://github.com/karwa/swift-url/tree/main/Benchmarks

[weburl-benchmark-averageurls]: https://github.com/karwa/swift-url/blob/186d0b3895a3601132a6fe2279359ccf40618a0c/Benchmarks/Sources/WebURLBenchmark/SampleURLs.swift#L39