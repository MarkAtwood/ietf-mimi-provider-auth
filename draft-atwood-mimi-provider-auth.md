---
title: "Provider Authentication for More Instant Messaging Interoperability (MIMI)"
abbrev: "MIMI Provider Authentication"
category: std
docname: draft-atwood-mimi-provider-auth-00
submissiontype: IETF
number:
date:
v: 3
ipr: trust200902
area: "Applications and Real-Time"
workgroup: "More Instant Messaging Interoperability"
venue:
  group: "More Instant Messaging Interoperability"
  type: "Working Group"
  mail: "mimi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mimi/"
  github: "MarkAtwood/mimi-provider-auth"
keyword:
 - MIMI
 - federation
 - HTTP Message Signatures
 - DANE
 - client authentication

author:
 -
    fullname: Mark Atwood
    organization: Review Commit, LLC
    city: Seattle
    region: WA
    country: US
    email: mark@reviewcommit.com

normative:
  RFC1035:
  RFC4033:
  RFC5234:
  RFC5280:
  RFC6698:
  RFC7671:
  RFC8032:
  RFC8552:
  RFC9110:
  RFC9421:
  RFC9525:
  RFC9530:
  I-D.ietf-mimi-protocol:
  I-D.ietf-dance-client-auth:

informative:
  RFC6125:
  RFC6335:
  RFC6376:
  RFC7942:
  RFC8792:
  RFC9457:
  CRP:
    title: "Chrome Root Program Policy, Version 1.8"
    target: https://googlechrome.github.io/chromerootprogram/crp/policy
    author:
      org: Google
    date: 2026-02-05
  LE-CLIENTAUTH:
    title: "Ending TLS Client Authentication Certificate Support in 2026"
    target: https://letsencrypt.org/2025/05/14/ending-tls-client-authentication/
    author:
      org: Let's Encrypt
    date: 2025-05-14

--- abstract

The MIMI protocol requires each provider to authenticate the provider that
sends it a request. It specifies mutually authenticated TLS for this, but
not where the client certificates come from or which trust anchors check
them. The public Web PKI has stopped issuing such certificates. This
document defines how MIMI providers authenticate each other without it.
The mechanism every provider implements is a key published in DNS under
the provider's domain, used to sign each request with HTTP Message
Signatures. DANE client certificates, client certificates from configured
PKIX trust anchors, and locally configured trust lists are defined as
additional mechanisms that a provider can accept by local policy.

--- middle

# Introduction

In the MIMI protocol {{I-D.ietf-mimi-protocol}}, providers send each other
HTTPS requests: to fetch key material, to submit messages to a room's hub,
and for the hub to fan messages out to the other providers in the room.
Every one of those requests is acted on because of who sent it. A hub
accepts a Commit because it came from a provider whose users are in the
room. A follower accepts a notification because it came from the room's
hub. If the source of a request is not authenticated, any host on the
Internet can claim to be any provider.

Section 4.1 of {{I-D.ietf-mimi-protocol}} says the TLS connection "MUST be
mutually authenticated" and that the certificates "MUST authenticate the
source and target provider domains". It does not say who issues the
source provider's client certificate or which trust anchors the target
uses to check it. {{webpki}} explains why the answer cannot be "the
public Web PKI". As a result, every deployment today either runs a
private CA and asks each peer to install it, or does not authenticate
peers at all. Neither is open federation, and neither interoperates
without out-of-band setup that the protocol does not describe.

This document fills that gap. It defines:

* A mandatory-to-implement mechanism ({{dnskey}}): the source provider
  publishes a public key in DNS under its own domain and signs each
  request with HTTP Message Signatures {{RFC9421}}. TLS is
  server-authenticated only, using the provider's ordinary server
  certificate. This is the same trust model DKIM {{RFC6376}} uses for
  email.

* Three optional mechanisms: DANE client certificates ({{dane}}), PKIX
  client certificates checked against trust anchors the receiver
  configures ({{pkix}}), and trust lists of keys or certificates
  configured per peer ({{lists}}).

* How a receiver combines them by local policy ({{policy}}), and what it
  answers when authentication fails ({{errors}}).

{{protocol-text}} proposes replacement text for Section 4.1 of
{{I-D.ietf-mimi-protocol}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms "provider", "hub" and "follower" as defined
in {{I-D.ietf-mimi-protocol}}.

Source provider:
: The provider that sends a MIMI protocol request.

Target provider:
: The provider that receives it.

Source domain:
: The domain name of the source provider, as named in the request's
  `From` header field (Section 4.1 of {{I-D.ietf-mimi-protocol}}).

Authenticated domain:
: The domain name that a mechanism in this document proves the request
  came from.

# The Web PKI Has Left the Federation {#webpki}

The obvious source of a provider's client certificate used to be the
public CA that issues its server certificate. For years those CAs added
the id-kp-clientAuth extended key usage to server certificates as a matter
of course, and servers presented them as client certificates to their
peers. SMTP with mutual TLS, XMPP server-to-server federation, and a
long list of HTTP APIs relied on this. It was never a promise. It was a
side effect, and it has now been removed.

The Chrome Root Program decides which CAs Chrome trusts, and so,
in practice, which CAs can sell a server certificate. Version 1.8 of its
policy {{CRP}} states the goal plainly: "to align all PKI hierarchies
included in the Chrome Root Store on the principle of serving only TLS
server authentication use cases". It then sets dates:

* Subordinate CA certificates disclosed on or after 15 June 2026 may
  assert only id-kp-serverAuth.

* Subscriber certificates issued on or after 15 March 2027 may assert
  only id-kp-serverAuth.

* From 15 June 2026, hierarchies found in violation are phased out of
  Chrome, 90 days after the violation is detected. Multi-purpose roots
  are being removed from the store.

The enforcement is worth reading twice. Chrome does not forbid a CA from
issuing client certificates. It only stops trusting the CA's server
certificates if it does. Since server certificates are the CA's business,
that is not a choice. Let's Encrypt removed the client-authentication
EKU from its default profile on 11 February 2026 and retired its last
profile offering it on 8 July 2026 {{LE-CLIENTAUTH}}. The other public CAs
are following the same calendar.

None of this was aimed at MIMI, or at SMTP, or at XMPP. Those protocols
were not consulted so much as noticed, the way a road crew notices
whatever happens to be parked on the road. The policy is coherent from
the point of view of a browser: a root store with exactly one purpose is
easier to run, and every change to it now breaks only browsers. It is not
a PKI that any other protocol should build on, and its operator has said
so in writing.

So a MIMI provider cannot obtain a publicly trusted client certificate.
What remains is a private CA, installed by hand at every peer, which turns
an open federation into a set of bilateral agreements. This document
takes the other road. A provider already proves control of its domain to
get a server certificate, and a domain-validated certificate is issued on
the strength of DNS. The provider can publish a key in that same DNS and
leave the CA out of the client side entirely.

# Overview {#overview}

The source provider always sends its domain in the `From` header field
(Section 4.1 of {{I-D.ietf-mimi-protocol}}). The target provider runs the
mechanisms it accepts, in an order it chooses. A mechanism either
produces an authenticated domain or fails. The request is authenticated
if some mechanism's authenticated domain equals the source domain.

| Mechanism | Credential | Where the trust comes from | DNSSEC | Section |
|---|---|---|---|---|
| DNS keys | RFC 9421 request signature | DNS, under the source domain | Optional | {{dnskey}} |
| DANE | TLS client certificate | TLSA record, under the source domain | Required | {{dane}} |
| PKIX | TLS client certificate | Trust anchors the receiver configures | No | {{pkix}} |
| Trust lists | Signature or client certificate | Per-peer configuration | No | {{lists}} |
{: title="Provider authentication mechanisms"}

Every provider MUST implement both signing and verification for the DNS
key mechanism. This is the only requirement that exists for
interoperability: without one mechanism that every provider implements,
two providers that each conform could have no mechanism in common.
Implementing it does not oblige a provider to accept it. Which mechanisms
a provider accepts, from which peers, is local policy ({{policy}}).

Responses are not signed. The source provider authenticates the target
provider through the TLS server certificate, as with any HTTPS request.

# DNS-Published Keys with HTTP Message Signatures {#dnskey}

## Key Records {#key-record}

A provider publishes each signing key as a DNS TXT record {{RFC1035}} at
the owner name:

~~~
<selector>._mimi.<provider-domain>
~~~

The selector is a single DNS label chosen by the provider. It MUST NOT
begin with an underscore. A provider MAY publish several keys under
different selectors, for rotation or for separate servers.

The record data is a list of tag-value pairs. If the record consists of
several character-strings, they are concatenated without separators
before parsing, as in DKIM {{RFC6376}}. The syntax, in ABNF {{RFC5234}}:

~~~ abnf
key-record = %s"v=MIMI1" *( *WSP ";" *WSP tag ) *WSP [";"] *WSP
tag        = tag-name *WSP "=" *WSP tag-value
tag-name   = ALPHA *( ALPHA / DIGIT / "_" )
tag-value  = *( %x21-3A / %x3C-7E )   ; any VCHAR except ";"
WSP        = SP / HTAB
~~~

The version tag `v=MIMI1` MUST come first. A record that does not begin
with it is not a MIMI key record and is ignored. This specification
defines two further tags:

k:
: The key type. This document defines `ed25519`, an Ed25519 public key
  {{RFC8032}}. Verifiers MUST support `ed25519`. Other key types MAY be
  defined later.

p:
: The public key, in base64 with padding. For `ed25519` it is the 32-byte
  public key. An empty value means the key under this selector has been
  revoked.

Verifiers MUST ignore tags they do not recognize, so that later
specifications can add tags.

An example, using the Ed25519 test key from Appendix B.1.4 of
{{RFC9421}}:

~~~ dns
s1._mimi.a.example. 3600 IN TXT (
    "v=MIMI1; k=ed25519; "
    "p=JrQLj5P/89iXES9+vFgrIy29clF9CC/oPPsw3c5D0bs=" )
~~~

## Signing Requests {#signing}

The source provider signs each MIMI protocol request with HTTP Message
Signatures {{RFC9421}}. The signature:

* MUST cover the derived components `@method` and `@target-uri`, and the
  `from` header field. Covering `from` binds the signature to the source
  domain; without it, a valid signature from one provider could be
  attached to a request that claims to come from another.

* MUST cover the `content-digest` field {{RFC9530}} if the request has
  content. The `Content-Digest` field SHOULD use `sha-256`.

* MUST include the `created` parameter and the `keyid` parameter, whose
  value is the selector of the key that made the signature.

* SHOULD include the `tag` parameter with the value `mimi`, so that a
  verifier can find the MIMI signature on a request that carries others.

* MAY cover other components and include other parameters, such as
  `expires` or `nonce`.

The signer MAY include the `alg` parameter. If present, it MUST match the
key type in the key record (`ed25519` for `k=ed25519`).

The `@target-uri` is the URL the source provider requests, taken from the
target provider's directory (Section 5.1 of {{I-D.ietf-mimi-protocol}}).

{{example}} shows a complete signed request.

## Verifying Requests {#verifying}

To verify a request, the target provider:

1. Takes the source domain from the `From` header field.

2. Selects a signature from the `Signature-Input` field whose `tag`
   parameter is `mimi`, or, if there is none, any signature that covers
   the components listed in {{signing}}. A verifier MAY limit how many
   signatures it tries.

3. Checks that the signature covers the components required by
   {{signing}} and has `created` and `keyid` parameters.

4. Checks `created`. Verifiers SHOULD reject a signature created more than
   300 seconds in the past or more than 60 seconds in the future. They
   MAY configure other limits. If `expires` is present and has passed, the
   signature MUST be rejected.

5. Queries TXT records at `<keyid>._mimi.<source-domain>`, and uses the
   first record that is a MIMI key record ({{key-record}}). If there is
   none, or its `p` tag is empty, verification fails.

6. If the request has content, checks that `Content-Digest` matches it
   {{RFC9530}}.

7. Verifies the signature over the signature base {{RFC9421}} with the
   public key.

If all steps succeed, the authenticated domain is the source domain.

A verifier SHOULD use a DNSSEC-validating resolver {{RFC4033}} when one is
available. If validation of the key record's response is "bogus",
verification fails. A verifier MAY, by local policy, require the response
to be "secure".

A verifier MAY cache key records for their DNS TTL. It SHOULD NOT cache a
negative or failed lookup for longer than the negative-caching TTL, so
that a newly published key works promptly.

## Key Rotation and Revocation {#rotation}

To rotate keys, a provider publishes the new key under a new selector,
waits at least the TTL of the record, starts signing with the new
selector, and then removes the old record. To revoke a key at once, it
replaces the old record's `p` value with an empty value; this takes effect
as caches expire. A short TTL on key records shortens both delays.

# DANE Client Certificates {#dane}

A provider that prefers TLS-layer authentication MAY present a client
certificate that is authenticated by a DANE TLSA record, as specified in
{{I-D.ietf-dance-client-auth}}. It uses Format 1 of that specification,
with the service label `_mimi`:

~~~
_mimi._client.<provider-domain>
~~~

The TLSA record data and its matching rules are as in {{RFC6698}} and
{{RFC7671}}. The source domain's zone MUST be DNSSEC-signed, and the
target provider MUST treat the TLSA records as usable only if their
validation status is "secure"; that is what DANE's security rests on.

The target provider requests a client certificate in the TLS handshake.
It may not know the source domain until it reads the `From` header field
of the first request. It then looks up the TLSA records for that domain
and checks the client certificate against them. If they match, the
authenticated domain is the source domain. The source provider MAY send
the DANE Client Identity TLS extension defined in
{{I-D.ietf-dance-client-auth}} so that the target can fetch the records
during the handshake.

# PKIX Client Certificates from Configured Trust Anchors {#pkix}

Some providers operate where a PKIX hierarchy that still issues client
certificates is required or already deployed: government PKIs, sector and
enterprise PKIs, and federation operators that run a CA for their
members. Such a provider MAY present a client certificate from that
hierarchy, which is the mechanism Section 4.1 of
{{I-D.ietf-mimi-protocol}} describes today.

The certificate MUST contain a subjectAltName dNSName that is the provider
domain. If it contains an extended key usage extension, that extension
MUST include id-kp-clientAuth.

The target provider validates the certificate path per {{RFC5280}} against
trust anchors it configures for this purpose, applying whatever
certificate policy constraints its deployment requires. It matches the
provider domain against the certificate as a DNS-ID per {{RFC9525}}; it
does not use the subject common name. If both succeed, the authenticated
domain is the matched DNS name.

A target provider SHOULD NOT use a public Web PKI root store, such as the
one its operating system or TLS library ships for server authentication,
as the trust anchors for this mechanism. The CAs in those stores no longer
issue client certificates ({{webpki}}), and the ones that still might are
the ones the store is about to drop.

# Trust Lists {#lists}

A target provider MAY configure, per peer domain, the keys or certificates
it accepts from that peer:

* An Ed25519 public key and its selector. Requests are verified as in
  {{verifying}}, except that the key comes from configuration rather than
  DNS.

* A certificate, a public key, or a CA certificate. The client certificate
  is checked against it at the TLS layer.

The authenticated domain is the peer domain the entry is configured for.
This is how most deployments authenticate peers today, and it remains
useful for private federations and testing. It requires out-of-band setup
for every peer, so a provider that relies only on it cannot reach
providers it has not been configured for.

# Local Policy {#policy}

Each target provider decides which mechanisms it accepts, in which order,
and whether the choice differs per peer. Some examples:

* An open provider accepts DNS keys from anyone and the other mechanisms
  from peers that use them.

* A provider in a regulated environment accepts only PKIX certificates
  under its sector's trust anchors, and DNS keys only with DNSSEC.

* A private federation accepts only trust-listed peers.

A target provider MAY accept a request without authentication during a
transition, for example to log which peers would fail before it starts
refusing them. This is outside this specification, and such a provider
gets none of the protection in {{security}}.

Whatever mechanism is used, the authenticated domain MUST equal the source
domain. This matters especially for the TLS-layer mechanisms: an HTTP
connection authenticated with one provider's client certificate MUST NOT
be used to authenticate a request whose `From` header field names another
provider. Without this check, a provider with any valid credential could
impersonate any other.

Authentication says only which provider sent the request. Whether that
provider is allowed to make it, for example whether a notification really
comes from the room's hub, is decided by {{I-D.ietf-mimi-protocol}} and by
the room's policy.

# Errors {#errors}

If no accepted mechanism authenticates the source domain, the target
provider answers with status 403 (Forbidden) {{RFC9110}}. It MAY include an
`Accept-Signature` field (Section 5.1 of {{RFC9421}}) describing the
signature it requires, and MAY include a problem details body {{RFC9457}}
saying why authentication failed.

Status 401 (Unauthorized) is not used, because {{RFC9110}} requires a 401
response to carry a `WWW-Authenticate` challenge, and none of these
mechanisms is an HTTP authentication scheme.

A target provider SHOULD NOT say in its response which mechanisms it
tried or why each one failed beyond what the source provider needs to fix
its configuration.

# Replacement Text for the MIMI Protocol {#protocol-text}

This section proposes replacing the last paragraph of Section 4.1 of
{{I-D.ietf-mimi-protocol}} ("The TLS connection underlying the HTTPS
connection MUST be mutually authenticated...") with:

> The TLS connection underlying the HTTPS connection authenticates the
> target provider domain, according to [RFC9525]. The target provider
> authenticates the source provider domain using one of the mechanisms
> in [this document]. Every provider implements the DNS key mechanism of
> [this document], Section 5; which mechanisms a provider accepts is local
> policy.

It also replaces the reference to {{RFC6125}}, which {{RFC9525}}
obsoletes. The second sentence of the paragraph in Section 9 of
{{I-D.ietf-mimi-protocol}} that relies on "mutually-authenticated TLS"
would change to refer to provider authentication generally.

# Security Considerations {#security}

## DNS as the Root of Trust

Without DNSSEC, the DNS key mechanism is as strong as the target
provider's view of DNS for the source domain. An attacker who can forge
DNS responses to the target, or who controls the source domain's
delegation, can substitute a key. This is the same exposure DKIM has. It
is also the exposure domain-validated certificates have: a CA validates
domain control through DNS or through HTTP to an address DNS gave it. An
attacker who controls DNS for a domain can get a public server
certificate for it. The DNS key mechanism removes the CA from the client
side; it does not remove a protection that the Web PKI provided.

With DNSSEC, the key is authenticated by the zone's signature, and an
off-path attacker cannot forge it. Deployments that need that assurance
can require it ({{verifying}}) or use DANE ({{dane}}).

## Replay

A signature is valid for any copy of the request within the `created`
window. An attacker who records a signed request can replay it in that
window, to the same target, provided the attacker can see the request at
all; it is sent over TLS. Signers MAY include a `nonce` parameter and
verifiers MAY track nonces for the window. Most MIMI requests are either
idempotent or carry MLS messages that the hub will reject the second
time, which limits the damage.

## Key Compromise

A provider whose signing key is compromised revokes it as described in
{{rotation}}. Until caches expire, the attacker can sign requests as the
provider. The record's TTL bounds that time.

## Downgrade Between Mechanisms

An attacker cannot downgrade a target provider from a mechanism it
requires to one it does not accept, since the target decides alone. A
target that accepts several mechanisms is as strong as the weakest of
them for any peer it accepts them from.

## Same-Connection Impersonation

See the requirement in {{policy}} that the authenticated domain equal the
source domain on every request, including requests on a connection whose
client certificate was already accepted.

# Privacy Considerations

Key lookups tell the target provider's DNS resolver, and anyone who can
observe its queries, which source providers the target provider hears
from. This is the same information the source provider's TLS connection
already reveals to a network observer, and it is about providers, not
users. Caching reduces it.

# IANA Considerations

## Underscored and Globally Scoped DNS Node Names

IANA is requested to add the following entry to the "Underscored and
Globally Scoped DNS Node Names" registry {{RFC8552}}:

| RR Type | _NODE NAME | Reference |
|---|---|---|
| TXT | _mimi | RFC XXXX |
{: title="Underscored node name"}

## Service Name

IANA is requested to register the following in the "Service Name and
Transport Protocol Port Number Registry" {{RFC6335}}, for the DANE client
identity owner name in {{dane}}:

Service Name:
: mimi

Transport Protocol:
: tcp

Assignee:
: IESG

Contact:
: IETF Chair

Description:
: More Instant Messaging Interoperability (MIMI) protocol

Reference:
: RFC XXXX

Port Number:
: none

--- back

# Example Signed Request {#example}

The source provider a.example sends a notification to b.example. The
content (`example MIMI notify body`, shown as text for brevity) stands in
for a real MIMI message. The key is the one shown in {{key-record}}, whose
private part is the Ed25519 test key from Appendix B.1.4 of {{RFC9421}}.

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

POST /v1/notify/a.example/r/clubhouse HTTP/1.1
Host: b.example
From: mimi@a.example
Content-Type: application/mimi-protocol
Content-Digest: \
  sha-256=:ltvFi/ijqMH79/ThrwZ/JykItjIYFA06v8rfqBdj7dY=:
Signature-Input: mimi=("@method" "@target-uri" "content-digest" \
  "from");created=1794700800;keyid="s1";tag="mimi"
Signature: mimi=:OBXnNmTPY+Gcxbn+BKcfqCpK7GrkCGO+Ab0/PS956byMzzcArSV\
  EeJWA95WiYjfLJ9vMsEaCRab3geIFr5FpCw==:

example MIMI notify body
~~~

The signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@method": POST
"@target-uri": https://b.example/v1/notify/a.example/r/clubhouse
"content-digest": sha-256=:ltvFi/ijqMH79/ThrwZ/JykItjIYFA06v8rfqBdj7dY=:
"from": mimi@a.example
"@signature-params": ("@method" "@target-uri" "content-digest" "from")\
  ;created=1794700800;keyid="s1";tag="mimi"
~~~

The target provider b.example reads `a.example` from `From`, looks up
`s1._mimi.a.example`, and verifies the Ed25519 signature over this base.
Ed25519 signatures are deterministic, so an implementation can check its
signer against these bytes.

# Open Issues

* Section 4.1 of {{I-D.ietf-mimi-protocol}} has an editor's note asking
  whether `From` is the right field for the source domain. If the WG
  replaces it, the signature covers the new field instead.

* The directory document could advertise which mechanisms a provider
  accepts, so a source provider knows before its first request. This
  document does not define that.

* Keys could also be published in the directory document instead of DNS.
  That would rest their trust on the target's TLS server certificate,
  which is to say on the Web PKI, which is the thing this document is
  trying to stop depending on for the client side.

* Whether nonces should be required rather than optional.

# Implementation Status

{::boilerplate rfc7942info}

mimid (Review Commit, LLC): an open-source MIMI provider. It implements
RFC 9421 signing and verification with the profile in {{signing}}, with
keys from trust lists ({{lists}}). The DNS key mechanism, PKIX and DANE
client certificates are being implemented. It signs every outbound request
and can either refuse or log unauthenticated inbound requests.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
