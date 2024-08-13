# Known Customer Credential Issuance <!-- omit in toc -->

<!-- TOC -->
- [Introduction](#introduction)
  - [Known Customer Credential](#known-customer-credential)
  - [Known Customer Credential Schema](#known-customer-credential-schema)
  - [Know Your Customer Background](#know-your-customer-background)
    - [Identity Verification](#identity-verification)
    - [IDV Vendor Integrations](#idv-vendor-integrations)
      - [PII Collected by IDV Vendor](#pii-collected-by-idv-vendor)
      - [PII Collected by PFI](#pii-collected-by-pfi)
- [Requirements](#requirements)
- [Implementation Details](#implementation-details)
  - [Context](#context)
  - [Assumptions](#assumptions)
  - [Participants](#participants)
    - [PFI](#pfi)
    - [Mobile App](#mobile-app)
    - [Web View](#web-view)
  - [Multiple Credential Types](#multiple-credential-types)
- [Credential Application Flow](#credential-application-flow)
  - [Discover Initiation Endpoint](#discover-initiation-endpoint)
  - [Initiate Application](#initiate-application)
    - [Request](#request)
    - [Response](#response)
      - [Client Metadata](#client-metadata)
      - [URI Encoding](#uri-encoding)
  - [Authenticate Applicant DID](#authenticate-applicant-did)
    - [Request](#request-1)
      - [ID Token](#id-token)
    - [Response](#response-1)
      - [Credential Offer](#credential-offer)
      - [Grants](#grants)
      - [Grant Type: urn:ietf:params:oauth:grant-type:pre-authorized\_code](#grant-type-urnietfparamsoauthgrant-typepre-authorized_code)
  - [IDV Collection](#idv-collection)
    - [IDV Vendor Collects PII](#idv-vendor-collects-pii)
    - [PFI collects PII](#pfi-collects-pii)
  - [Credential Issuance](#credential-issuance)
    - [Metadata Endpoints](#metadata-endpoints)
      - [Well Known Endpoints](#well-known-endpoints)
      - [Credential Issuer Metadata](#credential-issuer-metadata)
        - [`credential_configurations_supported`](#credential_configurations_supported)
      - [Authorization Server Metadata](#authorization-server-metadata)
    - [Token Endpoint](#token-endpoint)
      - [Token Request](#token-request)
      - [Successful Token Response](#successful-token-response)
        - [`access_token` JOSE Header](#access_token-jose-header)
        - [`access_token` Claims](#access_token-claims)
      - [Token Error Response](#token-error-response)
        - [Error Codes](#error-codes)
    - [Credential Endpoint](#credential-endpoint)
      - [Credential Request](#credential-request)
        - [`proof`](#proof)
          - [`proof.jwt` JOSE Headers](#proofjwt-jose-headers)
          - [`proof.jwt` Claims](#proofjwt-claims)
      - [Credential Response](#credential-response)
      - [Credential Error Response](#credential-error-response)
        - [Authorization Errors](#authorization-errors)
      - [Credential Request Errors](#credential-request-errors)
    - [Deferred Credential Endpoint](#deferred-credential-endpoint)
      - [Deferred Credential Request](#deferred-credential-request)
      - [Successful Deferred Credential Response](#successful-deferred-credential-response)
      - [Deferred Credential Error Response](#deferred-credential-error-response)
- [Other Considerations](#other-considerations)
<!-- TOC -->


# Introduction
This document proposes a standardized means for PFIs (Participating Financial Institution) to perform KYC (Know Your Customer) and issue a subsequent KCC (Known Customer Credential) on a DID (Decentralized Identifer) controlled by a retail customer for the purpose of providing financial services to that DID in accordance to regulatory requirements.

## Known Customer Credential

KCC (Known Customer Credential) is a [VC (Verifiable Credential)](https://www.w3.org/TR/vc-data-model-2.0/) which is intended to be utilized by a retail customer as proof of an actively compliant KYC verification.

## Known Customer Credential Schema
The json schema for the KCC can be found here: [KCC Schema](kcc-schema.json). This schema defines the structure and requirements for a Known Customer Credential, ensuring all necessary information is included and correctly formatted.

## Know Your Customer Background
In the financial industry, KYC (Know Your Customer) is term used to describe a set of policies, procedures, and processes that financial institutions use to determine the true identity of a customer, and assess the on-going risk that a customer poses to an organization during the life-time of a customer relationship. KYC typically encompasses:
* Customer identification and verification (IDV);
* Understanding the nature and purpose of customer relationships to develop a customer risk profile; and
* Ongoing monitoring for reporting suspicious transactions and, using a risk-based approach, maintaining and updating customer information. 

Regulatory KYC requirements can vary by region with respect to the information that needs to be collected, the information that needs to be validated and verified, how long that information needs to be retained, and how often various aspects of KYC need to be repeated (e.g. sanctions screening, refreshing IDV, re-collecting source of funds information). 

### Identity Verification
IDV (Identity Verification) is a critical component of KYC wherein the PII (Personally Identifying Information) collected from an individual is verified using third party resources. IDV often includes steps such as verification of a valid government-issued photo ID, liveness checks, and verification of user-submitted PII against authoritative databases.

Financial institutions often leverage IDV Vendors to streamline the IDV process.

### IDV Vendor Integrations

Integration with IDV vendors happens in 1 of 2 ways:

#### PII Collected by IDV Vendor
1. IDV Vendor provides an SDK that takes control of the user interface for PII collection 
2. PII is submitted directly to the vendor's backend system 
3. IDV Vendor notifies the PFI via webhook request when IDV is complete
4. PFI requests IDV result and PII from IDV Vendor 

#### PII Collected by PFI
1. PII is submitted directly to the PFI's backend system
2. PII is subsequently sent to the IDV vendor via the backend system for verification

> [!IMPORTANT]
> The 2 styles of integration provide flexibility for PFI identity verification systems, but they don't impact Mobile Apps at all, as
> they have the same external behaviour. 

# Requirements
This body of work is an extension of the work being done for tbDEX. In effect, this proposal considers the following as requirements:


1. **Must support all IDV flows described in the [IDV Vendor Integrations](#idv-vendor-integrations) section of this document**

Ensuring that this is possible is essential to reduce friction or pain points for financial institutions interested in providing liquidity on tbDEX.

---

2. **Must support the ability to provide other Verifiable Credentials or Verifiable Claims as input for IDV (e.g. mDL, eIDAS, VCs from other issuers)**

This requirement is critical to the value proposition of using Verifiable Credentials within the context of KYC. Performing KYC has a non-negligible cost for financial institutions which can be drastically reduced by receiving the necessary PII in a format that has been provably verified by a trusted third party.

> [!IMPORTANT]
> We will need to consider scenarios wherein an individual possesses one or more "Identity Wallet" mobile applications that store credentials 

> [!IMPORTANT]
> We will need to consider issuance _to_ and presentation _from_ identity wallets that did not initiate the flow


---

3. **Must ensure that applications initiating KCC issuance / IDV flows for PFIs **do not** have to store, handle, or relay PII through the initating application's backend systems**

Necessitating that PII come in contact with an application's backend systems introduces undesired complexity for many use-cases (e.g. self-custodial wallets that wish to provide on/off-ramps to their users without owning customer relationships)

---

4. **Must prevent individuals who already have a pre-existing account with a PFI from having to go through IDV again**

A PFI could be a bank that an individual already has an account with. In this scenario, The individual should be able to _log in_ to their pre-existing account via webview vs. having to go through IDV again. 

---

5. **Must make use of pre-existing standards wherever possible.**

Interoperability is critical for the ecosystem as a whole. This is also an implied requirement  in order to leverage credentials that are already being issued/presented using pre-existing standards 

---

# Implementation Details

Concretely, the objective is to implement a solution that allows a mobile application to initiate an IDV flow with a PFI used to perform KYC that results in a Known Customer Credential issued by the PFI. This KCC can be presented by the holder to the PFI to utilize financial services (e.g. tbDEX value exchange)

## Context
As a means to provide clarity, many of the examples in this section will refer to an imaginary mobile application named Mobile App.

Mobile App is a mobile application that can be used by individuals to:
* Purchase Stablecoin using fiat currency from a PFI via tbDEX. The PFI acts as the custodian of the purchased Stablecoin
* Send custodied Stablecoin to anyone via tbDEX
* Sell Stablecoin for fiat currency through a PFI via tbDEX.

Mobile App acts as a **self-custodial** Identity Wallet that:
* creates a DID for each individual and securely stores private keys directly on device. 
* stores Verifiable Credentials issued to the user directly on device

## Assumptions
* The PFI controls a DID whose method supports `service` endpoints
* The Holder controls their own DID
* The Holder has already discovered the PFI's DID


## Participants

This implementation involves 3 distinct participants that have different responsibilities:

### PFI

The PFI operates an Issuer, implementing this protocol. 
  
### Mobile App

The Mobile app is responsible for:
* Initiating the IDV flow of a PFI
* Acquiring Credential offered by PFI


### Web View

> [!NOTE]
> Triggered by the mobile app as a byproduct of initiating the IDV flow 

The Web view is utilized to:
* Render IDV flow of a PFI
* Return an [OID4VCI Credential Offer](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#name-credential-offer) back to Mobile App which is then used to acquire a credential


> [!IMPORTANT]
> This implementation has chosen to use Web Views to render IDV flows for the following reasons:
> * provides PFI with maximal flexibility as to _how_ they collect Personal Information from an individual
> * prevents Wallets / Mobile Apps from having to update source code in order to integrate with different PFIs
> * establish a clear distinction between the application initiating the flow and a PFI

## Multiple Credential Types

Financial service providers often perform KYC incrementally in response to customer's expressed intent to access services. For example, minimal upfront diligence on newly onboarded customers, followed by additional diligence when the customer's activity passes a threshold.

A KCC Issuer may define multiple KCC types to represent different levels of KYC diligence, and describe each with a Presentation Definition. Mobile Apps which hold a previously issued KCC may apply for new credentials to acquire a higher level KCC (see [Initiate Application](#request)) 

# Credential Application Flow

```mermaid
sequenceDiagram

actor A as Applicant
participant W as Webview
participant D as Mobile App
participant P as PFI
participant I as IDV

rect rgba(0, 0, 0, 0.1)
    D->>+P: Initiate KCC application
    Note right of I: Initiate Application
    P-->>-D: IDV Request
end
rect rgba(0, 0, 0, 0.1)
    D->>+W: Load URL
    Note right of I: Collect IDV
    A->>W: Provide identity data
    W->>+I: Applicant identity data
    deactivate W
end
rect rgba(0, 0, 0, 0.1)
    D->>+P: Request KCC
    Note right of I: Credential Issuance
    P-->>-D: Issue KCC
end
```

## Discover Initiation Endpoint

PFI's can become publicly discoverable by advertising their IDV endpoint as a [Service](https://www.w3.org/TR/did-core/#services) within their DID Document. In order to increase the likelihood of being discovered, the `service` entry **SHOULD** include the following properties:

| Property          | Value                                                                                                                                                                                            |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`              | MUST be equal to the DID URI with an appended `idv` fragment, example `did:example:123#idv`, therefore the service MAY be [dereferenced](https://www.w3.org/TR/did-core/#did-url-dereferencing). |
| `type`            | `IDV`                                                                                                                                                                                            |
| `serviceEndpoint` | PFI's publicly addressable IDV endpoint for usage in [Initiate Application](#initiate-application).                                                                                              |

> [!NOTE]
>
> _Decentralized_ discoverability is dependent upon whether the underlying [verifiable registry](https://www.w3.org/TR/did-core/#dfn-verifiable-data-registry) of the selected [DID Method](https://www.w3.org/TR/did-core/#methods) is crawlable

## Initiate Application

The application flow is initiated with a [SIOPv2](https://openid.github.io/SIOPv2/openid-connect-self-issued-v2-wg-draft.html) interaction that authenticates the applicant's DID.

```mermaid
sequenceDiagram
autonumber

participant W as Webview
participant D as Mobile App
participant P as PFI

D->>+P: GET
P->>P: Construct SIOPv2 Authorization Request
P-->>-D: SIOPv2 Authorization Request
D->>D: Construct SIOPv2 Authorization Response
D->>+P: SIOPv2 Authorization Response
P->>P: Construct IDV Request
P-->>-D: IDV Request
D->>D: Verify IDV Request
D->>W: Load URL in IDV Request
```

1. Mobile App sends an HTTP GET request to the `serviceEndpoint` specified in [Discover Initiation Endpoint](#discover-initiation-endpoint)
2. PFI constructs a [SIOPv2 Authorization Request](#response)
3. PFI URI-encodes SIOPv2 Authorization Request and returns in HTTP response
4. Mobile App verifies integrity of SIOPv2 Authorization Request and constructs a [SIOPv2 Authorization Response](#request-1)
5. Mobile App POSTs SIOPv2 Authorization Response to the `response_uri` from the SIOPv2 Authorization Request
6. PFI verifies integrity of SIOPv2 Authorization Response and constructs [IDV Request](#response-1)
7. PFI returns IDV Request in HTTP response
8. Mobile App verifies integrity of IDV Request
9. Mobile App loads URL provided in IDV Request in Webview

### Request

An HTTP GET request begins the IDV and KCC issuance flow.

| Query Parameter              | Description                                                          | Required | References                                                                                                                                                                                                                    | Comments                                            |
| :--------------------------- | :------------------------------------------------------------------- | :------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------- |
| `presentation_definition_id` | The ID of a presentation definition describing the KCC to be issued. | n        | [Presentation&nbsp;Exchange&nbsp;2.0.0](https://identity.foundation/presentation-exchange/spec/v2.0.0/#presentation-definition) [tbDEX&nbsp;Offering](https://github.com/TBD54566975/tbdex/tree/main/specs/protocol#offering) | If not provided, the PFI chooses which KCC to issue |

### Response

The response is a [SIOPv2 Authorization Request](https://openid.github.io/SIOPv2/openid-connect-self-issued-v2-wg-draft.html#name-self-issued-openid-provider-a).

| Field                     | Description                                                                                  | Required | References                                                                                                                                                                                   | Comments                                                  |
| :------------------------ | :------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------- |
| `client_id`               | The DID of the Relying Party (the PFI)                                                       | y        |                                                                                                                                                                                              |                                                           |
| `scope`                   | What's being requested. 'openid' indicates ID Token is being requested                       | y        | [OIDC](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest)                                                                                                                    |                                                           |
| `response_type`           | What sort of response the RP is expecting. MUST include `id_token`. MAY include `vp_token`   | y        | [OIDC](https://openid.net/specs/openid-connect-core-1_0.html#Authentication)                                                                                                                 |                                                           |
| `response_uri`            | The URI to which the SIOPv2 Authorization Response will be sent                              | y        | [OID4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.2-7.2)                                                                                                |                                                           |
| `response_mode`           | The mode in which the SIOPv2 Authorization Response will be sent. MUST be `direct_post`      | y        | [OID4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.2-1)                                                                                                  |                                                           |
| `presentation_definition` | Used by PFI to request VCs as input to IDV process                                           | n        | [OID4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#name-presentation_definition-par)                                                                               | If present, Response Type `vp_token` MUST also be present |
| `nonce`                   | A nonce which MUST be included in the ID Token provided in the SIOPv2 Authorization Response | y        |                                                                                                                                                                                              |                                                           |
| `client_metadata`         | A JSON object containing the Verifier metadata values                                        | y        | [OIDC](https://openid.net/specs/openid-connect-registration-1_0.html) [SIOPv2](https://openid.github.io/SIOPv2/openid-connect-self-issued-v2-wg-draft.html#name-relying-party-client-metada) |                                                           |

#### Client Metadata
| Field                            | Description                                                                                                 | Required | References                                                                                              | Comments                         |
| :------------------------------- | :---------------------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------ | :------------------------------- |
| `subject_syntax_types_supported` | Array of strings, each a DID method supported for the subject of ID Token                                   | y        | [SIOPv2](https://openid.github.io/SIOPv2/openid-connect-self-issued-v2-wg-draft.html#section-7.5-2.1.1) | Example `["did:dht", "did:jwk"]` |
| `client_name`                    | Human-readable string name of the client to be presented to the end-user during authorization               | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                                  |
| `client_uri`                     | URI of a web page providing information about the client                                                    | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                                  |
| `logo_uri`                       | URI of an image logo for the client                                                                         | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                                  |
| `contacts`                       | Array of strings representing ways to contact people responsible for this client, typically email addresses | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                                  |
| `tos_uri`                        | URI that points to a terms of service document for the client                                               | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                                  |
| `policy_uri`                     | URI that points to a privacy policy document                                                                | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                                  |

> [!IMPORTANT]
> Include `vp_formats` in Client Metadata?  https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-9.1-2.2

> [!IMPORTANT]
> the inclusion of `presentation_definition` as per [OID4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#name-presentation_definition-par) allows for other verifiable credentials to be provided as input for IDV.

#### URI Encoding

The SIOPv2 Authorization Request is encoded as a URI before being returned to Mobile App, as per [SIOPv2](https://openid.github.io/SIOPv2/openid-connect-self-issued-v2-wg-draft.html#section-5). No `authorization_endpoint` is used in the URI, so it is the query parameter portion of the URI only.

## Authenticate Applicant DID

The Mobile App responds with a [Cross-Device SIOPv2 Authorization Response](https://openid.github.io/SIOPv2/openid-connect-self-issued-v2-wg-draft.html#name-cross-device-self-issued-open). 

### Request 
| Field                     | Description                                                                                                        | Required | References                                                                                                                                                     | Comments |
| :------------------------ | :----------------------------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------- |
| `id_token`                | A self issued, signed JWT which responds to the SIOPv2 Authorization Request                                       | y        | [JWT](https://www.rfc-editor.org/info/rfc7519) [SIOPv2](https://openid.github.io/SIOPv2/openid-connect-self-issued-v2-wg-draft.html#name-self-issued-id-token) |          |
| `vp_token`                | A Verifiable Presentation or an array of VPs in response to `presentation_definition`                              | n        | [OIDV4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.1-2.2)                                                                 |          |
| `presentation_submission` | A Presentation Submission that contains mappings between the requested VC and where to find them within `vp_token` | n        | [OIDV4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.1-2.4)                                                                 |          |

#### ID Token
| Field   | Description                                                                                    | Required | References | Comments |
| :------ | :--------------------------------------------------------------------------------------------- | :------- | :--------- | :------- |
| `iss`   | Issuer MUST match the value of `sub` (Applicant's DID)                                         | y        |            |          |
| `sub`   | Subject. The DID of the customer applying for KCC                                              | y        |            |          |
| `aud`   | Audience MUST match the value of `client_id` from the SIOPv2 Authorization Request (PFI's DID) | y        |            |          |
| `nonce` | Nonce MUST match the value of `nonce` from the SIOPv2 Authorization Request                    | y        |            |          |
| `exp`   | Expiry time                                                                                    | y        |            |          |
| `iat`   | Issued at time                                                                                 | y        |            |          |

### Response

The response is an IDV Request.

| Field              | Description                     | Required | References                                                                                                                            | Comments                                                                                       |
| :----------------- | :------------------------------ | :------- | :------------------------------------------------------------------------------------------------------------------------------------ | :--------------------------------------------------------------------------------------------- |
| `url`              | URL of form used to collect PII | y        |                                                                                                                                       | Required for now until we figure out how to support exclusively providing credentials as input |
| `credential_offer` |                                 | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#name-credential-offer-parameters) |                                                                                                |

#### Credential Offer
| Field                          | Description                                                                                                                                                         | Required | References                                                                                                             | Comments |
| :----------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :------- | :--------------------------------------------------------------------------------------------------------------------- | :------- |
| `credential_issuer`            | The URL of the Credential Issuer that the Mobile App will interact with in subsequent steps                                                                         | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-2.1) |          |
| `credential_configuration_ids` | Array of unique strings that each identify a credential being offered. Mobile App can use these to request metadata                                                 | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-2.2) |          |
| `grants`                       | Object containing Grant Types that the Credential Issuer will accept for this credential offer. MUST contain `urn:ietf:params:oauth:grant-type:pre-authorized_code` | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-2.3) |          |

#### Grants 
| Field                                                  | Description                                                                                          | Required | References                                                                                                               | Comments |
| :----------------------------------------------------- | :--------------------------------------------------------------------------------------------------- | :------- | :----------------------------------------------------------------------------------------------------------------------- | :------- |
| `urn:ietf:params:oauth:grant-type:pre-authorized_code` | Grant Type that allows the Mobile App to follow a Pre-Authorized Code Flow to collect the credential | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-4.2.1) |          |


#### Grant Type: urn:ietf:params:oauth:grant-type:pre-authorized_code
| Field                 | Description                                                                                           | Required | References                                                                                                                 | Comments |
| :-------------------- | :---------------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------------------------- | :------- |
| `pre-authorized_code` | The code representing the Credential Issuer's authorization for the Mobile App to obtain a credential | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-4.2.2.1) |          |

> [!WARNING] 
> TODO: explain rationale behind providing `credential_offer` at this stage

## IDV Collection

The IDV Request sent to the Mobile App contains a `url` field. The Mobile App MUST load this URL in an embedded 
webview. The Applicant is guided through whatever steps are necessary to collect identity data and submit to the IDV 
server. The webview MUST close itself when the application steps are complete.

> [!IMPORTANT]
> Whether the PFI is utilizing an IDV vendor is entirely opaque from the originating mobile app's perspective.

### IDV Vendor Collects PII
```mermaid
sequenceDiagram
autonumber

actor A as Applicant 
participant W as Webview
participant D as Mobile App
participant P as PFI
participant V as IDV Vendor


D->>W: Load URL
A->>W: Provide PII, Submit
W->>V: PII
V->>V: Process
V->>W: Callback URI or 200
W->>W: Close
```

### PFI collects PII
```mermaid
sequenceDiagram
autonumber

actor A as Applicant 
participant W as Webview
participant D as Mobile App
participant P as PFI


D->>W: Load URL
loop until 200 response
    A->>W: Provide PII, Submit
    W->>P: PII
    P->>P: Process
    P->>W: 200 or 400 with errors
end
W->>W: Close
```

## Credential Issuance

Credential Issuance is an [OID4VCI Pre-Authorized Code flow](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#name-pre-authorized-code-flow):

```mermaid
sequenceDiagram
autonumber

participant D as Mobile App
participant P as PFI

D->>+P: GET Authorization metadata
P-->>-D: Authorization Server metadata
D->>+P: Token Endpoint
P-->>-D: Access Token
D->>+P: GET Credential metadata
P-->>-D: Credential Issuer metadata
D->>+P: Issue credential
P-->>-D: Credential
```

1. GET well-known oauth-authorization-server endpoint
2. PFI returns [Authorization Server Metadata](#authorization-server-metadata)
3. Token Request containing `pre-authorized_code` 
4. Token Response containing an Access Token
5. GET well-known openid-credential-issuer endpoint
6. PFI returns [Credential Issuer Metadata](#credential-issuer-metadata)
7. Request credentials
8. PFI issues credential 

### Metadata Endpoints

Metadata resources are hosted by the Credential Issuer as a means of informing clients of endpoint locations, technical capabilities, feature support and (internationalized) display information.

#### Well Known Endpoints

Credential Issuers are required to set up a number of well known endpoints to facilitate authorization and credential issuance as follows. 

URLs to retrieve both [Credential Issuer Metadata](#credential-issuer-metadata) and [Authorization Server Metadata](#authorization-server-metadata) are dynamically constructed by the client [using `.well-known` URI's](https://www.rfc-editor.org/rfc/rfc5785).

- [Credential Issuer Metadata](#credential-issuer-metadata) URL: `credential_issuer` + `/.well-known/openid-credential-issuer`
- [Authorization Server Metadata](#authorization-server-metadata) URL: `credential_issuer` + `/.well-known/oauth-authorization-server`

Where `credential_issuer` originates from within the [Credential Offer](#credential-offer) from within the [IDV Request](#idv-request).

#### Credential Issuer Metadata

The Credential Issuer Metadata informs clients of endpoint locations, technical capabilities, supported Credentials, and (internationalized) display information.

| Field                                 | Description                                                               | Required | References                                                                                              | Comments                                                                               |
| :------------------------------------ |:--------------------------------------------------------------------------| :------- | :------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------- |
| `credential_issuer`                   | URL of the Credential Issuer                                              | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2.3-2.1) | Same value as the `credential_issuer` within the [Credential Offer](#credential-offer) |
| `credential_endpoint`                 | URL for the [Credential Endpoint](#credential-endpoint)                   | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2.3-2.3) |                                                                                        |
| `deferred_credential_endpoint`        | URL for the [Deferred Credential Endpoint](#deferred-credential-endpoint) | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2.3-2.5) |                                                                                        |
| `credential_configurations_supported` | Object which defines the specifics of the credentials being issued        | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2.3-2.3) |                                                                                        |

##### `credential_configurations_supported`

The `credential_configurations_supported` is an object which defines the specifics of the credentials being issued. The `credential_configurations_supported` is a key/value object wherein each key corresponds to a value within the `credential_configuration_ids` from the [Credential Offer](#credential-offer) and the value is defined with the following fields.

| Field                                     | Description                                        | Required | References                                                                                                     | Comments                                                                       |
| :---------------------------------------- | :------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------- |
| `format`                                  | Format for the given credential                    | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2.3-2.11.2.1)   | MUST be `jwt_vc_json`                                                          |
| `cryptographic_binding_methods_supported` | List of supported DID Methods                      | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2.3-2.11.2.3)   | MUST be `["did:web", "did:jwk", "did:dht"]`                                    |
| `credential_signing_alg_values_supported` | List of supported cryprographic signing algorithms | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2.3-2.11.2.4)   | MUST be `EdDSA` or `ES256K`                                                    |
| `proof_types_supported`                   | Object that describes the supported key proof      | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2.3-2.11.2.5.1) | MUST be `{"jwt": {"proof_signing_alg_values_supported": ["EdDSA", "ES256K"]}}` |

#### Authorization Server Metadata

The Credential Issuer's Authorization Server Metadata informs clients of its endpoint locations and authorization server capabilities.

| Field            | Description                                 | Required | References                                                         | Comments                          |
| :--------------- | :------------------------------------------ | :------- | :----------------------------------------------------------------- | :-------------------------------- |
| `issuer`         | URL of then Credential Issuer               | y        | [RFC8414](https://datatracker.ietf.org/doc/html/rfc8414#section-2) | Same value as `credential_issuer` |
| `token_endpoint` | URL for the [Token Request](#token-request) | y        | [RFC8414](https://datatracker.ietf.org/doc/html/rfc8414#section-2) |                                   |

[Reference](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.3)

> [!WARNING]
> TODO we need to consider additional fields, such as `authorization_endpoint` (once we support the Auth Flow) or `response_types_supported`

### Token Endpoint

The Token Endpoint issues an Access Token in exchange for the `pre-authorized_code` from the [Credential Offer](#credential-offer).
The `pre-authorized_code` must be invalidated immediately upon issue of an Access Token. 
The Access Token can be used with the [Credential Endpoint](#credential-endpoint) and [Deferred Credential Endpoint](#deferred-credential-endpoint).

The URL is the `token_endpoint` field of the [Authorization Server Metadata](#authorization-server-metadata).

#### Token Request

| Field                 | Description                                                                       | Required | References                                                                                                                 | Comments                                                       |
| :-------------------- | :-------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------- |
| `grant_type`          |                                                                                   | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-4.1.1-4.2.1)                   | MUST be `urn:ietf:params:oauth:grant-type:pre-authorized_code` |
| `pre-authorized_code` | The value of `pre-authorized_code` from the [Credential Offer](#credential-offer) | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-4.2.2.1) |                                                                |
| `client_id`           | The client DID                                                                    | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-6.1-5)                         |                                                                |

[Reference](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1.3)

> [!WARNING]
> TODO we only support pre-auth flow now, but once we support auth flow then `code` will be used instead of `pre-authorized_code`

#### Successful Token Response

Clients must use the fields from token response in subsequent calls to the [Credential Endpoint](#credential-endpoint) and [Deferred Credential Endpoint](#deferred-credential-endpoint).

| Field                | Description                                                                          | Required | References                                                                                           | Comments                                                                                                    |
| :------------------- | :----------------------------------------------------------------------------------- | :------- | :--------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------- |
| `access_token`       | The access token granted                                                             | y        | [RFC6749](https://datatracker.ietf.org/doc/html/rfc6749#section-4.2.2)                               | `access_token`'s are [Compact Serialized JWT's](https://datatracker.ietf.org/doc/html/rfc7515#section-3.1). |
| `token_type`         |                                                                                      | y        | [RFC6749](https://datatracker.ietf.org/doc/html/rfc6749#section-4.2.2)                               | MUST be `bearer`                                                                                            |
| `expires_in`         | Seconds from issue until the access token expires                                    | y        | [RFC6749](https://datatracker.ietf.org/doc/html/rfc6749#section-4.2.2)                               |                                                                                                             |
| `c_nonce`            | A nonce for use in the subsquent call to [Credential Endpoint](#credential-endpoint) | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-6.2-4.1) |                                                                                                             |
| `c_nonce_expires_in` | Seconds from issue until the `c_nonce` expires                                       | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-6.2-4.2) |                                                                                                             |

Additionally the token response MUST contain HTTP headers "Cache-Control: no-store" and "Pragma: no-cache" as per [RC6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-5.1)

> [!WARNING]
> TODO we need to define refresh token flows

##### `access_token` JOSE Header

The `access_token` granted by the Credential Issuer contains the following [JOSE Header](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1) fields.

| Field | Description                                                  | Required | References                                                             | Comments                                                                    |
| :---- | :----------------------------------------------------------- | :------- | :--------------------------------------------------------------------- | :-------------------------------------------------------------------------- |
| `alg` | (Algorithm) The cryptographic algorithm used to sign the JWT | y        | [RFC7515](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.1) | MUST be `EdDSA` or `ES256K`                                                 |
| `kid` | (KeyID) The fully qualified DID Key ID of the signer         | y        | [RFC7515](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.4) | In the form `did:{method}:{identifier}#{key_id}`                            |
| `typ` | (Type) The explicit JWT type                                 | y        | [RFC7515](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.9) | MUST be `at+jwt` per [RFC9068](https://www.rfc-editor.org/rfc/rfc9068.html) |

##### `access_token` Claims

The `access_token` granted by the Credential Issuer contains the following [JWT Claim](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1) fields.

| Field            | Description                                                          | Required | References                                                                                           | Comments |
| :--------------- | :------------------------------------------------------------------- | :------- | :--------------------------------------------------------------------------------------------------- | :------- |
| `iss`            | (Issuer) The DID of the Authorization Server                         | y        | [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.1)                               |          |
| `aud`            | (Audience) The resource indicator (DID) of the Credential Issuer     | y        | [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.3)                               |          |
| `sub`            | (Subject) The DID of the customer applying for KCC (Applicant DID)   | y        | [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.2)                               |          |
| `client_id`      | (Client ID) The DID of the customer applying for KCC (Applicant DID) | y        | [RFC8693](https://www.rfc-editor.org/rfc/rfc8693#section-4.3)                                        |          |
| `exp`            | (Expiration) The time at which the `access_token` expires            | y        | [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.4)                               |          |
| `iat`            | (IssuedAt) The time at which the `access_token` was granted          | y        | [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.6)                               |          |
| `jti`            | (JWT ID) A unique identifier for the token                           | y        | [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.7)                               |          |
| `c_nonce`        | A nonce to be used in the [Proof JWT](#proofjwt-claims)              | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-6.2-4.1) |          |
| `c_nonce_expiry` | Expiry time of the c_nonce                                           | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-6.2-4.1) |          |

#### Token Error Response

The authorization server responds with an HTTP 400 (Bad Request), and an error object in the response body.

| Field               | Description                                                | Required | References                                                           | Comments |
| :------------------ | :--------------------------------------------------------- | :------- | :------------------------------------------------------------------- | :------- |
| `error`             | A string error code from below list                        | y        | [RFC6749](https://datatracker.ietf.org/doc/html/rfc6749#section-5.2) |          |
| `error_description` | A human-readable description of the error                  | n        | [RFC6749](https://datatracker.ietf.org/doc/html/rfc6749#section-5.2) |          |
| `error_uri`         | A URI that provides additional information about the error | n        | [RFC6749](https://datatracker.ietf.org/doc/html/rfc6749#section-5.2) |          |

##### Error Codes

Error codes are defined in [OID4VCI Token Error Response](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-6.3).

- `invalid_request`
- `invalid_grant`
- `invalid_client`
- `unauthorized_client`
- `unsupported_grant_type`

### Credential Endpoint

This endpoint issues a Credential (or a Transaction ID) upon presentation of a valid Access Token.
The URL is the `credential_endpoint` field of the [Credential Issuer Metadata](#credential-issuer-metadata).

```mermaid
sequenceDiagram
autonumber

participant D as Mobile App
participant P as PFI
participant V as IDV Vendor

alt Immediate Issuance
  D->>+P: Credential Request
  P->>-D: Issued Credential
else Deferred Issuance
  D->>+P: Credential Request
  P->>-D: transaction_id
  loop until credential received
    D->>+P: Deferred Credential Request
    P->>-D: issuance_pending
  end
  V->>+P: Webhook Request w. results
  P->>P: Evaluate results and Issue Credential or Reject
  D->>+P: Deferred Credential Request
  P->>-D: Issued Credential
end 
```

#### Credential Request

The `access_token` must be passed as an HTTP `Authorization` header (i.e. `Authorization: Bearer {access_token}`)

| Field    | Description                                    | Required | References                                                                                             | Comments              |
| :------- | :--------------------------------------------- | :------- | :----------------------------------------------------------------------------------------------------- | :-------------------- |
| `format` | The format of the credential issued            | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2-2.1)   | MUST be `jwt_vc_json` |
| `proof`  | Proof of possession of cryptographic materials | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2-2.2.1) |                       |

##### `proof`

`proof` is an object which contains proof of possession of the client's cryptographic materials.

| Field        | Description       | Required | References                                                                                               | Comments      |
| :----------- | :---------------- | :------- | :------------------------------------------------------------------------------------------------------- | :------------ |
| `proof_type` | The type of proof | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2-2.2.2.1) | MUST be `jwt` |
| `jwt`        | The proof JWT     | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2.1.1)     |               |

###### `proof.jwt` JOSE Headers

For the client to prove possession of their cryptographic materials, they must construct a JWT with the following [JOSE Header](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1) fields.

| Field | Description                                                  | Required | References                                                                                                   | Comments                                         |
| :---- | :----------------------------------------------------------- | :------- | :----------------------------------------------------------------------------------------------------------- | :----------------------------------------------- |
| `alg` | (Algorithm) The cryptographic algorithm used to sign the JWT | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2.1.1-2.1.2.1) | MUST be `EdDSA` or `ES256K`                      |
| `typ` | (Type) The explicit JWT type                                 | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2.1.1-2.1.2.2) | MUST be `openid4vci-proof+jwt`                   |
| `kid` | (KeyID) The fully qualified DID Key ID of the signer         | y        | [OID4VCI](openid4vci-proof+jwt)                                                                              | In the form `did:{method}:{identifier}#{key_id}` |

###### `proof.jwt` Claims

For the client to prove possession of their cryptographic materials, they must construct a JWT with the following [JWT Claim](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1) fields.

| Field   | Description                                                                      | Required | References                                                                                                   | Comments |
| :------ | :------------------------------------------------------------------------------- | :------- | :----------------------------------------------------------------------------------------------------------- | :------- |
| `iss`   | (Issuer) The DID of the customer applying for KCC (Applicant DID)                | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2.1.1-2.2.2.1) |          |
| `aud`   | (Audience) The DID of the Credential Issuer                                      | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2.1.1-2.2.2.2) |          |
| `iat`   | (IssuedAt) The time at which the key proof was created                           | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2.1.1-2.2.2.3) |          |
| `nonce` | The value of the `c_nonce` from the [Token Response](#successful-token-response) | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2.1.1-2.2.2.4) |          |

#### Credential Response

Credential Response can be immediate or deferred. If the issuer can immediately issue the requested credential, it will
return it in the `credential` field, with an HTTP 200 (OK) status code. 

If the issuer cannot immediately issue a credential it returns a `transaction_id`, which is to be used in a subsequent call 
in the [Deferred Credential Request](#deferred-credential-request). The HTTP status code MUST be 202 (Accepted).

| Field            | Description                                                                                | Required | References                                                                                           | Comments                                      |
| :--------------- | :----------------------------------------------------------------------------------------- | :------- | :--------------------------------------------------------------------------------------------------- | :-------------------------------------------- |
| `credential`     | The credential in `jwt_vc_json` format                                                     | n        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.3-6.1) | If missing, `transaction_id` must be present  |
| `transaction_id` | ID used for subsequent call to [Deferred Credential Request](#deferred-credential-request) | n        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.3-6.2) | If missing, then `credential` must be present |

#### Credential Error Response

##### Authorization Errors

If the Credential Request does not contain a valid Access Token, the response is an authorization error response, as per [RFC6750](https://www.rfc-editor.org/rfc/rfc6750.html#section-3).

#### Credential Request Errors

| Field               | Description                               | Required | References                                                                                                 | Comments |
| :------------------ | :---------------------------------------- | :------- | :--------------------------------------------------------------------------------------------------------- | :------- |
| `error`             | A string error code from below list       | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.3.1.2-3.1.1) |          |
| `error_description` | A human-readable description of the error | n        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.3.1.2-3.2)   |          |

Error codes are defined by [OID4VCI Credential Request Errors](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.3.1.2-3.1.1)
- `invalid_credential_request`
- `unsupported_credential_type`
- `invalid_proof`
- `invalid_encryption_parameters`
- `invalid_request`

### Deferred Credential Endpoint

This endpoint issues a credential previously requested via the [Credential Endpoint](#credential-endpoint), in cases where
the credential issuer was not able to immediately issue the credential. 
The URL is the `deferred_credential_endpoint` field of the [Credential Issuer Metadata](#credential-issuer-metadata).

#### Deferred Credential Request

The `access_token` must be passed as an HTTP `Authorization` header (i.e. `Authorization: Bearer {access_token}`).

| Field            | Description                                                                | Required | References                                                                                                            | Comments |
| :--------------- | :------------------------------------------------------------------------- | :------- | :-------------------------------------------------------------------------------------------------------------------- | :------- |
| `transaction_id` | Transaction ID returned by the [Credential Response](#credential-response) | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#name-deferred-credential-request) |          |

#### Deferred Credential Response

| Field        | Description                            | Required | References                                                                                           | Comments |
| :----------- | :------------------------------------- | :------- | :--------------------------------------------------------------------------------------------------- | :------- |
| `credential` | The credential in `jwt_vc_json` format | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-9.1-3.1) |          |

#### Deferred Credential Error Response

When the Deferred Credential Request is invalid or the credential is not available yet, the response is an HTTP 400 (Bad Request), 
and an error object in the response body.

Response body is defined in [Credential Error Response](#credential-error-response)

# Other Considerations

It may very well be the case that this approach works for identity verification in general even outside the purposes of performing KYC but it's far too early to say or have that discussion. Just something to keep in the back of our minds

---
