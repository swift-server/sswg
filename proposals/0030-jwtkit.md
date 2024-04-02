# JWTKit SSWG Proposal

* Proposal: [SSWG-0030](0030-jwtkit.md)
* Authors: [Paul Toffoloni](https://github.com/ptoffy), [Tim Condon](https://github.com/0xTim), [Gwynne Raskind](https://github.com/gwynne) 
* Review Manager: [Konrad 'ktoso' Malawski](https://github.com/ktoso)
* Status: **Graduated**
* Implementation: [JWTKit](https://github.com/vapor/jwt-kit)
* Forum Threads: [Initial pitch](https://forums.swift.org/t/pitch-jwtkit/70205)

## Package Description

A JWT library that allows signing and verifying tokens with all modern algorithms.

| **Package Name**            | `jwt-kit`                                                    |
| --------------------------- | ------------------------------------------------------------ |
| **Module Name**             | `JWTKit`                                                     |
| **Proposed Maturity Level** | Incubating |
| **License**                 | [MIT](https://choosealicense.com/licenses/mit/)                            |
| **Dependencies**            | [swift-crypto 3.0.0](https://github.com/apple/swift-crypto.git)<br />[swift-certificates 1.2.0](https://github.com/apple/swift-certificates.git)<br />[BigInt 5.3.0](https://github.com/attaswift/BigInt.git) |

## Introduction

JWTs are an extremely powerful and popular tool used for safe data transfer between two parties in a distributed system and any modern server-side ecosystem provides its users with a JWT library.

JWTKit aims to fill this gap in the server-side Swift world by providing users with every possible need regarding JWTs, all based on SwiftCrypto.

## Motivation

JWTKit has been the go-to library for handling JWTs in a server-side Swift environment for quite some time. 

The library has recently undergone a major version upgrade which removed the use of BoringSSL and replaced it with SwiftCrypto, making it the perfect fit for filling the gap of a missing JWT library for the server-side Swift ecosystem.

## Proposed Solution

JWTKit provides the user with most operations needed to process JWTs, and a customisation API for the user to add options if they're not present.

As of version 5, JWTKit implements cryptographic operations using SwiftCrypto rather than BoringSSL, which is a significant improvement given the library being now Swift native. There are a couple of other JWT libraries for server-side Swift but none of them provides a completely platform-agnostic solution. JWTKit also implements `Sendable`, making it safe to use across concurrency-domains.

JWTKit's API revolves around the `JWTKeyCollection` type, which, as the name suggests, is a collection of keys which can sign and verify tokens. It's an `actor`, which means that access to its state is `async`:

```swift
let keys = JWTKeyCollection()
try await keys.addES256(key: ES256PrivateKey(pem: yourPEMString))
```

after creating a payload type:

```swift
struct TestPayload: JWTPayload { ... }
```

this collection can both sign:

```swift
let payload = TestPayload( ... )
let tokenStringRepresentation = try await keys.sign(payload)
```

and verify:

```swift
let payload = try await keys.verify(tokenStringRepresentation, as: TestPayload.self)
```

## Detailed Design

As explained above, the API provides a `JWTKeyCollection` type which contains all keys used by the user which can be filtered using a `kid` (Key Identifier) provided when adding the key to the collection.

### Algorithms

The key types try to follow SwiftCrypto's key structure, and the following algorithms are provided:

|  JWS  | Algorithm | Description                        |
| :---: | :-------: | :--------------------------------- |
| HS256 |  HMAC256  | HMAC with SHA-256                  |
| HS384 |  HMAC384  | HMAC with SHA-384                  |
| HS512 |  HMAC512  | HMAC with SHA-512                  |
| RS256 |  RSA256   | RSASSA-PKCS1-v1_5 with SHA-256     |
| RS384 |  RSA384   | RSASSA-PKCS1-v1_5 with SHA-384     |
| RS512 |  RSA512   | RSASSA-PKCS1-v1_5 with SHA-512     |
| PS256 | RSA256PSS | RSASSA-PSS with SHA-256            |
| PS384 | RSA384PSS | RSASSA-PSS with SHA-384            |
| PS512 | RSA512PSS | RSASSA-PSS with SHA-512            |
| ES256 | ECDSA256  | ECDSA with curve P-256 and SHA-256 |
| ES384 | ECDSA384  | ECDSA with curve P-384 and SHA-384 |
| ES512 | ECDSA512  | ECDSA with curve P-521 and SHA-512 |
| EdDSA |   EdDSA   | EdDSA with Ed25519                 |
| none  |   None    | No digital signature or MAC        |

> Note:
>
> While provided, the RSA key is gated behing an `Insecure` namespace to discourage new users from using it.

Each of these algorithms has its own, type-safe method for adding keys to the key collection.

### Payload and Claims

The package provides a `JWTPayload` protocol which can be conformed to when creating your own payload type:

```swift
struct ExamplePayload: JWTPayload {
    var sub: SubjectClaim
    var exp: ExpirationClaim
    var admin: BoolClaim

    func verify(using key: JWTAlgorithm) throws {
        try self.exp.verifyNotExpired()
    }
}
```

This type can then be used to sign and verify tokens.

There are some default claims as listed here:

| Claim | Type              | Verify Method                       |
| :---- | :---------------- | :---------------------------------- |
| `aud` | `AudienceClaim`   | `verifyIntendedAudience(includes:)` |
| `exp` | `ExpirationClaim` | `verifyNotExpired(currentDate:)`    |
| `jti` | `IDClaim`         | n/a                                 |
| `iat` | `IssuedAtClaim`   | n/a                                 |
| `iss` | `IssuerClaim`     | n/a                                 |
| `nbf` | `NotBeforeClaim`  | `verifyNotBefore(currentDate:)`     |
| `sub` | `SubjectClaim`    | n/a                                 |

but there's also the possibility to create custom claims using the `JWTClaim` type.

### JWKs

JWTKit provides support for using JWKs, which are keys in a JSON format:

```swift
let json = """
{
    "keys": [
        {"kty": "RSA", "alg": "RS256", "kid": "a", "n": "\(rsaModulus)", "e": "AQAB"},
        {"kty": "RSA", "alg": "RS512", "kid": "b", "n": "\(rsaModulus)", "e": "AQAB"},
    ]
}
"""

let keyCollection = try await JWTKeyCollection().use(jwksJSON: json)
// then use the keyCollection as always
```

### Customisation

JWTKit aims to encompass all possible needs of users, not by implementing all of the specs defined in the JOSE standards, but providing an API to let users implement their _own_ customisations. 

#### Header

For example, while some of the most common header fields are present as type-safe extensions, the header is a dictionary which uses `@_dynamicMemberLookup`, which means that the user can add any custom header.

```swift
let token = try await keyCollection.sign(payload, header: ["b64": true])
```

#### Parsing and Serialising

Some header fields, like the `b64` one, require that the payload may be encoded some custom, way. In particular, when the `b64` header field is set to `false`, the payload should _not_ be base64URL encoded. JWTKit provides an API to customise this behaviour by creating a custom parser and serialiser:

```swift
struct CustomSerializer: JWTSerializer {
    // Here you can set a custom encoder or just leave this as default
    var jsonEncoder: JWTJSONEncoder = .defaultForJWT

    // This method should return the payload in the way you want/need it
    func serialize(_ payload: some JWTPayload, header: JWTHeader) throws -> Data {
        // Check if the b64 header is set. If it is, base64URL encode the payload, don't otherwise
        if header.b64?.asBool == true {
            try Data(jsonEncoder.encode(payload).base64URLEncodedBytes())
        } else {
            try jsonEncoder.encode(payload)
        }
    }
}

struct CustomParser: JWTParser {
    // Here you can set a custom decoder or just leave this as default
    var jsonDecoder: JWTJSONDecoder = .defaultForJWT

    // This method parses the token into a tuple containing the various token's elements
    func parse<Payload>(_ token: some DataProtocol, as: Payload.Type) throws -> (header: JWTHeader, payload: Payload, signature: Data) where Payload: JWTPayload {
        // A helper method is provided to split the token correctly
        let (encodedHeader, encodedPayload, encodedSignature) = try getTokenParts(token)

        // The header is usually always encoded the same way
        let header = try jsonDecoder.decode(JWTHeader.self, from: .init(encodedHeader.base64URLDecodedBytes()))

        // If the b64 header field is non present or true, base64URL decode the payload, don't otherwise
        let payload = if header.b64?.asBool ?? true {
            try jsonDecoder.decode(Payload.self, from: .init(encodedPayload.base64URLDecodedBytes()))
        } else {
            try jsonDecoder.decode(Payload.self, from: .init(encodedPayload))
        }

        // The signature is usually also always encoded the same way
        let signature = Data(encodedSignature.base64URLDecodedBytes())

        return (header: header, payload: payload, signature: signature)
    }
}
```

These can be simply added to the key collection:

```swift
let keyCollection = await JWTKeyCollection()
            .addHS256(key: "secret", parser: CustomParser(), serializer: CustomSerializer())
```

### Maturity Justification

Incubating or Graduated. The package follows [minimum requirements](https://www.swift.org/sswg/incubation-process.html#minimal-requirements) and most of the [graduation ones](https://www.swift.org/sswg/incubation-process.html#graduation-requirements).

### Alternatives Considered

Kitura's [Swift-JWT](https://github.com/Kitura/Swift-JWT), old and not maintained.

[JWSETKit](https://github.com/amosavian/JWSETKit) not (yet) platform agnostic due to dependencies.
