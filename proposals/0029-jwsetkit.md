# JWSETKit

 * Proposal: [SSWG-0029](0029-jwsetkit.md)
 * Authors: [Amir-Abbas Mousavian](https://github.com/amosavian)
 * Review Manager: [Konrad 'ktoso' Malawski](https://github.com/ktoso)
 * Status: **Implemented**
 * Implementation: [amosavian/JWSETKit](https://github.com/amosavian/JWSETKit)
 * Forum Threads: [Pitch](https://forums.swift.org/t/pitch-jwt-jws-jwt-support-kit/68113), [Discussion](https://forums.swift.org/), [Review](https://forums.swift.org/)

## Package Description

Lightweight, modular, modern, extensible JWS/JWE/JWT framework written in Swift.

 |  |  |
 |--|--|
 | **Package Name** | `JWSETKit` |
 | **Module Name** | `JWSETKit` |
 | **Proposed Maturity Level** | [Sandbox](https://www.swift.org/sswg/incubation-process.html) |
 | **License** | [MIT](https://github.com/amosavian/JWSETKit/blob/main/LICENSE) |
 | **Dependencies** | [AnyCodable](https://github.com/Flight-School/AnyCodable), [CryptoSwift](https://github.com/krzyzanowskim/CryptoSwift), [swift-asn1](https://github.com/apple/swift-asn1), [swift-certificates](https://github.com/apple/swift-certificates), [swift-crypto](https://github.com/apple/swift-crypto) |

## Introduction

We propose that the [JWSETKit](https://github.com/amosavian/JWSETKit) library be considered for incubation by the Swift Server Workgroup. JWSETKit is a Swift library that provides a comprehensive suite of tools for handling JSON Web Encryption (JWE), JSON Web Signatures (JWS), and JSON Web Tokens (JWT), which are crucial for secure data transmission and authentication in server-side Swift applications.

## Motivation

Security is a paramount concern in server-side development. The JWSETKit library provides a robust and Swift-native solution for handling JWE, JWS, and JWT. These are widely used standards for securing data and managing user authentication and authorization. By incubating JWSETKit, the Swift Server Workgroup can help to ensure that server-side Swift developers have access to high-quality, community-vetted tools for secure data handling and user authentication.

## Detailed Design

JWSETKit library provides following objectives according to RFCs:

### JWA

All algorithms defined in [RFC7518](https://tools.ietf.org/html/rfc7518) and
[RFC8037](https://tools.ietf.org/html/rfc8037) except `ECDH-ES` related
algorithms are supported. Algorithms can be categorized into three types,
which are supported by this library:

* `JSONWebSignatureAlgorithm` for JWS algorithms.
* `JSONWebKeyEncryptionAlgorithm` for JWE key encryption algorithms.
* `JSONWebContentEncryptionAlgorithm` for JWE content encryption algorithms.

All types conform to `JSONWebAlgorithm` and can be type-erased using `AnyJSONWebAlgorithm`.

New algorithms can be defined and registered using a `register` method of each type by declaring public and private key classes and associated data.

### JWK

All key types including CommonCrypto's `SecKey` and `SecCertificate` and
CryptoKit's types including `P256`, `P384`, `P521` and `SymmetricKey` can be
encoded to JWK using `JSONEncoder` or be deserialized from JWK using `JSONDecoder`.

This library also provides custom key types:
1. To support other key types such as `AES-CBC-HMAC`, `AES-GCM` and `HMAC`.
2. To provide container types for `RSA` and `EC` private and public keys.

#### JWK Functionality

`JSONWebKey` is a generic type that defines general key structure. For functionality, it's divided into following protocols:

* `JSONWebEncryptingKey` acts as a public key for encryption.
* `JSONWebDecryptingKey` acts as a private key for decryption.
* `JSONWebSealingKey` acts as a symmetric key for authenticated encryption.
* `JSONWebValidatingKey` acts as a public key for signature verification.
* `JSONWebSigningKey` acts as a private key for signing.

#### JWK Set

`JSONWebKeySet` is a container for `JSONWebKey` objects compliant to [RFC7517](https://tools.ietf.org/html/rfc7517). It conforms to `Collection` protocol.

### JWS

`JSONWebSignature` is a generic type that can be used to sign and verify any
payload type conforming to `JWSPayload` protocol.
This library provides `ProtectedWebContainer` and `TypedProtectedWebContainer`
protocols to support different payload types.

These concrete implementations of Web Containers are provided:
* `ProtectedDataWebContainer` for raw/untyped data.
* `ProtectedJSONWebContainer` for payloads that should be encoded as JSON using `JSONWebContainer` as base protocol.

`JSONWebContainer` has a storage property which contains data as dictionary.
It's easy to extend conformed types using `@dynamicMemberLookup` to define various accessors to registered keys.

### JWT

JWT is specialized `JSONWebSignature` with `JSONWebTokenClaims` as payload.
Some convenience methods are provided to create, parse and verify JWTs.

Default implementation in library contains following claims defined in [IANA JSON Web Token Claims Registry](https://www.iana.org/assignments/jwt/jwt.xhtml):
* Claims registered in [RFC7519](https://tools.ietf.org/html/rfc7519)
  are defined in `JSONWebTokenClaimsRegisteredParameters`.
* Claims registered in [RFC8693](https://tools.ietf.org/html/rfc8693)
  are defined in `JSONWebTokenClaimsOAuthParameters`.
* Claims for `id_token` registered in [OpenID Connect Core Section 2](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)
  are defined in `JSONWebTokenClaimsPublicOIDCAuthParameters`.
* Claims for `id_token` registered in [OpenID Connect Core Section 5.1](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims)
  are defined in `JSONWebTokenClaimsPublicOIDCStandardParameters` with localization support.

Other claims can easily be defined by declaring parameters using
`JSONWebContainerParameters` and and extending `JSONWebTokenClaims`
to implement dynamic member lookup for parameters.

### JWE

JWE is supported using `JSONWebEncryption` type encryption and decryption of data using given keys.

A new JWE can be created using `init(protected:content:additionalAuthenticatedData:keyEncryptingAlgorithm:keyEncryptionKey:contentEncryptionAlgorithm:contentEncryptionKey:)` method with given data and keys.

An existing JWE can be decrypted `decrypt(using:keyId:)` method.

## Impact

Incubating JWSETKit will have several positive impacts on the server-side Swift community:

- It will provide developers with a robust, Swift-native solution for handling JWE, JWS, and JWT.
- It will help to promote best practices for secure data handling and user authentication in server-side Swift development.
- It will contribute to the ongoing growth and maturity of the server-side Swift ecosystem.

## Maturity Justification

 Requirements for maturity level "Sandbox":

 **General**
 - ✅ Has relevance to Swift on Server specifically
 - ✅ Publicly accessible source managed by an SCM such as github.com or similar
 - ✅ Prefer to use main as the default branch name, in line with Swift's guidelines
 - 🔳 Adopt the Swift Code of Conduct

 **Ecosystem**
 - ✅ Uses SwiftPM
 - ✅ Integrated with critical SSWG ecosystem building blocks, e.g., Logging and Metrics APIs, SwiftNIO for IO

 **Longevity**
 - 🔳 Must be from a team that has more than one public repository (or similar indication of experience)
 - 🔳 SSWG should have access / authorization to graduated repositories in case of emergency
 - 🔳 Adopt the SSWG Security Best Practices

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

## Alternatives considered

The only viable alternative which supports Linux is  Vapor's [jwt-kit], which is a mature library that supports JWT.
However, it does not support JWE and JWS. Also It does not provides interoperability with CommonCrypto and CryptoKit.
