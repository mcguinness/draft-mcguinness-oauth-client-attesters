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

OAuth 2.0 Attestation-Based Client Authentication requires an
authorization server to trust the attester that vouches for a client
instance, but does not define how a client identifies the attesters
authorized to speak for it. This specification closes that gap with a
client metadata member, usable by registered clients and Client ID
Metadata Documents, that names endorsed attesters and their
verification-key locations. It defines how an authorization server
validates endorsements and processes their withdrawal while retaining
control over whether to trust them. It introduces no new credential or
client authentication method.

--- middle

# Introduction

Attestation-Based Client Authentication {{ATTEST}} lets a Client
Attester make security-relevant statements about a client instance and
the key it holds. Before an authorization server (AS) relies on such an
attestation, it needs answers to two distinct questions: is the
attester trusted ({{ATTEST, Section 7.1}}), and is that attester
authorized to speak for this particular client?

ATTEST defines the Client Attestation format, presentation, and
validation, and intentionally leaves establishment of Client Attester
trust to deployments ({{ATTEST, Section 10.8}}). It defines no
relationship by which a client identifies the attesters authorized to
attest its instances.

That relationship matters when a client has many independently
provisioned instances, uses a platform or workload attester, migrates
between attesters, or is identified by a Client ID Metadata Document
(CIMD) {{CIMD}} rather than by pre-established bilateral configuration.
If the AS configures every client-to-attester association, the client
cannot withdraw or narrow its attesters without the AS.

This specification makes the relationship explicit with a Client
Attester Endorsement in client metadata:

~~~ ascii-art
Client metadata --endorses--> Attester --attests--> Client Instance
       \__________________ AS validates __________________/
~~~

An endorsement says that the client publisher authorizes the named
Client Attester to speak for the client. It does not make that attester
trusted by the AS, which still decides whether to accept the endorsed
attester and how to trust its verification keys ({{trust}}). The
`client_attesters` client metadata member ({{metadata}}) carries the
endorsed attesters and their verification-key locations. It serves
registered and CIMD clients alike and can hold several endorsements,
for example across heterogeneous platforms or during attester
migration.

This separates two authorities that are otherwise easily conflated:

* the client publisher determines which attesters are authorized to
  speak for the client; and
* the AS determines which of those endorsements it is willing to trust.

The client publisher thus manages its attester associations without gaining
control of AS trust policy. It can always withdraw or narrow its
endorsements. Adding an attester or moving its key location takes
effect without AS action only where the AS authorizes the publisher to
select keys; where the AS configures attester trust itself, the AS also
has to accept the change ({{trust}}).

The profile builds directly on ATTEST and introduces no new credential
or client authentication method. ATTEST, this profile, and the optional
Client Instance ID profile {{INSTANCE-ID}} address separate layers:

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

Deployments that can manage client-to-attester associations entirely
through AS configuration can use {{ATTEST}} without this profile. The
profile is for deployments where the client publisher expresses and
maintains that association, subject to AS policy ({{acceptance}}).

# Conventions and Trust Model {#trust}

{::boilerplate bcp14-tagged}

OAuth terms follow {{RFC6749}}; Client Attester, Client Attestation,
and Client Instance Key follow {{ATTEST}}. The client publisher controls
the authoritative metadata for a `client_id`.

Client Attester Endorsement
: A statement in the authoritative client metadata for a `client_id`
  identifying a Client Attester whose Client Attestations naming that
  `client_id` are eligible for acceptance under this profile, subject to
  AS policy ({{trust}}). It expresses the publisher's authorization for that
  attester to speak for the client; it does not say which Client
  Instances the attester may attest, which {{processing}} leaves to the
  attester. An endorsement delegates
  attestation authority for the named client only. It does not delegate
  OAuth authorization, user authority, or authority to further delegate
  attestation.

## Acceptance Policy {#acceptance}

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
the endorsed set, and authorizing a publisher to select keys
({{key-resolution}}) permits each attester that publisher endorses.
AS policy MUST NOT add an unendorsed attester or accept an
attestation through another attester-trust mechanism. Where the
attestation is optional, proceeding on a companion client
authentication method without it ({{errors}}) is not such a fallback.
An endorsement does not by itself establish that the client is trusted
or authorized to access a resource.

Two key-trust policies exist, and AS policy determines which one
applies. The choice is not free per association: AS-configured
attester trust is keyed by exact issuer string and, once configured for
any client, governs that issuer string for every client.
Publisher-authorized key selection is keyed by the client publisher and
needs no per-attester configuration. {{key-resolution}} gives the
procedure that selects between them:

* **Publisher-authorized key selection:** the AS authorizes the
  publisher of specified clients to select both the attester and its
  key source, so the endorsed `jwks_uri` supplies the keys. For CIMD,
  the AS configures exact client URLs or HTTPS origins, optionally
  restricted to a path prefix. A prefix matches only at a `/` segment
  boundary. A client URL whose path contains `\`, `;`, or a
  percent-encoded `/`, `\`, or `.` matches no prefix, because a server
  can decode or route such a path to a different document than the one
  compared. Client identifier comparison itself stays exact. A path
  prefix is a publisher boundary only where the host serves each path
  under it from the publisher it names; shared hosting requires such a
  boundary. Successful metadata retrieval does not establish this
  authorization. For a registered client, the publisher is the party
  authorized to set endorsements under {{registered}}, and the AS
  configures whether that party's endorsements select keys.
* **AS-configured attester trust:** the AS independently trusts a
  particular attester and configures its key source. The endorsement
  authorizes that attester to act for the client; it cannot supply the
  trust anchor ({{key-resolution}}).

Publisher-authorized key selection covers what AS configuration does
not reach: an AS serving many CIMD clients, each published by a
different operator and attested by that operator's own platform
attester, would otherwise need a configured entry for every attester of
every client before any of them could authenticate.

When combined with {{INSTANCE-ID}}, the same two conditions establish
attester authority; instance continuity remains independent.

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
authorize setting or replacing `client_attesters`. Removing the member,
including by omitting it from an update that {{RFC7592}} treats as a
deletion request, counts as replacing it.

An AS that does not accept a submitted endorsement MUST either reject
the request with `invalid_client_metadata`
({{RFC7591, Section 3.2.2}}) or omit `client_attesters` from the
stored metadata and from the client information response
({{RFC7591, Section 3.2.1}}), so that the response never shows an
endorsement the AS has not accepted.

## Applicability and Scope {#profile-selection}

AS policy, which can be scoped per client, determines whether this
profile applies to a request; this out-of-band determination satisfies
{{ATTEST, Section 13}}. The presence or absence of `client_attesters`
does not select applicability, and publishing it does not require an
AS to apply this profile. The profile is independent of how CIMD
handles unrecognized metadata, which CIMD leaves unspecified. An AS
advertises the capability with `client_attester_endorsement_supported`
({{as-metadata}}). Because that parameter is AS-wide, deployments
relying on endorsement enforcement establish through a trust agreement
that the AS applies this profile to their clients. A trust agreement is
the out-of-band arrangement between the AS operator and the client
publisher or attester operator that fixes which policies the AS
applies.

The profile applies at AS endpoints that accept Client Attestations for
client authentication or as an additional security signal: typically
the token endpoint, the pushed authorization request endpoint
{{RFC9126}}, the device authorization endpoint {{RFC8628}}, and the
introspection {{RFC7662}} and revocation {{RFC7009}} endpoints. The
authorization endpoint does not authenticate clients and is out of
scope. A party authenticating at any of these endpoints acts as a
client, including a resource server presenting a Client Attestation to
the introspection endpoint. Where a flow authenticates more than once,
each presentation is evaluated on its own under {{as-processing}}. The
profile retains ATTEST's wire format, proof methods, and token binding.

A resource server that accepts a Client Attestation presented to it
({{ATTEST, Section 7.6}}) relies on configured attester trust; this
profile does not define endorsement discovery or acceptance there. This
keeps client metadata resolution and endorsement policy at the AS
rather than at each resource server, with one consequence: withdrawing
an endorsement ({{updates}}) does not reach a resource server that
validates attestations directly.

The AS conveys its decision through the artifacts it issues, not
through endorsement data, and what they carry depends on the
deployment's token-binding method. In ATTEST's combined mode
({{ATTEST, Section 5.2}}), the Demonstrating Proof of Possession (DPoP)
key {{RFC9449}} and the attested Client Instance Key are one key, so
the issued token's confirmation claim names the attested key. Where
DPoP is used alongside a separate Client Attestation proof, that
section does not require the DPoP key to match the attestation's `cnf`,
and the token is bound to the DPoP key instead. Either way, a
confirmation claim reports a binding, not an endorsement verdict, and
introspection {{RFC7662}} reports the token's state, not how the AS
evaluated the endorsement. Neither tells a resource server whether an
endorsement was accepted, which is another reason endorsement policy
stays at the AS.

## Conformance

Conformance is role-specific:

* Client publishers publish and maintain `client_attesters` under
  {{metadata}}, and withdraw an endorsement by updating that metadata
  ({{updates}}).
* Client Attesters and clients implement their issuance and presentation
  requirements in {{processing}}.
* Authorization servers implement key-trust policy selection, metadata
  and key validation, processing, and withdrawal, and can advertise
  support under {{as-metadata}}.

An implementation serving several roles satisfies each role's
requirements. Instance identification is optional.

# Client Metadata {#metadata}

The `client_attesters` member is OPTIONAL client metadata, usable in
registered client metadata (including {{RFC7591}}) or a CIMD. Its value
is an array of objects, each a Client Attester Endorsement ({{trust}})
for the `client_id` whose metadata contains it:

| Member | Requirement | Meaning |
|---|---|---|
| `issuer` | REQUIRED, nonempty StringOrURI {{RFC7519}} | Exact `iss` of the endorsed Client Attester |
| `jwks_uri` | REQUIRED, HTTPS URL without userinfo or fragment | Location of the attester's public JSON Web Key (JWK) Set {{RFC7517}} |
{: title="Members of a client_attesters entry"}

An `issuer` identifies a namespace, not a discovery endpoint. An issuer
MUST NOT occur more than once in the array. A missing or empty array
authorizes no attester. An entry is malformed if it violates the
requirements in the table above. The AS MUST:

* reject `client_attesters` for this profile if an entry is malformed
  or an issuer occurs more than once, using no endorsement from the
  list; and
* ignore unrecognized members.

Rejection under this profile does not affect the client's other
authentication methods and does not by itself make a CIMD invalid or
uncacheable under {{CIMD}}. An extension to this member is safe only if
an implementation that ignores it reads the endorsement the same way;
this profile defines no way to mark an extension critical.

Endorsed keys authenticate attesters, not clients. A key obtained from
an endorsement MUST NOT be used to verify a client authentication
assertion, and a key from the client's own `jwks` or `jwks_uri` MUST
NOT be used to verify a Client Attestation. An entry whose `jwks_uri`
is identical to the client's own `jwks_uri` is therefore malformed.

An endorsement identifies a Client Attester by both issuer and key
location. The publisher cannot know which key-trust policy the AS
applies, so each entry carries a complete issuer-to-key-location
mapping that means the same under either policy; {{key-resolution}}
specifies how each policy uses the location.

{{ATTEST, Section 10.8}} recommends, among other options, resolving `kid`
through client metadata `jwks_uri`. This profile extends that option
with a separate key location for each endorsed issuer. A top-level
`jwks_uri` can hold several issuers' keys, but it neither associates
them with named attesters nor separates them from client
authentication keys, so it does not replace `client_attesters`.

SPIFFE client authentication {{SPIFFE-OAUTH}} publishes one
verification-key location, `spiffe_bundle_endpoint`, per trust domain.
Under AS-configured attester trust, the endorsed `jwks_uri` plays that
role for each named attester, but the key source is established out of
band and the endorsed location is only compared against it.
Publisher-authorized key selection lets the publisher name the
location, so it is no substitute for SPIFFE bundle configuration.

Clients using attestation as client authentication select
`attest_jwt_client_auth` or `attest_jwt_client_auth_dpop` under
{{ATTEST, Section 9}}. When attestation supplements another method,
that method remains required under {{ATTEST, Section 7.6}}. The member
does not select a grant, proof method, or the optional
instance-identification profile.

# Authorization Server Metadata {#as-metadata}

In addition to the parameters in {{ATTEST, Section 8}}, this profile
defines one OPTIONAL authorization server metadata parameter
{{RFC8414}}:

`client_attester_endorsement_supported`
: Boolean. `true` indicates that the AS can process
  `client_attesters` under this profile. The default is `false`.

Whether the profile governs a particular client, and which key-trust
policy applies to an attester, remain AS policy ({{profile-selection}}).

# Attestation and AS Processing {#processing}

This section specifies Client Attestation content, the AS validation
order, key selection, and error reporting.

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
key sets: the issuer selects the endorsement and key source before
`kid` is resolved, and `kid` identifies a key within a set, not a
Client Attester or a trust relationship.

This profile keeps the default `client_id` to `sub` equality of
{{ATTEST, Section 7.5}} without relaxation, so an endorsement for one
client cannot validate an attestation naming another. Other claims and
proof requirements follow {{ATTEST}}.

## Authorization Server Processing {#as-processing}

For each presentation, the AS MUST:

1. Before evaluating endorsements, select the authoritative metadata
   source for the requested `client_id` using AS registration or
   discovery policy. Obtain metadata from that source or a fresh cache,
   following CIMD resolution and validation or registered metadata
   policy, including {{registered}}. The AS MUST NOT combine endorsement
   lists from different sources or switch sources because endorsement
   validation fails. A client identifier with both a registration and a
   reachable CIMD is resolved from the single source this step selects;
   an endorsement failure from that source is final.
2. Validate `client_attesters` and select the entry whose `issuer`
   exactly matches the attestation's nonempty `iss`; because an issuer
   occurs at most once in the array ({{metadata}}), the selection is
   unique. Verify that AS policy permits that client-to-attester
   association; selecting an entry does not by itself authorize it.
   Policy evaluates the selected entry, including its `jwks_uri`, not
   the issuer alone. Agreement between that `jwks_uri` and a configured
   key source is checked in step 3 ({{key-resolution}}), not here.
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
   authenticates the same client identifier; a mismatch is a failure of
   that method. Where the companion method also establishes a
   confirmation key, for example mutual TLS {{RFC8705}}, AS
   configuration selects which key binds the issued token; the AS MUST
   NOT bind a token to the attested key on the strength of an
   attestation it did not accept.
5. Apply grant and authorization policy independently of the endorsement.

## Key Source Selection {#key-resolution}

The AS MUST select keys according to the key-trust policy governing
the client-to-attester association ({{trust}}). If the AS has
configured attester trust for the attestation's exact issuer string
for any client, AS-configured attester trust governs that issuer for
every client, even if the requesting client's publisher is also
authorized to select keys. Otherwise, publisher-authorized key
selection applies if the publisher is so authorized. If neither
applies, no key source is available and the endorsement fails.

After a configured entry is removed, the AS MUST NOT verify an
attestation under that issuer with publisher-selected keys unless an
operator has since decided that publishers may select keys for that
issuer; otherwise removing a configured entry would hand key selection
to the publisher. Restoring a configured key source for the issuer
returns it to AS-configured attester trust. Until one of these
happens, no key source is available and the endorsement fails.

* **AS-configured attester trust:** use only the independently
  configured key source for the exact issuer; no origin relationship
  between that source and the issuer is required. The endorsed
  `jwks_uri` MUST equal that source's URI or one of its configured
  aliases. This check surfaces, rather than silently resolves,
  disagreement between the endorsement and AS configuration, including
  endorsement of a different key set behind a shared issuer string. An
  alias is an endorsed URI that the AS treats as equivalent to the
  issuer's configured key source; it does not change where keys are
  retrieved. Aliases are issuer-wide: an alias applies to every client
  that endorses the issuer, not only the client whose endorsement
  prompted it. A configured alias MUST preserve the endorsed
  attestation authority, including tenant scope; a shared issuer or
  origin alone does not establish equivalence, and tenant isolation
  ({{security}}) depends on this. An endorsed `jwks_uri` MUST NOT
  select, override, or provide a fallback for the configured key
  source. The AS MUST NOT retrieve the endorsed `jwks_uri` under this
  policy; the endorsed value is compared, never fetched.
* **Publisher-authorized key selection:** use the endorsed `jwks_uri`.
  The `issuer` MUST be an HTTPS URL and `jwks_uri` MUST have the same
  origin {{RFC6454}}. This origin check neither isolates tenants sharing
  an origin nor establishes trust in an issuer name. A non-HTTPS issuer
  has no HTTPS origin binding and so requires AS-configured attester
  trust.

The AS MUST use exact, case-sensitive string comparison, without URI
normalization, for issuer identifiers, for client identifiers, and when
comparing endorsed `jwks_uri` values with configured source URIs and
aliases. An alternative spelling of a location requires an explicit
alias, and origin comparison does not change identifier comparison.

Key selection MUST bind a key to the client identifier, issuer,
selected key source, and applicable trust policy, so that a key
selected under one entry never verifies an attestation evaluated under
another; `kid` alone or a union of keys from different entries is
insufficient. The binding applies when a key is selected, not when it
is fetched, so a shared HTTP cache keyed by JWK Set URL is compatible
with it. The AS MUST ignore the `jku`, `x5u`, `x5c`, and `jwk` JOSE
header parameters for key selection under this profile and MUST
resolve only `kid` against the selected source.

A key is eligible when all of the following hold:

* it is the only key in the selected JWK Set whose `kid` equals the
  header `kid` by octet comparison;
* it is an asymmetric public key whose type is consistent with the
  header `alg`;
* its `use`, if present, is `sig`;
* its `key_ops`, if present, includes `verify`; and
* its `alg`, if present, equals the header `alg`.

More than one key matching the `kid` is a failure; the AS MUST NOT try
candidate keys in turn.

When retrieving a JWK Set or client metadata, the AS MUST authenticate
the HTTPS server and MUST NOT follow redirects. Bounding response size
and request time, and blocking prohibited network destinations, are
local defenses; see {{security}}. The AS SHOULD advertise
`client_attestation_signing_alg_values_supported` consistent with the
algorithm restrictions in step 3 of {{as-processing}}
({{ATTEST, Section 8}}).

## Errors {#errors}

Endorsement validation covers these parts of {{as-processing}}:

* selecting a permitted endorsement in step 2;
* selecting the key source and resolving `kid` in step 3 under
  {{key-resolution}}, including an endorsed `jwks_uri` that matches
  neither the configured key source nor a configured alias; and
* the case where no eligible key is available after any refresh
  permitted by {{updates}}.

How a failure is reported depends on the role the Client Attestation
plays in the request.

Attestation is the client authentication method:
: An endorsement validation failure MUST produce
  `invalid_client_attestation`. {{ATTEST, Section 7.4}} defines that
  code alongside the more general `invalid_client`; this profile
  requires the specific code so the response identifies the Client
  Attestation, not another client credential, as the cause. The code
  does not distinguish endorsement failures from other attestation
  failures. The response MUST NOT expose policy details.

  This profile does not change the HTTP status code an endpoint assigns
  to a client authentication failure. The token endpoint responds 400
  by default and requires 401 only when the client authenticated
  through the `Authorization` header field {{RFC6749, Section 5.2}},
  which a Client Attestation does not use. The introspection endpoint
  requires 401 {{RFC7662, Section 2.3}}. Other endpoints follow their
  own specifications.

  A client library that recognizes only `invalid_client` treats this
  code as an unrecognized failure rather than a credential failure.

Attestation is an additional security signal:
: Where the Client Attestation accompanies another client
  authentication method ({{ATTEST, Section 7.6}}), an endorsement
  validation failure leaves no attestation signal for that request.
  The AS MUST NOT treat the failed attestation as a satisfied signal.
  If the deployment requires an attestation alongside that method, the
  request fails. A server signals that requirement by advertising
  `client_attestation_pop_methods_supported` without the value `none`;
  a list containing `none` leaves the attestation optional. Where the
  attestation is optional, whether the request proceeds on the
  companion method alone is AS policy.

Whenever an endorsement validation failure causes the AS to reject the
request, the AS MUST return `invalid_client_attestation`, whether the
Client Attestation served as the client authentication method or as an
additional security signal.

A fresh attestation does not correct an endorsement validation failure
caused by disagreement between the endorsement and AS configuration,
such as an endorsed `jwks_uri` matching neither the configured key
source nor a configured alias ({{key-resolution}}). Because the
response deliberately carries no policy detail, a client cannot tell
that case from one a fresh attestation would fix, or from a
transient retrieval failure ({{updates}}) that a later presentation
can clear. The client publisher and the AS operator
resolve a configuration disagreement outside the protocol, for example
under the trust agreement ({{profile-selection}}), not by client retry.

Everything else keeps its own error. Signature verification with a
resolved key and the remaining attestation and proof checks follow
{{ATTEST, Section 7.4}}, including challenge and freshness responses. A
companion client authentication method that fails, or that
authenticates a different client identifier, produces the error defined
by its own specification. Other metadata-discovery, registration,
authentication, and grant errors follow their base specifications. The
no-fallback rule in {{acceptance}} applies.

# Updates and Withdrawal {#updates}

A publisher withdraws an endorsement by removing it from the
authoritative client metadata ({{metadata}}). The removal reaches the
AS as cached copies expire (below), binds from the next presentation
once retrieved ({{endorsement-changes}}), and leaves issued grants in
place unless separately revoked ({{existing-grants}}).

## Cache Freshness and Removal

The AS MUST:

* enforce configured finite maximum ages for cached endorsement metadata and
  JWK Sets, applying CIMD and HTTP caching constraints {{RFC9111}} when
  stricter; and
* revalidate or refresh expired entries before use, rejecting stale
  entries if that operation fails.

Configured maximum ages bound withdrawal latency: a withdrawn
endorsement or key can remain acceptable until the applicable age
expires. The AS operator chooses maximum ages that keep this latency
within the deployment's security requirements; short ages, for example
one hour, keep it small. This profile specifies no ceiling, so a
publisher cannot predict withdrawal latency from the protocol alone; a
deployment that needs a predictable bound states one in its trust
agreement. Fresh entries need not be retrieved on each request. These
limits expire cached copies, not authoritative client registrations.

On an unknown `kid`, the AS SHOULD refresh the selected key source's
JWK Set once and retry key selection, subject to rate limits. The AS
MUST rate-limit these refreshes per selected key source, independently
of `kid`, and MUST reject the attestation if no eligible key is
available. Where several clients or publishers endorse one key source,
the AS SHOULD also limit refreshes per endorsing client and per
publisher, so that no client or publisher can exhaust another's
allowance. Rate-limit parameters are deployment-specific. An `iss`
matching no endorsement MUST NOT cause a client-metadata refresh; the
metadata maximum age bounds the delay before a newly published
endorsement takes effect, as it bounds withdrawal.

On observing that a CIMD or a selected JWK Set has been removed (HTTP
404 or 410), the AS MUST stop using previously cached endorsements or
keys from that document, and MUST NOT use them again unless a later
retrieval of that document succeeds. A retrieval failure that is not a
removal, such as a timeout or a 5xx status, does not by itself
invalidate an unexpired cached copy. While the AS serves an unexpired
cache, it cannot observe a removal. Deleting a client registration
removes its endorsements.

## Endorsement and Key Changes {#endorsement-changes}

Once the AS has retrieved or stored a metadata or key update, it MUST
use it on the next presentation. Removing an endorsement or a
verification key, or publishing an empty list, prevents acceptance
under that entry or key, including for attestations issued before the
update. Denial by AS policy MUST take effect immediately on subsequent
requests, without waiting for cache expiration.

For planned key rotation, the attester adds the new key to the JWK Set
the AS reads (the configured key source or the endorsed `jwks_uri`,
{{key-resolution}}) and waits for JWK Set cache lifetimes to elapse
before signing with it. It keeps the old key published until
attestations signed with it expire, since removing the key rejects them.
A new key location takes effect only after the publisher updates the
endorsement and metadata caches refresh. Under AS-configured attester
trust, it also fails endorsement validation until the AS configures a
matching alias or updates its configured key source, so the attester
and publisher coordinate the move with the AS operator.

## Existing Grants {#existing-grants}

Endorsement withdrawal is prospective: it prevents future client
authentication under the removed endorsement but does not revoke
existing grants or access tokens. Refresh requests that require a
Client Attestation are checked again under {{processing}}. A deployment
that requires termination separately revokes the affected grants and
their tokens and stops further refresh issuance. Introspection
{{RFC7662}} then reports the revoked tokens inactive; offline
validation needs a separate revocation mechanism or token expiration.

# Security Considerations {#security}

The considerations in {{ATTEST}}, {{CIMD}}, and {{RFC8725}} apply.

* **Publisher compromise:** whoever controls a CIMD host or a client's
  registration administration can change endorsements, within AS
  policy. The protocol gives the AS no signal of this, so operators
  generally monitor endorsement changes and treat new attesters as
  policy changes. A separately specified signed-metadata mechanism with
  independently trusted signing keys could bind publisher intent apart
  from the HTTPS host; this profile defines none.
* **Attester compromise:** a client or tenant of a shared attester
  could obtain attestations naming another, so the attester needs
  issuance controls that prevent this. Under AS-configured attester
  trust, the agreement check in {{key-resolution}} isolates tenants
  only if the configured key source and its aliases preserve the
  endorsed tenant scope, which is the AS operator's responsibility.
* **Key retrieval:** under publisher-authorized key selection, an
  authorized publisher chooses both the issuer and the key location,
  and so can direct an AS request to an origin of its choosing.
  {{key-resolution}} constrains that request, deliberately extending
  CIMD's no-automatic-redirect rule to key retrieval, but does not
  remove it. An AS can also bound response size and request time and
  block prohibited network destinations. Endorsed URLs remain subject
  to server-side request forgery (SSRF) defenses; endorsement does not
  make a network location safe.
* **Withdrawal latency:** cached copies outlast withdrawal
  ({{updates}}), so removing a key at its origin is not instantaneous
  revocation. Urgent incidents require denial by AS policy
  ({{endorsement-changes}}) or another revocation channel.
* **Omitted attestation:** `client_attesters` does not itself require
  attestation, so a thief of the client's other credentials can
  authenticate without one unless the deployment requires attestation,
  either through an attestation `token_endpoint_auth_method` or by
  advertising `client_attestation_pop_methods_supported` without `none`
  ({{errors}}); the latter keeps mutual TLS or `private_key_jwt` in
  place. A registration access token can change
  `token_endpoint_auth_method` or `jwks` through {{RFC7592}} even where
  it cannot change `client_attesters` ({{registered}}), so a deployment
  relying on endorsement restricts those changes too or requires
  attestation by AS policy.
* **Unscoped endorsement:** an endorsement carries no audience. Under
  publisher-authorized key selection, one endorsement selects the
  attester and its keys at every AS whose policy covers that publisher,
  so a compromised attester authenticates the client at all of them
  until the endorsement is withdrawn.
* **Privacy:** public metadata exposes client-to-attester relationships.
  Such metadata need not enumerate instances or their keys, and this
  profile gives no reason to do so; it requires no stable instance
  identifier. Caching reduces the request-timing information observable
  at metadata and key sources.

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

This example is informative. The AS has the following configuration,
which is AS policy, not protocol metadata:

| Setting | Value |
|---|---|
| Permitted origin for CIMD retrieval | `https://platform.example` |
| Independently trusted attester | `https://attester.example/tenant/acme` |
| Key-trust policy for that attester | AS-configured attester trust |
| Configured key source for that attester | `https://attester.example/tenant/acme/jwks` |
| Maximum metadata and key cache ages | 3600 seconds each |
{: title="AS configuration for this example"}

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

The configured key source publishes this illustrative JWK Set, whose
signing key is distinct from the Client Instance Key in `cnf.jwk`:

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
      "x": "9iHztmYIeKeyta94k1y5Dya5cab3-_H_yw_3v0p6K80",
      "y": "NsrICJ4xFvOs5xaDM4sF1yDijCxN5LWailjw5EsERwI"
    }
  }
}
~~~

1. The Client Instance proves to the attester that it is authorized
   to use this client identifier and holds its instance key.
2. The attester issues the Client Attestation shown above.
3. After user authorization, the Client Instance redeems its code with
   that `client_id`, the Client Attestation, and a combined
   Demonstrating Proof of Possession (DPoP) proof {{RFC9449}}.
4. The AS validates the CIMD, endorsement, attestation, proof, and
   grant before issuing the access token.

There is one client metadata document, not one per installation. An
endorsement for this client does not let the attester authenticate
another client, even if both use the same attestation service. The
flow does not require `client_instance_id` or an `act` claim.

Under AS-configured attester trust, keys come only from the configured
key source, and the endorsed `jwks_uri` has to equal it, as it does
here. An attestation from an unendorsed issuer, an endorsement naming
the trusted issuer with a different key location, or a `kid` that
resolves to no key in the configured key source produces the response
below. {{RFC6749, Section 5.2}} requires 401 only for a client that
authenticated through the `Authorization` header field, which this
client does not use, so the example shows the default 400:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{"error": "invalid_client_attestation"}
~~~

Had the AS instead authorized `https://platform.example` for
publisher-authorized key selection and never configured trust for the
issuer, the same document would also succeed, with keys retrieved
from the endorsed `jwks_uri`, which shares the issuer's origin. An
entry whose `jwks_uri` had a different origin from the issuer would
then fail endorsement validation with the same error.

# Registered Client Example {#registered-example}

This example is informative. It repeats {{example}}, with the same AS
configuration, for an opaque client identifier. Under AS-configured
attester trust, neither endorsement nor key selection depends on the
identifier's shape: `client_attesters` travels with the client's
metadata, and the endorsement names the key location outright, so no
origin is derived from the client identifier. Publisher authorization
does differ between the two forms ({{trust}}), but not here, because
the AS trusts this attester independently.

An authenticated, authorized administrator registers the client, for
example through {{RFC7591}}, and the AS returns this client information:

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
      "x": "9iHztmYIeKeyta94k1y5Dya5cab3-_H_yw_3v0p6K80",
      "y": "NsrICJ4xFvOs5xaDM4sF1yDijCxN5LWailjw5EsERwI"
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
