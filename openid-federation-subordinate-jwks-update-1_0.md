%%%
title = "OpenID Federation Subordinate JWKS Update 1.0 - draft 00"
abbrev = "openid-federation-subordinate-jwks-update"
ipr = "none"
workgroup = "OpenID Connect Working Group"
keyword = ["security", "openid", "federation", "jwks", "key-rotation"]

[seriesInfo]
name = "Internet-Draft"
value = "openid-federation-subordinate-jwks-update"
status = "standard"

[[author]]
initials="G."
surname="Zachmann"
fullname="Gabriel Zachmann"
organization="Karlsruhe Institute of Technology"
    [author.address]
    email = "gabriel.zachmann@kit.edu"
%%%

.# Abstract

This specification is an extension to
[@!OpenID.Federation]. It defines two optional federation endpoints — the
Subordinate JWKS Update Trigger Endpoint and the
Subordinate JWKS Update Endpoint — with which a Subordinate Entity keeps the
JWK Set published about it in Subordinate Statements up to date at its Superior
Entities. The two endpoints complement each other: the Subordinate JWKS Update Trigger
Endpoint informs a Superior that the Subordinate's Entity Configuration should
be re-fetched, and the Subordinate JWKS Update Endpoint lets a Subordinate push its new
federation keys to a Superior directly as a signed JWK Set.

{mainmatter}

# Introduction

OpenID Federation 1.1 [@!OpenID.Federation] specifies that a Superior Entity
(Trust Anchor or Intermediate) issues a Subordinate Statement for each of its
Subordinate Entities and that this statement carries the Subordinate's
federation keys in the `jwks` Claim. Trust Chains are resolved based
on these statements.

When a Subordinate Entity rotates its federation keys, this must be reflected
in the `jwks` Claim of Subordinate Statements issued by the Superior Entities.
If the Superior continues to issue statements including the old keys, the Subordinate's Entity Configuration
can no longer be verified and therefore also Trust Chains involving that Entity break.

To assure that a Superior always knows about the current JWK Set of its
Subordinates, it may poll the
Entity Configurations of its Subordinates without any involvement of
the Subordinate: it periodically fetches the Entity Configuration of a
Subordinate from the location defined in [@!OpenID.Federation] (typically
`https://<entity_id>/.well-known/openid-federation`), compares the keys in the
`jwks` Claim with its Stored JWKS, and updates the Stored JWKS when the keys
have changed. This approach requires no support from the Subordinate and picks
up key changes even when the Subordinate cannot reach its Superior.
The details of a periodic polling approach, in particular the polling interval,
are out of scope of this specification.

This specification defines two complementary mechanisms as an alternative to
periodic polling by the Superior. With these mechanisms a Subordinate
can keep the keys that its Superiors store about it up to date:

- The **Subordinate JWKS Update Trigger Endpoint**
  (`federation_subordinate_jwks_update_trigger_endpoint`) lets a Subordinate signal to a
  Superior that the Superior should re-fetch the Subordinate's Entity
  Configuration and adopt the current keys.
- The **Subordinate JWKS Update Endpoint** (`federation_subordinate_jwks_update_endpoint`) lets a
  Subordinate submit its new federation keys directly to a Superior as a signed
  JWK Set, without requiring the Superior to fetch anything.

## Requirements Notation and Conventions

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT
RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP
14 [@!RFC2119] [@!RFC8174] when, and only when, they appear in all capitals, as shown here.

# Terminology

This specification uses the terms "Entity", "Entity Configuration", "Federation
Entity", "Federation Entity Key", "Subordinate Entity", "Superior Entity",
"Trust Anchor", "Intermediate", "Subordinate Statement", and "Trust Chain" as
defined in [@!OpenID.Federation]. It uses the terms "JSON Web Token" (JWT),
"JWT Claims Set", and "Claim" as defined in [@!RFC7519], "JSON Web Signature"
(JWS) as defined in [@!RFC7515], the term "JSON Web Key Set" [@!RFC7517]
(referred to interchangeably as "JWK Set" and "JWKS" in this document; "JWKS"
is also used in the names of the endpoints and of the "Stored JWKS"), and the
notion of client authentication of the OAuth 2.0 framework [@!RFC6749], on
which the entity authentication of OpenID Federation builds.

The term **Stored JWKS** refers to the copy of a Subordinate's federation keys
that a Superior holds in order to include them in the Subordinate Statements it
issues. A key in the Stored JWKS is **expired** if its `exp` member is present
and its value is strictly in the past of the point in time at which the key is
evaluated. A key without an `exp` member is
never considered expired on the basis of the `exp` member.

# Key Rotation Procedure

A Superior keeps, for each of its Subordinate Entities, a copy of the last set
of federation keys it has adopted — the **Stored JWKS** — and uses it for two
purposes: to publish the Subordinate's keys in Subordinate Statements, and to
verify documents presented by the Subordinate, in particular the Subordinate's
Entity Configuration and the signed JWK Sets submitted to the Subordinate JWKS
Update Endpoint.

An expired or otherwise not valid key MUST NOT be used to verify a document
presented by a Subordinate. The mechanisms of this specification therefore
cannot be used to rotate keys with an already-expired, revoked or otherwise not
valid key: a Subordinate that rotates its keys MUST present its new key (or its
new Entity Configuration) in a document that is signed with a Stored JWKS key
that is **not expired**, and it MUST do so before the signing key expires. If
there is no valid key in the Stored JWKS, the new JWKS MUST be communicated in
another way that is out of scope of this specification.

The following normative procedure MUST be followed, in the order given, by a
Subordinate that rotates its federation keys with the mechanisms of this
specification, and by its Superior. The
procedure MUST be completed while the old key is still valid; the validity
periods of the documents involved MUST additionally be taken into account, see
[Validity Periods](#validity-periods).

1. **Generate the new key(s).** The Subordinate MUST generate a new federation
   key (or key set) and assign each new key a unique `kid` that has not been
   used by this Subordinate before. The new key(s) MUST NOT be used for signing
   before step 6.

2. **Publish the new key(s) alongside the old key(s).** The Subordinate MUST
   publish an Entity Configuration whose `jwks` Claim contains both the new
   key(s) and the old key(s) that are still valid, and that is signed with a
   key the Superior currently accepts, i.e., a key from the Stored JWKS that is
   not expired (the old key). The Subordinate MUST continue to sign its Entity
   Configuration with the old key at least until the release time `T` defined
   in [Validity Periods](#validity-periods), so that cached copies of the
   Entity Configuration that do not contain the new key(s) remain verifiable.

3. **Notify the Superior.** The Subordinate MUST notify the Superior of the new
   key(s) by one of the following means:
   - The Subordinate creates a signed JWK Set containing all new and old
     key(s). This JWK Set SHOULD be the same as the one published in the Entity
     Configuration (step 2). The JWK Set MUST be signed with an old key. The
     Subordinate sends the JWK Set to the Subordinate JWKS Update Endpoint; or
   - the Subordinate sends a request to the Subordinate JWKS Update Trigger
     Endpoint, which causes the Superior to re-fetch the Entity Configuration
     of step 2; or
   - if the Superior supports neither endpoint, the Subordinate waits for the
     Superior's next periodic poll (see [Introduction](#introduction)).

4. **The Superior adopts the new key(s).** The Superior MUST validate the
   signed JWK Set or the re-fetched Entity Configuration as required by
   [Processing](#update-processing) and [Validation Requirements](#validation-requirements) and, on success, MUST include the
   new key(s) in its Stored JWKS. Subordinate Statements issued by the Superior
   after the adoption MUST contain the new key(s). The Subordinate Statements
   still MUST contain the old key(s) until the Subordinate has switched to the
   new signing key (step 6).

5. **Confirm adoption.** The Subordinate MUST consider the new key(s) adopted
   only after it has received `204 No Content` from the Subordinate JWKS Update
   Endpoint, after the response of the Subordinate JWKS Update Trigger Endpoint
   indicated that the Stored JWKS has been updated, or after it has observed
   the new key(s) in a Subordinate Statement issued by the Superior. After the
   adoption has been confirmed, the Subordinate MUST NOT yet switch to the new
   signing key (step 6): it MUST wait until the release time `T` defined in
   [Validity Periods](#validity-periods), so that other entities no longer rely
   on cached copies of the Entity Configuration or of the Subordinate
   Statements that do not contain the new key(s).

6. **Switch signature.** At the release time `T` (see [Validity
   Periods](#validity-periods)), the Subordinate MUST sign all subsequent
   documents — in particular its Entity Configuration and any further signed
   JWK Sets — with the new key(s).

7. **Phase out the old key(s).** The Subordinate MAY keep the old key(s) in its
   Entity Configuration for a transition period, but MUST remove them from the
   Entity Configuration once they have expired.

8. **Preserve verifiability of old material.** For documents that were signed
   with the old key(s) — for example Entity Configurations or Subordinate
   Statements predating the rotation — to remain verifiable, the Subordinate
   MAY publish the old key(s) at its historical keys endpoint as defined in
   [@!OpenID.Federation].

9. **The Superior phases out the old key(s).** The Superior MUST NOT include
   the old key(s) in Subordinate Statements once they have expired.

A Superior MAY retain expired keys in its Stored JWKS (for example for
historical purposes) but MUST NOT use them for the verification
of documents presented by the Subordinate, and MUST NOT include them in
Subordinate Statements, whose `jwks` Claim MUST contain only keys that are not
expired at the time the statement is issued.

## Validity Periods {#validity-periods}

**Release time.** Other entities may hold copies of the documents involved
that predate the rotation: an Entity Configuration that was fetched before the
new key(s) were published, and Subordinate Statements that were obtained before
the Superior included the new key(s). Such copies remain in circulation for
the validity period of the respective document. The Subordinate
MUST continue to sign its Entity Configuration with the old key (step 2) and
MUST NOT switch to the new signing key (step 6) until those copies have expired
and Entities had the opportunity to fetch the new keys.
The Superior MUST continue to include the old key(s) in Subordinate Statements
(step 4) until the Subordinate has switched to the new signing key (step 6).

The Subordinate MUST therefore keep the old key usable until the **release
time** `T`:

    T = max( t_ec + L_ec ,  t_stmt + L_stmt )

where

- `t_ec` is the time at which the Subordinate first issued an Entity
  Configuration that includes the new key(s) (step 2), and `L_ec` is the
  validity period of Entity Configurations issued by the Subordinate;
- `t_stmt` is the time at which the Superior first issued a Subordinate
  Statement that includes the new key(s) (step 4), and `L_stmt` is the
  validity period of Subordinate Statements issued by the Superior for this
  Subordinate.

The Subordinate MUST continue to sign its Entity Configuration with the old
key (step 2) and MUST NOT switch to signing with the new key (step 6) before
the release time `T`.

If the lifetime of Entity Configurations issued by the Subordinate is not
static the Subordinate still MUST ensure that all Entity Configurations issued
before the new key(s) were published are expired before switching to the new
signing key. The same applies to the lifetime of the Subordinate Statements
issued by the Superior: if the lifetime of Subordinate Statements issued by the Superior is not
static the Subordinate still MUST ensure that all Subordinate Statements issued
before the new key(s) were included are expired before switching to the new
signing key. 

For periodic, scheduled rotation it is RECOMMENDED that the keys a Subordinate
publishes carry `iat`, `nbf`, and `exp` members (as defined in [@!RFC7519]).
The `exp` member determines whether a key is still valid and enables the
Superior to exclude expired keys when issuing Subordinate Statements (step 9).
A key MUST NOT be used after its `exp` and MUST NOT be used before its `nbf`.

# Entity Metadata

A Superior publishes the endpoints defined by this specification in the
`federation_entity` metadata of its Entity Configuration, using the
`federation_subordinate_jwks_update_trigger_endpoint` and `federation_subordinate_jwks_update_endpoint`
members. The endpoints MUST use the `https` scheme and MAY include port, path, and query parameter components encoded in `application/x-www-form-urlencoded` format. They MUST NOT contain a fragment component.

The following is a non-normative example of an Entity Configuration payload
for a Superior that advertises both endpoints:

```json
{
  "iss": "https://ta.example.org",
  "sub": "https://ta.example.org",
  "iat": 1590000000,
  "exp": 1590086400,
  "jwks": {
    "keys": [
      {
        "kty": "RSA",
        "kid": "key1",
        "use": "sig",
        "n": "n4EPtAOCc9AlkeQHPzHStgAbgs7bTZLwUBZdR8_KuKPEHLd4rHVTeT",
        "e": "AQAB"
      }
    ]
  },
  "metadata": {
    "federation_entity": {
      "federation_fetch_endpoint": "https://ta.example.org/fetch",
      "federation_subordinate_jwks_update_trigger_endpoint": "https://ta.example.org/jwks/update-trigger",
      "federation_subordinate_jwks_update_endpoint": "https://ta.example.org/jwks/update"
    }
  }
}
```

# Subordinate JWKS Update Trigger Endpoint {#trigger-endpoint}

The Subordinate JWKS Update Trigger Endpoint is an optional endpoint published by a Superior
Entity. It allows a Subordinate Entity to request that the Superior re-fetch the
Subordinate's Entity Configuration from the location defined in
[@!OpenID.Federation] (typically `https://<entity_id>/.well-known/openid-federation`)
and update the Stored JWKS for that Subordinate when the keys have changed.

## Request

The request MUST be an HTTP `POST`. The target Subordinate is identified either
by client authentication or by a `sub` parameter, as follows.

### Client Authentication

The endpoint MAY require client authentication. When client authentication is
required, the Subordinate MUST authenticate with the `private_key_jwt` client
authentication method using its federation keys, as defined in
[@!OpenID.Federation]. The authenticated Subordinate is the target of the
request; any `sub` parameter in the request body is ignored.

### Without Client Authentication

When the endpoint does not require client authentication, the target
Subordinate is identified by the `sub` parameter, whose value MUST be the
Subordinate's Entity Identifier. The request body is encoded in
`application/x-www-form-urlencoded` format.

The following is a non-normative example of a request:

```
POST /jwks/update-trigger HTTP/1.1
Host: ta.example.org
Content-Type: application/x-www-form-urlencoded

sub=https%3A%2F%2Fsub.example.org
```

## Processing

Upon receiving a request, the Superior:

1. determines the target Subordinate, either from the authenticated client
   entity or from the `sub` parameter;
2. verifies that the target is a known Subordinate and is eligible for updates;
3. fetches the Entity Configuration of the target from the location defined in
   [@!OpenID.Federation];
4. validates the fetched Entity Configuration (see [Validation
   Requirements](#validation-requirements));
5. compares the keys in the validated Entity Configuration with the Stored
   JWKS; if they differ, updates the Stored JWKS to the JWK Set in the `jwks`
   Claim of the Subordinate's Entity Configuration and uses the updated Stored
   JWKS in subsequent Subordinate Statements issued by the Superior;
6. responds to the request.

## Validation Requirements {#validation-requirements}

For the Subordinate JWKS Update Trigger Endpoint, the fetched Entity Configuration is valid
if and only if all of the following hold:

0. The Entity Configuration MUST be a signed JWT.
0. The Entity Configuration MUST have a `typ` header parameter with the value `entity-statement+jwt`.
0. The Entity Configuration MUST have an `alg` (algorithm) header parameter with a value that is an acceptable JWS signing algorithm; it MUST NOT be `none`.
0. The Entity Identifier of the Subordinate MUST match the value of the `sub` (subject) Claim.
0. The Entity Identifier of the Subordinate MUST match the value of the `iss` (issuer) Claim.
0. The current time MUST be after the time represented by the `iat` (issued at) Claim (possibly allowing for some small leeway to account for clock skew).
0. The current time MUST be before the time represented by the `exp` (expiration) Claim (possibly allowing for some small leeway to account for clock skew).
0. The `jwks` (JWK Set) Claim MUST be present, with a value that is a valid JWK Set [@!RFC7517].
0. The Entity Configuration's kid (Key ID) header parameter value MUST be a non-zero length string and MUST exactly match the kid value for a key in the Stored JWKS for the Subordinate, using only keys that are **not expired**.
0. The Entity Configuration's signature MUST validate using the key from the Stored JWKS identified by the kid value.

If any of these conditions does not hold, the Superior MUST NOT update the
Stored JWKS.

## Response

### Successful Response

A successful response MUST use the HTTP status code 200 and the content type `application/json`.

The response is a JSON object with the following Claims:

- **sub**: REQUIRED. The Entity Identifier of the target Subordinate, i.e., the
  Subordinate whose Stored JWKS was evaluated. Its value MUST equal the Entity
  Identifier determined during request processing.
- **changed**: REQUIRED. A boolean indicating whether the Superior replaced its
  Stored JWKS for the Subordinate. `true` means the keys in the validated
  Entity Configuration differed from the Stored JWKS and the Stored JWKS was
  replaced; `false` means the keys are identical and the Stored JWKS was left
  unchanged.

The following is a non-normative example of a successful response:

```json
{
  "sub": "https://sub.example.org",
  "changed": true
}
```

### Error Response {#trigger-error-response}

On failure, the Superior responds with an HTTP error status code and an error
response in the format defined in [Error Response Format](#error-response-format). The
following error codes are used by this endpoint:

- **invalid_request**: The request is malformed, for example the `sub`
  parameter is missing or the request body cannot be parsed. The HTTP response
  status code SHOULD be 400 (Bad Request).
- **invalid_client**: The fetched Entity Configuration could not be
  authenticated against the Stored JWKS, either because a signature
  verification failed or because the Stored JWKS contains no key that is not
  expired. The HTTP response status code SHOULD be 401 (Unauthorized).
- **not_found**: The target Subordinate is not known to the Superior or is not
  eligible for updates. The HTTP response status code SHOULD be 404 (Not
  Found).
- **server_error**: The Entity Configuration could not be fetched from the
  Subordinate, or the fetched Entity Configuration is invalid (for example, it
  is expired, has an `iss`/`sub` mismatch, or is malformed). The HTTP response
  status code SHOULD be 502 (Bad Gateway).

# Subordinate JWKS Update Endpoint {#update-endpoint}

The Subordinate JWKS Update Endpoint is an optional endpoint published by a Superior Entity.
It allows a Subordinate Entity to push its new federation keys to the Superior
directly as a signed JWK Set, replacing the Stored JWKS for that Subordinate.

## Request {#update-endpoint-request}

The request MUST be an HTTP `POST` with content type `application/jwk-set+jwt`.
The request body is a signed JSON Web Token in compact serialization, reusing
the signed JWK Set format defined for the `signed_jwks_uri` in
[@!OpenID.Federation]. The JWT MUST have a JOSE header with a `typ` value of
`jwk-set+jwt` and a `kid` identifying the signing key.

The JWT payload contains the following Claims:

- **keys**: REQUIRED. A JSON array of JWK values — the Subordinate's new
  federation key set. Each key MUST have a unique, non-empty `kid`.
- **iss**: REQUIRED. The issuer — the Subordinate's Entity Identifier.
- **sub**: REQUIRED. The owner of the keys — the Subordinate's Entity
  Identifier, equal to `iss`.
- **iat**: OPTIONAL. The time at which the signed JWK Set was issued. It is
  evaluated only at acceptance time by the Subordinate JWKS Update Endpoint
  (the Superior MUST NOT accept a signed JWK Set whose `iat` is in the future)
  and carries no meaning beyond that instant.
- **exp**: OPTIONAL. The time at which the signed JWK Set expires. It is
  evaluated only at acceptance time by the Subordinate JWKS Update Endpoint
  (the Superior MUST NOT accept a signed JWK Set whose `exp` is in the past)
  and carries no meaning beyond that instant; in particular, it is NOT the
  expiration time of the keys in the `keys` Claim, which is governed by the
  `exp` member of each individual JWK.

The Subordinate JWKS Update Endpoint does not use client authentication for requests. Instead, the
signature on the signed JWK Set is the authentication mechanism: the Superior
only accepts the request if the signature verifies against the Subordinate's
Stored JWKS. This is described in [Processing](#update-processing).

## Processing {#update-processing}

Upon receiving a request, the Superior:

1. verifies the `Content-Type` header and that the body parses as a signed JWK
   Set (with the correct `typ` header);
2. reads the `sub` Claim to determine the target Subordinate and verifies it
   is a known and eligible Subordinate;
3. verifies that `iss` equals `sub`;
4. verifies the signature of the signed JWK Set against the Stored JWKS of the
   Subordinate, using only keys that are **not expired**;
5. if the signed JWK Set carries an `exp` Claim, verifies that it is not in
   the past;
6. if the signed JWK Set carries an `iat` Claim, verifies that it is not in
   the future;
7. either replaces the Stored JWKS with the `keys` from the signed JWK Set,
   or, if the Superior supports deferred updates, holds the signed JWK Set
   pending manual approval without replacing the Stored JWKS. Which of the two
   the Superior performs determines the response, see
   [Response](#update-endpoint-response).

A signed JWK Set that is held pending approval MUST NOT be used to replace the
Stored JWKS, MUST NOT be used to verify documents presented by the Subordinate,
and MUST NOT be included in Subordinate Statements until it has been approved.
The approval process itself, including how the approver is notified and how
approval is granted or denied, is out of scope of this specification. A
Superior MAY discard a signed JWK Set that is pending approval at any time;
whether a pending signed JWK Set eventually expires is a matter for the
approval process.

## Response {#update-endpoint-response}

### Successful Response

On success, the Superior responds either with HTTP status `204 No Content`
when it has replaced the Stored JWKS, or with `202 Accepted` when it has
accepted the signed JWK Set but the update awaits manual approval and the
Stored JWKS has not been replaced. In both cases the response MUST NOT contain
a body.

A Subordinate that receives `202 Accepted` MUST NOT yet rely on the new keys
being in effect: the Superior will continue to verify documents and issue
Subordinate Statements with the previous keys until the update has been
approved. The Subordinate MUST keep using its previous federation key for
signing until it observes the new keys in a Subordinate Statement issued by
the Superior.

### Error Response {#update-error-response}

On failure, the Superior responds with an HTTP error status code and an error
response in the format defined in [Error Response Format](#error-response-format). The
following error codes are used by this endpoint:

- **invalid_request**: The request body is not a well-formed signed JWK Set,
  the `iss` and `sub` Claims differ, a `kid` is missing or duplicated, or a
  REQUIRED Claim is absent. The HTTP response status code SHOULD be 400 (Bad
  Request).
- **invalid_client**: The signature could not be verified against the
  Subordinate's Stored JWKS, including the case where every stored key is
  expired. The HTTP response status code SHOULD be 401 (Unauthorized).
- **not_found**: The target Subordinate is not known to the Superior or is not
  eligible for updates. The HTTP response status code SHOULD be 404 (Not
  Found).
- **unsupported_media_type**: The `Content-Type` of the request is not
  `application/jwk-set+jwt`. The HTTP response status code SHOULD be 415
  (Unsupported Media Type).

# Error Response Format {#error-response-format}

If a request cannot be processed, the response body MUST be a JSON object with
the content type `application/json`. In compliance with [@!RFC6749] and
[@!OpenID.Federation], the following standardized error format MUST be used:

- **error**: (REQUIRED) A single ASCII error code. Error codes in the IANA
  "OAuth Extensions Error Registry" [@!IANA.OAuth.Parameters] MAY be used; in
  addition, this specification uses the error codes defined for the respective
  endpoints below.
- **error_description**: (REQUIRED) Human-readable text providing additional
  information used to assist the developer in understanding the error that
  occurred.

The following is a non-normative example of an error response:

```http
400 Bad Request
Content-Type: application/json

{
  "error": "invalid_request",
  "error_description": "The sub parameter is required but missing."
}
```

The error codes used by the Subordinate JWKS Update Trigger Endpoint are
defined in [Trigger Endpoint Error Response](#trigger-error-response), and
those used by the Subordinate JWKS Update Endpoint in [Update Endpoint Error
Response](#update-error-response).


# Security Considerations

The security considerations of OpenID Federation 1.1 [@!OpenID.Federation]
apply. This specification adds the following considerations.

## Expired Material MUST NOT Be Accepted

Both endpoints MUST reject requests that cannot be authenticated with a
not expired key, and the Subordinate JWKS Update Trigger Endpoint MUST reject an expired or
otherwise invalid Entity Configuration. Accepting expired material would allow
a Subordinate (or an attacker with a compromised expired key) to continue being
authenticated after the validity of its key has lapsed. A Subordinate whose
keys have all expired cannot use this specification to rotate its keys.


{backmatter}

<reference anchor="RFC2119" target="https://www.rfc-editor.org/info/rfc2119">
  <front>
    <title>Key words for use in RFCs to Indicate Requirement Levels</title>
    <author initials="S." surname="Bradner" fullname="Scott Bradner">
      <organization>Harvard University</organization>
    </author>
    <date year="1997" month="March"/>
  </front>
</reference>

<reference anchor="RFC6749" target="https://www.rfc-editor.org/info/rfc6749">
  <front>
    <title>The OAuth 2.0 Authorization Framework</title>
    <author initials="D." surname="Hardt" fullname="Dick Hardt">
      <organization>Microsoft</organization>
    </author>
    <date year="2012" month="October"/>
  </front>
</reference>

<reference anchor="RFC7515" target="https://www.rfc-editor.org/info/rfc7515">
  <front>
    <title>JSON Web Signature (JWS)</title>
    <author initials="M." surname="Jones" fullname="Michael B. Jones">
      <organization>Microsoft</organization>
    </author>
    <author initials="J." surname="Bradley" fullname="John Bradley">
      <organization>Ping Identity</organization>
    </author>
    <author initials="N." surname="Sakimura" fullname="Nat Sakimura">
      <organization>NRI</organization>
    </author>
    <date year="2015" month="May"/>
  </front>
</reference>

<reference anchor="RFC7517" target="https://www.rfc-editor.org/info/rfc7517">
  <front>
    <title>JSON Web Key (JWK)</title>
    <author initials="M." surname="Jones" fullname="Michael B. Jones">
      <organization>Microsoft</organization>
    </author>
    <date year="2015" month="May"/>
  </front>
</reference>

<reference anchor="RFC7519" target="https://www.rfc-editor.org/info/rfc7519">
  <front>
    <title>JSON Web Token (JWT)</title>
    <author initials="M." surname="Jones" fullname="Michael B. Jones">
      <organization>Microsoft</organization>
    </author>
    <author initials="J." surname="Bradley" fullname="John Bradley">
      <organization>Ping Identity</organization>
    </author>
    <author initials="N." surname="Sakimura" fullname="Nat Sakimura">
      <organization>NRI</organization>
    </author>
    <date year="2015" month="May"/>
  </front>
</reference>

<reference anchor="RFC8174" target="https://www.rfc-editor.org/info/rfc8174">
  <front>
    <title>Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words</title>
    <author initials="B." surname="Leiba" fullname="Barry Leiba">
      <organization>Huawei Technologies</organization>
    </author>
    <date year="2017" month="May"/>
  </front>
</reference>

<reference anchor="OpenID.Federation" target="https://openid.net/specs/openid-federation-1_1.html">
    <front>
        <title>OpenID Federation 1.1</title>
        <author fullname="R. Hedberg, Ed.">
            <organization>independent</organization>
        </author>
        <author fullname="Michael B. Jones">
            <organization>Self-Issued Consulting</organization>
        </author>
        <author fullname="A. Solberg">
            <organization>Sikt</organization>
        </author>
        <author fullname="John Bradley">
            <organization>Yubico</organization>
        </author>
        <author fullname="Giuseppe De Marco">
            <organization>independent</organization>
        </author>
        <author fullname="Vladimir Dzhuvinov">
            <organization>Connect2id</organization>
        </author>
        <date day="5" month="May" year="2026"/>
    </front>
</reference>

<reference anchor="IANA.OAuth.Parameters" target="https://www.iana.org/assignments/oauth-parameters">
    <front>
        <title>OAuth Parameters</title>
        <author fullname="IANA">
        </author>
        <date day="25" month="March" year="2026"/>
    </front>
</reference>

# Notices

Copyright (c) 2026 The OpenID Foundation.

The OpenID Foundation (OIDF) grants to any Contributor, developer,
implementer, or other interested party a non-exclusive, royalty free,
worldwide copyright license to reproduce, prepare derivative works from,
distribute, perform and display, this Implementers Draft, Final
Specification, or Final Specification Incorporating Errata Corrections
solely for the purposes of (i) developing specifications,
and (ii) implementing Implementers Drafts, Final Specifications,
and Final Specification Incorporating Errata Corrections based
on such documents, provided that attribution be made to the OIDF as the
source of the material, but that such attribution does not indicate an
endorsement by the OIDF.

The technology described in this specification was made available
from contributions from various sources, including members of the OpenID
Foundation and others. Although the OpenID Foundation has taken steps to
help ensure that the technology is available for distribution, it takes
no position regarding the validity or scope of any intellectual property
or other rights that might be claimed to pertain to the implementation
or use of the technology described in this specification or the extent
to which any license under such rights might or might not be available;
neither does it represent that it has made any independent effort to
identify any such rights. The OpenID Foundation and the contributors to
this specification make no (and hereby expressly disclaim any)
warranties (express, implied, or otherwise), including implied
warranties of merchantability, non-infringement, fitness for a
particular purpose, or title, related to this specification, and the
entire risk as to implementing this specification is assumed by the
implementer. The OpenID Intellectual Property Rights policy
(found at openid.net) requires
contributors to offer a patent promise not to assert certain patent
claims against other contributors and against implementers.
OpenID invites any interested party to bring to its attention any
copyrights, patents, patent applications, or other proprietary rights
that may cover technology that may be required to practice this
specification.

# Acknowledgements

We would like to thank the following individuals for their contributions to this specification:
Niels van Dijk,
Harry Kodden,
Henri Mikkonen,
and Davide Vaghetti

# Document History

[[ To be removed from the final specification ]]

-00

* Initial version defining the Subordinate JWKS Update Trigger Endpoint and the
  Subordinate JWKS Update Endpoint for keeping a Superior's Stored JWKS for a
  Subordinate up to date.
