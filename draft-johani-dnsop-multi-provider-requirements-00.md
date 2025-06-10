# Introduction

Multi-provider DNS setups are increasingly common, not least because
this is the recommended Best Current Practice.  Furthermore, the zones
that acquire multi-provider support are, quite naturally, among the
more important and critical zones on the public Internet. However, in
a very large fraction of the multi-provider setups there are manual
steps, which is error-prone and carries operational risks. In other
cases, the steps may be automated, but in the absence of a "standard"
for DNS multi-provider setups the exact steps may be a function of
exactly which DNS providers are involved, as their internal mechanisms
and processes may be different.

A smaller fraction of the multi-provider setups involve multiple
DNSSEC signers, i.e. the DNSSEC signer as a single point of failure
has been eliminated at the cost of further complexity which likely
require further automation to be possible to operate in practice.

Because of the additional synchronization requirements that arise in a
multi-signer setup (in addition to what is needed for every
multi-provider setup) there is a risk that in practice the step where
the single-signer (or unsigned) multi-provider setup is generalized to
a multi-signer setup will not be taken. If so, the last single
point-of failure remains.

To navigate the pros and cons of complexity versus operational needs
it is necessary to sort out exactly what the requirements are (as
opposed to what other possibly desirable features are).

## Requirements Notation

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**",
"**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",
"**NOT RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document
are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Terminology

DNS Provider: A provider of DNS services such as DNSSEC signing of an
unsigned zone and/or authoritative publication of a DNS zone.

Signing Party: A DNS provider responsible for signing a zone

Publishing Party: A DNS provider responsible for publishing a zone

# Goal

The intent is to make it possible for a zone owner to specify what
role each DNS provider should fulfill in sufficient detail to
henceforth not be forced to get involved in operations of the DNS
providers or in synchronization between them.

I.e. once the multi-provider setup is in operation (and as long as the
zone owner does not explicitly change the instructions) the DNS
operators should be able to manage all changes to the zone without
involving the zone owner.

The zone owner should only have to be involved when explicitly changing
the instructions to the DNS providers, including the operation of adding
or removing a DNS provider from the zone.

# Multi-provider Scenarios

Some examples of multi-provider scenarios:

1. A zone owner has multiple DNS providers for an unsigned zone. All
   providers serve the unsigned zone.

2. A zone owner has multiple DNS providers for a signed zone. All
   providers serve the signed zone.

3. A zone owner has multiple DNS providers. One provider signs the
   zone. All providers serve the signed zone.

4. A zone owner has multiple DNS providers. Two providers sign the
   zone. Each publishing party serves one of the signed zones.

# Multi-provider Complexity

All multi-provider setups inevitably add complexity (having multiple
providers is more complex than having only one). The goal is to localize
the complexity by making the zone owner intent and the technical
requirements as explicit as possible.

The more explicit the intent and requirements are (i.e. fewer assumptions)
the easier it will be to fully automate implementations.

## Multi-provider Synchronization

One of the complexities of the multi-provider scenarios is the need
for synchronization of data across providers. For example, if a
publishing party changes the NS RRset, this must be communicated to
all other DNS providers, preferably using a secure and authenticated
mechanism.

## Multi-signer Key Rollovers

The multi-signer scenario has an additional need for synchronization
between the signing parties during each step of the key rollover process.
For example, a signing party MUST NOT use a new ZSK for signing until all
signing parties have published the new ZSK.

# Mandatory Requirements

A multi-provider architecture must fulfill the following requirements
to be able to fully support all multi-provider scenarios:

1. Each party (each DNS provider) must be able to identify and
   authenticate all other DNS providers via a secure mechanism without
   manual handholding by the zone owner.

2. All publishing parties must be able to contribute to the NS RRset in
   the zone.

3. All publishing parties must be able to trigger the publication of a
   CSYNC record.

4. All signing parties must be able to contribute to the DNSKEY RRset
   in the zone.

5. All signing parties must be able to contribute to the CDS RRset in
   the zone.

6. All signing parties must be able to perform multi-signer key
   rollovers (ZSK/KSK/CSK).

7. All DNS providers must be able to initiate synchronization of
   changed data by notifying the other providers.

8. All DNS providers must be able to fetch data from another DNS
   provider using a secure mechanism.

9. DNS service for unsigned zones must be supported.

10. Responsibility for updates to the delegation information in the
    parent zone must be explicit.

11. The zone owner must be able to add and remove DNS providers from
    the multi-provider setup, and the providers' infrastructure must
    handle such changes automatically without manual intervention.

# Desirable Features (i.e. not Requirements)

1. A signing DNS provider should be able to use a "standard" DNSSEC
   signer application. I.e. it should be possible to use signers that
   are not specifically aware of the multi provider setup. "Standard
   DNSSEC signer" is defined as a bump-on-the-wire DNSSEC signer with
   support for multi-signer key rollovers.

# Security Considerations

The integrity of the data that is used to modify the zone (i.e. the
data synchronized between multiple providers, like the NS RRset and,
in a multi-signer context, the DNSKEY RRset) is of the utmost
importance. Therefore, all such data must either be directly
verifiable or received through secure and authenticated communication.

The requirements above are designed to fulfill this need.

# IANA Considerations

None.
