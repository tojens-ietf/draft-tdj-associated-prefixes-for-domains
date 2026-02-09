---
title: "Associated IP Prefixes for Domain Names"
abbrev: "associated-prefixes"
category: std

docname: draft-tdj-dnsop-associated-prefixes-for-domains-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
v: 3
keyword:
 - DNS
 - IP prefixes
 - SVCB


author:
 -
    fullname: Tommy Jensen
    email: tojens.ietf@gmail.com
 -
    fullname: David Redekop
    organization: Adam Networks Inc.
    email: david.ietf@adamnet.works
 -
    fullname: John Todd
    organization: Quad9
    email: jtodd@quad9.net

normative:

informative:


--- abstract

RFC9000 defines a mechanism that allows servers to migrate clients
to another IP address without name resolution. The new address may 
not be discoverable as A/AAAA records for that domain name. This draft defines a
mechanism that allows a client to get advance notice of associated IP addresses
for a domain name as part of the DNS query.


--- middle

# Introduction

It is common for services to be associated with domain names
even if not every IP address used by the service are represented
in A and AAAA records for the service's domain names. One common example is
teleconferencing, which often uses a media streaming protocol whose peer
addresses are negotiated within a connection, such as the use of WebRTC.
Another possible but less likely example is QUIC's use of preferred
addresses, defined in {{Section 9.6 of
?RFC9000}}, which allows a server to migrate a client to another server IP
address which may or may not have been resolvable from the domain name
resolved to initiate the original connection.

This document defines a mechanism for domain name owners to advertise
any IP address prefixes that are associated with a domain name. This allows
client peers to predict which IP addresses may end up in use when contacting
a given service.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Terminology

TBD


# Mechanism

## Client Behavior

When a client wishes to discover the IP prefixes that are associated with a
given domain name, it SHOULD issue a SVCB query for the domain name (which it
may already be doing for other reasons). If the "cidrs" key is present, then
it SHOULD issue another query of type CIDRS to retrieve the associated IP
prefixes.

Clients MAY choose to ignore claims of association by CIDRS records
with prefixes shorter than a preconfigured minimum length per IP version to
avoid attacks where a name maliciously claims association with IP address
space it has no association with. This version of the text does not suggest
defined values for minimum prefix lengths.

## Server Behavior

When a server wishes to provide the associated IP prefixes for a given name, it
SHOULD create CIDRS records for the prefixes associated with the name as well as
add a "cidrs" key to a new or existing SVCB record.

Servers SHOULD avoid claiming very short prefixes in CIDRS records. It is not
expected that a single domain name is legitimately associated with a short
prefix. Associated IP addresses SHOULD be restricted to IP addresses a
server reasonably expects a client to interact with for the functionality
provided services using the domain name. For example, name owners
SHOULD NOT create CIDRS records that include all IP ranges owned by a company
for the company's primarily recognizable domain name (example-company.example.
having a CIDRS record listing every IP address owned by Example Company would
be inappropriate).

When configuring the TTL of CIDRS and SVCB records, name owners SHOULD avoid
SVCB records containing the "cidrs" key having a shorter TTL than the TTL of the
CIDRS records themselves. This will avoid unnecessary follow-up CIDRS queries by
clients which still have valid records. The TTL values of CIDRS records
SHOULD NOT be any shorter than the expected lifetime of traffic flows of typical
service usage.

Servers SHOULD truncate responses to avoid creating risk of
effective DDoS attacks, even if the CIDRS record would fit in a single UDP
packet. This means in effect that CIDRS records SHOULD NOT ever be sent using
unencrypted DNS over UDP.


# CIDRS record format

The CIDRS RR type is designed to convey an IP prefix, an associated port range,
and a protocol number as defined in the Assigned Internet Protocol Numbers IANA
registry. The port range can be defined as 0-65535 and the protocol number as
255 if these fields are not applicable for the record owner (because only the
IP prefix is interesting).

## Wire format

CIDRS RR data on the wire uses the following format:

                        1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 2 3 3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Family |  Rsvd | Prefix Length |            Prefix             |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
   |          ... (variable, padded to an octet boundary)          |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |       Port Range Begin        |        Port Range End         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Protocol Number|
   +-+-+-+-+-+-+-+-+

- IP family (4 bits): indicates which version of the Internet Protocol the prefix belongs to

- Reserved for future use (4 bits)

- Length of Prefix (8 bits): number of bits of the IP prefix.

- Prefix (variable length): the IP prefix itself. The length of this field is always an integer of bytes (begins and ends on octet boundaries) just large enough to contain a prefix of length equal to the Length of Prefix

- Port Range Begin (16 bits): the lowest port number in the included range

- Port Range End (16 bits): the highest port number in the included range

- Protocol Number (8 bits): a value from the Assigned Internet Protocol Numbers IANA registry (255 if no associated protocol should be defined)


## Zone file format

TODO


## Justification for new RR type

Because the list of associated IP addresses for a given name is likely to be
somewhat large, at least relative to the typical A, AAAA, and SVCB/HTTPS queries
a client is likely to make, it makes sense to create a new type that ensures
DNS query issuers can opt into receiving this payload. Additionally, inclusion
within SVCB or HTTPS records is a dubious reuse given there is no guarantee that
services at associated IP addresses can authenticate a name (if the resulting
protocol does not perform name verification).


## Verification

CIDRS records MUST be DNSSEC signed. This is because unlike A and AAAA records,
there is no expectation that the resulting traffic the querying client will
send to these IP addresses will be able to prevent attacker impersonation via
secure peer validation such as that provided by TLS certificates. By definition,
the addresses in these CIDRs are used in association with services that use the
domain name but cannot validate claims of the domain name. DNSSEC validation
will provide assurance that the IP addresses are those expected by the valid
owner of the domain name.


# Security Considerations

## Suspiciously large IP address sets

This version of the text does not suggest defined values for minimum prefix
lengths. Some reasonable considerations when defining associated prefixes include
not having a single prefix span multiple ASNs and identifying if a single prefix
is associated with functionality provided by services unrelated to this domain name.
It can hardly be called an association if a short prefix that spans all media services
provided by Example Company (which include video streaming, image hosting, and
music library organiztion) is listed in the CIDRS records for the domain name
very-specific-to.image-hosting.example-company.example.

## Large payload sizes

It is expected that in most common use cases, CIDRS records will need more than
one CIDR value, possibly many (balancing this against guidance given in
{{prefixcounts}}). The more CIDRS records in a zone, the greater the amplification
of client compute to server processing.

# Operational Considerations

## TTLs

Having short TTLs on CIDRS records can risk the records expiring before the
expected usage time of the service(s) the domain name provides. This would encourage
DNS stub resolvers and the processes calling DNS stub resolver APIs to ignore
TTL values in favor of supporting performant user experiences.

## Prefix lengths and counts {#prefixcounts}

Having many prefixes associated with a domain name increases the payload size of
a CIDRS response. However, reducing that size by sending an over-inclusive prefix
risks claiming IP addresses are associated with the name when they are not, they
just happen to fall under the common shorter prefix of the real values. Balancing
this trade-off is something name owners need to be mindful of when managing
name mappings.

## Complex messages {#complexity}

{{?RFC3123}} defines an experimental mechanism by which the APL RR type can both
define CIDRs that are somehow associated with a domain name as well as negate
subsets of the CIDR. This is specifically not supported for the CIDRS RR type
because of the added complexity this creates for implementors and the support
for defining non-consecutive subnets for the same purpose. Implementors SHOULD
reduce the number of CIDRs needed for a given domain name rather than have
many long prefixes that cannot be grouped under fewer CIDRs without needing to
define the non-included gaps within them.

## Transition period workarounds

Before deployment of this document is common, clients will frequently run into
the problem of wanting to discover associated IP addresses for a given zone, but
the zone owner does not yet support this document. During this ramp up of
deployment, administrators might turn to workarounds, such as creating a zone
they control to distribute CIDRs the administrator knows to be associated with
services they depend on but do not yet support this document.

How to accomplish such a mapping is left to implementors as a non-standard
mechanism. This is out of scope for this document, which only defines
advertisement of IP addresses associated with a given name directly.


# IANA Considerations

TODO: new DNS RR type "CIDRS"

TODO: new SVCB key "cidrs"


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
