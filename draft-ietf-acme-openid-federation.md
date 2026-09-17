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
  evaluating the trust to the Requestor.

- The Requestor doesn't provide a Trust Chain in the challenge solution.

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
    If the Requestor cannot construct a Trust Chain to one of the Trust Anchors
    indicated by the Issuer, or if no Trust Anchors were indicated, it MAY use
    some other Trust Anchor that it believes the Issuer trusts.
    If the Requestor cannot construct a Trust Chain to any Trust Anchor, it MAY
    omit the `trustChain` field from the challenge response.

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
relative="#section-10"}) to obtain a Trust Chain for the Requestor.

The Issuer MUST verify that the Trust Chain is valid according to
{{Section 4 of OPENID-FED}}{: relative="#section-4"} and that it terminates at
a Trust Anchor the Issuer is configured to trust. The Issuer MUST NOT accept a
Trust Chain that terminates at any other Trust Anchor, including a Trust Anchor
selected by the Requestor that the Issuer does not itself trust.

Once it has obtained a valid Trust Chain, the Issuer evaluates the entity's
Resolved Metadata, and verifies:

* That the requested `openid-federation` ACME Identifier value matches the `sub`
  parameter of the Requestor's Entity Configuration.

* That there is a key in the `acme_requestor` metadata ({{requestor-metadata}})
  of the Requestor's Resolved Metadata with a `kid` matching the `kid` claim in
  the challenge response.

* That the `sig` field of the payload is the compact JSON serialization of a
  valid JWS signing the key authorization, using the above public key.

* That the JWS protected header contains a `typ` header parameter with the
  value "signed-acme-challenge+jwt".

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

The Requestor MUST publish in its Entity Configuration an `acme_requestor`
metadata containing a JWK set, according to {{Section 5.2.1 of
OPENID-FED}}{: relative="#section-5.2.1"}. The keys in the set are used to
respond to ACME challenges.

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

Before using a discovered Issuer, the Requestor MUST obtain a Trust Chain from
the Issuer to a Trust Anchor that the Requestor is configured to trust, and MUST
verify that the Issuer's Resolved Metadata contains `acme_issuer` metadata.
Requestors MUST use the ACME Directory URL from that Resolved Metadata for
client configuration of ACME endpoints.

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

# Certificate Lifecycle {#certificate-lifecycle}

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

The extensions in this document build upon the Security Considerations and
threat model of ACME ({{Section 10 of !RFC8555}}) and of OpenID Federation 1.0
({{Section 18 of OPENID-FED}}{: relative="#section-18"}). This section describes
the additional considerations, deployment requirements, and composition risks that
arise from using OpenID Federation 1.0 as the identifier validation method for
ACME.

## Trust Model

The `openid-federation-01` challenge proves two properties at the time of
validation: that the Requestor controls a private key attested in its
`acme_requestor` metadata, and that a Trust Chain binds that Requestor to a
Trust Anchor the Issuer is configured to trust.

This is a different control demonstration than `http-01`, `dns-01`, or
`tls-alpn-01`. Those methods prove control of a DNS name or IP address that
typically appears in the issued certificate. This method proves membership in a
federation and control of a federation-attested key. It does not, by itself,
prove control of any DNS name, IP address, or other identifier that a
Certification Authority might place in an X.509 certificate.

Issuers MUST be configured with an explicit set of Trust Anchors, and MUST
reject any Trust Chain that does not terminate at one of those Trust Anchors.
Each such Trust Anchor MUST be administered under a federation policy that is
appropriate for the certificates the Issuer will produce, including the
certificate profile, the population of entities that may obtain certificates,
and any additional authorization criteria (for example, Trust Marks or
federation policy constraints on `acme_requestor` metadata). A Trust Anchor
that is acceptable for other OpenID Federation applications is not automatically
acceptable for X.509 issuance.

Requestors that discover Issuers through the federation MUST likewise be
configured with an explicit set of Trust Anchors, and MUST authenticate a
candidate Issuer as a federation member before using its ACME Directory
({{issuer-metadata}}). Automatic discovery without a configured Trust Anchor
allows a Requestor to enroll with an attacker-controlled ACME server.

OpenID Federation Entity Statements are signed JWTs. Their integrity and the
resulting Trust Chain can be evaluated independently of the TLS channel used to
transport them ({{OPENID-FED}}). Implementations MUST treat Trust Chain
validation, not Web PKI validation of the HTTPS server that hosts an Entity
Configuration, as the source of trust for this challenge. TLS still provides
confidentiality and server authentication for ACME and for Federation Entity
Discovery; it does not substitute for federation trust evaluation.

## Integrity of Authorizations

ACME requires that a validation response be bound to the ACME account key, so
that a man-in-the-middle on the ACME channel cannot obtain an authorization for
its own account by replaying a legitimate client's response
({{Section 10.2 of !RFC8555}}). The `openid-federation-01` challenge meets this
requirement by signing the ACME key authorization, which includes the token and
the account key thumbprint, rather than signing the token alone.

That binding is especially important for this challenge type. Unlike `http-01`,
`dns-01`, or `tls-alpn-01`, the signature that demonstrates control is sent on
the ACME channel rather than observed on a separate validation channel. A
Requestor that discovers Issuers automatically may be willing to complete a
challenge presented by an ACME server it has not preconfigured. Signing the key
authorization prevents such a server from relaying a token issued by a different
ACME server and reusing the Requestor's signature to obtain a certificate under
an account the attacker controls.

The `openid-federation` identifier type MUST NOT be validated except by
`openid-federation-01`, and `openid-federation-01` MUST NOT be used to validate
any other identifier type ({{identifier-type}}, {{challenge-type}}). Hosting an
HTTP resource at an Entity Identifier URL, or completing a DNS challenge for a
name derived from that URL, does not demonstrate federation membership. Completing
`openid-federation-01` does not demonstrate control of a DNS name.

The Issuer MUST take `acme_requestor` keys from the Requestor's Resolved
Metadata after Trust Chain evaluation, not from an Entity Configuration that has
not been bound to a trusted Trust Anchor. Federation policy along the Trust
Chain can constrain or replace metadata. If the Requestor and the Issuer would
derive different Resolved Metadata because they use different Trust Anchors or
different policy, the Issuer's view is authoritative for issuance: the Issuer
MUST NOT accept keys or other metadata from a Trust Chain that it does not
itself trust. A mismatch that causes issuance to fail is preferable to allowing
the Requestor's choice of Trust Anchor to weaken the Issuer's policy.

## Trust Chain Handling

A `trustChain` supplied in the challenge response is a hint that saves the
Issuer from performing Federation Entity Discovery. It is not a substitute for
validation. The Issuer MUST verify every Entity Statement in a client-supplied
Trust Chain and MUST confirm that the chain terminates at a Trust Anchor the
Issuer is configured to trust, exactly as it would for a chain the Issuer
discovered itself.

The `trustAnchors` field in the challenge object tells the Requestor which Trust
Anchors the Issuer is prepared to accept. It is an optimization for the
Requestor. The Requestor MUST NOT treat those values as Trust Anchors for its
own discovery or authentication of the Issuer. The Issuer MUST reject a
client-supplied chain that ends at a Trust Anchor it did not configure.

A Trust Chain MUST NOT be relied upon past its expiration
({{Section 10.4 of OPENID-FED}}{: relative="#section-10.4"}).
{{certificate-lifecycle}} requires that the `notBefore` and `notAfter` values of
issued certificates fall before that expiration. Separately, Issuers that reuse
ACME authorizations for later orders ({{Section 7.1.3 of !RFC8555}}) MUST NOT
treat an authorization as valid after the Trust Chain that justified it has
expired. Reusing a stale authorization would allow issuance without a current
attestation of federation membership.

## Certification Authority Policy

This document does not define an X.509 certificate profile. Fields of issued
certificates, including Common Name, Subject Alternative Names, Key Usage, and
validity period, are determined by Certification Authority policy and by the
systems those certificates must interoperate with.

Because `openid-federation-01` validates only an `openid-federation` identifier,
an Issuer that copies other names from a CSR, from federation metadata, or from
local policy into the certificate is asserting identifier bindings that this
challenge does not prove. Issuers MUST NOT include a DNS name, IP address,
email address, or other identifier in an issued certificate unless that
identifier has been validated by an appropriate ACME challenge or is justified
by the Issuer's certificate policy and the federation's trust framework. This
challenge type is intended as a bridge into existing, often closed, PKIs. It is
not a substitute for domain control validation in a publicly trusted Web PKI.

If the Issuer includes an `id-on-OpenIdFederationEntityId` Subject Alternative
Name ({{openidfed-othername-id}}), the value MUST be the Entity Identifier that
was validated. Including a different Entity Identifier would bind the
certificate to an entity whose control was not demonstrated.

Successful validation attests federation membership and key control at
validation time. It does not, by itself, track later changes in membership. The
constraint in {{certificate-lifecycle}} limits certificate validity so that it
does not extend beyond the Trust Chain observed at issuance, but it does not
revoke a certificate if the Requestor later leaves the federation or if an
attested key is compromised. Deployments that need issued certificates to follow
federation membership after issuance MUST use the certificate lifecycle
mechanisms of their X.509 ecosystem, such as CRLs, OCSP, or short-lived
certificates, according to the Issuer's certificate policy.

Publishing issued certificates in the `x5c` parameter of keys in a Subordinate
Statement ({{publish-cert}}) is a federation policy decision. Relying parties
that consume `x5c` in that context MUST still evaluate those certificates
according to the applicable PKI and federation policy. A stale `x5c` member can
advertise a certificate that has since expired or been revoked.

## Issuer Discovery

The `directory_url` in `acme_issuer` metadata is reached only after the
Requestor has authenticated the Issuer as a federation member. Using a
`directory_url` taken from an untrusted Entity Configuration, from search, or
from user input without Trust Chain evaluation exposes the Requestor to an
attacker-controlled ACME server. The key-authorization binding described above
limits the harm of completing a challenge at such a server; it does not make
that server safe to use for account registration, order creation, or certificate
download.

The Requestor MUST use the `directory_url` from the Issuer's Resolved Metadata,
so that federation policy along the Trust Chain can constrain which ACME server
the Requestor contacts.

## Key Management

This protocol involves three distinct key roles: the ACME account key
({{!RFC8555}}), the keys published in `acme_requestor` metadata and used to sign
challenge responses, and the subject keys in Certificate Signing Requests and
issued certificates. These keys SHOULD be distinct. In particular, the
cryptographic keys in the `acme_requestor` metadata SHOULD NOT be reused for any
purpose other than signing `openid-federation-01` challenge responses, including
as Federation Entity Keys, as ACME account keys, or as the subject key of an
issued certificate.

Reusing a key across protocols requires a cross-protocol analysis. Domain
separation for the challenge response depends on the JWS `typ` value
"signed-acme-challenge+jwt"; verifiers MUST reject responses that omit this
`typ` or use a different value ({{?RFC8725}}). Distinct keys remain the most
reliable defense.

Compromise of an `acme_requestor` private key allows an attacker to complete
`openid-federation-01` challenges for that Entity Identifier until the key is
removed from the Requestor's Resolved Metadata and any authorizations granted
under it have expired. Requestors MUST protect these private keys with at least
the same care as ACME account keys, and SHOULD rotate them periodically. After
a challenge has been completed, the Requestor MAY remove `acme_requestor`
metadata from its Entity Configuration ({{requestor-metadata}}), which limits
the time during which the corresponding public keys are advertised.

## Denial-of-Service Considerations

If the Requestor omits `trustChain`, the Issuer performs Federation Entity
Discovery, which can cause the Issuer to make multiple outbound requests
({{Section 18.1 of OPENID-FED}}{: relative="#section-18.1"}). ACME account
credentials are required to submit a challenge response, but account creation
is often open. Issuers SHOULD prefer client-supplied Trust Chains, SHOULD bound
the time, depth, and number of `authority_hints` they are willing to follow, and
SHOULD apply the same denial-of-service controls they apply to other outbound
validation methods in {{!RFC8555}}.

The `token` value in the challenge object MUST have at least 128 bits of
entropy, as specified in {{challenge-type}}, so that challenge responses cannot
be predicted or reused across challenges.

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
