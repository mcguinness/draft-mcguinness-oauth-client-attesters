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
  RFC8705:
  RFC8725:
  RFC9111:
informative:
  RFC7009:
  RFC7592:
  RFC8628:
  RFC9126:
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
Client ID Metadata Documents (CIMDs) {{CIMD}}. Either can endorse one or
more
attesters, for example across platforms or during migration.

Publisher-authorized key selection exists for the case that AS
configuration does not reach. Consider an AS serving many clients
identified by metadata documents, each published by a different
operator and attested by that operator's own platform attester. That AS
would otherwise need a configured entry for every attester of every
client before any of them could authenticate.

Deployments that can manage attester trust entirely
through AS configuration do not need this
profile and can continue to use {{ATTEST}}.

This profile adds `client_attesters`: the client's endorsements of
attesters and their verification-key locations. Under AS-configured
attester trust this plays the role that the bundle endpoint of SPIFFE
client authentication plays {{SPIFFE-OAUTH}}, with the key source
established out of band and the endorsed location only compared against
it. Publisher-authorized key selection deliberately does the opposite,
letting the publisher name the location, so it is not a substitute for
SPIFFE bundle configuration. An AS accepts an endorsement only under
its own trust policy. The resulting chain is:

~~~ ascii-art
Client metadata --endorses--> Attester --attests--> Client Instance
       |                         |                       |
       +----- AS accepts endorsement and validates proof-+
~~~

The profile applies at authorization server (AS) endpoints accepting
Client Attestations for
client authentication or as an additional security signal, using the
profiling hook in {{ATTEST, Section 13}}. In a typical deployment those
are the token endpoint, the pushed authorization request endpoint
{{RFC9126}}, the device authorization endpoint {{RFC8628}}, and the
introspection {{RFC7662}} and revocation {{RFC7009}} endpoints; the
authorization endpoint does not authenticate clients and is out of
scope. A party authenticating at any of these is acting as a client,
including a resource server presenting a Client Attestation to the
introspection endpoint. Where a flow authenticates more than once, each
presentation is evaluated on its own under {{as-processing}}. The
profile retains ATTEST's wire format, proof methods, and token binding.

This profile, {{ATTEST}}, and {{INSTANCE-ID}} answer three separate
questions in layers:

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

A resource server that accepts a Client Attestation presented to it
({{ATTEST, Section 7.6}}) relies on configured attester trust; this
profile does not define endorsement discovery or acceptance there. That
keeps client metadata resolution and endorsement-policy evaluation at
the AS rather than distributing those functions to resource servers,
and it has a consequence: withdrawing an endorsement ({{updates}})
does not reach a resource server validating attestations directly.
This is distinct from a resource server authenticating to an AS
endpoint, which acts as a client and is in scope above.

The AS conveys what it decided through the artifacts it issues rather
than through endorsement data, and what those artifacts carry depends
on the token-binding method the deployment selects. In ATTEST's
combined mode ({{ATTEST, Section 5.2}}) the Demonstrating Proof of
Possession (DPoP) key {{RFC9449}} and the attested Client Instance Key
are one key, so the issued token's confirmation claim names the
attested key. Where DPoP is used alongside
a separate Client Attestation proof, that same section does not require
the DPoP key to match the attestation's `cnf`, and the token is bound
to the DPoP key instead.

A confirmation claim reports a binding, not an endorsement verdict.
Introspection {{RFC7662}} likewise reports the token's state rather
than how the AS evaluated the endorsement. Neither tells a resource
server whether an endorsement was accepted, which is why endorsement
policy stays at the AS.

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
another trust mechanism. An endorsement does not by itself establish
that the client is trusted or authorized to access a resource.

Two key-trust policies exist, and the AS determines from its configured
policy which one applies. The choice is not free per association:
AS-configured attester trust is keyed by exact issuer string and, once
configured for any client, governs that issuer string for every client.
Publisher-authorized key selection is keyed by the client publisher and
needs no per-attester configuration. The two policies are defined
below; {{key-resolution}} gives the procedure that selects between
them:

* **Publisher-authorized key selection:** the AS authorizes the
  publisher of specified clients to select both the attester and its
  key source, so the endorsed `jwks_uri` supplies the keys. For CIMD,
  configure exact client URLs or HTTPS origins, optionally restricted
  to path segments. Successful metadata retrieval does not establish
  this authorization. Shared hosting requires a boundary that excludes
  other publishers. For a registered client, the publisher is the party
  authorized to set endorsements under {{registered}}, and the AS
  configures whether that party's endorsements select keys.
* **AS-configured attester trust:** the AS independently trusts a
  particular attester and configures its key source. The endorsement
  authorizes that attester to act for the client; it cannot supply the
  trust anchor ({{key-resolution}}).

When combined with {{INSTANCE-ID}}, the same two requirements establish
attester authority; instance continuity remains independent. Other ATTEST
deployments can use configured trust without this profile.

## Registered Endorsements {#registered}

Registered endorsements MUST originate from a party authenticated and
authorized to set them for that client, or be covered by a validated
software statement from an issuer approved for that purpose under
{{RFC7591}}. Open registration alone supplies neither assurance; issuing
a client credential does not retroactively approve its endorsements.
The same restriction applies to endorsement updates, including updates
made through the registration management protocol {{RFC7592}}.
Possession of a registration access token establishes control of the
registration, not authority to endorse, and MUST NOT by itself
authorize setting or replacing `client_attesters`.

An AS that does not accept a submitted endorsement MUST either reject
the request with `invalid_client_metadata`
({{RFC7591, Section 3.2.2}}) or omit `client_attesters` from the
stored metadata and from the client information response
({{RFC7591, Section 3.2.1}}), so that the response never shows an
endorsement the AS has not accepted.

## Profile Selection {#profile-selection}

The AS determines that this profile applies to a request by local
policy, which can be scoped per client; this out-of-band determination
satisfies {{ATTEST, Section 13}}. Applicability is not selected by the
presence or absence of `client_attesters`. An AS advertises the
capability with `client_attester_endorsement_supported`
({{as-metadata}}). Because that parameter is AS-wide, deployments
relying on endorsement enforcement establish that the AS applies this
profile to their clients through a trust agreement: the out-of-band
arrangement between the AS operator and the client publisher or
attester operator that fixes which policies the AS applies.

CIMD leaves handling of unrecognized metadata unspecified; this profile
is independent of that choice. Publishing `client_attesters` does not
require an AS to apply this profile.

## Conformance

Conformance is role-specific:

* Client publishers publish and maintain `client_attesters` under
  {{metadata}}, and withdraw an endorsement by updating that metadata
  ({{updates}}).
* Client Attesters and clients implement their issuance and presentation
  requirements in {{processing}}.
* Authorization servers implement trust-policy selection, metadata and
  key validation, processing, and withdrawal, and can advertise support
  under {{as-metadata}}.

An implementation serving several roles satisfies each role's
requirements. Instance identification is optional.

# Client Metadata {#metadata}

The `client_attesters` member is OPTIONAL client metadata, usable in
registered client
metadata (including {{RFC7591}}) or a CIMD. Its value is an array of objects:

| Member | Requirement | Meaning |
|---|---|---|
| `issuer` | REQUIRED, nonempty StringOrURI {{RFC7519}} | Exact `iss` of the endorsed Client Attester |
| `jwks_uri` | REQUIRED, HTTPS URL without userinfo or fragment | Location of the attester's public JSON Web Key (JWK) Set {{RFC7517}} |

An issuer MUST NOT occur more than once in the array. A missing or
empty array authorizes no attester. An entry is malformed if it
violates the requirements in the table above. The AS MUST:

* reject `client_attesters` for this profile if an entry is malformed
  or an issuer occurs more than once, using no endorsement from the
  list; and
* ignore unrecognized members.

Rejection under this profile does not affect the client's other
authentication methods and does not by itself make a CIMD invalid or
uncacheable under {{CIMD}}.

An extension to this member is safe only if an implementation that
ignores it reads the endorsement the same way. This profile defines no
mechanism for marking an extension critical.

Endorsed keys authenticate attesters, not clients. A key obtained from
an endorsement MUST NOT be used to verify a client authentication
assertion, and a key from the client's own `jwks` or `jwks_uri` MUST
NOT be used to verify a Client Attestation.

Every entry carries a complete issuer-to-key-location mapping, so an
endorsement has the same meaning regardless of the AS policy that
evaluates it, which the publisher cannot know. An endorsement
therefore identifies a Client Attester by both its issuer and its key
location. How each key-trust policy uses that location is specified in
{{key-resolution}}.

Each entry is a Client Attester Endorsement ({{trust}}) for the
`client_id` whose metadata contains it. An `issuer` identifies a
namespace, not a discovery endpoint.

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
The member does not select a grant, proof method, or the optional
instance-identification profile.

# Authorization Server Metadata {#as-metadata}

In addition to the parameters in {{ATTEST, Section 8}}, this profile
defines one OPTIONAL authorization server metadata parameter
{{RFC8414}}:

`client_attester_endorsement_supported`
: Boolean. `true` indicates that the AS is capable of processing
  `client_attesters` under this profile. The default is `false`.

Whether the profile governs a particular client, and which key-trust
policy applies to an attester, remain AS policy ({{profile-selection}}).

# Attestation and AS Processing {#processing}

This section covers what an attester must establish and put in a Client
Attestation, the order in which an AS validates one, how the AS selects
the key that verifies it, and how failures are reported.

## Issuance and Presentation

Requests under this profile MUST include `client_id` to select the
client metadata; the parameter alone is not authentication.

The Client Attester MUST establish that the requesting Client Instance
is authorized to obtain a Client Attestation naming the specified
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
another. Other claims and proof requirements follow {{ATTEST}}.

## Authorization Server Processing {#as-processing}

For each presentation, the AS MUST:

1. Select the authoritative metadata source for the requested `client_id`
   using AS registration or discovery policy, before evaluating endorsements.
   Obtain metadata from that source or a fresh cache, following CIMD
   resolution and validation or registered metadata policy, including
   {{trust}}. The AS MUST NOT combine endorsement lists from different
   sources or switch sources because endorsement validation fails. A
   client identifier that has both a registration and a reachable CIMD
   is resolved from whichever single source this step selects; an
   endorsement failure from that source is final, and the AS does not
   then consult the other.
2. Validate `client_attesters` and select the entry whose `issuer`
   exactly matches the attestation's nonempty `iss`. Verify AS policy
   permits that client-to-attester association. The entry's `issuer`
   and `jwks_uri` are matched together rather than the issuer alone,
   so that an endorsement naming a different key location behind a
   shared issuer string does not match; the matching identifies the
   entry and does not by itself authorize it.
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
   that method. Where the companion method also establishes a
   confirmation key, for example mutual TLS {{RFC8705}}, the
   configuration selects which key binds the issued token; the AS MUST
   NOT bind a token to the attested key on the strength of an
   attestation it did not accept.
5. Apply grant and authorization policy independently of the endorsement.

## Key Source Selection {#key-resolution}

The AS MUST select keys according to the key-trust policy governing
the client-to-attester association ({{trust}}). If the AS has
configured attester trust for the attestation's exact issuer string
for any client, AS-configured attester trust governs that issuer for
every client, regardless of whether the requesting client's publisher
is also authorized to select keys. Otherwise, publisher-authorized key
selection applies if the publisher is so authorized. If neither
applies, no key source is available and the endorsement fails.

Removing a configured entry MUST NOT by itself make its issuer eligible
for publisher-authorized key selection. An issuer the AS has configured
remains governed by AS-configured attester trust until an operator
records a policy decision for that issuer; until then no key source is
available and the endorsement fails. Otherwise an edit made to reduce
trust would instead hand key selection to the publisher.

* **AS-configured attester trust:** use only the independently
  configured key source for the exact issuer. The endorsed `jwks_uri`
  MUST equal that source's URI or one of its configured aliases. An
  alias is an endorsed URI that the AS is configured to treat as
  equivalent to the issuer's configured source; it does not change
  where keys are retrieved. An alias belongs to the issuer's configured
  key source, so it applies to every client that endorses that issuer
  and is not scoped to the client whose endorsement prompted it.
  A configured alias MUST preserve the endorsed attestation authority,
  including tenant scope; a shared issuer or origin alone does not
  establish equivalence. Because an alias applies to every client
  endorsing the issuer, this bounds what an alias may map to, and it is
  the rule the tenant-isolation argument in {{security}} rests on.
  This check
  surfaces disagreement between the endorsement and AS configuration,
  including endorsement
  of a different key set behind a shared issuer string, instead of
  resolving it silently. An endorsed `jwks_uri` MUST NOT select,
  override, or provide a fallback for the configured source. The AS
  MUST NOT retrieve the endorsed `jwks_uri` under this policy; the
  endorsed value is compared, never fetched. No origin relationship is
  required between
  the configured key source and the issuer.
* **Publisher-authorized key selection:** use the endorsed `jwks_uri`.
  The `issuer` MUST be an HTTPS URL and `jwks_uri` MUST have the same
  origin {{RFC6454}}. This origin check neither isolates tenants sharing
  an origin nor establishes trust in an issuer name. Because
  AS-configured trust governs any issuer string it is configured for,
  publisher-selected keys are never accepted under an issuer string the
  AS trusts or has configured.

A non-HTTPS issuer requires AS-configured attester trust because it has
no HTTPS origin binding.

The AS MUST use exact, case-sensitive string comparison, without URI
normalization, for issuer identifiers, for client identifiers, and when
comparing endorsed `jwks_uri` values with configured source URIs and
aliases. An alternative spelling of a location requires an explicit
alias. Key selection and caches MUST bind keys to
the client identifier, issuer, selected key source, and applicable trust
policy; `kid` alone or a union of keys from different entries is
insufficient. The AS MUST ignore the `jku`, `x5u`, `x5c`, and `jwk`
JOSE header parameters for key selection under this profile and MUST
resolve only `kid` against the selected source. Origin comparison does
not change identifier comparison.

A key is eligible when all of the following hold:

* it is the only key in the selected JWK Set whose `kid` equals the
  header `kid` by octet comparison;
* it is an asymmetric public key whose type is consistent with the
  header `alg`;
* its `use`, if present, is `sig`;
* its `key_ops`, if present, includes `verify`; and
* its `alg`, if present, equals the header `alg`.

More
than one key matching the `kid` is a failure; the AS MUST NOT try
candidate keys in turn.

When retrieving a JWK Set or client metadata, the AS MUST authenticate
the HTTPS server and MUST NOT follow redirects. Bounding response size
and request time, and blocking prohibited network destinations, are
local defenses; see Security Considerations. The AS SHOULD advertise
`client_attestation_signing_alg_values_supported` consistent with the
algorithm restrictions in step 3 of {{as-processing}}
({{ATTEST, Section 8}}).

## Errors

Endorsement validation covers these parts of {{as-processing}}:

* selecting a permitted endorsement in step 2;
* selecting the key source and resolving `kid` in step 3 under
  {{key-resolution}}, including an endorsed `jwks_uri` that matches
  neither the configured source nor a configured alias; and
* the case where no eligible key is available after any refresh
  permitted by {{updates}}.

Its outcome is reported differently depending on the role the Client
Attestation plays in the request.

Attestation is the client authentication method:
: An endorsement validation failure MUST produce
  `invalid_client_attestation`. {{ATTEST, Section 7.4}} defines that
  code for use in addition to the more general `invalid_client`; this
  profile narrows the choice to the specific code so an endorsement
  failure is distinguishable from an ordinary credential failure. The
  response MUST NOT expose policy details.

  This profile does not change the HTTP status code any endpoint
  assigns to
  a client authentication failure. At the token endpoint
  {{RFC6749, Section 5.2}} responds 400 by default and requires 401
  only where the client authenticated through the `Authorization`
  header field, which carrying a Client Attestation does not do. At the
  introspection endpoint {{RFC7662, Section 2.3}} requires 401. Other
  endpoints follow their own specifications.

  A Client library that recognizes only `invalid_client` treats this as
  an unrecognized failure rather than a credential failure.

Attestation is an additional security signal:
: Where the deployment uses the Client Attestation alongside another
  client authentication method ({{ATTEST, Section 7.6}}), an
  endorsement validation failure means no attestation signal is
  available for that request. The AS MUST NOT treat the failed
  attestation as a satisfied signal, and whether the request proceeds
  on the companion method alone is AS policy.

Obtaining a fresh attestation does not correct an endorsement failure
caused by disagreement between the endorsement and AS configuration,
such as an endorsed `jwks_uri` matching neither the configured source
nor a configured alias ({{key-resolution}}). Because the response
deliberately carries no policy detail, a Client cannot tell that case
apart from one a fresh attestation would fix; it is resolved through
the operational channels in {{security}} rather than by client retry.

Everything else keeps its own error. Signature verification with a
resolved key and the remaining attestation and proof checks follow
{{ATTEST, Section 7.4}}, including challenge and freshness responses. A
companion client authentication method that fails, or that
authenticates a different client identifier, produces the error defined
by its own specification. Other metadata-discovery, registration,
authentication, and grant errors follow their base specifications. The
no-fallback rule in {{trust}} applies.

# Updates and Withdrawal {#updates}

A publisher withdraws an endorsement by removing it from the
authoritative client metadata ({{metadata}}). Three things then govern
when that takes effect and what it reaches: cached copies expire under
the maximum ages below, an accepted update binds from the next
presentation ({{endorsement-changes}}), and an issued grant keeps its
access tokens unless the deployment separately revokes them, though a
refresh that presents an attestation is checked again
({{existing-grants}}). This profile sets no ceiling on the withdrawal
latency that the maximum ages bound.

## Cache Freshness and Removal

The AS MUST:

* enforce configured finite maximum ages for cached endorsement metadata and
  JWK Sets, applying CIMD and HTTP caching constraints {{RFC9111}} when
  stricter; and
* revalidate or refresh expired entries before use, rejecting stale
  entries if that operation fails.

Configured maximum ages bound withdrawal latency: a withdrawn
endorsement or key can remain acceptable until the applicable age
expires. Configure maximum ages so that this latency stays within the
deployment's security requirements; short ages, for example one hour,
keep it small. This profile specifies no ceiling, so
a publisher cannot predict from the protocol alone how long a
withdrawal takes to bite; deployments that need a predictable bound
state one in their trust agreement. Fresh entries do not require
retrieval on each request. These limits expire cached copies, not authoritative client
registrations.

On an unknown `kid`, the AS SHOULD refresh the selected key source's
JWK Set once and retry key selection, subject to rate limits. The AS
MUST rate-limit these refreshes per selected key source, independently
of `kid`, and MUST reject the attestation if no eligible key is
available. Where several clients or publishers endorse one key source,
the AS SHOULD also limit refreshes per endorsing client and per
publisher, so that no client or publisher can exhaust another's
allowance. Rate-limit parameters are
deployment-specific. An `iss`
matching no endorsement MUST NOT cause a client-metadata refresh; the
metadata maximum age bounds the delay before a newly published
endorsement takes effect, as it bounds withdrawal.

On observing that a CIMD has been removed (HTTP 404 or 410), the AS MUST
stop using previously cached endorsements from that document, and MUST
NOT use them again unless a later retrieval of that document succeeds.
A retrieval failure that is not a removal, such as a timeout or a 5xx
status, does not by itself invalidate an unexpired cached copy. Removal
cannot be detected while the AS continues to use an unexpired cache.
Deleting a registered client's registration removes its endorsements
with it.

## Endorsement and Key Changes {#endorsement-changes}

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

## Existing Grants {#existing-grants}

Endorsement withdrawal is prospective with respect to client
authentication: it prevents future authentication under the removed
endorsement but does not itself revoke existing grants or access tokens
unless the deployment separately couples withdrawal to revocation.
Refresh requests requiring a Client Attestation are checked again under
{{processing}}.

Withdrawal alone does not terminate existing access. A deployment that
requires termination separately revokes the affected grants,
invalidates their access and refresh tokens, and prevents further
refresh issuance.
Introspection {{RFC7662}}
reports revoked tokens inactive. Offline validation requires a separate
revocation mechanism or token expiration.

# Security Considerations {#security}

The considerations in {{ATTEST}}, {{CIMD}}, and {{RFC8725}} apply.

* **Publisher compromise:** control of a CIMD host or a client's registration
  administration permits changing endorsements, within AS policy. The AS
  has no protocol signal for this, so operators generally monitor
  endorsement changes and evaluate new attesters as policy changes.
  A separately specified signed-metadata mechanism could bind publisher
  intent independently of the HTTPS host, if its signing keys have an
  independent trust basis; this profile defines no such mechanism.
* **Attester compromise:** attesters serving several clients or tenants
  need issuance controls preventing one from obtaining attestations
  for another. Under AS-configured attester trust, the agreement check
  in {{key-resolution}} establishes tenant isolation only if the
  configured source and its aliases preserve the endorsed tenant scope,
  which remains the AS operator's responsibility.
* **Key retrieval:** the retrieval rules in {{key-resolution}}
  deliberately extend CIMD's no-automatic-redirect rule to attester key
  retrieval. Under publisher-authorized key selection the publisher
  chooses both the issuer and the key location, so an authorized
  publisher can cause the AS to issue an outbound request to an origin
  of the publisher's choosing. {{key-resolution}} constrains where that
  request may go but does not remove it, and an AS defends itself
  further by bounding response
  size and request time and by blocking prohibited network
  destinations. Endorsed URLs remain subject to server-side request
  forgery (SSRF) defenses;
  endorsement does not make a network location safe.
* **Withdrawal latency:** cached acceptance persists as described in
  {{updates}}. Urgent incidents require local denial or another
  revocation channel; removing a key at its origin is not instantaneous
  revocation.
* **Omitted attestation:** `client_attesters` does not itself require
  attestation. A client whose other credentials are stolen can be
  authenticated without an attestation unless its registered
  `token_endpoint_auth_method` requires one; deployments relying on
  endorsement enforcement set that method accordingly.
* **Unscoped endorsement:** an endorsement carries no audience. Under
  publisher-authorized key selection, one public endorsement determines
  the attester and its keys at every AS whose policy covers that
  publisher, so a compromised attester authenticates the client at all
  of them until the endorsement is withdrawn.
* **Privacy:** public metadata exposes client-to-attester relationships.
  Such metadata need not enumerate instances or their keys, and this
  profile gives no reason to do so. Caching reduces the
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
* Metadata Description: Boolean indicating that the AS is capable of
  processing `client_attesters` under this profile
* Change Controller: IETF
* Specification Document(s): {{as-metadata}} of this document

--- back

# CIMD Deployment Example {#example}

This example is informative. The AS has the following local configuration;
these are policy settings, not new protocol metadata:

| Setting | Value |
|---|---|
| Permitted origin for CIMD retrieval | `https://platform.example` |
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
Set. This signing key is distinct from the Client Instance Key in
`cnf.jwk`:

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

The decoded payload names the endorsed issuer and the client:

~~~ json
{
  "iss": "https://attester.example/tenant/acme",
  "sub": "https://platform.example/oauth-client",
  "exp": 1789434000,
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "JYcxlWx7A9YIcr3Bb94ZHqhUX6ea7leTAeGx_WWjs0A",
      "y": "ppg4pVaOV7ANtw8fQoV8OWfe_6GhY13WPLpuWd_rHnc"
    }
  }
}
~~~

1. The Client Instance proves its authorization to use this client
   identifier and possession of its instance key to the attester.
2. The attester issues the Client Attestation shown above, naming its
   endorsed issuer, the client, an expiration, and the instance public
   key in `cnf.jwk`.
3. After obtaining user authorization, the Client Instance redeems its code
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
a `kid` that resolves to no key in the configured source, produces the
response below. {{RFC6749, Section 5.2}} reserves 401 for a client that
authenticated through the `Authorization` header field; this client
presents its attestation in the ATTEST header fields instead, so the
status here is 400:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{"error": "invalid_client_attestation"}
~~~

Had the AS instead authorized `https://platform.example` for
publisher-authorized key selection and configured no trust for the
issuer, the same document would succeed under the other policy: the AS
would retrieve keys from the endorsed `jwks_uri`, which shares the
issuer's origin, and apply the same processing steps. An entry whose
`jwks_uri` had a different origin from the issuer would then fail
endorsement validation with the same error.

# Registered Client Example {#registered-example}

This example is informative. It repeats {{example}} with an opaque
client identifier instead of a URL. The repetition shows that, under
this policy, neither endorsement nor key selection depends on the
identifier's shape: `client_attesters` travels with the client's
metadata
either way, and the endorsement names the attester's key location
outright, so no origin has to be derived from the client identifier.
How a publisher is authorized does differ between the two forms
({{trust}}), but that question does not arise here because the AS
trusts this attester independently. The AS configuration is the one in
{{example}}.

An authenticated, authorized administrator registers this metadata,
for example through {{RFC7591}}:

~~~ json
{
  "client_id": "s6BhdRkqt3",
  "client_name": "Managed Agent Harness",
  "redirect_uris": ["https://app.example/callback"],
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

The attester signs with the same key as in {{example}}, so the
attestation header is the same:

~~~ json
{
  "typ": "oauth-client-attestation+jwt",
  "alg": "ES256",
  "kid": "attester-1"
}
~~~

In the payload only `sub` differs:

~~~ json
{
  "iss": "https://attester.example/tenant/acme",
  "sub": "s6BhdRkqt3",
  "exp": 1789434000,
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "JYcxlWx7A9YIcr3Bb94ZHqhUX6ea7leTAeGx_WWjs0A",
      "y": "ppg4pVaOV7ANtw8fQoV8OWfe_6GhY13WPLpuWd_rHnc"
    }
  }
}
~~~

The client redeems its code with `client_id=s6BhdRkqt3`, that
attestation, and a combined DPoP proof. The AS reads the registered
metadata rather than fetching a CIMD, then runs the same steps of
{{as-processing}}: the endorsed issuer matches the attestation's `iss`,
AS-configured attester trust selects the configured key source, `kid`
resolves to `attester-1` there, and `sub` equals the requested
`client_id`. The failure cases and their error response are those of
{{example}}.

# Document History
{:numbered="false"}

*RFC EDITOR: Remove this section before publication.*

* Initial draft.
