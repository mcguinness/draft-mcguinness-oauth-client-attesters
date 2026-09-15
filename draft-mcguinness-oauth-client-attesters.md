---
title: "OAuth 2.0 Client Attester Endorsement"
abbrev: "Client Attester Endorsement"
category: std
docname: draft-mcguinness-oauth-client-attesters-latest
submissiontype: IETF
stand_alone: yes
ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - OAuth
 - client attestation
 - client metadata
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-client-attesters"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-client-attesters/draft-mcguinness-oauth-client-attesters.html"
author:
 - fullname: Karl McGuinness
   organization: Independent
   email: public@karlmcguinness.com
normative:
  ATTEST: I-D.ietf-oauth-attestation-based-client-auth
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  RFC6749:
  RFC6454:
  RFC7515:
  RFC7517:
  RFC7519:
  RFC7591:
  RFC7662:
  RFC8725:
  RFC9111:
informative:
  RFC9449:
  SPIFFE-OAUTH: I-D.ietf-oauth-spiffe-client-auth
  INSTANCE-ID:
    title: "Client Instance Identification for Attestation-Based Client Authentication"
    target: https://mcguinness.github.io/draft-mcguinness-oauth-client-instance-id/draft-mcguinness-oauth-client-instance-id.html
    author:
      - fullname: Karl McGuinness
    date: 2026-09-14
--- abstract

This specification profiles OAuth 2.0 Attestation-Based Client
Authentication with client metadata identifying endorsed attesters and
their verification-key locations. It defines authorization-server
validation and withdrawal rules for endorsements in Client ID Metadata
Documents or registered client metadata. It introduces no new credential
or authentication method.

--- middle

# Introduction

A client identified by one metadata URL can have many installations,
each holding a different key. Attestation-Based Client Authentication
{{ATTEST}} allows an attester to authenticate those instances. ATTEST
requires an attestation to verify under the key of a known and trusted
Client Attester ({{ATTEST, Section 7.1}}) and leaves how that trust is
established to deployments ({{ATTEST, Section 10.8}}). Client ID Metadata
Documents {{CIMD}} publish client metadata but do not specify how the client
endorses an attester for this purpose.

This profile targets clients without a registration channel for configuring
attester trust at the authorization server (AS): primarily CIMD clients,
and registered clients whose registration cannot express multiple attesters
across platforms or during migration. Where operators can configure
attester trust per client, ATTEST's existing key-resolution options suffice
and this profile adds nothing.

This profile adds `client_attesters`: the client's endorsements of
attesters and their verification keys, generalizing SPIFFE client
authentication's bundle endpoint {{SPIFFE-OAUTH}}. An AS accepts an
endorsement only under its own trust policy. The resulting chain is:

~~~ ascii-art
Client metadata --endorses--> Attester --attests--> Client Instance
       |                         |                       |
       +----- AS accepts endorsement and validates proof-+
~~~

The profile applies at AS endpoints accepting Client Attestations for
client authentication or as an additional security signal, using the
profiling hook in {{ATTEST, Section 13}}. It retains ATTEST's wire
format, proof methods, and token binding. Stable instance identification
is defined separately in {{INSTANCE-ID}}; neither endorsement nor
instance identification establishes user delegation.

Resource servers validating Client Attestations directly rely on
configured attester trust; this profile does not define endorsement
discovery or acceptance for those endpoints.

# Conventions and Trust Model {#trust}

{::boilerplate bcp14-tagged}

OAuth terms follow {{RFC6749}} and Client Attester, Client Attestation,
and Client Instance Key follow {{ATTEST}}. The client publisher controls
the authoritative metadata for a `client_id`; an endorsement authorizes
an attester to issue Client Attestations naming that client.

## Approval Policy

The AS MUST configure whether this profile applies and the basis for
approving endorsements:

* **Publisher approval:** the AS authorizes a publisher to select
  attesters for specified clients. For CIMD, configure exact client URLs
  or HTTPS origins, optionally restricted to path segments. Successful
  metadata retrieval does not establish publisher approval. Shared
  hosting requires a boundary that excludes other publishers.
* **Attester approval:** the AS independently trusts a particular
  attester and configures its key source. The endorsement authorizes
  that attester to act for the client; it cannot supply the trust anchor.

For requests governed by this profile, both a current client endorsement
and AS policy approval are REQUIRED. AS policy can narrow the endorsed
set; it MUST NOT add an unendorsed attester or fall back to another trust
mechanism. An endorsement MUST NOT by itself establish that the client
is trusted or authorized to access a resource.

When combined with {{INSTANCE-ID}}, the same two requirements establish
attester authority; instance continuity remains independent. Other ATTEST
deployments can use configured trust without this profile.

## Registered Endorsements

Registered endorsements MUST originate from a party authenticated and
authorized to set them for that client, or be covered by a validated
software statement from an issuer approved for that purpose under
{{RFC7591}}. Open registration alone supplies neither assurance; issuing
a client credential does not retroactively approve its endorsements.
The same restriction applies to endorsement updates.

## Profile Selection

Profile applicability is an AS policy decision, not selected by the
presence or absence of `client_attesters`. ATTEST authentication-method
discovery does not signal this trust policy; deployments relying on
client endorsement enforcement establish that the AS applies this
profile through their trust agreement.

CIMD leaves handling of unrecognized metadata unspecified; this profile
is independent of that choice. Publishing `client_attesters` does not
require an AS to apply this profile.

## Conformance

Conformance is role-specific:

* Client publishers implement {{metadata}} and {{updates}}.
* Client Attesters and clients implement their issuance and presentation
  requirements in {{processing}}.
* Authorization servers implement trust-policy selection, metadata and
  key validation, processing, and withdrawal.

An implementation serving several roles satisfies each role's
requirements. Instance identification is optional.

# Client Metadata {#metadata}

`client_attesters` is OPTIONAL client metadata, usable in a CIMD or
registered metadata under {{RFC7591}}. Its value is an array of objects:

| Member | Requirement | Meaning |
|---|---|---|
| `issuer` | REQUIRED, nonempty StringOrURI {{RFC7519}} | Exact `iss` of the endorsed Client Attester |
| `jwks_uri` | REQUIRED, HTTPS URL without userinfo or fragment | Location of the attester's public JSON Web Key (JWK) Set {{RFC7517}} |

An issuer occurs at most once in the array. A missing or empty array
authorizes no attester. The AS MUST:

* reject the metadata for this profile if an entry is malformed or an
  issuer occurs more than once, without accepting a partial list; and
* ignore unrecognized members.

Extensions MUST NOT weaken an endorsement's meaning for implementations
that ignore them.

Each entry endorses only attestation for the `client_id` whose metadata
contains it. It does not authorize further delegation or reuse for
another client. An `issuer` identifies a namespace, not a discovery
endpoint. Key-source constraints are specified in {{key-resolution}}.

{{ATTEST, Section 10.8}} recommends, among other options, resolving `kid`
through client metadata `jwks_uri`. This profile extends that option
with a separate key location for each endorsed issuer. A top-level
`jwks_uri` can contain several issuers' keys, but does not associate
them with named attesters or separate them from client authentication
keys. It does not replace `client_attesters` under this profile.

SPIFFE's `spiffe_bundle_endpoint` publishes a verification-key location
for one trust domain. `client_attesters` extends this pattern to multiple
named attesters, with AS approval policy selecting the published or
independently configured key source ({{key-resolution}}).

Clients using attestation as client authentication select
`attest_jwt_client_auth` or `attest_jwt_client_auth_dpop` under
{{ATTEST, Section 9}}. When attestation supplements another method,
that method remains required under {{ATTEST, Section 7.6}}.
`client_attesters` does not select a grant, proof method, or the optional
instance-identification profile.

# Attestation and AS Processing {#processing}

## Issuance and Presentation

Requests under this profile MUST include `client_id` to select the
client metadata; the parameter alone is not authentication.

The Client Attester MUST verify that the requesting runtime is
authorized as an instance of the specified client, using authenticated
enrollment or platform evidence. Knowledge of a public `client_id` or
possession of a newly generated key alone is insufficient.

The attester MUST include:

* `iss`: its endorsed `issuer` value;
* `sub`: the exact client identifier; and
* `kid`: a nonempty header parameter {{RFC7515}} identifying its
  signing key.

Other claims and proof requirements follow ATTEST.

## Authorization Server Processing

For each presentation, the AS MUST:

1. Obtain authoritative metadata, from a fresh cache or by retrieval,
   for the requested `client_id`, following CIMD resolution and validation
   or its registered metadata policy, including {{trust}}.
   Selection between registered and fetched metadata follows CIMD;
   the AS MUST NOT combine their endorsement lists.
2. Validate `client_attesters` and select the entry whose `issuer`
   exactly matches the attestation's nonempty `iss`. Verify AS policy
   permits that client-to-attester association.
3. Select the key source under {{key-resolution}}. Resolve `kid` to one
   eligible public key and verify the signature using an acceptable
   asymmetric algorithm. Symmetric keys, private keys, and `alg=none`
   MUST NOT be accepted under this profile.
4. Verify `sub` exactly equals the requested `client_id`, then validate
   the remaining attestation and proof under the selected ATTEST method.
5. Apply grant and authorization policy independently of the endorsement.

## Key Source Selection {#key-resolution}

The AS MUST select keys according to its approval policy:

* **Attester approval:** use the independently configured key source for
  the exact issuer and require the endorsed `jwks_uri` to match that URI
  exactly or an explicitly configured alias. An alias does not change
  the key source. The AS MUST NOT substitute or fall back to
  publisher-selected keys, even when the publisher is also approved.
  The configured key source MAY use a different HTTPS origin from the
  issuer.
* **Publisher approval only:** use the endorsed key source. The `issuer`
  MUST be an HTTPS URL and `jwks_uri` MUST have the same origin
  {{RFC6454}}. This origin check neither isolates tenants sharing an
  origin nor establishes trust in an issuer name. Publisher-selected
  keys MUST NOT inherit an independently trusted attester's assurance
  merely because issuer strings match.

A non-HTTPS issuer requires an independently configured issuer-to-key
association because it has no HTTPS origin binding.

Issuer and client identifiers MUST use exact, case-sensitive comparison
without URI normalization. Key selection and caches MUST bind keys to
the client identifier, issuer, approved key source, and applicable trust
policy; `kid` alone or a union of keys from different entries is
insufficient. Token-controlled key locations MUST NOT override that
source. Origin comparison does not change identifier comparison.

## Errors

Endorsement or attestation validation failures MUST produce
`invalid_client_attestation`, without exposing policy details. This
profile deliberately reuses that error when an otherwise valid
attestation lacks an accepted endorsement. Other metadata-discovery,
registration, authentication, and grant errors follow their base
specifications. The no-fallback rule in {{trust}} applies.

# Updates and Withdrawal {#updates}

## Cache Freshness

The AS MUST:

* enforce configured finite maximum ages for endorsement metadata and
  JWK Sets, applying CIMD and HTTP caching constraints {{RFC9111}} when
  stricter; and
* revalidate or refresh expired entries before use, rejecting stale
  entries if that operation fails.

Maximum ages SHOULD NOT exceed one hour; longer intervals increase
withdrawal delay. Fresh entries do not require retrieval on each request.

On observing that a CIMD has been removed (HTTP 404 or 410), the AS MUST
stop using previously cached endorsements from that document. Removal
cannot be detected while the AS continues to use an unexpired cache.

## Endorsement and Key Changes

Once a metadata or key update is accepted, the AS MUST use it on the
next presentation. Removing an endorsement, removing a verification
key, or publishing an empty list prevents acceptance under that entry
or key, including for attestations issued before the update. Local
policy denial MUST take effect immediately on subsequent requests,
without waiting for cache expiration.

For planned key rotation, publish the new key before using it and
retain the old key while attestations signed with it should remain
acceptable. Metadata caches and JWK Set caches have separate propagation
windows; the AS's configured maximum ages bound stale acceptance.

## Existing Grants

Endorsement withdrawal prevents future authentication under the removed
endorsement; it does not itself revoke existing grants or access tokens.
Refresh requests requiring a Client Attestation are checked again under
{{processing}}.

Deployments using withdrawal to terminate existing access MUST configure
the AS to revoke affected grants, invalidate their access and refresh
tokens, and prevent further refresh issuance. Introspection {{RFC7662}}
reports revoked tokens inactive. Offline validation requires a separate
revocation mechanism or token expiration.

# Security Considerations

The considerations in {{ATTEST}}, {{CIMD}}, and {{RFC8725}} apply.

* **Publisher compromise:** control of a CIMD URL permits changing its
  endorsements, within AS policy. The AS SHOULD monitor and alert on
  endorsement changes and evaluate new attesters as policy changes.
  A separately specified signed-metadata mechanism could bind publisher
  intent independently of the HTTPS host, if its signing keys have an
  independent trust basis; this profile defines no such mechanism.
* **Attester compromise:** attesters serving several clients or tenants
  need issuance controls preventing one from obtaining attestations
  for another.
* **Key retrieval:** the AS MUST authenticate HTTPS servers, limit
  response sizes and request time, and prevent retrieval from prohibited
  network destinations. It MUST NOT follow redirects for `jwks_uri`
  retrieval. This deliberately extends CIMD's no-automatic-redirect
  rule to attester key retrieval. Endorsed URLs remain subject to SSRF
  defenses; endorsement does not make a network location safe.
* **Withdrawal latency:** an already cached endorsement or key can
  remain acceptable until its allowed age expires. Urgent incidents
  require local denial or another revocation channel; removing a key
  at its origin is not instantaneous revocation.
* **Privacy:** public metadata exposes client-to-attester relationships.
  It SHOULD NOT enumerate instances or their keys. Caching reduces the
  request-timing information observable at metadata and key endpoints.
  No stable instance identifier is required by this profile.

# IANA Considerations

This document requests registration in the OAuth Dynamic Client
Registration Metadata registry established by {{RFC7591}}:

* Client Metadata Name: `client_attesters`
* Client Metadata Description: Attesters endorsed to issue Client
  Attestations for this client, with their verification-key locations
* Change Controller: IETF
* Specification Document(s): {{metadata}} of this document

--- back

# CIMD Deployment Example {#example}
{:numbered="false"}

This example is informative. The AS has the following local configuration;
these are policy settings, not new protocol metadata:

| Setting | Value |
|---|---|
| Permitted CIMD publisher origin | `https://platform.example` |
| Independently trusted attester | `https://attester.example/tenant/acme` |
| Configured key source for that attester | `https://attester.example/tenant/acme/jwks` |
| Maximum metadata and key cache ages | 3600 seconds each |

At `https://platform.example/oauth-client`, the publisher serves:

~~~ json
{
  "client_id": "https://platform.example/oauth-client",
  "client_name": "Managed Agent Harness",
  "redirect_uris": ["https://platform.example/callback"],
  "grant_types": ["authorization_code"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "attest_jwt_client_auth_dpop",
  "client_attesters": [
    {
      "issuer": "https://attester.example/tenant/acme",
      "jwks_uri": "https://attester.example/tenant/acme/jwks"
    }
  ]
}
~~~

The attester's configured key endpoint publishes this illustrative JWK
Set. This signing key is distinct from the runtime key in `cnf.jwk`:

~~~ json
{
  "keys": [{
    "kty": "EC",
    "crv": "P-256",
    "kid": "attester-1",
    "use": "sig",
    "alg": "ES256",
    "x": "axfR8uEsQkf4vOblY6RA8ncDfYEt6zOg9KE5RdiYwpY",
    "y": "T-NC4v4af5uO5-tKfA-eFivOM1drMV7Oy7ZAaDe_UfU"
  }]
}
~~~

The decoded Client Attestation header selects that key:

~~~ json
{
  "typ": "oauth-client-attestation+jwt",
  "alg": "ES256",
  "kid": "attester-1"
}
~~~

1. The runtime proves its authorization to use this client identifier
   and possession of its instance key to the attester.
2. The attester issues an ATTEST credential with
   `iss=https://attester.example/tenant/acme`,
   `sub=https://platform.example/oauth-client`, an expiration, and the
   instance public key in `cnf.jwk`.
3. After obtaining user authorization, the runtime redeems its code
   with that `client_id`, the Client Attestation, and a combined
   Demonstrating Proof of Possession (DPoP) proof {{RFC9449}}.
4. The AS validates the CIMD, accepted endorsement, attestation,
   proof, and grant before issuing the access token.

There is one client metadata document, not one per installation.
An endorsement for this client does not let the attester authenticate
another client, even if both use the same attestation service.
The flow does not require `client_instance_id` or an `act` claim.

An attestation from an unendorsed issuer, or an endorsement naming the
trusted issuer with an unapproved key location, produces the same error:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{"error": "invalid_client_attestation"}
~~~

# Document History
{:numbered="false"}

*RFC EDITOR: Remove this section before publication.*

* Initial draft.
