---
title: "Discovery of Designated Resolvers"
abbrev: DDR
docname: draft-ietf-add-ddr-latest
date:
category: std

ipr: trust200902
area: Internet
workgroup: ADD

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: T. Pauly
    name: Tommy Pauly
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: tpauly@apple.com
  -
    ins: E. Kinnear
    name: Eric Kinnear
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: ekinnear@apple.com
  -
    ins: C. A. Wood
    name: Christopher A. Wood
    org: Cloudflare
    street: 101 Townsend St
    city: San Francisco
    country: United States of America
    email: caw@heapingbits.net
  -
    ins: P. McManus
    name: Patrick McManus
    org: Fastly
    email: mcmanus@ducksong.com
  -
    ins: T. Jensen
    name: Tommy Jensen
    org: Microsoft
    email: tojens@microsoft.com

--- abstract

This document defines Discovery of Designated Resolvers (DDR), a
mechanism for DNS clients to use DNS records to discover a resolver's encrypted
DNS configuration. This mechanism can be used to move from unencrypted DNS to
encrypted DNS when only the IP address of a resolver is known. This mechanism is
designed to be limited to cases where unencrypted resolvers and their designated
resolvers are operated by the same entity or cooperating entities. It can also be used
to discover support for encrypted DNS protocols when the name of an encrypted resolver
is known.

--- middle

# Introduction

When DNS clients wish to use encrypted DNS protocols such as DNS-over-TLS (DoT)
{{!RFC7858}} or DNS-over-HTTPS (DoH) {{!RFC8484}}, they require additional
information beyond the IP address of the DNS server, such as the resolver's
hostname, non-standard ports, or URL paths. However, common configuration
mechanisms only provide the resolver's IP address during configuration. Such
mechanisms include network provisioning protocols like DHCP {{?RFC2132}} and
IPv6 Router Advertisement (RA) options {{?RFC8106}}, as well as manual
configuration.

This document defines two mechanisms for clients to discover designated
resolvers using DNS server Service Binding (SVCB, {{I-D.ietf-dnsop-svcb-https}})
records:

1. When only an IP address of an Unencrypted Resolver is known, the client
queries a special use domain name (SUDN) {{!RFC6761}} to discover DNS SVCB
records associated with one or more Encrypted Resolvers the Unencrypted
Resolver has designated for use when support for DNS encryption is
requested ({{bootstrapping}}). 

2. When the hostname of an Encrypted Resolver is known, the client requests
details by sending a query for a DNS SVCB record. This can be used to discover
alternate encrypted DNS protocols supported by a known server, or to provide
details if a resolver name is provisioned by a network ({{encrypted}}).

Both of these approaches allow clients to confirm that a discovered Encrypted
Resolver is designated by the originally provisioned resolver. "Designated" in
this context means that the resolvers are operated by the same entity or
cooperating entities; for example, the resolvers are accessible on the same
IP address, or there is a certificate that claims ownership over both resolvers.

## Specification of Requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{?RFC2119}} {{?RFC8174}} when, and only when,
they appear in all capitals, as shown here.

# Terminology

This document defines the following terms:

DDR:
: Discovery of Designated Resolvers. Refers to the mechanisms defined
in this document.

Designated Resolver:
: A resolver, presumably an Encrypted Resolver, designated by another resolver
for use in its own place. This designation can be verified with TLS certificates.

Encrypted Resolver:
: A DNS resolver using any encrypted DNS transport. This includes current
mechanisms such as DoH and DoT as well as future mechanisms.

Unencrypted Resolver:
: A DNS resolver using TCP or UDP port 53.

# DNS Service Binding Records

DNS resolvers can advertise one or more Designated Resolvers that
may offer support over encrypted channels and are controlled by the same
entity.

When a client discovers Designated Resolvers, it learns information such as
the supported protocols and ports. This information is provided in Service
Binding (SVCB) records for DNS Servers. The formatting of these records,
including the DNS-unique parameters such as "dohpath", are defined by
{{!I-D.ietf-add-svcb-dns}}.

The following is an example of an SVCB record describing a DoH server discovered
by querying for `_dns.example.net`:

~~~
_dns.example.net.  7200  IN SVCB 1 example.net. (
     alpn=h2 dohpath=/dns-query{?dns} )
~~~

The following is an example of an SVCB record describing a DoT server discovered
by querying for `_dns.example.net`:

~~~
_dns.example.net  7200  IN SVCB 1 dot.example.net (
     alpn=dot port=8530 )
~~~

If multiple Designated Resolvers are available, using one or more
encrypted DNS protocols, the resolver deployment can indicate a preference using
the priority fields in each SVCB record {{I-D.ietf-dnsop-svcb-https}}.

To avoid name lookup deadlock, Designated Resolvers SHOULD follow the guidance
in Section 10 of {{?RFC8484}} regarding the avoidance of DNS-based references
that block the completion of the TLS handshake.

This document focuses on discovering DoH and DoT Designated Resolvers.
Other protocols can also use the format defined by {{!I-D.ietf-add-svcb-dns}}.
However, if any protocol does not involve some form of certificate validation,
new validation mechanisms will need to be defined to support validating
designation as defined in {{verified}}.

# Discovery Using Resolver IP Addresses {#bootstrapping}

When a DNS client is configured with an Unencrypted Resolver IP address, it
SHOULD query the resolver for SVCB records for "dns://resolver.arpa" before
making other queries. Specifically, the client issues a query for
`_dns.resolver.arpa` with the SVCB resource record type (64)
{{I-D.ietf-dnsop-svcb-https}}.

Because this query is for an SUDN, which no entity can claim ownership over,
the SVCB response MUST NOT use the "." value for the TargetName. Instead,
the domain name used for DoT or used to construct the DoH template MUST be provided.

The following is an example of an SVCB record describing a DoH server discovered
by querying for `_dns.resolver.arpa`:

~~~
_dns.resolver.arpa  7200  IN SVCB 1 doh.example.net (
     alpn=h2 dohpath=/dns-query{?dns} )
~~~

The following is an example of an SVCB record describing a DoT server discovered
by querying for `_dns.resolver.arpa`:

~~~
_dns.resolver.arpa  7200  IN SVCB 1 dot.example.net (
     alpn=dot port=8530 )
~~~

If the recursive resolver that receives this query has one or more Designated
Resolvers, it will return the corresponding SVCB records. When responding
to these special queries for "dns://resolver.arpa", the recursive resolver
SHOULD include the A and AAAA records for the name of the Designated Resolver
in the Additional Answers section. This will allow the DNS client to make
queries over an encrypted connection without waiting to resolve the Encrypted
Resolver name per {{!I-D.ietf-dnsop-svcb-https}}. If no A/AAAA records or SVCB
IP address hints are included, clients will be forced to delay use of the
Encrypted Resolver until an additional DNS lookup for the A and AAAA records
can be made to the Unencrypted Resolver (or some other resolver the DNS client
has been configured to use).

If the recursive resolver that receives this query has no Designated Resolvers,
it SHOULD return NODATA for queries to the "resolver.arpa" SUDN.

## Use of Designated Resolvers

When a client discovers Designated Resolvers from an Unencrypted Resolver IP
address, it can choose to use these Designated Resolvers either automatically,
or based on some other policy, heuristic, or user choice.

This document defines two preferred methods to automatically use Designated
Resolvers:

- Verified Discovery {{verified}}, for when a TLS certificate can
be used to validate the resolver's identity.
- Opportunistic Discovery {{opportunistic}}, for when a resolver is accessed
using a non-public IP address.

A client MAY additionally use a discovered Designated Resolver without
either of these methods, based on implementation-specific policy or user input.
Details of such policy are out of scope of this document. Clients SHOULD NOT
automatically use a Designated Resolver without some sort of validation,
such as the two methods defined in this document or a future mechanism.

## Verified Discovery {#verified}

Verified Discovery is a mechanism that allows automatic use of a
Designated Resolver that supports DNS encryption that performs a TLS handshake.

In order to be considered a verified Designated Resolver, the
TLS certificate presented by the Designated Resolver MUST contain the IP address
of the designating Unencrypted Resolver in a subjectAltName extension.
If the certificate can be validated, the client SHOULD use the discovered
Designated Resolver for any cases in which it would have otherwise used the
Unencrypted Resolver. If the Designated Resolver has a different IP
address than the Unencrypted Resolver and the TLS certificate does not cover the
Unencrypted Resolver address, the client MUST NOT automatically use the discovered
Designated Resolver. Additionally, the client SHOULD suppress any further queries for
Designated Resolvers using this Unencrypted Resolver for the length of
time indicated by the SVCB record's Time to Live (TTL).

If the Designated Resolver and the Unencrypted Resolver share an IP
address, clients MAY choose to opportunistically use the Designated Resolver even
without this certificate check ({{opportunistic}}).

If resolving the name of a Designated Resolver from an SVCB record yields an
IP address that was not presented in the Additional Answers section or ipv4hint
or ipv6hint fields of the original SVCB query, the connection made to that IP
address MUST pass the same TLS certificate checks before being allowed to replace
a previously known and validated IP address for the same Designated Resolver name.

## Opportunistic Discovery {#opportunistic}

There are situations where Verified Discovery of encrypted DNS
configuration over unencrypted DNS is not possible. This includes Unencrypted
Resolvers on non-public IP addresses such as those defined in {{!RFC1918}} whose
identity cannot be confirmed using TLS certificates.

Opportunistic Privacy is defined for DoT in Section 4.1 of {{!RFC7858}} as a
mode in which clients do not validate the name of the resolver presented in the
certificate. A client MAY use information from the SVCB record for
"dns://resolver.arpa" with this "opportunistic" approach (not validating the
names presented in the SubjectAlternativeName field of the certificate) as long
as the IP address of the Encrypted Resolver does not differ from the IP address
of the Unencrypted Resolver. Clients SHOULD use this mode only for resolvers
using non-public IP addresses. This approach can be used for any encrypted DNS
protocol that uses TLS.

# Discovery Using Resolver Names {#encrypted}

A DNS client that already knows the name of an Encrypted Resolver can use DDR
to discover details about all supported encrypted DNS protocols. This situation
can arise if a client has been configured to use a given Encrypted Resolver, or
if a network provisioning protocol (such as DHCP or IPv6 Router Advertisements)
provides a name for an Encrypted Resolver alongside the resolver IP address.

For these cases, the client simply sends a DNS SVCB query using the known name
of the resolver. This query can be issued to the named Encrypted Resolver itself
or to any other resolver. Unlike the case of bootstrapping from an Unencrypted
Resolver ({{bootstrapping}}), these records SHOULD be available in the public
DNS.

For example, if the client already knows about a DoT server
`resolver.example.com`, it can issue an SVCB query for
`_dns.resolver.example.com` to discover if there are other encrypted DNS
protocols available. In the following example, the SVCB answers indicate that
`resolver.example.com` supports both DoH and DoT, and that the DoH server
indicates a higher priority than the DoT server.

~~~
_dns.resolver.example.com.  7200  IN SVCB 1 resolver.example.com. (
     alpn=h2 dohpath=/dns-query{?dns} )
_dns.resolver.example.com.  7200  IN SVCB 1 resolver.example.com. (
     alpn=dot )
~~~

Clients MUST validate that for any Encrypted Resolver discovered using a
known resolver name, the TLS certificate of the resolver contains the
known name in a subjectAltName extension. In the example above,
this means that both servers need to have certificates that cover
the name `resolver.example.com`. Often, the various supported encrypted
DNS protocols will be specified such that the SVCB TargetName matches the
known name, as is true in the example above. However, even when the
TargetName is different (for example, if the DoH server had a TargetName of
`doh.example.com`), the clients still check for the original known resolver
name in the certificate.

Note that this resolver validation is not related to the DNS resolver that
provided the SVCB answer.

As another example, being able to discover a Designated Resolver for a known
Encrypted Resolver is useful when a client has a DoT configuration for
`foo.resolver.example.com` but is on a network that blocks DoT traffic. The
client can still send a query to any other accessible resolver (either the local
network resolver or an accessible DoH server) to discover if there is a designated
DoH server for `foo.resolver.example.com`.

# Deployment Considerations

Resolver deployments that support DDR are advised to consider the following
points.

## Caching Forwarders

A DNS forwarder SHOULD NOT forward queries for "resolver.arpa" upstream. This
prevents a client from receiving an SVCB record that will fail to authenticate
because the forwarder's IP address is not in the upstream resolver's Designated
Resolver's TLS certificate SAN field. A DNS forwarder which already acts as a
completely blind forwarder MAY choose to forward these queries when the operator
expects that this does not apply, either because the operator knows the upstream
resolver does have the forwarder's IP address in its TLS certificate's SAN field
or that the operator expects clients of the unencrypted resolver to use the SVCB
information opportunistically.

Operators who choose to forward queries for "resolver.arpa" upstream should note
that client behavior is never guaranteed and use of DDR by a resolver does not
communicate a requirement for clients to use the SVCB record when it cannot be
verified.

## Certificate Management

Resolver owners that support Verified Discovery will need to list valid
referring IP addresses in their TLS certificates. This may pose challenges for
resolvers with a large number of referring IP addresses.

## Server Name Handling

Clients MUST NOT use "resolver.arpa" as the server name either in the TLS
Server Name Indication (SNI) ({{?RFC8446}}) for DoT or DoH connections,
or in the URI host for DoH requests.

When performing discovery using resolver IP addresses, clients MUST
use the IP address as the URI host for DoH requests.

Note that since IP addresses are not supported by default in the TLS SNI,
resolvers that support discovery using IP addresses will need to be
configured to present the appropriate TLS certificate when no SNI is present
for both DoT and DoH. 

# Security Considerations

Since clients can receive DNS SVCB answers over unencrypted DNS, on-path
attackers can prevent successful discovery by dropping SVCB packets. Clients
should be aware that it might not be possible to distinguish between resolvers
that do not have any Designated Resolver and such an active attack.
To limit the impact of discovery queries being dropped either maliciously or
unintentionally, clients can re-send their SVCB queries periodically.

DoH resolvers that allow discovery using DNS SVCB answers over unencrypted
DNS MUST NOT provide differentiated behavior based on the HTTP path alone,
since an attacker could modify the "dohpath" parameter.

While the IP address of the Unencrypted Resolver is often provisioned over
insecure mechanisms, it can also be provisioned securely, such as via manual
configuration, a VPN, or on a network with protections like RA guard
{{?RFC6105}}. An attacker might try to direct Encrypted DNS traffic to itself by
causing the client to think that a discovered Designated Resolver uses
a different IP address from the Unencrypted Resolver. Such a Designated Resolver
might have a valid certificate, but be operated by an attacker that is trying to
observe or modify user queries without the knowledge of the client or network.

If the IP address of a Designated Resolver differs from that of an
Unencrypted Resolver, clients applying Verified Discovery ({{verified}}) MUST
validate that the IP address of the Unencrypted Resolver is covered by the
SubjectAlternativeName of the Designated Resolver's TLS certificate.

Clients using Opportunistic Discovery ({{opportunistic}}) MUST be limited to cases
where the Unencrypted Resolver and Designated Resolver have the same IP address.

The constraints on the use of Designated Resolvers specified here apply
specifically to the automatic discovery mechanisms defined in this document, which are
referred to as Verified Discovery and Opportunistic Discovery. Clients
MAY use some other mechanism to verify and use Designated Resolvers discovered
using the DNS SVCB record. However, use of such an alternate mechanism needs
to take into account the attack scenarios detailed here.

# IANA Considerations {#iana}

## Special Use Domain Name "resolver.arpa"

This document calls for the addition of "resolver.arpa" to the Special-Use
Domain Names (SUDN) registry established by {{!RFC6761}}. This will
allow resolvers to respond to queries directed at themselves rather than a
specific domain name. While this document uses "resolver.arpa" to return SVCB
records indicating designated encrypted capability, the name is generic enough
to allow future reuse for other purposes where the resolver wishes to provide
information about itself to the client.

The "resolver.arpa" SUDN is similar to "ipv4only.arpa" in that the querying
client is not interested in an answer from the authoritative "arpa" name
servers. The intent of the SUDN is to allow clients to communicate with the
Unencrypted Resolver much like "ipv4only.arpa" allows for client-to-middlebox
communication. For more context, see the rationale behind "ipv4only.arpa" in
{{?RFC8880}}.

--- back

# Rationale for using SVCB records {#rationale}

This mechanism uses SVCB/HTTPS resource records {{!I-D.ietf-dnsop-svcb-https}}
to communicate that a given domain designates a particular Designated
Resolver for clients to use in place of an Unencrypted Resolver (using a SUDN)
or another Encrypted Resolver (using its domain name).

There are various other proposals for how to provide similar functionality.
There are several reasons that this mechanism has chosen SVCB records:

- Discovering encrypted resolver using DNS records keeps client logic for DNS
self-contained and allows a DNS resolver operator to define which resolver names
and IP addresses are related to one another.

- Using DNS records also does not rely on bootstrapping with higher-level
application operations (such as {{?I-D.schinazi-httpbis-doh-preference-hints}}).

- SVCB records are extensible and allow definition of parameter keys. This makes
them a superior mechanism for extensibility as compared to approaches such as
overloading TXT records. The same keys can be used for discovering Designated
Resolvers of different transport types as well as those advertised by
Unencrypted Resolvers or another Encrypted Resolver.

- Clients and servers that are interested in privacy of names will already need
to support SVCB records in order to use Encrypted TLS Client Hello
{{?I-D.ietf-tls-esni}}. Without encrypting names in TLS, the value of encrypting
DNS is reduced, so pairing the solutions provides the largest benefit.

- Clients that support SVCB will generally send out three queries when accessing
web content on a dual-stack network: A, AAAA, and HTTPS queries. Discovering a
Designated Resolver as part of one of these queries, without having to
add yet another query, minimizes the total number of queries clients send. While
{{?RFC5507}} recommends adding new RRTypes for new functionality, SVCB provides
an extension mechanism that simplifies client behavior.
