---
title: "Automatic Certificate Management Environment (ACME) with OpenID Federation 1.0"
abbrev: "ACME OpenID Federation"
category: std

docname: draft-ietf-acme-openid-federation-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Automated Certificate Management Environment"
keyword:
 - OpenID Federation
 - ACME
venue:
  group: "Automated Certificate Management Environment"
  type: "Working Group"
  mail: "acme@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/acme/"
  github: "ietf-wg-acme/draft-ietf-acme-openid-federation"

author:
 -
    fullname: Giuseppe De Marco
    ins: G. De Marco
    organization: independent
    email: demarcog83@gmail.com
 -
    name: Brandon Pitman
    email: bran@bran.land
 -
    name: Tim Geoghegan
    org: ISRG
    email: timgeog+ietf@gmail.com
 -
    name: David Cook
    org: ISRG
    email: divergentdave@gmail.com
 -
    name: J.C. Jones
    org: ISRG
    email: ietf@insufficient.coffee

contributor:
 -
    name: Ameer Ghani
    org: ISRG
    email: inahga@letsencrypt.org

normative:
  OPENID-FED:
    title: "OpenID Federation 1.0"
    target: https://openid.net/specs/openid-federation-1_0.html
    date: 2026-02-17
    author:
      -
        ins: R. Hedberg
        name: Roland Hedberg
      -
        ins: M.B. Jones
        name: Michael Jones
      -
        ins: A.Å. Solberg
        name: Andreas Solberg
      -
        ins: J. Bradley
        name: John Bradley
      -
        ins: G. De Marco
        name: Giuseppe De Marco
      -
        ins: V. Dzhuvinov
        name: Vladimir Dzhuvinov

  IANA-ACME:
    title: "Automated Certificate Management Environment (ACME) Protocol"
    target: https://www.iana.org/assignments/acme
    author:
      organization: IANA

  IANA-OAUTH:
    title: "OAuth Parameters"
    target: https://www.iana.org/assignments/oauth-parameters
    author:
      organization: IANA

--- abstract

The Automatic Certificate Management Environment (ACME) protocol allows server
operators to obtain TLS certificates for their websites, based on a
demonstration of control over the website's domain via a fully-automated
challenge/response protocol.

OpenID Federation 1.0 defines how to build a trust infrastructure using a
trusted third-party model. It uses a trust evaluation mechanism to attest to the
possession of private keys, protocol specific metadata and miscellaneous
administrative and technical information related to a specific entity.

This document defines how X.509 certificates associated with a given OpenID
Federation Entity can be issued by an X.509 Certification Authority through the
ACME protocol to the organizations which are part of a federation built on top
of OpenID Federation 1.0.

--- middle

# Introduction

This document describes extensions to the ACME protocol that integrate with
OpenID Federation 1.0, allowing an ACME server to issue X.509 Certificates
associated with a given OpenID Federation entity. An entity can be a school,
government agency, corporation, automated system, or any participant that
wishes to join the Federation.

In a multilateral federation, composed of thousands of entities belonging to
different organizations, all the participants adhere to the same regulation or
trust framework. OpenID Federation 1.0 allows each participant to recognize
other participants using a trust evaluation mechanism, with RESTful services and
cryptographic materials.

Federation members declare what kind of Entities they are using a basic OpenID
Federation component called an Entity Configuration, a signed JSON Web Token.
This document defines new OpenID Federation
Entity Types for certificate Requestors and Issuers, facilitating automated
discovery of an issuer's ACME API.

X.509 Certificates can be provided to one or more such entities, without having
pre-established any direct relationship or contract.

These Certificates are useful for integrating with existing systems and
protocols which require X.509 Certificates. For example, RADIUS ({{?RFC2865}})
deployments often require Certificates to authenticate clients, or these
Certificates could also be used for TLS {{?RFC8446}}. In order to accommodate a
variety of use cases, this document adds no requirement that CAs conform to any
particular issuance profile, because fields of the X.509 Certificates like the
Common Names, Subject Alternative Names or Key Usages may depend on details of
those existing systems.

This document extends {{!RFC5280}}, ACME ({{!RFC8555}}) and OpenID Federation
1.0 ({{OPENID-FED}}) in the following ways:

- It defines a new ACME Identifier type called `openid-federation`
  ({{identifier-type}}).

- It defines a new ACME challenge type called `openid-federation-01`
  ({{challenge-type}}).

- It defines new OpenID Federation 1.0 Entity Types `acme_issuer`
  ({{issuer-metadata}}) and `acme_requestor` ({{requestor-metadata}}).

- It defines how OpenID Federation 1.0 Superior Entities can use Subordinate
  Statements to publish X.509 Certificates previously issued with ACME
  ({{publish-cert}}).

- It defines a new SubjectAlternativeName type `id-on-OpenIdFederationEntityId`
  and a corresponding OID so that OpenID Federation 1.0 Entity Identifiers can
  be included in X.509 Certificates if an Issuer wishes
  ({{openidfed-othername-id}}).

# Terminology

The terms "Federation Entity", "Trust Anchor", "Entity Configuration",
"Subordinate Statement", "Superior Entity", "Immediate Superior Entity",
"Federation Entity Keys", "Federation Entity Discovery", "Trust Mark", "Trust
Chain", and "Resolved Metadata" used in this document are defined in
{{Section 1.2 of OPENID-FED}}{: relative="#section-1.2"}.

The term "Certificate Signing Request" (CSR) used in this document is defined as
a "Certification Request" in {{!RFC2986}}. The term "Certification Authority"
used in this document is defined in {{!RFC5280}}. The terms "ACME Client" and
"ACME Server" are defined in {{!RFC8555}}.

The specification also defines the following terms:

ACME Identifier:
: An identifier in the sense of {{!RFC8555}}. Some identifier that the ACME
  client proves control of to the ACME server.

Entity Identifier:
: A globally unique string identifier that is bound to one Entity in OpenID
  Federation, as defined in
  {{Section 1.2 of OPENID-FED}}{: relative="#section-1.2-3.4"}.

Requestor:
: A Federation Entity which wants to request X.509 Certificates. It operates an
  ACME client, extended according to this document.

Certificate Issuer (or Issuer):
: A Federation Entity which issues X.509 Certificates. It operates an ACME
  server, extended according to this document.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# OpenID Federation ACME Identifier {#identifier-type}

This document defines a new ACME Identifier type for OpenID Federation Entities,
`openid-federation`, whose value is the Entity Identifier of the Requestor (the
`sub` parameter of the Requestor's Entity Configuration, as defined in
{{Section 3.1.1 of OPENID-FED}}{: relative="#section-3.1.1"}).

For example, the ACME Identifier corresponding to the example Entity
Configuration in {{requestor-metadata}} is:

~~~~
{"type": "openid-federation", "value": "https://requestor.example.com"}
~~~~

The `openid-federation` ACME Identifier type MUST NOT be validated except by the
`openid-federation-01` challenge.

# OpenID Federation Challenge Type {#challenge-type}

The OpenID Federation challenge type allows a Requestor to prove control of a
Federation Entity using the trust evaluation mechanism provided by
{{OPENID-FED}}. The Requestor demonstrates control of a cryptographic public key
published in its OpenID Federation Entity Configuration.

There are two ways the Certificate Issuer is able to check if a Requestor is
part of the federation:

- The Requestor provides a Trust Chain when solving the ACME challenge. This is
  RECOMMENDED since it reduces the effort of the Certificate Issuer in
  evaluating the trust to the Requestor. The Trust Chain includes the
  Requestor's Entity Configuration, so the Requestor need not publish that
  document at its configuration endpoint ({{requestor-metadata}}).

- The Requestor doesn't provide a Trust Chain in the challenge solution. In
  that case the Requestor MUST publish its Entity Configuration at the
  configuration endpoint so the Issuer can perform Federation Entity Discovery.

The openid-federation-01 ACME challenge object has the following format:

type (required, string):  The string "openid-federation-01"

token (required, string):  A random value that uniquely identifies the
    challenge. This value MUST have at least 128 bits of entropy. It MUST NOT
    contain any characters outside the base64url alphabet as described in
    {{Section 5 of !RFC4648}}. Trailing '=' padding characters MUST be stripped.
    See {{!RFC4086}} for additional information on randomness requirements.

trustAnchors (optional, array of string):  An array of strings containing the
    Entity Identifiers of the Issuer's Trust Anchors. When solving the
    challenge, the Requestor can construct a Trust Chain from itself to one of
    these Trust Anchors. It is RECOMMENDED that the Issuer includes this field
    to make it easier for the Requestor to construct a Trust Chain.

A non-normative example of a challenge with `trustAnchors` specified:

~~~~
   {
     "type": "openid-federation-01",
     "url": "https://issuer.example.com/acme/chall/prV_B7yEyA4",
     "status": "pending",
     "token": "LoqXcYV8q5ONbJQxbmR7SCTNo3tiAXDfowyjxAjEuX0",
     "trustAnchors": [
       "https://trust-anchor-1.example.com",
       "https://trust-anchor-2.example.com"
     ]
   }
~~~~

The `openid-federation-01` challenge MUST NOT be used to validate possession of
any ACME Identifiers except `openid-federation` ACME Identifiers.

The Requestor responds to the challenge with an object with the following
format:

sig (required, string):  the compact JSON serialization (as described in
    {{Section 7.1 of !RFC7515}}) of a JWS, signing the key authorization
    encoded in UTF-8.
    The key authorization is computed from the token in the challenge and the
    Requestor's ACME account key, as defined in {{Section 8.1 of !RFC8555}}.
    The signature must be made by one of the keys published in the Requestor's
    `acme_requestor` metadata in its Entity Configuration, as specified in
    {{requestor-metadata}}.
    The JWS MUST include a `kid` header parameter corresponding to the key used
    to sign the key authorization and a `typ` header parameter set to
    "signed-acme-challenge+jwt".

trustChain (optional, array of string):  an array of strings containing signed
    JWTs, representing a Trust Chain from the Requestor to one of the Issuer's
    Trust Anchors (see {{Section 4 of OPENID-FED}}{: relative="#section-4"}).
    The Resolved Metadata of the Trust Chain subject MUST contain
    `acme_requestor` metadata that contains the key used to compute `sig`.
    It is RECOMMENDED that the Requestor includes this field.
    If the Requestor does not publish its Entity Configuration at the
    configuration endpoint ({{requestor-metadata}}), it MUST include
    `trustChain`.
    If the Requestor cannot construct a Trust Chain to one of the Trust Anchors
    indicated by the Issuer, or if no Trust Anchors were indicated, it MAY use
    some other Trust Anchor that it believes the Issuer trusts.
    If the Requestor publishes its Entity Configuration and cannot construct a
    Trust Chain to any Trust Anchor, it MAY omit the `trustChain` field from the
    challenge response.

A non-normative example for an authorization with `trustChain` specified:

~~~~
   POST /acme/chall/prV_B7yEyA4
   Host: issuer.example.com
   Content-Type: application/jose+json

   {
     "protected": base64url({
       "alg": "ES256",
       "kid": "https://issuer.example.com/acme/acct/evOfKhNU60wg",
       "nonce": "UQI1PoRi5OuXzxuX7V7wL0",
       "url": "https://issuer.example.com/acme/chall/prV_B7yEyA4"
     }),
     "payload": base64url({
      "sig": "wQAvHlPV1tVxRW0vZUa4BQ...",
      "trustChain": ["eyJhbGciOiJFU...", "eyJhbGci..."]
     }),
     "signature": "Q1bURgJoEslbD1c5...3pYdSMLio57mQNN4"
   }
~~~~

On receiving a challenge response, the Certificate Issuer verifies that the
Requestor is trusted. If the Requestor did not provide a `trustChain`, the
Issuer MUST perform Federation Entity Discovery ({{Section 10 of OPENID-FED}}{:
relative="#section-10"}) to obtain a Trust Chain for the Requestor. That
discovery requires the Requestor to have published its Entity Configuration at
the configuration endpoint ({{Section 9 of OPENID-FED}}{:
relative="#section-9"}).

Once it has obtained a Trust Chain, the Issuer evaluates the entity's Resolved
Metadata, and verifies:

* That the requested `openid-federation` ACME Identifier value matches the `sub`
  parameter of the Requestor's Entity Configuration.

* That there is a key in the `acme_requestor` metadata ({{requestor-metadata}})
  of the Requestor's Resolved Metadata with a `kid` matching the `kid` claim in
  the challenge response.

* That the `sig` field of the payload is the compact JSON serialization of a
  valid JWS signing the key authorization, using the above public key.

If all of the above checks succeed, then the validation is successful.
Otherwise, it has failed. In either case, the Certificate Issuer responds
according to {{Section 7.5.1 of !RFC8555}}. If the Issuer fails to verify OpenID
Federation trust, the problem document SHOULD contain a subproblem of type
`urn:ietf:params:acme:error:openIDFederationEntity` constructed as discussed in
{{error-type}}.

A non-normative example for the challenge object post-validation:

~~~~
   {
     "type": "openid-federation-01",
     "url": "https://issuer.example.com/acme/chall/prV_B7yEyA4",
     "status": "valid",
     "validated": "2024-10-01T12:05:13.72Z",
     "token": "LoqXcYV8q5ONbJQxbmR7SCTNo3tiAXDfowyjxAjEuX0"
   }
~~~~

# Requestor Entity Configuration Metadata {#requestor-metadata}

The Requestor's Entity Configuration MUST include `acme_requestor` metadata
containing a JWK set, according to {{Section 5.2.1 of
OPENID-FED}}{: relative="#section-5.2.1"}. The keys in the set are used to
respond to ACME challenges.

{{Section 9 of OPENID-FED}}{: relative="#section-9"} requires Trust Anchors and
Intermediate Entities to publish their Entity Configuration at the configuration
endpoint, and says Leaf Entities SHOULD do so. This document uses the profile
exception in that section. A Requestor that includes a `trustChain` in the
`openid-federation-01` challenge response ({{challenge-type}}) conveys its
Entity Configuration as the first statement of that Trust Chain, so it MAY omit
publication at `/.well-known/openid-federation`.

A Requestor that omits `trustChain` MUST publish its Entity Configuration at the
configuration endpoint so the Issuer can perform Federation Entity Discovery.

The following is a non-normative example of an Entity Configuration including
the `acme_requestor` metadata and using the `jwks` metadata parameter.

~~~~
{
  "iss": "https://requestor.example.com",
  "sub": "https://requestor.example.com",
  "iat": 1516239022,
  "exp": 1516298022,
  "jwks": {
    "keys": [
      {
        "kty": "RSA",
        "alg": "RS256",
        "use": "sig",
        "kid": "NzbLsXh8uDCcd-6MNwXF4W_7noWXFZAfHkxZsRGC9Xs",
        "n": "pnXBOusEANuug6ewezb9J_...",
        "e": "AQAB"
      }
    ]
  },
  "metadata": {
    "acme_requestor": {
      "jwks": {
        "keys": [
          {
            "kty": "RSA",
            "kid": "SUdtUndEWVY2cUFDeD...",
            "n": "y_Zc8rByfeRIC9fFZrD...",
            "e": "AQAB"
          },
          {
            "kty": "EC",
            "kid": "MFYycG1raTI4SkZvVDBIMF9CNGw3VEZYUmxQLVN2T21nSWlkd3",
            "crv": "P-256",
            "x": "qAOdPQROkHfZY1daGofOmSNQWpYK8c9G2m2Rbkpbd4c",
            "y": "G_7fF-T8n2vONKM15Mzj4KR_shvHBxKGjMosF6FdoPY"
          }
        ]
      }
    }
  }
}
~~~~

The Issuer MUST only use the Requestor's `acme_requestor` to validate an ACME
challenge. Therefore, after completing the challenge, the Requestor MAY remove
the `acme_requestor` metadata from its Entity Configuration.

# Issuer Discovery

The Requestor's ACME client may either be configured to use a particular ACME
server, or to automatically discover a Certificate Issuer through the OpenID
Federation.

In order to be discoverable, the Issuer MUST publish the entity type
`acme_issuer` in its Entity Configuration, according to {{issuer-metadata}}.

{{OPENID-FED}} describes a variety of patterns and generic mechanisms for
discovering Federation Entities (see {{Section 17.2 of OPENID-FED}}{:
relative="#section-17.2"}). This section describes how Issuers may make
themselves discoverable in a Federation by Requestors.

## Issuer Metadata

The Issuer MUST publish its Entity Configuration including the `acme_issuer`
Entity Type metadata within it. The `acme_issuer` metadata contains one
parameter, `directory_url`, which is the URL of the ACME Directory, as defined
in {{Section 7.1.1 of !RFC8555}}.

Requestors MUST use the ACME Directory provided in the Issuer's Entity
Configuration for client configuration of ACME endpoints.

The following is a non-normative example of an Entity Configuration including
the `acme_issuer` metadata:

~~~~
{
  "iss": "https://issuer.example.com",
  "sub": "https://issuer.example.com",
  "iat": 1516239022,
  "exp": 1516298022,
  "jwks": {
    "keys": [
      {
        "kty": "RSA",
        "alg": "RS256",
        "use": "sig",
        "kid": "NzbLsXh8uDCcd-6MNwXF4W_7noWXFZAfHkxZsRGC9Xs",
        "n": "pnXBOusEANuug6ewezb9J_...",
        "e": "AQAB"
      }
    ]
  },
  "metadata": {
    "acme_issuer": {
      "directory_url": "https://issuer.example.com/acme/directory"
    }
  }
}
~~~~

# Publication of the Certificates within the Federation {#publish-cert}

The X.509 Certificates issued by federation Immediate Superior Entities
pertaining to one or more Federation Entity Keys in control of their
Subordinates MAY publish this information by including the `x5c` member
in each JWK contained within the matching Subordinate Statement. The contents
of the published `x5c` member, including whether it contains a full or
partial Trust Chain, and if so, to what Trust Anchor, are policy decisions
out of scope for this document.

# CSR and Certificate Fields {#openidfed-othername-id}

Depending on the Certificate Issuer's X.509 Certificate profile, the CSR and
X.509 Certificate MAY associate the X.509 Certificate to the Federation Entity
by including the Entity Identifier in the X.509 Certificate.

To do so, the Issuer includes a Subject Alternative Name extension containing an
`otherName` with a `type-id` of `id-on-OpenIdFederationEntityId`. The value of
the name is an Octet String containing the UTF-8 encoding of the Entity
Identifier (i.e., the corresponding `openid-federation` ACME Identifier from
the `newOrder` request).

~~~~
   id-on-OpenIdFederationEntityId OBJECT IDENTIFIER ::= { id-on XXX }

   OpenIdFederationEntityId ::= UTF8String
~~~~

# Certificate Lifecycle

The identity of the Requestor is verified through proof of possession of a
private key corresponding to a public key attested within a Trust Chain. The
Trust Chain has an expiration time, and its content MUST NOT be trusted past the
expiration time ({{Section 10.4 of OPENID-FED}}{: relative="#section-10.4"}).

The `notBefore` and `notAfter` fields of issued certificates MUST represent
dates before the Trust Chain's expiration time. If the Requestor's newOrder
request ({{Section 7.4 of !RFC8555}}) contains `notAfter` or `notBefore` fields
that make this impossible, the Certificate Issuer MUST reply with an error of
type `urn:ietf:params:acme:error:openIDFederationCertificateValidity`.

# Errors {#error-type}

This document defines two new error type URIs to be used in problem documents
{{!RFC9457}}, as described in {{Section 6.7 of !RFC8555}}.

The error type `urn:ietf:params:acme:error:openIDFederationEntity` is used to
encapsulate any OAuth error code returned while resolving OpenID Federation
Entities. The title of this error type is "OpenID Federation Error". The
`detail` member of the problem document MAY include the description of the
particular OAuth error code that caused the error. The problem document for this
error type SHOULD include an extension member named `error_code`, which MUST be
set to the OAuth error code, taken from the error codes defined in
{{Section 8.9 of OPENID-FED}}{: relative="#section-8.9"}.

The error type `urn:ietf:params:acme:error:openIDFederationCertificateValidity`
is used to indicate that a Certificate could not be issued because of a mismatch
between the requested period of validity and the expiration of the OpenID
Federation Trust Chain.

# Security Considerations

The `openid-federation-01` challenge defined in {{challenge-type}} defends
against replay attacks by malicious ACME servers because the signature in
challenge responses is over an ACME key authorization, which binds the ACME
account key.

The cryptographic keys in the `acme_requestor` metadata SHOULD NOT be reused
for other purposes than signing responses to `acme-federation-01` challenges.
For example, the same keys SHOULD NOT be reused in the issued X.509 Certificate.
If the keys are reused for other purposes, cross-protocol attacks MUST be
considered.

The cryptographic keys in the `acme_requestor` metadata SHOULD be rotated
periodically.

# IANA Considerations

IANA is kindly asked to make the following updates to registries:

## ACME Registry Group

The following updates are all assignments in the "Automated Certificate
Management Environment (ACME) Protocol" registry group {{IANA-ACME}}.

### ACME Identifier Types

IANA is asked to add to the "ACME Identifier Types" registry, defined in
{{Section 9.7.7 of !RFC8555}}, the entry below, as specified here in
{{identifier-type}}:

|Label|Reference|
|-----|---------|
|openid-federation|this document|

### ACME Validation Methods

IANA is also asked to add to the "ACME Validation Methods" registry, defined
in {{Section 9.7.8 of !RFC8555}}, the entry below, as specified here
in {{challenge-type}}:

|Label|Identifier Type|Reference|
|-----|---------------|---------|
|openid-federation-01|openid-federation|this document|

### ACME Error Types

IANA is also asked to add to the "ACME Error Types" registry, defined
in {{Section 9.7.4 of !RFC8555}}, the entry below, as specified here in
{{error-type}}:

|Type|Description|Reference|
|----|-----------|---------|
|openIDFederationEntity|An error occurred while resolving an OpenID Federation entity|this document|
|openIDFederationCertificateValidity|A certificate was requested whose validity period is not before the OpenID Federation Trust Chain's expiry|this document|

## Assign X.509 PKIX Other Name

IANA is asked to add to the "PKIX Other Name Forms" registry
([1.3.6.1.5.5.7.8](https://www.iana.org/assignments/smi-numbers/smi-numbers.xhtml#smi-numbers-1.3.6.1.5.5.7.8)) the entry below, as
specified here in {{openidfed-othername-id}}

|Decimal|Description|Reference|
|-------|-----------|---------|
|TBA    |id-on-OpenIdFederationEntityId|this document|

--- back

# End-to-end Issuance Flow

~~~ BEGIN EDNOTE ~~~

This appendix is to be removed before publication.

This section contains explanatory material that recaps a lot of RFC 8555. It is
included here for the benefit of readers who are familiar with OpenID Federation
but not with ACME, and want to see at a glance how the whole thing fits
together.

The following diagram illustrates a successful interaction between Issuer and
Requestor to retrieve an X.509 Certificate. The diagram assumes the Requestor
has already discovered the Issuer, and the Requestor has already created an
ACME account with the Issuer.

This section is informational and non-normative. The remainder of this document,
{{!RFC8555}} and {{OPENID-FED}} should be considered correct should this
appendix disagree with any of them.

~~~~ ascii-art
,-----------------.
|Requestor's      |  ,-----------.
|OpenID Federation|  |Requestor's|               ,------------------------.  ,-----------------------.
| Web Server      |  |ACME Client|               |X.509 Certificate Issuer|  |Federation Trust Anchor|
`--------+--------'  `-----+-----'               `-----------+------------'  `-----------+-----------'
        |                  |           POST /acme/new-order  |                           |
        |                  |--------------------------------->                           |
        |                  |                                 |                           |
        |                  |Authorization at                 |                           |
        |                  |/acme/authz/[authz-id]           |                           |
        |                  |Finalize at                      |                           |
        |                  | /acme/order/[order-id]/finalize |                           |
        |                  |<- - - - - - - - - - - - - - - - -                           |
        |                  |                                 |                           |
        |                  | POST /acme/authz/[authz-id]     |                           |
        |                  |--------------------------------->                           |
        |                  |                                 |                           |
        |                  |  openid-federation-01 Challenge |                           |
        |                  |  at /acme/chall/[chall-id]      |                           |
        |                  |<- - - - - - - - - - - - - - - - -                           |
        |                  |                                 |                           |
        |                  ----.                             |                           |
        |                  |   | Sign challenge token        |                           |
        |                  |   | with private key            |                           |
        |                  <---'                             |                           |
        |                  |                                 |                           |
        |                  | POST /acme/chall/[chall-id]     |                           |
        |                  | with signed                     |                           |
        |                  | token and entity ID             |                           |
        |                  | set to Requestor's ID           |                           |
        |                  |--------------------------------->                           |
        |                  |                                 |                           |
        |                  |      Challenge validation       |                           |
        |                  |      beginning                  |                           |
        |                  |<- - - - - - - - - - - - - - - - -                           |
        |                  |                                 |                           |
        |          GET /.well-known/openid-federation        |                           |
        |<----------------------------------------------------                           |
        |                  |                                 |                           |
        |           Requestor's Entity Configuration         |                           |
        | - - -  - - - - - - - - - - - - - - - - - - - - - - >                           |
        |                  |                                 |                           |
        |                  |           ______________________________________________________
        |                  |           ! OPT  /  If Requestor didn't provide Trust Chain |  !
        |                  |           !_____/               |                           |  !
        |                  |           !                     |  Determine Trust Chain    |  !
        |                  |           !                     |  from Issuer's            |  !
        |                  |           !                     |  Trust Anchor to Requestor|  !
        |                  |           !                     |  (Federation Discovery)   |  !
        |                  |           !                     | <------------------------>|  !
        |                  |           !~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~!
        |                  |                                 |                           |
        |                  |                                 |----.                      |
        |                  |                                 |    | Evaluate Trust Chain |
        |                  |                                 |<---'                      |
        |                  |                                 |                           |
        |                  |                                 |----.                      |
        |                  |                                 |    | Check                |
        |                  |                                 |    | Entity Configuration |
        |                  |                                 |    | sub matches          |
        |                  |                                 |    | Entity Identifier    |
        |                  |                                 |<---' in the order         |
        |                  |                                 |                           |
        |                  |                                 |                           |
        |                  |                                 |----.                      |
        |                  |                                 |    | Check challenge      |
        |                  |                                 |    | sig is signed        |
        |                  |                                 |    | with key in          |
        |                  |                                 |<---' Entity Configuration |
        |                  |                                 |                           |
        |  _________________________________________________________________________     |
        |  ! LOOP  /  Poll until authz status                |                      !    |
        |  !      /  is "valid" or "invalid"                 |                      !    |
        |  !_____/         |                                 |                      !    |
        |  !               |    POST-as-GET                  |                      !    |
        |  !               |    /acme/authz/[authz-id]       |                      !    |
        |  !               |--------------------------------->                      !    |
        |  !               |                                 |                      !    |
        |  !               |           Current authz status  |                      !    |
        |  !               |<---------------------------------                      !    |
        |  !~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~!    |
        |                  |                                 |                           |
        |                  |                                 |                           |
        |  ___________________________________________________________________________   |
        |  ! OPT  /  If the authz status is "valid"          |                        !  |
        |  !_____/         |                                 |                        !  |
        |  !               | POST                            |                        !  |
        |  !               | /acme/orders/[order-id]/finalize|                        !  |
        |  !               | with CSR                        |                        !  |
        |  !               |--------------------------------->                        !  |
        |  !               |                                 |                        !  |
        |  !               |                                 |----.                   !  |
        |  !               |                                 |    | Check CSR         !  |
        |  !               |                                 |    | validity          !  |
        |  !               |                                 |    | according to      !  |
        |  !               |                                 |    | protocol          !  |
        |  !               |                                 |<---' and CA policy     !  |
        |  !               |                                 |                        !  |
        |  !               |                                 |                        !  |
        |  !               |  Order object with certificate  |                        !  |
        |  !               |  at /acme/cert/[cert-id]        |                        !  |
        |  !               |<- - - - - - - - - - - - - - - - -                        !  |
        |  !               |                                 |                        !  |
        |  !               |       POST /acme/cert/[cert-id] |                        !  |
        |  !               |--------------------------------->                        !  |
        |  !               |                                 |                        !  |
        |  !               |  Newly issued X.509 Certificate |                        !  |
        |  !               |<- - - - - - - - - - - - - - - - -                        !  |
        |  !~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~!  |
,--------+--------.  ,-----+-----.               ,-----------+------------.  ,-----------+-----------.
|Requestor's      |  |Requestor's|               |X.509 Certificate Issuer|  |Federation Trust Anchor|
|OpenID Federation|  |ACME Client|               `------------------------'  `-----------------------'
| Web Server      |  `-----------'
`-----------------'
~~~~

~~~ END EDNOTE ~~~

# Acknowledgments
{:numbered="false"}

Aaron Gable and Mike Ounsworth, for early and thorough reviews of this draft.
