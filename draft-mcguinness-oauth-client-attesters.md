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
  RFC8414:
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
validation and withdrawal rules for endorsements in registered client
metadata or Client ID Metadata Documents, and an authorization-server
metadata parameter advertising support. It introduces no new credential
or authentication method.

--- middle

# Introduction

A client can have many installations, each holding a different key.
Attestation-Based Client Authentication
{{ATTEST}} allows an attester to authenticate those instances. ATTEST
requires an attestation to verify under the key of a known and trusted
Client Attester ({{ATTEST, Section 7.1}}) and leaves how that trust is
established to deployments ({{ATTEST, Section 10.8}}).

This profile applies to both registered clients and clients identified by
Client ID Metadata Documents {{CIMD}}. Either can endorse one or more
attesters, for example across platforms or during migration. Deployments
that manage attester trust entirely through authorization server (AS)
configuration can continue to use ATTEST without this profile.

This profile adds `client_attesters`: the client's endorsements of
attesters and their verification-key locations, generalizing SPIFFE client
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
format, proof methods, and token binding.

This profile, ATTEST, and {{INSTANCE-ID}} answer three separate questions
in layers:

~~~ ascii-art
Client Attester Endorsement    Who may attest for this client?
             |
             v
ATTEST                         Is this a legitimate instance holding
             |                 this key now?
             v
Client Instance ID (optional)  Which persistent instance is this?
~~~

Endorsement carries no instance semantics, and instance identification
does not establish attester trust. Neither establishes user delegation.

Resource servers validating Client Attestations directly rely on
configured attester trust; this profile does not define endorsement
discovery or acceptance for those endpoints. This keeps client metadata
resolution and endorsement-policy evaluation at the AS rather than
distributing those functions to resource servers.

# Conventions and Trust Model {#trust}

{::boilerplate bcp14-tagged}

OAuth terms follow {{RFC6749}} and Client Attester, Client Attestation,
and Client Instance Key follow {{ATTEST}}. The client publisher controls
the authoritative metadata for a `client_id`.

Client Attester Endorsement
: A statement in the authoritative client metadata for a `client_id`
  expressing the publisher's authorization for a specified Client
  Attester to issue Client Attestations naming that `client_id`. An
  endorsement delegates
  attestation authority for the named client only. It does not delegate
  OAuth authorization, user authority, or authority to further delegate
  attestation, and it does not extend to any other client.

## Acceptance Policy

For requests governed by this profile, the AS MUST accept a Client
Attestation only when both of the following hold:

1. **Client endorsement:** the authoritative metadata for the requested
   `client_id` currently endorses the attestation's issuer
   ({{metadata}}).
2. **AS attester acceptance:** AS policy permits that attester for that
   client and determines how the attester's verification keys are
   trusted ({{key-resolution}}).

Endorsement alone does not make an attester trusted, and AS trust in an
attester alone does not authorize it for a client. AS policy can narrow
the endorsed set; it MUST NOT add an unendorsed attester or fall back to
another trust mechanism. An endorsement MUST NOT by itself establish
that the client is trusted or authorized to access a resource.

The AS MUST determine, from its configured policy, which of two
key-trust policies governs each client-to-attester association.
Publisher-authorized key selection MAY be established by a policy
covering the client publisher, without configuring each attester
individually:

* **Publisher-authorized key selection:** the AS authorizes the
  publisher of specified clients to select both the attester and its
  key source, so the endorsed `jwks_uri` supplies the keys. For CIMD,
  configure exact client URLs or HTTPS origins, optionally restricted
  to path segments. Successful metadata retrieval does not establish
  this authorization. Shared hosting requires a boundary that excludes
  other publishers.
* **AS-configured attester trust:** the AS independently trusts a
  particular attester and configures its key source. The endorsement
  authorizes that attester to act for the client; it cannot supply the
  trust anchor, and the endorsed `jwks_uri` is checked against the
  configured source rather than used to select it ({{key-resolution}}).

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
presence or absence of `client_attesters`. An AS advertises support with
`client_attester_endorsement_supported` ({{as-metadata}}). Because that
parameter is AS-wide and enforcement can vary per client, deployments
relying on client endorsement enforcement establish that the AS applies
this profile to their clients through their trust agreement.

CIMD leaves handling of unrecognized metadata unspecified; this profile
is independent of that choice. Publishing `client_attesters` does not
require an AS to apply this profile.

## Conformance

Conformance is role-specific:

* Client publishers implement {{metadata}} and {{updates}}.
* Client Attesters and clients implement their issuance and presentation
  requirements in {{processing}}.
* Authorization servers implement trust-policy selection, metadata and
  key validation, processing, and withdrawal, and can advertise support
  under {{as-metadata}}.

An implementation serving several roles satisfies each role's
requirements. Instance identification is optional.

# Client Metadata {#metadata}

`client_attesters` is OPTIONAL client metadata, usable in registered client
metadata (including {{RFC7591}}) or a CIMD. Its value is an array of objects:

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

Every entry carries a complete issuer-to-key-location mapping, so an
endorsement has the same meaning regardless of the AS policy that
evaluates it, which the publisher cannot know. An endorsement
therefore identifies a Client Attester by both its issuer and its key
location. Under AS-configured
attester trust the AS does not treat that location as the key source;
it retrieves keys from the configured source and uses the endorsed
value only to confirm that the two agree ({{key-resolution}}).

Each entry is a Client Attester Endorsement ({{trust}}) for the
`client_id` whose metadata contains it. An `issuer` identifies a
namespace, not a discovery endpoint. Key-source constraints are
specified in {{key-resolution}}.

{{ATTEST, Section 10.8}} recommends, among other options, resolving `kid`
through client metadata `jwks_uri`. This profile extends that option
with a separate key location for each endorsed issuer. A top-level
`jwks_uri` can contain several issuers' keys, but does not associate
them with named attesters or separate them from client authentication
keys. It does not replace `client_attesters` under this profile.

SPIFFE's `spiffe_bundle_endpoint` publishes a verification-key location
for one trust domain. `client_attesters` extends this pattern to multiple
named attesters, with AS key-trust policy selecting the published or
AS-configured key source ({{key-resolution}}).

Clients using attestation as client authentication select
`attest_jwt_client_auth` or `attest_jwt_client_auth_dpop` under
{{ATTEST, Section 9}}. When attestation supplements another method,
that method remains required under {{ATTEST, Section 7.6}}.
`client_attesters` does not select a grant, proof method, or the optional
instance-identification profile.

# Authorization Server Metadata {#as-metadata}

In addition to the parameters in {{ATTEST, Section 8}}, this profile
defines one OPTIONAL authorization server metadata parameter
{{RFC8414}}:

`client_attester_endorsement_supported`
: Boolean. `true` indicates that the AS processes `client_attesters`
  under this profile for clients presenting Client Attestations. The
  default is `false`.

This parameter does not indicate which key-trust policy applies to any
attester or whether the profile governs a particular client. Those
remain AS policy established through the trust agreement ({{trust}}).

# Attestation and AS Processing {#processing}

## Issuance and Presentation

Requests under this profile MUST include `client_id` to select the
client metadata; the parameter alone is not authentication.

The Client Attester MUST establish that the requesting runtime is
authorized to obtain a Client Attestation naming the specified
`client_id`. The attester MUST NOT treat knowledge of the `client_id`
or possession of a newly generated key, alone or together, as
sufficient. How the attester establishes this authorization is outside
the scope of this specification.

The attester MUST include:

* `iss`: its endorsed `issuer` value;
* `sub`: the exact client identifier; and
* `kid`: a nonempty header parameter {{RFC7515}} identifying its
  signing key.

{{ATTEST, Section 4}} does not require `iss`. This profile requires it
because a client can endorse several Client Attesters with independent
key sets: the issuer selects the applicable endorsement and key source
before `kid` is resolved, and `kid` alone identifies a key within a
set, not a Client Attester or a trust relationship.

This profile retains the default `client_id` to `sub` equality of
{{ATTEST, Section 7.5}} and does not relax it. The binding prevents an
endorsement for one client from validating an attestation naming
another. Other claims and proof requirements follow ATTEST.

## Authorization Server Processing {#as-processing}

For each presentation, the AS MUST:

1. Select the authoritative metadata source for the requested `client_id`
   using AS registration or discovery policy, before evaluating endorsements.
   Obtain metadata from that source or a fresh cache, following CIMD
   resolution and validation or registered metadata policy, including
   {{trust}}. The AS MUST NOT combine endorsement lists from different
   sources or switch sources because endorsement validation fails.
2. Validate `client_attesters` and select the entry whose `issuer`
   exactly matches the attestation's nonempty `iss`. Verify AS policy
   permits that client-to-attester association.
3. Select the key source under {{key-resolution}}. Resolve `kid` to one
   eligible public key, refreshing on an unknown `kid` only as {{updates}}
   permits, and verify the signature using an acceptable asymmetric
   algorithm. Symmetric keys, private keys, and `alg=none` MUST NOT be
   accepted under this profile.
4. Verify `sub` exactly equals the requested `client_id`, then validate
   the remaining attestation and proof under the selected ATTEST method.
   When the attestation is an additional security signal alongside
   another client authentication method ({{ATTEST, Section 7.6}}),
   validate that method under its own specification and verify that it
   authenticates that same client identifier. A mismatch is a failure of
   that method.
5. Apply grant and authorization policy independently of the endorsement.

## Key Source Selection {#key-resolution}

The AS MUST select keys according to the key-trust policy governing
the client-to-attester association ({{trust}}). If AS-configured
attester trust applies to the issuer, it governs regardless of whether
the publisher is also authorized to select keys. Otherwise,
publisher-authorized key selection applies if the publisher is so
authorized. If neither applies, no key source is available and the
endorsement fails.

* **AS-configured attester trust:** use only the independently
  configured key source for the exact issuer. The endorsed `jwks_uri`
  MUST equal that source's URI or one of its configured aliases. An
  alias is an endorsed URI that the AS is configured to treat as
  equivalent to the issuer's configured source; it does not change
  where keys are retrieved. Configured aliases MUST preserve the
  endorsed attestation authority, including tenant scope; a shared
  issuer or origin alone does not establish equivalence. This check
  surfaces disagreement
  between the endorsement and AS configuration, including endorsement
  of a different key set behind a shared issuer string, instead of
  resolving it silently. An endorsed `jwks_uri` MUST NOT select,
  override, or provide a fallback for the configured source. The
  configured key source MAY use a different HTTPS origin from the
  issuer.
* **Publisher-authorized key selection:** use the endorsed `jwks_uri`.
  The `issuer` MUST be an HTTPS URL and `jwks_uri` MUST have the same
  origin {{RFC6454}}. This origin check neither isolates tenants sharing
  an origin nor establishes trust in an issuer name. Publisher-selected
  keys MUST NOT inherit an independently trusted attester's assurance
  merely because issuer strings match.

A non-HTTPS issuer requires AS-configured attester trust because it has
no HTTPS origin binding.

Issuer and client identifiers, and endorsed `jwks_uri` values compared
with configured source URIs and aliases, MUST use exact, case-sensitive
string comparison without URI normalization; an alternative spelling of
a location requires an explicit alias. Key selection and caches MUST bind keys to
the client identifier, issuer, selected key source, and applicable trust
policy; `kid` alone or a union of keys from different entries is
insufficient. Token-controlled key locations MUST NOT override that
source. Origin comparison does not change identifier comparison.

## Errors

Endorsement validation failures MUST produce `invalid_client_attestation`,
without exposing policy details. Endorsement validation covers selecting
a permitted endorsement in step 2 of {{as-processing}} and selecting the
key source and resolving `kid` in step 3 under {{key-resolution}},
including an endorsed `jwks_uri` that matches neither the configured
source nor a configured alias, and the case where no eligible key is
available after any refresh permitted by {{updates}}. Signature verification with a resolved key and
the remaining attestation and proof checks follow {{ATTEST, Section 7.4}},
including challenge and freshness responses. A companion client
authentication method that fails, or that authenticates a different
client identifier, produces the error defined by its own specification.
Other metadata-discovery, registration, authentication, and grant errors
follow their base specifications. The no-fallback rule in {{trust}} applies.

# Updates and Withdrawal {#updates}

## Cache Freshness

The AS MUST:

* enforce configured finite maximum ages for cached endorsement metadata and
  JWK Sets, applying CIMD and HTTP caching constraints {{RFC9111}} when
  stricter; and
* revalidate or refresh expired entries before use, rejecting stale
  entries if that operation fails.

Configured maximum ages bound withdrawal latency: a withdrawn
endorsement or key can remain acceptable until the applicable age
expires. The AS MUST be configured with maximum ages that keep this
latency within the deployment's security requirements; short ages, for
example one hour, keep it small. Fresh entries do not require retrieval on each
request. These limits expire cached copies, not authoritative client
registrations.

On an unknown `kid`, the AS SHOULD refresh the selected key source's
JWK Set once and retry key selection, subject to rate limits. The AS
MUST rate-limit these refreshes per selected key source, independently
of `kid`, and MUST reject
the attestation if no eligible key is available.

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

For planned key rotation, publish the new key and allow the applicable
JWK Set cache lifetimes to elapse before using it. Retain the old key while
attestations signed with it should remain acceptable. Changes to
endorsed key locations need time for metadata-cache propagation and,
under AS-configured attester trust, cause endorsement failures until
the AS configures a matching alias or updates its configured source.
The AS's configured maximum ages bound stale acceptance.

## Existing Grants

Endorsement withdrawal is prospective with respect to client
authentication: it prevents future authentication under the removed
endorsement but does not itself revoke existing grants or access tokens
unless the deployment separately couples withdrawal to revocation.
Refresh requests requiring a Client Attestation are checked again under
{{processing}}.

Deployments using withdrawal to terminate existing access MUST configure
the AS to revoke affected grants, invalidate their access and refresh
tokens, and prevent further refresh issuance. Introspection {{RFC7662}}
reports revoked tokens inactive. Offline validation requires a separate
revocation mechanism or token expiration.

# Security Considerations

The considerations in {{ATTEST}}, {{CIMD}}, and {{RFC8725}} apply.

* **Publisher compromise:** control of a CIMD host or a client's registration
  administration permits changing endorsements, within AS policy. The AS
  SHOULD monitor and alert on endorsement changes and evaluate new attesters
  as policy changes.
  A separately specified signed-metadata mechanism could bind publisher
  intent independently of the HTTPS host, if its signing keys have an
  independent trust basis; this profile defines no such mechanism.
* **Attester compromise:** attesters serving several clients or tenants
  need issuance controls preventing one from obtaining attestations
  for another. Under AS-configured attester trust, the configured key
  source determines whose keys are accepted. The required agreement
  with the endorsed `jwks_uri` ({{key-resolution}}) detects
  disagreement between the endorsement and AS configuration; it
  establishes tenant isolation only if the configured source and its
  aliases preserve the endorsed tenant scope, which remains the AS
  operator's responsibility.
* **Key retrieval:** the AS MUST authenticate HTTPS servers, limit
  response sizes and request time, and prevent retrieval from prohibited
  network destinations. It MUST NOT follow redirects for JWK Set
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

This document also requests registration in the OAuth Authorization
Server Metadata registry established by {{RFC8414}}:

* Metadata Name: `client_attester_endorsement_supported`
* Metadata Description: Boolean indicating that the AS processes
  `client_attesters` under this profile
* Change Controller: IETF
* Specification Document(s): {{as-metadata}} of this document

--- back

# CIMD Deployment Example {#example}
{:numbered="false"}

This example is informative. The AS has the following local configuration;
these are policy settings, not new protocol metadata:

| Setting | Value |
|---|---|
| Permitted CIMD publisher origin | `https://platform.example` |
| Independently trusted attester | `https://attester.example/tenant/acme` |
| Key-trust policy for that attester | AS-configured attester trust |
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

Because the AS applies AS-configured attester trust, keys come only
from the configured source, and the endorsed `jwks_uri` is required to
equal it, as it does here. An attestation from an unendorsed issuer, an
endorsement naming the trusted issuer with a different key location, or
a `kid` that resolves to no key in the configured source, produces:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{"error": "invalid_client_attestation"}
~~~

# Registered Client Example
{:numbered="false"}

An authenticated, authorized administrator registers `s6BhdRkqt3` with the
same `client_attesters` and `token_endpoint_auth_method` as {{example}}. The AS
applies the same AS-configured attester trust and key source.

The client sends `client_id=s6BhdRkqt3` with an attestation whose
`sub` is `s6BhdRkqt3` and `iss` is `https://attester.example/tenant/acme`,
plus its DPoP proof. The AS loads the registered metadata and applies
the same endorsement and proof checks; no CIMD is fetched.

Had the administrator instead registered the URL `client_id` from
{{example}}, the AS would process the request from the single source its
policy selected in step 1, the registration or the CIMD, and never from a
union of both. An endorsement failure from the selected source produces
`invalid_client_attestation`. The AS does not then consult the other
source.

# Document History
{:numbered="false"}

*RFC EDITOR: Remove this section before publication.*

* Initial draft.
