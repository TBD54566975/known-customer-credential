# Known Customer Credential Issuance <!-- omit in toc -->

<!-- TOC -->
- [Introduction](#introduction)
  - [Known Customer Credential](#known-customer-credential)
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
    - [Mobile App](#mobile-app)
    - [Web View](#web-view)
  - [Initiating IDV Flow](#initiating-idv-flow)
    - [SIOPv2 Authorization Request](#siopv2-authorization-request)
      - [Client Metadata](#client-metadata)
      - [URI Encoding](#uri-encoding)
    - [SIOPv2 Authorization Response](#siopv2-authorization-response)
      - [ID Token](#id-token)
    - [IDV Request](#idv-request)
      - [Credential Offer](#credential-offer)
      - [Grants](#grants)
      - [Grant Type: urn:ietf:params:oauth:grant-type:pre-authorized\_code](#grant-type-urnietfparamsoauthgrant-typepre-authorized_code)
  - [IDV](#idv)
    - [IDV Vendor Collects PII](#idv-vendor-collects-pii)
    - [PFI collects PII](#pfi-collects-pii)
  - [Credential Issuance](#credential-issuance)
    - [1. Metadata Endpoints](#1-metadata-endpoints)
      - [Well Known Endpoints](#well-known-endpoints)
      - [Credential Issuer Metadata](#credential-issuer-metadata)
        - [`credential_configurations_supported`](#credential_configurations_supported)
      - [Authorization Server Metadata](#authorization-server-metadata)
    - [2. Authorization Endpoints](#2-authorization-endpoints)
      - [Token Request](#token-request)
      - [Token Response](#token-response)
        - [`access_token` JOSE Header](#access_token-jose-header)
        - [`access_token` Claims](#access_token-claims)
    - [3. Issuance Endpoints](#3-issuance-endpoints)
      - [Credential Request](#credential-request)
        - [`proof`](#proof)
          - [`proof.jwt` JOSE Headers](#proofjwt-jose-headers)
          - [`proof.jwt` Claims](#proofjwt-claims)
      - [Credential Response](#credential-response)
      - [Deferred Credential Request](#deferred-credential-request)
      - [Deferred Credential Response](#deferred-credential-response)
- [Other Considerations](#other-considerations)
<!-- TOC -->


# Introduction
This document proposes a standardized means for PFIs (Participating Financial Institution) to perform KYC (Know Your Customer) and issue a subsequent KCC (Known Customer Credential) on a DID (Decentralized Identifer) controlled by a retail customer for the purpose of providing financial services to that DID in accordance to regulatory requirements.

## Known Customer Credential

KCC (Known Customer Credential) is a [VC (Verifiable Credential)](https://www.w3.org/TR/vc-data-model-2.0/) which is intended to be utilized by a retail customer as proof of an actively compliant KYC verification.

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
> Web-based IDV is necessary for reasons that are explained in this document

# Requirements
This body of work is an extension of the work being done for tbDEX. In effect, this proposal considers the following as requirements:


1. **Must support all IDV flows described in the [IDV Vendor Integrations](#idv-vendor-integrations) section of this document**

Ensuring that this is possible is essential to reduce friction or pain points for financial institutions interested in providing liquidity on tbDEX

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
As a means to provide clarity, many of the examples in this section will refer to an imaginary mobile application named Mobile Wallet.

Mobile Wallet is a mobile application that can be used by individuals to:
* Purchase Stablecoin using fiat currency from a PFI via tbDEX. The PFI acts as the custodian of the purchased Stablecoin
* Send custodied Stablecoin to anyone via tbDEX
* Sell Stablecoin for fiat currency through a PFI via tbDEX.

Mobile Wallet acts as a **self-custodial** Identity Wallet that:
* creates a DID for each individual and securely stores private keys directly on device. 
* stores Verifiable Credentials issued to the user directly on device

## Assumptions
* The PFI controls a DID whose method supports `service` endpoints
* The Holder controls their own DID
* The Holder has already discovered the PFI's DID


## Participants

This implementation involves 3 distinct participants that have different responsibilities:

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


## Initiating IDV Flow

Initiating the IDV flow is done using [SIOPv2](https://openid.github.io/SIOPv2/openid-connect-self-issued-v2-wg-draft.html).

```mermaid
sequenceDiagram
autonumber

participant W as Webview
participant D as Mobile Wallet
participant P as PFI

D->>+P: GET did:ex:pfi?service=IDV
P->>P: Construct SIOPv2 Authorization Request
P-->>-D: SIOPv2 Authorization Request
D->>D: Construct SIOPv2 Authorization Response
D->>+P: SIOPv2 Authorization Response
P->>P: Construct IDV Request
P-->>-D: IDV Request
D->>D: Verify IDV Request
D->>W: Load URL in IDV Request
```

1. Mobile App resolves the PFI's DID and sends an HTTP GET Request to the `serviceEndpoint` of the first `IDV` service found in the resolved DID Document
2. PFI constructs a [SIOPv2 Authorization Request](#siopv2-authorization-request)
3. PFI URI encodes SIOPv2 Authorization Request and returns in HTTP response
4. Mobile Wallet verifies integrity of SIOPv2 Authorization Request and constructs a [SIOPv2 Authorization Response](#siopv2-authorization-response)
5. Mobile Wallet POSTs SIOPv2 Authorization Response to the `response_uri` from the SIOPv2 Authorization Request 
6. PFI verifies integrity of SIOPv2 Authorization Response and constructs IDV Request
7. PFI returns IDV Request in HTTP response
8. Mobile Wallet verifies integrity of IDV Request
9. Mobile Wallet loads URL provided in IDV Request in Webview


> [!WARNING]
> I don't know if we're breaking OIDC conformance here by using the response returned by RP to convey use-case specific information

### SIOPv2 Authorization Request

| Field                     | Description                                                                                  | Required | References                                                                                                                                                                                   | Comments                                                  |
| :------------------------ | :------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------- |
| `client_id`               | The DID of the RP, which is us (the PFI)                                                     | y        |                                                                                                                                                                                              |                                                           |
| `scope`                   | What's being requested. 'openid' indicates ID Token is being requested                       | y        | [OIDC](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest)                                                                                                                    |                                                           |
| `response_type`           | What sort of response the RP is expecting. MUST include `id_token`. MAY include `vp_token`   | y        | [OIDC](https://openid.net/specs/openid-connect-core-1_0.html#Authentication)                                                                                                                 |                                                           |
| `response_uri`            | The URI to which the SIOPv2 Authorization Response will be sent                              | y        | [OID4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.2-7.2)                                                                                                |                                                           |
| `response_mode`           | The mode in which the SIOPv2 Authorization Response will be sent. MUST be `direct_post`      | y        | [OID4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-6.2-1)                                                                                                  |                                                           |
| `presentation_definition` | Used by PFI to request VCs as input to IDV process                                           | n        | [OID4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#name-presentation_definition-par)                                                                               | If present, Response Type `vp_token` MUST also be present |
| `nonce`                   | A nonce which MUST be included in the ID Token provided in the SIOPv2 Authorization Response | y        |                                                                                                                                                                                              |                                                           |
| `client_metadata`         | A JSON object containing the Verifier metadata values                                        | y        | [OIDC](https://openid.net/specs/openid-connect-registration-1_0.html) [SIOPv2](https://openid.github.io/SIOPv2/openid-connect-self-issued-v2-wg-draft.html#name-relying-party-client-metada) |                                                           |

#### Client Metadata
| Field                            | Description                                                                                                 | Required | References                                                                                              | Comments                  |
| :------------------------------- | :---------------------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------ | :------------------------ |
| `subject_syntax_types_supported` | Space separated list of DID methods supported for the subject of ID Token                                   | y        | [SIOPv2](https://openid.github.io/SIOPv2/openid-connect-self-issued-v2-wg-draft.html#section-7.5-2.1.1) | Example `did:dht did:jwk` |
| `client_name`                    | Human-readable string name of the client to be presented to the end-user during authorization               | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                           |
| `client_uri`                     | URI of a web page providing information about the client                                                    | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                           |
| `logo_uri`                       | URI of an image logo for the client                                                                         | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                           |
| `contacts`                       | Array of strings representing ways to contact people responsible for this client, typically email addresses | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                           |
| `tos_uri`                        | URI that points to a terms of service document for the client                                               | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                           |
| `policy_uri`                     | URI that points to a privacy policy document                                                                | n        | [RFC7591](https://www.rfc-editor.org/rfc/rfc7591.html#section-2)                                        |                           |

> [!IMPORTANT]
> Include `vp_formats` in Client Metadata?  https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#section-9.1-2.2

> [!IMPORTANT]
> the inclusion of `presentation_definition` as per [OID4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#name-presentation_definition-par) allows for other verifiable credentials to be provided as input for IDV.

#### URI Encoding

The SIOPv2 Authorization Request is encoded as a URI before being returned to Mobile Wallet, as per [SIOPv2](https://openid.github.io/SIOPv2/openid-connect-self-issued-v2-wg-draft.html#section-5). No `authorization_endpoint` is used in the URI, so it is the query parameter portion of the URI only.

### SIOPv2 Authorization Response

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

### IDV Request
| Field              | Description                     | Required | References                                                                                                                            | Comments                                                                                       |
| :----------------- | :------------------------------ | :------- | :------------------------------------------------------------------------------------------------------------------------------------ | :--------------------------------------------------------------------------------------------- |
| `url`              | URL of form used to collect PII | y        |                                                                                                                                       | Required for now until we figure out how to support exclusively providing credentials as input |
| `credential_offer` |                                 | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#name-credential-offer-parameters) |                                                                                                |

#### Credential Offer
| Field                          | Description                                                                                                                                                         | Required | References                                                                                                             | Comments |
| :----------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :------- | :--------------------------------------------------------------------------------------------------------------------- | :------- |
| `credential_issuer`            | The URL of the Credential Issuer that the Mobile Wallet will interact with in subsequent steps                                                                      | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-2.1) |          |
| `credential_configuration_ids` | Array of unique strings that each identify a credential being offered. Mobile Wallet can use these to request metadata                                              | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-2.2) |          |
| `grants`                       | Object containing Grant Types that the Credential Issuer will accept for this credential offer. MUST contain `urn:ietf:params:oauth:grant-type:pre-authorized_code` | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-2.3) |          |

#### Grants 
| Field                                                  | Description                                                                                             | Required | References                                                                                                               | Comments |
| :----------------------------------------------------- | :------------------------------------------------------------------------------------------------------ | :------- | :----------------------------------------------------------------------------------------------------------------------- | :------- |
| `urn:ietf:params:oauth:grant-type:pre-authorized_code` | Grant Type that allows the Mobile Wallet to follow a Pre-Authorized Code Flow to collect the credential | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-4.2.1) |          |


#### Grant Type: urn:ietf:params:oauth:grant-type:pre-authorized_code
| Field                 | Description                                                                                              | Required | References                                                                                                                 | Comments |
| :-------------------- | :------------------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------------------------- | :------- |
| `pre-authorized_code` | The code representing the Credential Issuer's authorization for the Mobile Wallet to obtain a credential | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-4.2.2.1) |          |

> [!WARNING] 
> TODO: explain rationale behind providing `credential_offer` at this stage

## IDV

> [!IMPORTANT]
> Whether the PFI is utilizing an IDV vendor is entirely opaque from the originating mobile app's perspective.

### IDV Vendor Collects PII
```mermaid
sequenceDiagram
autonumber

actor A as Alice 
participant D as Mobile Wallet
participant W as Webview
participant P as PFI
participant V as IDV Vendor


W->>W: Load URL
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

actor A as Alice 
participant D as Mobile Wallet
participant W as Webview
participant P as PFI


W->>W: Load URL
loop until 200 response
    A->>W: Provide PII, Submit
    W->>P: PII
    P->>P: Process
    P->>W: 200 or 400 with errors
end
W->>W: Close
```

## Credential Issuance

```mermaid
sequenceDiagram
autonumber

participant D as Mobile Wallet
participant P as PFI

D->>+P: Fetch metadata
P-->>-D: Metadata
D->>+P: Request authorization
P-->>-D: Authorize
D->>+P: Issue credential
P-->>-D: Credential
```

### 1. Metadata Endpoints

Metadata resources are hosted by the Credential Issuer as a means of informing client's of endpoint locations, technical capabilities, feature support and (internationalized) display information.

#### Well Known Endpoints

Credential Issuers are required to set up a number of well known endpoints to facilitate authorization and credential issuance as follows. 

URLs to retrieve both [Credential Issuer Metadata](#credential-issuer-metadata) and [Authorization Server Metadata](#authorization-server-metadata) are dynamically constructed by the client [using `.well-known` URI's](https://www.rfc-editor.org/rfc/rfc5785).

- [Credential Issuer Metadata](#credential-issuer-metadata) URL: `credential_issuer` + `/.well-known/openid-credential-issuer`
- [Authorization Server Metadata](#authorization-server-metadata) URL: `credential_issuer` + `/.well-known/oauth-authorization-server`

Where `credential_issuer` originates from within the [Credential Offer](#credential-offer) from within the [IDV Request](#idv-request)

#### Credential Issuer Metadata

The Credential Issuer Metadata informs clients of endpoint locations, technical capabilities, supported Credentials, and (internationalized) display information.

| Field                                                                         | Description                                                                                                 | Required | References                                                                                              | Comments                                                                               |
| :---------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------- |
| `credential_issuer`                                                           | URL of the Credential Issuer                                                                                | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2.3-2.1) | Same value as the `credential_issuer` within the [Credential Offer](#credential-offer) |
| `credential_endpoint`                                                         | URL for the [Credential Endpoint](#credential-endpoint)                                                     | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2.3-2.3) |                                                                                        |
| [`credential_configurations_supported`](#credential_configurations_supported) | Object which defines the specifics of the Credential's for which the Credential Issuer supports issuance of | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2.3-2.3) |                                                                                        |

##### `credential_configurations_supported`

The `credential_configurations_supported` is an object which defines the specifics of the Credential's for which the Credential Issuer supports issuance of. The `credential_configurations_supported` is a key/value object wherein each key corresponds to a value within the `credential_configuration_ids` from the [Credential Offer](#credential-offer) and the value is defined with the following fields.

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

### 2. Authorization Endpoints

Clients must authorize with the Credential Issuer prior-to interfacing with the [Issuance Endpoints](#3-issuance-endpoints) as a means of authorizing the Credential Issuer access to the client resources created during the [IDV](#idv) phase.

#### Token Request

Clients must request an access token in order to interface with the [3. Issuance Endpoints](#3-issuance-endpoints).

| Field                 | Description                                                                       | Required | References                                                                                                                 | Comments                                                       |
| :-------------------- | :-------------------------------------------------------------------------------- | :------- | :------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------- |
| `grant_type`          |                                                                                   | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-4.1.1-4.2.1)                   | MUST be `urn:ietf:params:oauth:grant-type:pre-authorized_code` |
| `pre-authorized_code` | The value of `pre-authorized_code` from the [Credential Offer](#credential-offer) | y        | [OID4VCI](https://openid.github.io/OpenID4VCI/openid-4-verifiable-credential-issuance-wg-draft.html#section-4.1.1-4.2.2.1) |                                                                |
| `client_id`           | The client DID                                                                    | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-6.1-5)                         |                                                                |

[Reference](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1.3)

> [!WARNING]
> TODO we only support pre-auth flow now, but once we support auth flow then `code` will be used instead of `pre-authorized_code`

#### Token Response

Clients must use the fields from token response in subsequent calls to the [3. Issuance Endpoints](#3-issuance-endpoints).

| Field                | Description                                                                          | Required | References                                                                                           | Comments                                                                                                    |
| :------------------- | :----------------------------------------------------------------------------------- | :------- | :--------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------- |
| `access_token`       | The access token granted                                                             | y        | [RFC6749](https://datatracker.ietf.org/doc/html/rfc6749#section-4.2.2)                               | `access_token`'s are [Compact Serialized JWT's](https://datatracker.ietf.org/doc/html/rfc7515#section-3.1). |
| `token_type`         |                                                                                      | y        | [RFC6749](https://datatracker.ietf.org/doc/html/rfc6749#section-4.2.2)                               | MUST be `bearer`                                                                                            |
| `expires_in`         | Seconds from issue until the access token expires                                    | y        | [RFC6749](https://datatracker.ietf.org/doc/html/rfc6749#section-4.2.2)                               |                                                                                                             |
| `c_nonce`            | A nonce for use in the subsquent call to [Credential Endpoint](#credential-endpoint) | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-6.2-4.1) |                                                                                                             |
| `c_nonce_expires_in` | Seconds from issue until the `c_nonce` expires                                       | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-6.2-4.2) |                                                                                                             |

> [!WARNING]
> TODO we need to define error responses https://datatracker.ietf.org/doc/html/rfc6749#section-4.2.2.1
> 
> TODO including `authorization_pending` https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#name-token-error-response
> 
>     the `authorization_pending` occurs here, and is approved by 

> [!WARNING]
> TODO we need to define refresh token flows

##### `access_token` JOSE Header

The `access_token` granted by the Credential Issuer contains the following [JOSE Header](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1) fields.

| Field | Description                                                  | Required | References                                                             | Comments                                                                                |
| :---- | :----------------------------------------------------------- | :------- | :--------------------------------------------------------------------- | :-------------------------------------------------------------------------------------- |
| `alg` | (Algorithm) The cryptographic algorithm used to sign the JWT | y        | [RFC7515](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.1) | MUST be `EdDSA` or `ES256K`                                                             |
| `kid` | (KeyID) The fully qualified DID Key ID of the signer         | y        | [RFC7515](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.4) | In the form `did:{method}:{identifier}#{key_id}`                                        |
| `typ` | (Type) The explicit JWT type                                 | y        | [RFC7515](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.9) | MUST be `application/at+jwt` per [RFC9068](https://www.rfc-editor.org/rfc/rfc9068.html) |

##### `access_token` Claims

The `access_token` granted by the Credential Issuer contains the following [JWT Claim](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1) fields.

| Field | Description                                                        | Required | References                                                             | Comments |
| :---- | :----------------------------------------------------------------- | :------- | :--------------------------------------------------------------------- | :------- |
| `iss` | (Issuer) The DID of the Credential Issuer                          | y        | [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.1) |          |
| `sub` | (Subject) The DID of the customer applying for KCC (Applicant DID) | y        | [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.2) |          |
| `exp` | (Expiration) The time at which the `access_token` expires          | y        | [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.4) |          |
| `iat` | (IssuedAt) The time at which the `access_token` was granted        | y        | [RFC7519](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.6) |          |

### 3. Issuance Endpoints

Clients interface with the following endpoints as a means of acquiring the credential.

```mermaid
sequenceDiagram
autonumber

participant D as Mobile Wallet
participant P as PFI
participant V as IDV Vendor

D->>+P: Credentials Request
P->>-D: txn_id

loop until credential received
    D->>+P: Deferred Cred Request
    P->>-D: issuance_pending
end
V->>+P: Webhook Request w. results
P->>P: evaluate results and Issue Credential or Reject
D->>P: Deferred Credential Request
P->>D: Credential Response w/ Credential
P->>-P: Invalidate preauth code
```

#### Credential Request

Clients use the Credential Request to request a credential.

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

| Field   | Description                                                           | Required | References                                                                                                   | Comments |
| :------ | :-------------------------------------------------------------------- | :------- | :----------------------------------------------------------------------------------------------------------- | :------- |
| `iss`   | (Issuer) The DID of the customer applying for KCC (Applicant DID)     | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2.1.1-2.2.2.1) |          |
| `aud`   | (Audience) The DID of the Credential Issuer                           | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2.1.1-2.2.2.2) |          |
| `iat`   | (IssuedAt) The time at which the key proof was created                | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2.1.1-2.2.2.3) |          |
| `nonce` | The value of the `c_nonce` from the [Token Response](#token-response) | y        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.2.1.1-2.2.2.4) |          |

#### Credential Response

The Credential Issuer will respond to a [Credential Request](#credential-request) with either the credential (in `jwt_vc_json` format) given the Credential Issuer is ready to issue, whereafter the credential may be used for the purpose of providing financial services to that DID in accordance to regulatory requirements, else, given the Credential Issuer is not ready to issue the credential, they must respond with a `transaction_id` which is to be used in a subsequent call in the [Deferred Credential Request](#deferred-credential-request).

| Field            | Description                                                                                | Required | References                                                                                           | Comments                                      |
| :--------------- | :----------------------------------------------------------------------------------------- | :------- | :--------------------------------------------------------------------------------------------------- | :-------------------------------------------- |
| `credential`     | The credential in `jwt_vc_json` format                                                     | n        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.3-6.1) | If missing, `transaction_id` must be present  |
| `transaction_id` | ID used for subsequent call to [Deferred Credential Request](#deferred-credential-request) | n        | [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-7.3-6.2) | If missing, then `credential` must be present |

> [!WARNING]
> TODO we need to define the c_nonce stuff for the deferred flow; right now `proof` is embedded under the Credential Request, but it's applicable against the deferred and batch credential requests

#### Deferred Credential Request

TODO

#### Deferred Credential Response

TODO

# Other Considerations

It may very well be the case that this approach works for identity verification in general even outside the purposes of performing KYC but it's far too early to say or have that discussion. Just something to keep in the back of our minds

---
