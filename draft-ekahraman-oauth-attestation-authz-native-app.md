---

title: "OAuth 2.0 Attestation Based Authorization for Native Applications"
abbrev: "Attested Authorization for Native Apps"
category: info

docname: draft-ekahraman-oauth-attestation-authz-native-app-latest
submissiontype: IETF
number:
date:
consensus: false
v: 3
area: SEC
workgroup: Individual Submission
keyword:
 - oauth
 - attestation
 - authorization
venue:
  mail: hello@mekarge.com
  github: mekarge/draft-ekahraman-oauth-attestation-authz-native-apps
  latest: https://mekarge.github.io/draft-ekahraman-oauth-attestation-authz-native-apps/

author:
 -
    fullname: Efe Kahraman
    organization: Mekarge
    email: efe@mekarge.com
    street: 190 Urla
    city: Izmir
    code: 35450
    country: TR

normative:
    RFC6749: RFC6749
    RFC9334: RFC9334
    RFC9126: RFC9126

informative:
    RFC7942: RFC7942
    RFC8252: RFC8252
    RFC9449: RFC9449
    RFC9421: RFC9421
    RFC8792: RFC8792
    RFC7638: RFC7638
    ZTA:
        author:
            org: "NIST"
        title: "Zero Trust Architecture"
        target: "https://doi.org/10.6028/NIST.SP.800-207"
    I-D.ietf-rats-ear: https://datatracker.ietf.org/doc/draft-ietf-rats-ear/
    I-D.ietf-oauth-browser-based-apps: https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps#name-backend-for-frontend-bff/
    I-D.ietf-oauth-attestation-based-client-auth: https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth/
    I-D.ietf-rats-ar4si: https://datatracker.ietf.org/doc/draft-ietf-rats-ar4si/
    IANA.OAuth.Parameters:
        target: https://www.iana.org/assignments/oauth-parameters
        title: OAuth Parameters
        author:
          - org: Internet Assigned Numbers Authority
    MekargeA3:
      title: "Mekarge A3"
      target: "https://a3.mekarge.com/site"
      author:
        -
          ins: Mekarge
      date: 2026
    MekargeA3.API:
      title: "Mekarge A3 Authorization API"
      target: "https://a3.mekarge.com/site/docs/api/authorization/"
      author:
        -
          ins: Mekarge
      date: 2026

--- abstract

This document defines an extension to OAuth 2.0 {{RFC6749}} that enables Authorization Servers to consider Attestation Results presented by Native Applications when issuing access grants. By incorporating information about the security characteristics of the application and its execution environment, this mechanism supports Authorization Policies that are tailored to the trustworthiness of the Native Application.

--- middle

# Introduction

This document defines an extension to OAuth 2.0 {{RFC6749}} that enables Authorization Servers to consider Attestation Results presented by Native Applications when issuing access grants. By incorporating information about the security characteristics of the application and its execution environment, this mechanism supports Authorization Policies that are tailored to the trustworthiness of the Native Application.

Consider a scenario where a Native Application is authorized to perform sensitive operations on behalf of a user, such as accessing financial information or initiating transactions. The application may execute on a device that has been modified or is running software capable of monitoring, intercepting, or influencing the application's execution environment. In such cases, information processed by the application or exchanged with remote services may be modified or misused by unauthorized parties. Consequently, authorization decisions based solely on the identity of the user may not accurately reflect the security posture of the requesting environment. 

The Zero Trust Architecture {{ZTA}} suggests that access to a protected resource should be granted by a policy which is evaluated on multiple attributes of the subject. In this regard, granting access to a resource may be determined by a set of attributes including device characteristics and software metadata. This leads to a need for a policy decision algorithm consuming vectors of attributes.

Thinking of Remote Attestation Procedures (RATS) Architecture {{RFC9334}} through the lens of Zero Trust Architecture brings a perspective to further break down the abstract concept of policy decision. It is then possible to conceptualize the subject as Attester which is the device being evaluated. And the Policy Decision Point can be mapped to the Relying Party. This mapping allows reuse of existing standardized roles and flows defined in RATS to design the policy decision functionality.

OAuth 2.0 {{RFC6749}} is an effective way to ground these abstract concepts into operational implementation. The combination of the Native Application, user's device, and relevant Attesting Environments may collectively act as the Attester. The Authorization Server on the other hand relates to the concept of Relying Party. 

This document defines a mechanism to pass the Attestation Result, signed by the Verifier, to the Authorization Server during token exchange for Native Application Clients. Access tokens are issued only with the scopes permitted by Authorization Decisions derived from the Attestation Result. The flow specified in this document relates to the "Passport Model" in RATS as the Attestation Result is transported to the Authorization Server without requiring direct communication between the Authorization Server and Verifier.

~~~ ascii-art
+---------------+               (A)   +---------------+
|               +-------------------->|               |
|    Client     |               (B)   |   Verifier    |
|               |<--------------------+               |
+-----------+---+                     +---------------+
    ^       |                                          
    |       |                                          
    |       |                         +---------------+
    |       |                   (C)   |               |
    |       +------------------------>| Authorization |
    |                           (D)   |     Server    |
    +---------------------------------+               |
                                      +---------------+
~~~

This flow includes the following steps:

(A) Client sends all collected Evidence to the Verifier. Client MAY sign the request with an asymmetric cryptographic key. If client signs the request, it's RECOMMENDED to send the public JWK in the request body. It's RECOMMENDED to use HTTP Message Signatures {{RFC9421}} for the signing implementation and HTTPS as the underlying protocol. The message structure is beyond the scope of this document.

(B) Verifier creates an Attestation Result based on the Evidence. Verifier MUST sign the Attestation Result with a cryptographic key. For asymmetric keys, Verifier MUST share the public key with Authorization Server. For both asymmetric and symmetric keys, key establishment protocol is beyond the scope of this document. Verifier MUST make the response uncacheable by adding a `Cache-Control` header set as `no-store`.

(C) Client sends the token request to Authorization Server with additional parameters including the Attestation Result. This document defines necessary extensions in [](#ext).

(D) The Authorization Server gathers Authorization Policies for each requested scope. For each Authorization Policy, Authorization Server obtains Trust Decisions by evaluating the corresponding Trust Assessment. Trust Assesments are predicates using the Attestation Result provided by the client. After each Authorization Policy is evaluated, Authorization Server determines scopes based on Authorization Decisions and issues the access token accordingly.

Client MAY cache the Attestation Result received from the Verifier for different token requests. In this case, client SHOULD cache the Attestation Result with a short expiry time. Authorization Server MAY reject the token request if the freshness of the Attestation Result doesn't meet system requirements.

This document does not standardize Evidence collection, Verifier interfaces, or Attestation Result formats. It defines only how an Authorization Server consumes Attestation Results during authorization.

## Related Works

A related approach is proposed in {{I-D.ietf-oauth-attestation-based-client-auth}}, which addresses a mechanism for proving client instance identity and handling authentication. This document addresses the distinct problem of authorization based on attestation.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology

The reader is assumed to be familiar with the vocabulary and concepts defined in OAuth 2.0.

The following term is imported from {{ZTA}}:

* Policy Decision Point

The following terms are imported from {{Section 4 of RFC9334}}:

* Attesting Environment
* Appraisal Policy for Attestation Result (APR)
* Attestation Result
* Attester
* Evidence
* Relying Party
* Verifier

The abbreviation APR is used throughout this document.

This document additionally defines the following terms:

Native Application (Native App)

>  An application that is installed by the user to their device. Different from the "native app" definition in {{Section 3 of RFC8252}}, Native Application is expected to provide device attributes and application metadata.

Native Application Client

> The OAuth 2.0 client requesting access token for the Native Application. Client can be a part of the Native Application acting as a Public client, or can be part of a backend service requesting access token on behalf of the Native Application as a part of the Backend-For-Frontend pattern. Throughout this document, Native Application Client will be referred to as "client".

Challenge

> A cryptographically random nonce which is generated and validated by the Verifier.

Appraisal Assertion

> Represents the outcome of evaluating a distinct aspect of the Attester. Each Appraisal Assertion MAY have its own status and associated claims. An Attestation Result MUST contain one or more Appraisal Assertions.

Trust Assessment

> Evaluation of an Attestation Result according to an APR. APR MAY have separate rule for each Appraisal Assertion forming the Attestation Result.

Trust Decision

> The outcome of a Trust Assessment. It is either "Allow" or "Deny".

Authorization Policy

> Specification of Trust Assessments required for a scope. Specification MAY require at least one of or all Trust Decisions to result in "Allow".

Authorization Decision

> Indicates if a particular scope is granted to a client after the evaluation of the associated Authorization Policy.

# Evidence Collection {#egen}

Native Applications collect Evidence for the Verifier. Verifier SHOULD provide a Challenge value to test the freshness of the Evidence. When provided, Native Application MUST use the Challenge value when collecting Evidence. Verifier SHOULD generate a Challenge value with sufficient entropy according to the system requirements. 

How Native Application fetches the Challenge is beyond the scope of this document. Verifier MAY offer an endpoint as shown below.

~~~ ascii-art
+---------------+               (A)   +---------------+
|               +-------------------->|               |
|  Native App   |               (B)   |   Verifier    |
|               |<--------------------+               |
+---------------+                     +---------------+
~~~

This flow includes the following steps:

(A) Native Application requests a Challenge value from Verifier. Native Application MAY sign the request by generating an asymmetric key pair in a secure enclave on the device or use a previously generated key pair. If request is signed, Native Application MUST send the public key JWK to the Verifier. It's RECOMMENDED to use HTTP Message Signatures {{RFC9421}} for the signing implementation and HTTPS as the underlying protocol.

(B) Verifier generates a fresh Challenge with sufficient entropy. If request is signed and Verifier receives public JWK from Native Application, it MUST bind the generated Challenge value to the public key JWK or to a stable identifier derived from that key, such as a JWK Thumbprint ({{RFC7638}}). Verifier MUST store the Challenge creation timestamp for the freshness check. Verifier MUST make the response uncacheable by adding a `Cache-Control` header set as `no-store`.

Native Application MAY contact with other parties when collecting the Evidence. In terms of RATS architecture, Evidence is created by Attesting Environments. The Attesting Environment can be a remote service provided by a vendor or platform. Message exchange is shown below.

~~~ ascii-art
+---------------+               (A)   +---------------+
|               +-------------------->|               |
|  Native App   |               (B)   |   Attesting   |
|               |<--------------------+  Environment  |
|               |                     |               |
+---------------+                     +---------------+
~~~

This flow includes the following steps:

(A) Native Application sends a request to Attesting Environment to get Evidence by providing necessary claims. Native Application SHOULD present the Challenge value supplied from Verifier in addition to the claims. The message structure is specific to the Attesting Environment and is beyond the scope of this document.

(B) Attesting Environment generates the Evidence. Attesting Environment MUST embed the Challenge value in Evidence when Challenge is present. Attesting Environment SHOULD sign the Evidence with a cryptographic key. 

# Evidence Verification {#ever}

Verifier creates the Attestation Result based on the Evidence sent by the client. Verifier MAY test the integrity and the authenticity of the Evidence using Attesting Environment as shown below.

~~~ ascii-art
+---------------+               (A)   +---------------+
|               +-------------------->|               |
|    Client     |               (D)   |    Verifier   |
|               |<--------------------+               |
+---------------+                     +-----------+---+
                                          ^       |
                                          |       |        
                                     (C)  |       | (B)
                                          |       v        
                                      +---+-----------+
                                      |               |
                                      |  Attesting    |
                                      |  Environment  |
                                      |               |
                                      +---------------+
~~~

This flow includes the following steps:

(A) Client sends Evidence to the Verifier. Evidence MUST include the Challenge value if Verifier has provided one during Evidence generation as described in [](#egen). When the Challenge value is used, client MUST send the public key JWK of the Native Application.

(B) If the Evidence is cryptographically signed, Verifier MUST validate the signature. The key establishment protocol for the cryptographic key between Verifier and Attesting Environment is beyond the scope of this document. Verifier calls the Attesting Environment for further checking the integrity and decoding the Evidence if necessary.

(C) Attesting Environment MAY run integrity checks on the Evidence and return the Evidence with claims useful for the Verifier. Attesting Environment MUST extract the Challenge from Evidence and return its value explicitly if it was supplied by the Native Application.

(D) Verifier processes the information returned from Attesting Environment. Verifier MUST test the Challenge value if it is returned from the Attesting Environment. Verifier MAY implement a lookup table to find the associated Challenge value via the public key JWK or to a stable identifier derived from that key, such as a JWK Thumbprint ([RFC7638]) which is received in the request (A). It's RECOMMENDED for Verifier to check the freshness of the Evidence when Challenge creation timestamp is known. Based on the validations, Verifier creates the Attestation Result.

# Attestation Result Requirements 

Attestation Result MUST be signed by the Verifier with a cryptographic key.

The structure of the Attestation Result is out of scope of this document. However, it is RECOMMENDED to include the elements defined by the {{I-D.ietf-rats-ar4si}}. Following information is REQUIRED for Attestation Result:

* One or more Appraisal Assertions where each Appraisal Assertion corresponding to a distinct aspect of the device or Native Application. Each Appraisal Assertion MAY have its own status and associated claims. Those claims can be implemented as the Trustworthiness Claims defined in {{I-D.ietf-rats-ar4si}}.
* Identity of the Verifier issuing the Attestation Result. This identity can be implemented as the Verifier ID defined in {{I-D.ietf-rats-ar4si}}.
* Intended Authorization Server. This value SHOULD be the issuer URI used by the Authorization Server.
* A timestamp value indicating when the Attestation Result is created.

The encoding of the Attestation Result is beyond the scope of this document. However implementer MAY choose EAR Tokens as defined in EAT Attestation Results {{I-D.ietf-rats-ear}}. Verifier MUST use the Attestation Result format and encoding supported by the Authorization Server.

# Authorization Server Processing

The Authorization Server MUST validate the Attestation Result's signature. Authorization Server MUST accept Attestation Result only from trusted Verifiers.

Only successfully validated Attestation Result is used when evaluating an Authorization Policy. Authorization Server, depending on the Authorization Decisions, MUST decide which scopes should be issued in access token.

Upon receiving the Attestation Result, Authorization Server MUST perform the following steps:

1. Checks if the cryptographic signature of the Attestation Result is valid. The key establishment protocol for the cryptographic key between Verifier and Authorization Server is beyond the scope of this document.
2. Checks the freshness of the Attestation Result using the creation timestamp. 
3. Checks if Verifier ID is present and is trusted by the system.
4. Checks if the intended Authorization Server points to the server itself.
5. Checks if Attestation Result contains all necessary Appraisal Assertions required by the Trust Assessments.
6. For each scope available to the client, Authorization Server evaluates the Authorization Policy, which is the specification of the Trust Assessments. How Authorization Policy is defined and associated to the scope is beyond the scope of this document.
7. Based on the Authorization Decision for each scope, Authorization Server adds or removes the particular scope from the access token. Authorization Server MAY issue a token containing a reduced set of scopes, or MAY reject the request entirely. If no scopes are allowed, Authorization Server SHOULD return the `invalid_scope` error as defined in {{Section 5.2 of RFC6749}}. If token is issued with a reduced set of scopes, the Authorization Server SHOULD return the `scope` parameter in the token response.

When access token issued successfully, Authorization Server MUST bind the Verifier ID to that access token and refresh token if requested by Client.

## Refresh Tokens

The Client MUST send an Attestation Result to the Authorization Server when using a refresh token grant according to the freshness window required by the Authorization Server. Because Attestation Results are snapshots of Native Application's runtime state at a single point in time, the Authorization Server MUST reject the request if Attestation Result is missing or stale.

# Protocol Extensions {#ext}

This specification adds two parameters to the Token Endpoint request for `authorization_code` and `refresh_token` grant types:

`attestation_result`

> REQUIRED. Contains the Attestation Result generated by the Verifier. The structure of the Attestation Result is beyond the scope of this document. However implementer MAY choose EAR Tokens as defined in EAT Attestation Results {{I-D.ietf-rats-ear}}.

`attestation_profile`

> OPTIONAL. References to the Authorization Policy which will evaluate the Attestation Result. Implementer may choose to mark this parameter as REQUIRED when Authorization Server supports definition of multiple Authorization Policies.

The following example uses "\\" line wrapping per {{RFC8792}} to show a token request. In this example Attestation Result is encoded in JWT as defined in {{I-D.ietf-rats-ear}}.

~~~
POST /token HTTP/1.1
     Host: server.example.com
     Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
     Content-Type: application/x-www-form-urlencoded
     grant_type=authorization_code\
     &code=SplxlOBeZQQYbYS6WxSbIA \
     &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb \
     &attestation_profile=android \
     &attestation_result=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlYXRfcHJvZmlsZSI6InRhZzpnaXRodWIuY29tLDIwMjM6dmVyYWlzb24vZWFyIiwiaWF0IjoxNjY2NTI5MzAwLCJlYXIudmVyaWZpZXItaWQiOnsiZGV2ZWxvcGVyIjoiaHR0cHM6Ly92ZXJpZmllci1zZXJ2aWNlIiwiYnVpbGQiOiIwLjAuMSJ9LCJzdWJtb2RzIjp7InBsYXktaW50ZWdyaXR5Ijp7ImVhci5zdGF0dXMiOiJhZmZpcm1pbmciLCJlYXIudHJ1c3R3b3J0aGluZXNzLXZlY3RvciI6eyJyZXF1ZXN0X3BhY2thZ2VfbmFtZSI6Mn19fX0.vSXHlSZokqGPxORoMrKY5Y-0Ge2kVdDKTnyugWD2lkk
~~~

These parameters are primarily intended for the Token Endpoint, unless the front-channel pre-flight optimizations described in [](#pub) are utilized.

# Public Client Considerations {#pub}

In this section client is a Public client and is part of the Native Application.

Authorization Server MAY precheck Attestation Result in early stages of the authorization flow in order to avoid unnecessary steps in case Attestation Result is invalid or doesn't meet requirements of target Trust Assesments. In this case, the parameters defined in [](#ext) can be used in the authorization request. However, the Attestation Result can contain some sensitive data that should not be transferred through the front-channel. The following approaches can be used:

* Verifier can cryptographically encrypt the Attestation Result. Contents of the Attestation Result will remain hidden all the way through the Authorization Server. But this option can lead to long URLs, which can be problematic due to size limitations that can be enforced from any intermediary hop.
* Client can use Pushed Authorization Requests {{RFC9126}} to send the Attestation Token through the back-channel.

When Attestation Result is received in authorization request, Authorization Server MUST perform the following steps:

1. Checks if the cryptographic signature of the Attestation Result is valid. The key establishment protocol for the cryptographic key between Verifier and Authorization Server is beyond the scope of this document.
2. Checks the freshness of the Attestation Result using the creation timestamp. 
3. Checks if Verifier ID is present and is trusted by the system.
4. Checks if the intended Authorization Server points to the server itself.
5. Checks if Attestation Result contains all necessary Appraisal Assertions required by the Trust Assessments.
6. Stores the Attestation Result for the Authorization Policy evaluation during the upcoming token request. Authorization Server SHOULD store the Attestation Result with an expiry time not longer than the combination of authorization code and request URI lifetimes.

If one of those steps fails, Authorization Server MUST respond with an error. The error SHOULD indicate `access_denied` as defined in {{Section 4.1.2.1 of RFC6749}}.

Authorization Server MUST respond with an error if it receives Attestation Result in both authorization request and token requests. The error SHOULD indicate `invalid_request` as defined in {{Section 5.2 of RFC6749}}

## Relationship with Proof of Possession

Native Application Client can use Proof of Possession method for the token request in the subsequent calls. OAuth 2.0 Demonstrating Proof of Possession (DPoP) {{RFC9449}} is RECOMMENDED for sender-constraining access tokens. Proof of Possession is not a dependency but can be used as complimentary to the mechanism defined in this document. 

# Backend-For-Frontend Pattern {#bff}

The proposed mechanism in this document can also be used for Native Applications connecting to a proxy backend acting as a confidential client. The Backend-For-Frontend pattern for OAuth 2.0 was introduced in OAuth 2.0 for Browser-Based Applications {{I-D.ietf-oauth-browser-based-apps}}. While the Backend-for-Frontend (BFF) pattern was originally designed to solve browser-based security vulnerabilities, it can be adapted to Native Applications. 

By placing a backend layer between Native Application and Authorization Server, responsibility of sending Attestation Result will be shifted to the new layer. From this point onwards this backend layer will be called the Attestation Server.

Attestation Server MAY embed the Verifier functionality, or use a remote Verifier for receiving the Attestation Result. 

## Communication Security

This document does not enforce any particular protocol for the messaging between Attestation Server and a remote Verifier. However the implementer MUST use TLS for securing the underlying protocol.

# Implementation Status

This section records the status of known implementations of the protocol defined by this specification at the time of posting of this Internet-Draft, and is based on a proposal described in {{RFC7942}}.  The description of implementations in this section is intended to assist the IETF in its decision processes in progressing drafts to RFCs.  Please note that the listing of any individual implementation here does not imply endorsement by the IETF.  Furthermore, no effort has been spent to verify the information presented here that was supplied by IETF contributors. This is not intended as, and must not be construed to be, a catalog of available implementations or their features.  Readers are advised to note that other implementations may exist.

According to {{RFC7942}}, "this will allow reviewers and working groups to assign due consideration to documents that have the benefit of running code, which may serve as evidence of valuable experimentation and feedback that have made the implemented protocols more mature.  It is up to the individual working groups to use this information as they see fit".

## Mekarge A3

The organization responsible for this implementation is Mekarge. {{MekargeA3}} is an Authorization Server designed to manage authentication, authorization, and access control for applications and services. By the time of writing this document, Mekarge A3 is available for beta access.

Mekarge A3 implements the mechanism offered in this document with incorporating Backend-For-Frontend pattern described in [](#bff). Mekarge A3 introduces the following concepts: 

* Permission as a unique combination of a Resource and one of its scopes. Each permission defines the specific actions and data access rights that can be granted to a client. 
* An Attestation Profile defines a set of Appraisal criteria for evaluating device and Native Application trustworthiness. Attestation Profiles are associated with the permissions. During authorization, Mekarge A3 Authorization Server evaluates all the Appraisals defined for the Attestation Profile and filters the client's granted permissions that are associated with the particular Attestation Profile.

Latest API documentation can be accessed via {{MekargeA3.API}}. The developers can be contacted through [hello@mekarge.com](hello@mekarge.com).

# Security Considerations

## Replay Attacks

Authorization Server MUST implement measures to detect replay attacks. Authorization Server MUST check the freshness of the Attestation Result using its creation timestamp. Authorization Server MAY implement additional measures to minimize the clock skew between the source (Verifier) and itself.

When receiving Attestation Result from public clients on token request, Authorization Server SHOULD use a short time window for checking freshness of the Attestation Result.

## Challenge Freshness

Verifier SHOULD offer a mechanism to provide a fresh Challenge value with sufficient entropy to Native Application. Verifier MUST check both freshness and the value of the Challenge in the Evidence before generating the Attestation Result if a Challenge was provided. In order to check the freshness, Verifier can store the timestamp when Challenge value is issued. Verifier SHOULD discard the Challenge value on its first occurrence in an Evidence.

## Downgrade Attacks

Authorization Server MUST reject the token request if `attestation_result` parameter is not provided and system has an Authorization Policy defined for at least one scope requested by client.

Authorization Server MUST reject the token request if there are multiple Authorization Policy defined for at least one scope requested by client and `attestation_profile` parameter is not provided.

## Verifier Compromise

As aforementioned, Authorization Server MUST check if Verifier ID is present and is trusted by the system when processing the Attestation Result. In case Verifier is known to be compromised, Authorization Server MUST reject all requests with Attestation Token that are created by the compromised Verifier. Authorization Server also MUST reject any token request using refresh token grant if the original token request issuing the refresh token has used the Attestation Result created by the compromised Verifier during Authorization Policy evaluation.

## Device Key Extraction

When client signs the Challenge request using an asymmetric key pair as described in [](#egen), the security of this model relies on the device's ability to prevent key extraction. It is therefore Verifier's responsibility to assess the risk accordingly if the device executing Native Application does not support a secure enclave or a similar hardware-based storage.

# Privacy Considerations

## Evidence Exposure

Depending on the Attesting Environment implementation, Evidence might contain sensitive data without cryptographic encryption. In some cases, such Evidence might include Personally Identifying Information (PII) as well. Clients are therefore responsible for securely sending Evidence to the Verifier. Client MUST use TLS based protocol and ensure Verifier server certificate is valid and trustable.

Native Application MUST avoid transmitting unnecessary device metadata to Attesting Environment. Similarly, Attesting Environment MUST only include claims required for the Verifier.

## Attestation Result Exposure

Attestation Result might contain information about the internal state of the device, Native Application software, and the Verifier software. Depending on the nature of the Native Application the Attestation Result can include Personally Identifying Information (PII) as well. Clients are therefore responsible for taking necessary measures for ensuring Attestation Result is safely stored if caching is implemented and it is sent to Authorization Server through back-channel. For public clients, if Native Application is intended to send Attestation Result before access token request as described in [](#pub), it is RECOMMENDED to use Pushed Authorization Requests {{RFC9126}}.

Authorization Server is responsible for how Attestation Result and associated decisions are logged for security and privacy audits. Authorization Server SHOULD NOT log Attestation Result if there is a risk of exposure of log data to unintended parties.

Attestation Results can contain stable identifiers. By accessing logs including Attestation Results, users or devices can be correlated using those stable identifiers. It is therefore Authorization Server's responsibility to handle such identifiers, such as hashing, anonymizing, or omitting during logging.

## Secondary Use

Attestation Results collected for authorization SHOULD not automatically be reused for analytics, profiling, or marketing by the Authorization Server.

# IANA Considerations

## OAuth Parameters Registration

This specification requests registration of the following values in the IANA "OAuth Parameters" registry of {{IANA.OAuth.Parameters}} established by {{RFC6749}}.

* Name: `attestation_result`

* Parameter Usage Location: authorization request, token request

* Reference: Section 8 of this document

> 

* Name: `attestation_profile`

* Parameter Usage Location: authorization request, token request

* Reference: Section 8 of this document

--- back


