# Introduction

Multi-provider DNS setups are increasingly common.  Furthermore, the
zones that acquire multi-provider support are, quite naturally, among
the more important and critical zones on the public Internet. However,
in a very large fraction of the multi-provider setups there are manual
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

Signing Party: A DNS provider responsible for signing a zone.

Publishing Party: A DNS provider responsible for publishing a zone.

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
providers is more complex than having only one).

## Multi-provider Synchronization

One of the complexities of the multi-provider scenarios is the need
for synchronization of data across providers. For example, if a
publishing party changes the NS RRset, this must be communicated to
all other DNS providers, preferably using a secure and authenticated
mechanism.

Data that requires synchronization includes the DNSKEY, CDNSKEY, and CDS
RRsets across providers. In addition to these DNSSEC records, the NS and
CSYNC RRsets should also be synchronized, to provide a consistent view
towards the parent.

Note 1: What about glue?

Note 2: The list of RRset types to be synchronized may be extended in the
future.

## Multi-signer Key Rollovers

The multi-signer scenario has an additional need for synchronization
between the signing parties during each step of the key rollover process.
For example, a signing party must not use a new ZSK for signing until all
signing parties have published the new ZSK.

# Hard Requirements

"Hard requirements" are those that must be followed by a successful
protocol {{?RFC5218}}, because violating them would present too much of
an obstacle for broad adoption.  These will primarily be related to
enabling automation for core multi-provider scenarios.

1. Each party (each DNS provider) MUST be able to identify and
   authenticate all other DNS providers via a secure mechanism without
   manual handholding by the zone owner.

2. The solution MUST interface cleanly with mechanisms that enable
   automatic updates of the parent-side NS RRset (once standardized).

3. All publishing parties MUST be able to trigger the publication of a
   CSYNC record.

4. All signing parties MUST be able to contribute to the DNSKEY RRset
   in the zone.

5. All signing parties MUST be able to contribute to the CDS and CDNSKEY
   RRsets in the zone.

6. All signing parties MUST be able to perform multi-signer-aware key
   rollovers (ZSK/KSK/CSK).

   TODO: Specify what a multi-signer key rollover is.

7. All DNS providers MUST be able to initiate synchronization of
   changed data by notifying the other providers.

8. All DNS providers MUST be able to fetch data from another DNS
   provider using a secure mechanism.

9. Updates to the delegation information in the parent zone are either
    applied automatically if supported by the parent, or otherwise
    by a designated party (likely the domain holder).

10. The zone owner MUST be able to add and remove DNS providers from
    the multi-provider setup, and each provider's infrastructure must
    handle such changes automatically without manual intervention.

# Soft Requirements

"Soft requirements" are those that are desirable, but the absence of
which does not intrinsically eliminate a design.  These will largely
be descriptive of the problems that are trying to be addressed with a
new method, or features that would ease adoption.

1. A signing DNS provider SHOULD be able to use a "standard DNSSEC
   signer" application. I.e. it should be possible to use signers that
   are not specifically aware of the multi provider setup. "Standard
   DNSSEC signer" is defined as a bump-on-the-wire DNSSEC signer with
   support for multi-signer key rollovers.
2. DNS service for unsigned zones MUST be supported.

# Scenario-based analysis

This section describes a system of analyzing scenarios with respect to
zone contents and then derive requirements from the generation and
modifications of those zones. The idea is that any mechanism can be linked
through the requirements to particular contents of changes to the contents
of a zone that are needed.

## Scenarios

This subsection lists the scenarios that are analyzed in order from low
to high complexity.

1) Simple multi-signer

   This scenario consists of two providers A and B that each both sign and
   provide the zone. Both providers receive an unsigned zone from the zone
   owner. The parent has a CDS/CDNSKEY/CSYNC scanner.

2) Simple unsigned zone with multiple providers

   This scenarios consists of two providers A and B that each provide the
   zone. Both providers receive the unsigned zone from the zone owner.
   There is currently no mechanism for the parent to automatically pick
   up changed to the NS RRset.

3) Future work to let the parent update the NS RRset for an unsigned zone.

(More scenarios are needed, also for parents without CDS/CDNSKEY/CSYNC scanner)

## Simple multi-signer Scenario

This subsection analyses the simple multi-signer scenario.

### Zone contents

Let's assume that each signer has a KSK and a ZSK called KSK_A, ZSK_A, KSK_B,
and ZSK_B. Each provider has NS records, NS_A for provider A and NS_B for
provider B.

The quiescent state is the following:

Provider A signs the zone (except for the DNSKEY, CDS and CDNSKEY RRsets) with
ZSK_A. Provider B signs with ZSK_B. Provider A's DNSKEY RRset contains at 
least KSK_A, ZSK_A, and ZSK_B. And is signed by at least KSK_A. Similarly,
provider B's DNSKEY RRset contains at least KSK_B, ZSK_A, and ZSK_B. This
RRset is signed by KSK_B. Both providers have an NS RRset that consists of
NS_A and NS_B. At the parent the DS records refer to KSK_A and KSK_B.

TODO: Isn't the quiescent state really when "everything is in sync", i.e.,
all ZSK_* are published at all providers, all KSK_* are reflected in
CDS/CDNSKEY records, and and the DS RRset refers to all CDS/CDNSKEY keys?
-- This definition is broader, because a mid-rollover situation would also
be called "quiescent" as long as the are no outstanding sync tasks between
providers (and all are waiting for the rolling provider to make the next move).

TODO: In the unsigned case, the quiescence definition includes NS and glue.
Why not here?

### ZSK rolls

We assume that the zone of provide A is currently signed with ZSK_A and
that provider wants to roll to ZSK_An. 

Changes to the DNSKEY RRset require coordination.
Other parts of the ZSK roll are local.

When (for example) provider A changes its DNSKEY RRset to consist of KSK_A, ZSK_An and ZSK_B then provider B is informed to replace ZSK_A with ZSK_An.
Provider A monitors the propagation of its own zone.
Provider A also need to get confirmation from provider B that ZSK_An is fully propagated.

### KSK rolls

We assume provider A wants to roll to KSK_An. 

Coordination is required regarding the contents of the CDS/CDNSKEY RRsets and
their signatures. Provider A and B update each other about changes to the
CDS/CDNSKEY RRsets including signatures.

### Updating the NS RRset at the parent

When provider A wants to change its NS RRset to NS_An, it can just update the
NS RRset at the apex of the zone locally (leaving other providers' NS in the
set unchanged). Provider A then tries to agree with
provider B on a new NS RRset. When agreement is reached provider A adds a
CSYNC record to the zone.

TODO: Shouldn't all providers add an equivalent CSYNC record?

Provider A monitors the parent NS RRset. When provider A notices the parent
NS RRset contains NS_An it can try to agree with provider B to remove the
CSYNC record.

TOOD: Shouldn't all providers monitor the parent NS RRset? (Provider B
should not remove the CSYNC record based on provider A's instruction, as
long as it hasn't been acted upon.)

### On-boarding a new provider

To on-board provider C, provider MUST first ensure that its ZSK_C is in the
DNSKEY RRsets of provider A and B. Then it MUST coordinate with provider A
and C to update the DS RRset at the parent to refer to KSK_C. Finally
provider C MUST coordinate with provider A and B to update the NS RRsets
at the apex of A and B and at the parent to include NS_C.

### Off-boarding 

Off-boarding goes in the opposite order of on-boarding.

### Requirements

1) The zone owner specifies that provider A and provider B both sign
   the zone and serve the zone. Providers A and B take full control over the 
   NS RRset and the parent zone scans for CDS/CDNSKEY/CSYNC.

2) The zone owner provides both providers with an unsigned copy of the zone.
   The zone SHOULD NOT include the apex CDS, CDNSKEY, CSYNC, DNSKEY, and
   NS RRsets. 

3) There is a mechanism that allows a provider to inform another provider of
   its current set of ZSKs.

4) There is a mechanism that allows a provider to wait until another provider
   has fully propagated the first provider's ZSKs.

5) There is a mechanism that allows a provider to propose changes to the 
   CDS/CDNSKEY RRsets and get agreement on the contents of the set.

7) There is a mechanism that allows a provider to propose changes to the 
   NS RRset and get agreement on the contents of the set.

## Simple unsigned zone with multiple providers

This subsection analyses the simple multi-provider scenario.

### Zone contents

Lets assume that each provider has NS records, NS_A for provider A and
NS_B for provider B.

The quiescent state is the following:

Both providers serve the zone with an NS RRset that consists of
NS_A and NS_B. At the parent the delegation NS RRset is the same.
Where needed, the parent has glue consistent with the data in the zone.

### Updating the NS RRset at the parent

When provider A wants to change its NS RRset to NS_An, it can just update the
NS RRset at the apex of the zone locally (leaving other providers' NS in the
set unchanged). Provider A then tries to agree with provider B on a new NS RRset.

Currently there is no mechanism to update the parent NS RRset automatically
for unsigned zones. 

An agent that operates on behalf of the zone owner has to be informed.
This agent can then interface with the registrar to update the NS
RRset and any required glue.

### On-boarding a new provider

To on-board provider C, provider C MUST coordinate with provider A and B to
update the NS RRsets at the apex of A and B and at the parent to include NS_C.

### Off-boarding 

Off-boarding goes in the opposite order of on-boarding.

### Requirements

1) The zone owner provides both providers with an unsigned copy of the zone.
   The zone SHOULD NOT include the apex NS RRset.

2) There is a mechanism that allows a provider to propose changes to the 
   NS RRset and get agreement on the contents of the set.

## Future work to let the parent update the NS RRset for an unsigned zone.

This subsection analyses the simple multi-provider scenario where the
parent has a mechanism to automatically update the NS RRset.

### Zone contents

Lets assume that each provider has NS records, NS_A for provider A and
NS_B for provider B.

The quiescent state is the following:

Both providers serve the zone with an NS RRset that consists of
NS_A and NS_B. At the parent the delegation NS RRset is the same.
Where needed, the parent has glue consistent with the data in the zone.

### Updating the NS RRset at the parent

When provider A wants to change its NS RRset to NS_An, it can just update the
NS RRset at the apex of the zone locally (leaving other providers' NS in the
set unchanged). Provider A then tries to agree with provider B on a new NS RRset.

Using ideas from RFC 9615 (Automatic DNSSEC Bootstrapping Using Authenticated
Signals from the Zone's Operator) a new protocol could be designed
to update the parent NS RRset.

If some of the current nameserver for the zone are in DNSSEC signed zones
then they can published the new NS RRset under something like:
_nsupdate.example.co.uk._signal.ns1.example.net

There are there two problems that need to be solved:
* The creation of a 'CNS' record that has a similar role as the CDS record.
* A name scheme for in-bailiwick glue records. For example
  ns1._nsupdate.example.co.uk._signalns1.example.net IN AAAA
  could be the glue for ns1.example.co.uk.

### On-boarding a new provider

To on-board provider C, provider C MUST coordinate with provider A and B to
update the NS RRsets at the apex of A and B and at the parent to include NS_C.

### Off-boarding 

Off-boarding goes in the opposite order of on-boarding.

### Requirements

1) A mechanism the signals the parent in a secure way that the parent
   NS RRset needs to be updated.

# Security Considerations

The integrity of the data that is used to modify the zone (i.e. the
data synchronized between multiple providers, like the NS RRset and,
in a multi-signer context, the DNSKEY RRset) is of the utmost
importance. Therefore, all such data must either be directly
verifiable or received through secure and authenticated communication.

The requirements above are designed to fulfill this need.

# IANA Considerations

None.
