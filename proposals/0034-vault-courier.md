# VaultCourier

* Proposal: [SSWG-0034](0034-vault-courier.md)
* Authors: [Javier Cuesta](https://github.com/jcubit)
* Review Manager: [Tim Condon](https://github.com/0xTim)
* Status: **Implemented**
* Implementation: [vault-courier](https://github.com/vault-courier/vault-courier)
* Forum Threads: [Pitch](https://forums.swift.org/t/pitch-vaultcourier/80367)

<!-- *During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Previous Revision(s): [1](https://github.com/swift-server/sswg/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal(s): [SSWG-XXXX](XXXX-filename.md)
-->

## Package Description
Swift client for interacting with Hashicorp Vault and OpenBao.

|  |  |
|--|--|
| **Package Name** | `vault-courier` |
| **Module Name** | `VaultCourier` |
| **Proposed Maturity Level** | [Sandbox](https://www.swift.org/sswg/incubation-process.html#process-diagram) |
| **License** | [Apache 2.0](https://choosealicense.com/licenses/apache-2.0/) |
| **Dependencies** | <ul><li>[swift-openapi-generator](https://github.com/apple/swift-openapi-generator)</li><li>[swift-openapi-runtime](https://github.com/apple/swift-openapi-runtime)</li><li>[swift-log](https://github.com/apple/swift-log)</li><li>[swift-openapi-async-http-client](https://github.com/swift-server/swift-openapi-async-http-client) (used only in integration tests)</li><li>[async-http-client](https://github.com/swift-server/async-http-client) (used only in integration tests)</li><li>[pkl-swift](https://github.com/apple/pkl-swift) (used only when the Pkl package trait is enabled)</li></ul>|

## Introduction

[Hashicorp Vault](https://developer.hashicorp.com/vault) and [OpenBao](https://openbao.org) are both tools for securely storing, auditing and managing access to secrets, such as API tokens, database credentials, and certificates. VaultCourier is a Swift package that can interact with Hashicorp Vault and OpenBao to retrieve and provision secrets.

## Motivation

As teams and organizations grow, managing secrets securely becomes increasingly complex. Three main challenges often arise:

- Secret Sprawl: Secrets end up scattered across infrastructure and repositories, making them hard to manage consistently.
- Lack of Auditability: Large teams often lack visibility into who has access to which secrets, and there's no reliable audit trail to track historical access. In particular, in the context of applications and machines.
- Difficult Secret Rotation: When secrets are spread everywhere, rotating them quickly in response to a breach becomes a serious operational hurdle.

Tools like Hashicorp Vault and OpenBao provide robust, battle-tested solutions to these issues. However, the Swift ecosystem lacks native support for integrating with these tools. While other languages have dedicated clients, Swift applications—especially server-side—could benefit from direct interaction with Vault. A native Swift client could also help for custom orchestration tools built entirely in Swift.

## Proposed solution

I've been working for about a year on VaultCourier, a Swift package that serves as a native client for HashiCorp Vault and OpenBao. Built using `swift-openapi`, it runs on Linux, macOS, and potentially other platforms (if you don't rely on the `swift-openapi-generator`). Unless otherwise specified, I'll refer to both HashiCorp Vault and OpenBao simply as Vault.

VaultCourier currently supports:

- Key/Value (KV v2) Secret Storage: you can store arbitrary JSON object secrets under named keys.
- Static and Dynamic Database Credential Generation. The latter avoids secret sharing by generating unique, short-lived credentials per application.
- Token and AppRole Authentication: AppRole provides identity-based authentication for applications, allowing fine-grained secret access control and auditability. In the event of a breach, compromised credentials can be rotated quickly, limiting the affected blast radius.
- Response Wrapping for Login: adds an extra layer of security by delivering secrets in a one-time-use, time-limited wrapper token.
- Pkl resource reader. More on this on the detailed design section.

A basic example of usage:

```swift
import Foundation
import VaultCourier
import OpenAPIAsyncHTTPClient
import Logging

struct AppSecrets: Decodable, Sendable {
    let apiToken: String
}

// 1. Create a vault client.
let vaultConfiguration = VaultClient.Configuration(
    apiURL: try URL(validatingOpenAPIServerURL: "http://127.0.0.1:8200/v1"),
    backgroundActivityLogger: Logger(label: "my-app")
)

let vaultClient = VaultClient(
    configuration: vaultConfiguration,
    client: Client(
        serverURL: vaultConfiguration.apiURL,
        transport: AsyncHTTPClientTransport()
    ),
    authentication: .token("education")
)
// 2. Authenticate with vault
try await vaultClient.authenticate()

// 3. Read secret
let secret: AppSecrets = try await vaultClient.readKeyValueSecret(key: "dev-eu-central-1")

// Use the secret to start your service or app...
```

To see more examples in detail please check out the [vault-courier-examples](https://github.com/vault-courier/vault-courier-examples) repository and the [tutorials](https://swiftpackageindex.com/vault-courier/vault-courier/main/tutorials/vault-courier). In particular, I have written an [end-to-end tutorial](https://swiftpackageindex.com/vault-courier/vault-courier/main/tutorials/vaultcourier/approle-hummingbird) of integrating VaultCourier into Hummingbird’s Todo example. An equivalent example in Vapor is also planned.


## Detailed design

VaultCourier is under development and currently supports the aforementioned features. The full API documentation can be found hosted in the [SPI](https://swiftpackageindex.com/vault-courier/vault-courier/main/documentation/vault-courier#API).

The actor `VaultClient` is VaultCourier's REST client for Hashicorp Vault and OpenBao. Before a client can interact with Vault, it must authenticate against an auth method. `VaultClient` protects the state of the mutating token during this process. Regardless of which authentication method was chosen during initialization, `VaultClient` always authenticates using the `authenticate()` method. In accordance with OpenAPI best practices, VaultCourier wraps an OpenAPI-generated client. Vault and OpenBao support a wide range of authentication methods. VaultCourier currently focuses on [AppRole](https://www.hashicorp.com/en/blog/how-and-why-to-use-approle-correctly-in-hashicorp-vault), as it's a cloud-agnostic method that works across diverse environments. With interest and contributions from the community, we're open to adding support for other authentication methods such as AWS, Google Cloud, Azure, AliCloud, GitHub, and more. 

During the initialization of a `VaultClient` we pass the desired `ClientTransport` and configuration parameters with the base URL of the Vault. In the configuration you can change the default paths to the secret engines or auth methods, otherwise the client is initialized with the default values.

As mentioned earlier, a key design decision in VaultCourier is the use of [swift-openapi-generator](https://github.com/apple/swift-openapi-generator). The client is built in a spec-driven manner, which brings several practical benefits. For example, the generated code is transport-agnostic, making it compatible with both Linux and macOS. It also supports mocking the API, which is useful for development and testing; developers can simulate Vault responses without needing a live Vault instance. `VaultCourier` includes such a mock client to simplify local development. For example:

```swift
let vaultClient: VaultClient
switch deploymentEnvironment {
    case .dev:
        vaultClient = try makeDevelopmentVaultClient()
    case .prod:
        vaultClient = try makeProductionVaultClient()
}

try await vaultClient.authenticate()
```

where the factory function for the VaultClient in the development enviroment uses the `MockClient`:

```swift
func makeDevelopmentVaultClient() throws -> VaultClient {
    let roleId = "59d6d1ca-47bb-4e7e-a40b-8be3bc5a0ba8"
    let secretId = "84896a0c-1347-aa90-a4f6-aca8b7558780"
    var client = MockClient()
    client.authApproleLoginAction = { input in 
      // ...
    }
    client.readKvSecretsAction = { input in
      // ...
    }

    let vaultConfiguration = // ...

    // Alternatively you can init directly with a token
    return .init(
        configuration: vaultConfiguration,
        client: client,
        authentication: .appRole(
            credentials: .init(roleID: roleID, secretID: secretID),
            isWrapped: false)
    )
}
```

In this snippet, `MockClient` responses are set using a closure to return a fixed value. These values can be loaded from an external source, such as a `.env` or `.pkl` file. Alternatively, different authentication methods can be used within these factory functions.

[Pkl](https://pkl-lang.org) also plays an important role in the project. The OpenAPI specification that VaultCourier uses is written using the [org.openapis.v3 Pkl package](https://pkl-lang.org/package-docs/pkg.pkl-lang.org/pkl-pantry/org.openapis.v3/current/index.html). Using Pkl helps avoid common mistakes that can occur when hand-writing OpenAPI in JSON or YAML, and it allows for modularizing the spec. This modularity becomes especially useful as OpenBao, the open-source fork of HashiCorp Vault, begins to diverge from Vault upstream. We maintain this specification in a separate repository along with common pkl templates for configuring Vault: [vault-courier-pkl](https://github.com/vault-courier/vault-courier-pkl).

Pkl is also used at runtime when the Pkl package trait is enabled in VaultCourier. This trait adds custom resource readers that allow Pkl configuration files to reference secrets stored in Vault without exposing them directly. For example, a Pkl config file might look like:

```pkl
amends "ServerConfig.pkl"

logLevel = "debug"

// Calls a Vault KV endpoint with version 2 of the secret
apiToken = read("vault:secrets/api_token?version=2").text

database {
  hostname = "localhost"
  
  port = 8080
  
  // Calls a Vault database endpoint
  credentials = read("vault:database/static-creds/server_role").text
}
```

The secrets here are read via resource URI schemes. VaultCourier provides an implementation for the default URI scheme "vault", though this name can be customized in the initialization of the Vault client. Moreover, users can customize how the URL is parsed by conforming to the protocol `ResourceReaderStrategy`. This setup allows applications to use Pkl as their primary configuration format while securely referencing secrets. A tutorial that extends the Todo app with this feature is available here: [Todos App with VaultCourier and Pkl](https://swiftpackageindex.com/vault-courier/vault-courier/main/tutorials/vaultcourier/approle-hummingbird-pkl). The class behind this feature is `VaultResourceReader` (available only if the Pkl trait is enabled) which implements the `PKlSwift.ResourceReader` protocol.

VaultCourier currently supports PostgreSQL vault integration, and with community contributions, support for more databases can be added. See the full [list of Hashicorp-Vault supported databases](https://developer.hashicorp.com/vault/docs/secrets/databases#database-capabilities); these list includes many of the databases already supported in the Swift ecosystem.


### Ideas for future directions

The Vault API used by application developers is relatively small compared to what is needed by operations developers (Vault admins). This package aims to support both use cases. We plan to further modularize the package and provide more targeted support for endpoints based on selected package traits, helping avoid including unnecessary code.

Currently, the package relies on a single OpenAPI specification. One idea is to split this specification into three separate documents: AuthMethods, SecretEngines, and SystemBackend APIs. This would result in three additional internal targets, each with its own OpenAPI document. These targets can also be refined using package traits to include only the required functionality. For example, AuthMethods always includes support for token and AppRole, while GitHub and AWS support would be optional and enabled via specific traits.

Since the OpenAPI specification is already generated using Pkl, we can continue leveraging Pkl for this structured approach. Note that we are already [forming the main document via importing such sub-documents](https://github.com/vault-courier/vault-courier-pkl/blob/3037eced438b5a970e20b1fb651347814df6f6b1/VaultOpenApi.pkl#L24).

## Maturity Justification

Sandbox. VaultCourier adheres to the minimal requirements of the SSWG Incubation Process. A stable version of the API is in development and feedback is welcomed.

## Alternatives considered

An alternative to building a Vault client with OpenAPI is to manually implement each endpoint offered by the Vault API directly in a Swift package. This is the approach taken by many clients in other languages, and also by a recent Swift package called [SwiftVault](https://swiftpackageindex.com/jmccloud827/VaultSwift).
While this method can work well, it comes with a few trade-offs:
It typically ties the client to a specific HTTP transport. For example, SwiftVault uses URLSession, which limits its compatibility with Linux, an important consideration for server-side Swift development.
It lacks built-in support for mocks, which can make development and testing more difficult.
SwiftVault does appear to support many Vault features. However, manually maintaining a large and evolving API without OpenAPI can make it harder to keep the client up to date and well-documented.
