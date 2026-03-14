# Java SAML Toolkit

## Framework Overview

### Purpose

`java-saml` is a **SAML 2.0 Service Provider (SP) toolkit** forked from [OneLogin's java-saml](https://github.com/onelogin/java-saml). It implements the SP side of the SAML protocol, enabling Nirmata services to authenticate users via external Identity Providers (IdPs) using Single Sign-On (SSO) and Single Logout (SLO).

### Problems Solved

- **SP-initiated SSO** — generates `AuthnRequest` XML, redirects users to the IdP, processes and validates `SAMLResponse` from the IdP, extracts user attributes and session data
- **SP-initiated SLO** — generates `LogoutRequest` XML, processes `LogoutResponse` from the IdP, handles IdP-initiated `LogoutRequest` by sending back a `LogoutResponse`
- **SP metadata generation** — produces SAML metadata XML for the SP, which IdPs consume during federation setup
- **IdP metadata parsing** — imports IdP configuration (entity ID, SSO/SLO URLs, certificates) from IdP metadata XML
- **XML signature verification** — validates XML signatures on SAML responses and assertions using Apache XML Security
- **Encrypted assertion support** — decrypts encrypted SAML assertions, with optional HSM (Azure Key Vault) support for key operations
- **Properties-based configuration** — all SP and IdP settings are loaded from a properties file (`onelogin.saml.properties`)

### Architectural Role

```
┌──────────────────────┐         ┌──────────────────────┐
│   Nirmata Service    │         │   Identity Provider   │
│   (Service Provider) │         │   (e.g., Okta, ADFS,  │
│                      │         │    Azure AD)           │
│  ┌────────────────┐  │         │                        │
│  │  Auth class    │  │  SAML   │                        │
│  │  (java-saml)   │──┼─────────┼──────────────────────  │
│  │                │  │ Redirect│                        │
│  │ login()        │──┼────────>│ SSO URL                │
│  │ processResponse│<─┼─────── │ SAMLResponse           │
│  │ logout()       │──┼────────>│ SLO URL                │
│  │ processSLO()   │<─┼─────── │ LogoutResponse         │
│  └────────────────┘  │         │                        │
└──────────────────────┘         └──────────────────────────┘
```

### Design Philosophy

- **Stateful per-request** — `Auth` instances are created per HTTP request and are **not thread-safe**
- **Properties-driven** — all configuration is loaded from a `.properties` file; no annotations or DI
- **Factory pattern** — `SamlMessageFactory` allows overriding default SAML message creation for customization
- **Framework-agnostic core** — the `core` module has no servlet dependency; the `toolkit` module bridges to `javax.servlet`

### Build System

**Maven** (not Gradle like other Nirmata Java repos).

| Property | Value |
|----------|-------|
| GroupId | `com.onelogin` |
| Version | `2.9.1-SNAPSHOT` |
| Java | 1.8+ (source/target) |
| Servlet API | `javax.servlet-api` 4.0.1 (provided) |
| XML Security | Apache `xmlsec` 3.0.2 |
| OWASP | dependency-check-maven, CVSS ≥7 fails build |

### Repository Structure

```
java-saml/
├── core/                    java-saml-core: protocol logic, settings, XML, crypto
│   ├── src/main/java/       31 Java source files
│   └── src/main/resources/  14 XSD schemas for SAML validation
├── toolkit/                 java-saml: servlet integration (Auth, ServletUtils)
│   └── src/main/java/       3 Java source files
├── samples/                 JSP sample web app
│   └── java-saml-tookit-jspsample/
├── pom.xml                  Parent POM
└── README.md
```

---

## Core Concepts

### SAML Flows

#### SP-Initiated SSO (Login)

```
1. Service calls Auth.login()
2. Auth creates AuthnRequest XML (with ID, issuer, ACS URL, etc.)
3. AuthnRequest is deflated, base64-encoded, and optionally signed
4. User is redirected to IdP SSO URL with SAMLRequest parameter
5. User authenticates at IdP
6. IdP sends SAMLResponse via HTTP POST to the ACS URL
7. Service calls Auth.processResponse()
8. Auth validates the SAMLResponse (signature, conditions, audience, etc.)
9. On success: Auth.isAuthenticated() = true, attributes available via Auth.getAttributes()
```

#### SP-Initiated SLO (Logout)

```
1. Service calls Auth.logout()
2. Auth creates LogoutRequest XML
3. LogoutRequest is deflated, base64-encoded, and optionally signed
4. User is redirected to IdP SLO URL with SAMLRequest parameter
5. IdP processes logout
6. IdP sends LogoutResponse via HTTP Redirect with SAMLResponse parameter
7. Service calls Auth.processSLO()
8. Auth validates LogoutResponse (signature, status, inResponseTo)
9. On success: session is invalidated
```

#### IdP-Initiated SLO

```
1. IdP sends LogoutRequest to SP's SLO URL via HTTP Redirect
2. Service calls Auth.processSLO()
3. Auth validates the incoming LogoutRequest
4. Auth creates LogoutResponse with status SUCCESS
5. User is redirected back to IdP SLO Response URL with SAMLResponse parameter
```

### Settings Configuration

All SAML configuration is loaded from a properties file (default: `onelogin.saml.properties` on the classpath). Key properties:

**SP Configuration:**

| Property | Description |
|----------|-------------|
| `onelogin.saml2.sp.entityid` | SP entity ID (unique identifier) |
| `onelogin.saml2.sp.assertion_consumer_service.url` | ACS URL (where IdP sends SAMLResponse) |
| `onelogin.saml2.sp.assertion_consumer_service.binding` | ACS binding (default: HTTP-POST) |
| `onelogin.saml2.sp.single_logout_service.url` | SLO URL (where IdP sends LogoutRequest/Response) |
| `onelogin.saml2.sp.single_logout_service.binding` | SLO binding (default: HTTP-Redirect) |
| `onelogin.saml2.sp.nameidformat` | NameID format (default: unspecified) |
| `onelogin.saml2.sp.x509cert` | SP certificate (PEM, single line) |
| `onelogin.saml2.sp.privatekey` | SP private key (PEM, single line) |

**IdP Configuration:**

| Property | Description |
|----------|-------------|
| `onelogin.saml2.idp.entityid` | IdP entity ID |
| `onelogin.saml2.idp.single_sign_on_service.url` | IdP SSO endpoint |
| `onelogin.saml2.idp.single_logout_service.url` | IdP SLO endpoint |
| `onelogin.saml2.idp.single_logout_service.response.url` | IdP SLO response endpoint |
| `onelogin.saml2.idp.x509cert` | IdP certificate for signature verification |
| `onelogin.saml2.idp.certfingerprint` | Alternative: certificate fingerprint |

**Security Settings:**

| Property | Description |
|----------|-------------|
| `onelogin.saml2.security.authnrequest_signed` | Sign AuthnRequests |
| `onelogin.saml2.security.logoutrequest_signed` | Sign LogoutRequests |
| `onelogin.saml2.security.logoutresponse_signed` | Sign LogoutResponses |
| `onelogin.saml2.security.want_messages_signed` | Require signed messages from IdP |
| `onelogin.saml2.security.want_assertions_signed` | Require signed assertions |
| `onelogin.saml2.security.want_assertions_encrypted` | Require encrypted assertions |
| `onelogin.saml2.security.want_nameid` | Require NameID in response |
| `onelogin.saml2.security.signature_algorithm` | Signature algorithm (default: RSA-SHA1) |
| `onelogin.saml2.security.digest_algorithm` | Digest algorithm (default: SHA1) |
| `onelogin.saml2.security.reject_deprecated_alg` | Reject deprecated algorithms |

Settings can also be constructed programmatically via `SettingsBuilder`:

```java
Saml2Settings settings = new SettingsBuilder()
    .fromFile("onelogin.saml.properties")
    .build();
```

Or from a `Map<String, Object>`:

```java
Map<String, Object> samlData = new HashMap<>();
samlData.put(SettingsBuilder.SP_ENTITYID_PROPERTY_KEY, "https://sp.example.com");
// ... more properties
Saml2Settings settings = new SettingsBuilder().fromValues(samlData).build();
```

### HttpRequest Abstraction

The `core` module uses `HttpRequest` (in `com.onelogin.saml2.http`) instead of `javax.servlet.http.HttpServletRequest` to remain framework-agnostic. `HttpRequest` holds:
- `requestURL` — the URL without query parameters
- `parameters` — `Map<String, List<String>>` of query/POST parameters
- `queryString` — the raw query string (needed for HTTP Redirect binding signature verification)

The `toolkit` module's `ServletUtils.makeHttpRequest()` converts `HttpServletRequest` → `HttpRequest`.

---

## Modules

### core (`java-saml-core`)

The protocol implementation with no servlet dependency.

**Packages:**

| Package | Purpose |
|---------|---------|
| `com.onelogin.saml2.authn` | `AuthnRequest` generation, `SamlResponse` parsing/validation |
| `com.onelogin.saml2.logout` | `LogoutRequest` and `LogoutResponse` generation/parsing |
| `com.onelogin.saml2.settings` | `Saml2Settings`, `SettingsBuilder`, `Metadata`, `IdPMetadataParser` |
| `com.onelogin.saml2.http` | `HttpRequest` — framework-agnostic HTTP request representation |
| `com.onelogin.saml2.model` | Data models: `Contact`, `Organization`, `KeyStoreSettings`, `SamlResponseStatus`, etc. |
| `com.onelogin.saml2.model.hsm` | HSM integration: `HSM` abstract class, `AzureKeyVault` implementation |
| `com.onelogin.saml2.exception` | `Error`, `SAMLException`, `SettingsException`, `ValidationError`, `XMLEntityException` |
| `com.onelogin.saml2.util` | `Constants` (SAML URIs), `Util` (XML/crypto/encoding), `SchemaFactory`, `Preconditions` |

**Resources:** 14 XSD schema files for XML validation (`saml-schema-protocol-2.0.xsd`, `saml-schema-assertion-2.0.xsd`, `xmldsig-core-schema.xsd`, etc.)

### toolkit (`java-saml`)

Servlet-based high-level API (3 files). Depends on `java-saml-core` and `javax.servlet-api` 4.0.1.

| Class | Purpose |
|-------|---------|
| `Auth` | Main entry point — orchestrates SSO and SLO flows |
| `ServletUtils` | Converts `HttpServletRequest` → `HttpRequest`, URL utilities, redirect helper |
| `SamlMessageFactory` | Factory interface for customizing SAML message creation |

### samples (`java-saml-tookit-jspsample`)

JSP-based demo web app showing SSO/SLO integration. No Java code — only JSP files that use the `Auth` class directly.

---

## Key Interfaces and Classes

### Class: `Auth`

**Package:** `com.onelogin.saml2` (toolkit module)

**Purpose:** Main entry point for SP SAML operations. Stateful, not thread-safe — create one per request.

**Constructors:**
- `Auth()` — loads settings from `onelogin.saml.properties`
- `Auth(String filename, HttpServletRequest, HttpServletResponse)` — custom settings file
- `Auth(Saml2Settings, HttpServletRequest, HttpServletResponse)` — programmatic settings
- Also accepts `KeyStoreSettings` for Java KeyStore-based certificate loading

**Key methods:**

| Method | Description |
|--------|-------------|
| `login()` | Initiates SSO: creates AuthnRequest, redirects to IdP |
| `login(String relayState, AuthnRequestParams params)` | SSO with custom parameters |
| `processResponse()` | Validates SAMLResponse from IdP, extracts user data |
| `logout()` | Initiates SLO: creates LogoutRequest, redirects to IdP |
| `logout(String relayState, LogoutRequestParams params)` | SLO with custom parameters |
| `processSLO()` | Handles incoming LogoutResponse or LogoutRequest from IdP |
| `isAuthenticated()` | Returns true after successful `processResponse()` |
| `getAttributes()` | Returns `Map<String, List<String>>` of SAML attributes |
| `getNameId()` | Returns the authenticated user's NameID |
| `getSessionIndex()` | Returns the SAML session index |
| `getErrors()` | Returns validation errors from the last operation |
| `getLastErrorReason()` | Returns the reason string for the last error |
| `setSamlMessageFactory(SamlMessageFactory)` | Overrides default SAML message creation |

**State after `processResponse()`:**
- `authenticated` — whether the response was valid
- `nameid`, `nameidFormat` — the user's NameID
- `attributes` — SAML attributes (e.g., email, groups)
- `sessionIndex`, `sessionExpiration` — SAML session data
- `lastMessageId`, `lastAssertionId` — IDs for replay prevention

### Class: `SamlResponse`

**Package:** `com.onelogin.saml2.authn` (core module)

**Purpose:** Parses and validates a SAML 2.0 Response received from the IdP.

**Validation checks (`isValid()`):**
- XML schema validation against SAML 2.0 XSD
- Response status code is `Success`
- Assertion exists and audience matches SP entity ID
- Time conditions (NotBefore, NotOnOrAfter) with clock skew tolerance (3 minutes)
- Subject confirmation (recipient URL, InResponseTo correlation)
- Signature verification on Response and/or Assertion
- Encrypted assertion decryption (if applicable)
- Issuer matches IdP entity ID

### Class: `AuthnRequest`

**Package:** `com.onelogin.saml2.authn` (core module)

**Purpose:** Generates SAML 2.0 AuthnRequest XML using string template substitution.

**Features:**
- Configurable ForceAuthn, IsPassive, NameIDPolicy
- Optional Subject element for specifying requested NameID
- RequestedAuthnContext for specifying authentication context classes
- Provider name from Organization display name
- Extensible via `postProcessXml()` override

### Class: `Saml2Settings`

**Package:** `com.onelogin.saml2.settings` (core module)

**Purpose:** Holds all SP and IdP configuration. Constructed by `SettingsBuilder`.

**Key fields:**
- SP: entity ID, ACS URL/binding, SLO URL/binding, NameID format, X.509 cert, private key
- IdP: entity ID, SSO URL/binding, SLO URL/binding, X.509 cert(s), cert fingerprint
- Security: signing flags, encryption flags, algorithm preferences, strict mode
- Compression: request/response deflate settings
- HSM: optional `HSM` instance for hardware key operations

### Class: `Metadata`

**Package:** `com.onelogin.saml2.settings` (core module)

**Purpose:** Generates SP metadata XML from `Saml2Settings`. Used for federation setup with IdPs. Metadata includes SP entity ID, ACS/SLO endpoints, certificate, organization, and contacts.

### Class: `IdPMetadataParser`

**Package:** `com.onelogin.saml2.settings` (core module)

**Purpose:** Parses IdP metadata XML to extract configuration values (entity ID, SSO/SLO URLs, certificates) into a `Map` compatible with `SettingsBuilder`.

### Interface: `SamlMessageFactory`

**Package:** `com.onelogin.saml2.factory` (toolkit module)

**Purpose:** Factory for creating SAML message objects. All methods have default implementations that create standard instances. Override to customize XML generation:

```java
auth.setSamlMessageFactory(new SamlMessageFactory() {
    @Override
    public AuthnRequest createAuthnRequest(Saml2Settings settings, AuthnRequestParams params) {
        return new CustomAuthnRequest(settings, params);
    }
});
```

### Class: `HSM` / `AzureKeyVault`

**Package:** `com.onelogin.saml2.model.hsm` (core module)

**Purpose:** Abstract HSM interface for cryptographic operations (key wrapping, encryption, decryption) with an Azure Key Vault implementation. Used for decrypting encrypted SAML assertions when the private key is stored in an HSM.

---

## Typical Usage

### SSO Integration

```java
// In the login endpoint
Auth auth = new Auth(request, response);
auth.login();  // redirects to IdP

// In the ACS (Assertion Consumer Service) endpoint
Auth auth = new Auth(request, response);
auth.processResponse();
if (auth.isAuthenticated()) {
    String nameId = auth.getNameId();
    Map<String, List<String>> attributes = auth.getAttributes();
    String email = attributes.get("email").get(0);
    // Create application session
} else {
    List<String> errors = auth.getErrors();
    String reason = auth.getLastErrorReason();
    // Handle error
}
```

### SLO Integration

```java
// In the logout endpoint
Auth auth = new Auth(request, response);
auth.logout();  // redirects to IdP

// In the SLS (Single Logout Service) endpoint
Auth auth = new Auth(request, response);
auth.processSLO();
List<String> errors = auth.getErrors();
if (errors.isEmpty()) {
    // Logout succeeded, session invalidated
} else {
    // Handle error
}
```

---

## Framework Conventions

### Configuration File

The default configuration file is `onelogin.saml.properties` on the classpath. Property keys use the prefix `onelogin.saml2.` with dot-separated hierarchical names.

### Certificate Format

SP and IdP certificates are specified as single-line PEM strings (without `-----BEGIN/END CERTIFICATE-----` headers) in the properties file.

### Binding Types

- **SSO**: Default IdP binding is HTTP-Redirect; default SP ACS binding is HTTP-POST
- **SLO**: Default binding is HTTP-Redirect for both request and response

### Strict Mode

When `onelogin.saml2.strict=true` (default), all validations are enforced. When `false`, some validations are relaxed — this should only be used during development.

### Clock Skew

SAML time condition validations allow a 3-minute clock drift (`Constants.ALOWED_CLOCK_DRIFT = 180` seconds).

### RelayState

When `relayState` is `null` in `login()` or `logout()`, it defaults to the current request URL. When empty string, no RelayState parameter is sent.

---

## Integration with Platform Infrastructure

### Identity Providers

Compatible with any SAML 2.0 IdP:
- Okta
- Azure AD / Entra ID
- ADFS
- OneLogin
- Keycloak
- PingFederate
- Shibboleth

### Azure Key Vault (HSM)

`AzureKeyVault` integrates with Azure Key Vault for secure key management:
- Uses `azure-security-keyvault-keys` SDK (4.7.0, optional dependency)
- Authenticates via `ClientSecretCredential` (client ID, secret, tenant ID)
- Supports key wrapping (RSA 1.5, RSA-OAEP, AES key wrap) and encryption

### Servlet Container

The `toolkit` module requires a `javax.servlet` 4.0.1 compatible container (Tomcat, Jetty, Undertow). The `core` module is container-independent.

### XML Security

Uses Apache XML Security (`xmlsec`) 3.0.2 for:
- XML signature generation and verification
- XML encryption and decryption
- Canonicalization (C14N, exclusive C14N)
- Supported algorithms: RSA-SHA1, RSA-SHA256, RSA-SHA384, RSA-SHA512, DSA-SHA1

---

## Important Code Paths

### Auth.login() Flow

```
Auth.login(relayState, authnRequestParams, stay, parameters)
→ SamlMessageFactory.createAuthnRequest(settings, params)
→ AuthnRequest constructor:
  → generates unique ID
  → builds XML via template substitution (StrSubstitutor)
  → calls postProcessXml() for customization
→ authnRequest.getEncodedAuthnRequest()
  → deflates XML (if compression enabled)
  → base64 encodes
→ if authnRequestsSigned:
  → buildRequestSignature() → Util.sign() with SP private key
  → adds SigAlg and Signature parameters
→ stores lastRequestId, lastRequest
→ ServletUtils.sendRedirect(response, ssoUrl, parameters, stay)
```

### Auth.processResponse() Flow

```
Auth.processResponse(requestId)
→ ServletUtils.makeHttpRequest(request)
→ extracts SAMLResponse parameter from POST
→ SamlMessageFactory.createSamlResponse(settings, httpRequest)
  → SamlResponse constructor: decodes, parses XML, decrypts if needed
→ samlResponse.isValid(requestId)
  → validates XML schema
  → checks status code = Success
  → verifies signatures (response and/or assertion)
  → validates audience, conditions, subject confirmation
  → validates InResponseTo correlation
→ if valid:
  → extracts NameID, attributes, sessionIndex, sessionExpiration
  → sets authenticated = true
→ if invalid:
  → stores error reason, validation exception
  → adds error codes to errors list
```

---

## Extension Points

### Custom SAML Messages via SamlMessageFactory

Override any SAML message creation by implementing `SamlMessageFactory`:

```java
public class CustomFactory implements SamlMessageFactory {
    @Override
    public AuthnRequest createAuthnRequest(Saml2Settings settings, AuthnRequestParams params) {
        return new MyAuthnRequest(settings, params);
    }
}

auth.setSamlMessageFactory(new CustomFactory());
```

### Custom AuthnRequest XML via postProcessXml

Extend `AuthnRequest` and override `postProcessXml()` to modify the generated XML:

```java
public class CustomAuthnRequest extends AuthnRequest {
    @Override
    protected String postProcessXml(String xml, AuthnRequestParams params, Saml2Settings settings) {
        // Add custom extensions, modify XML, etc.
        return modifiedXml;
    }
}
```

### Custom HSM Integration

Extend `HSM` to integrate with different hardware security modules:

```java
public class AwsCloudHSM extends HSM {
    @Override
    public void setClient() { /* AWS CloudHSM client setup */ }
    @Override
    public byte[] unwrapKey(String algorithmUrl, byte[] wrappedKey) { /* ... */ }
    // ... other methods
}
```

---

## Known Limitations

### javax.servlet (Not Jakarta)

The `toolkit` module uses `javax.servlet-api` 4.0.1, not `jakarta.servlet`. Quarkus 3.x and Spring Boot 3.x services using `jakarta.servlet` cannot directly use the `toolkit` module's `Auth` and `ServletUtils` classes. The `core` module works fine since it has no servlet dependency.

### Default Signature Algorithm is RSA-SHA1

`Saml2Settings` defaults `signatureAlgorithm` to `Constants.RSA_SHA1` and `digestAlgorithm` to `Constants.SHA1`. These are deprecated algorithms. Production deployments should set `onelogin.saml2.security.signature_algorithm` to RSA-SHA256 and `onelogin.saml2.security.digest_algorithm` to SHA256.

### Clock Drift Constant Typo

`Constants.ALOWED_CLOCK_DRIFT` is misspelled (should be `ALLOWED`). This is a public constant so fixing it would be a breaking change.

### Not Thread-Safe

`Auth` is stateful and not thread-safe. A new instance must be created for each HTTP request. Sharing an `Auth` instance across threads will cause data corruption.

### Java 8 Source Level

The project targets Java 8 (`source/target 1.8`), preventing use of newer Java features (records, sealed classes, pattern matching, etc.).

### StrSubstitutor Deprecated

`AuthnRequest` and other XML generators use `org.apache.commons.lang3.text.StrSubstitutor`, which is deprecated in favor of `org.apache.commons.text.StringSubstitutor` from the separate `commons-text` library.

### HttpRequest.unmodifiableCopyOf Missing Variable Declaration

In `HttpRequest.java`, the `unmodifiableCopyOf` method references a `copy` variable on line ~213 without a `Map<String, List<String>>` type declaration. This appears to be a compilation issue that may be masked by a different version of the file on disk.

### Module Name Typo

The samples module is named `java-saml-tookit-samples` and `java-saml-tookit-jspsample` (note: "tookit" instead of "toolkit").

---

## Migration Guidance

### Migrating to Jakarta Servlet

To use with Quarkus 3.x or Spring Boot 3.x:

1. **Core module works as-is** — it has no servlet dependency
2. **Toolkit module** requires `javax.servlet` → `jakarta.servlet` migration:
   - `Auth.java`: change `javax.servlet.http.*` → `jakarta.servlet.http.*`
   - `ServletUtils.java`: change `javax.servlet.http.*` → `jakarta.servlet.http.*`
3. **Alternative**: use the core module directly with a custom `HttpRequest` builder for your framework

### Upgrading Signature Algorithms

Replace deprecated SHA-1:

```properties
onelogin.saml2.security.signature_algorithm=http://www.w3.org/2001/04/xmldsig-more#rsa-sha256
onelogin.saml2.security.digest_algorithm=http://www.w3.org/2001/04/xmlenc#sha256
onelogin.saml2.security.reject_deprecated_alg=true
```

---

## Glossary

| Term | Definition |
|------|-----------|
| **SP** | Service Provider — the application that authenticates users via SAML (this library) |
| **IdP** | Identity Provider — the external service that authenticates users (e.g., Okta, Azure AD) |
| **SSO** | Single Sign-On — the login flow where the SP redirects to the IdP for authentication |
| **SLO** | Single Logout — the logout flow where the SP notifies the IdP to terminate the session |
| **AuthnRequest** | SAML authentication request XML sent from SP to IdP to initiate SSO |
| **SAMLResponse** | SAML response XML sent from IdP to SP containing the authentication assertion |
| **LogoutRequest** | SAML logout request XML sent to initiate SLO |
| **LogoutResponse** | SAML logout response XML confirming logout completion |
| **ACS** | Assertion Consumer Service — the SP endpoint that receives SAMLResponse (HTTP POST) |
| **SLS** | Single Logout Service — the SP endpoint that receives LogoutRequest/Response |
| **NameID** | The user identifier in the SAML assertion (e.g., email address, persistent ID) |
| **RelayState** | State parameter passed between SP and IdP to maintain context across the redirect |
| **Assertion** | The XML element within SAMLResponse containing user identity and attributes |
| **Metadata** | XML document describing the SP or IdP's SAML endpoints, certificates, and capabilities |
| **HSM** | Hardware Security Module — secure hardware for cryptographic key storage and operations |
| **Deflate** | Compression algorithm used for HTTP Redirect binding (SAML messages in URL parameters) |

---

## Framework Architecture Summary

`java-saml` is a SAML 2.0 Service Provider toolkit providing SP-initiated SSO and SLO. The `core` module implements protocol logic (AuthnRequest/SamlResponse/LogoutRequest/LogoutResponse generation and validation), settings management, XML signature verification, and encrypted assertion decryption with optional Azure Key Vault HSM support. The `toolkit` module adds servlet integration via the `Auth` class (the main entry point) and `ServletUtils` for `HttpServletRequest`/`HttpServletResponse` handling. Configuration is properties-based (`onelogin.saml.properties`), and SAML message creation is customizable via `SamlMessageFactory`. The library uses `javax.servlet` 4.0.1, not Jakarta.

## Key Classes

| # | Class | Module | Role |
|---|-------|--------|------|
| 1 | `Auth` | toolkit | Main entry point: orchestrates login/logout flows, holds session state |
| 2 | `SamlResponse` | core | Parses and validates SAML responses from IdP |
| 3 | `AuthnRequest` | core | Generates SAML authentication request XML |
| 4 | `LogoutRequest` | core | Generates/parses SAML logout request XML |
| 5 | `LogoutResponse` | core | Generates/parses SAML logout response XML |
| 6 | `Saml2Settings` | core | Holds all SP/IdP/security configuration |
| 7 | `SettingsBuilder` | core | Builds `Saml2Settings` from properties files or maps |
| 8 | `Metadata` | core | Generates SP metadata XML for federation setup |
| 9 | `IdPMetadataParser` | core | Parses IdP metadata XML into settings map |
| 10 | `HttpRequest` | core | Framework-agnostic HTTP request representation |
| 11 | `ServletUtils` | toolkit | Converts `HttpServletRequest` to `HttpRequest`, URL/redirect utilities |
| 12 | `SamlMessageFactory` | toolkit | Factory interface for customizing SAML message creation |
| 13 | `Constants` | core | SAML protocol URIs (namespaces, bindings, algorithms, status codes) |
| 14 | `Util` | core | XML parsing, signature generation/verification, encoding/decoding utilities |
| 15 | `HSM` | core | Abstract class for HSM-based cryptographic operations |
| 16 | `AzureKeyVault` | core | Azure Key Vault HSM implementation |
| 17 | `AuthnRequestParams` | core | Input parameters for AuthnRequest (ForceAuthn, IsPassive, etc.) |
| 18 | `LogoutRequestParams` | core | Input parameters for LogoutRequest (sessionIndex, NameID, etc.) |
| 19 | `SchemaFactory` | core | Creates XML Schema objects for SAML response validation |
| 20 | `SamlResponseStatus` | core | Holds SAML response status code and sub-status code |
