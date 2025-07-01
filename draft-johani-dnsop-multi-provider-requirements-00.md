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

(More scenarios are needed)

## Simple multi-signer Scenario

This subsection analyses the simple multi-signer scenario

### Zone contents

Lets assume that each signer has a KSK and a ZSK called KSK_A, ZSK_A, KSK_B,
and ZSK_B. Each provider has NS records, NS_A for provider A and NS_B for
provider B.

The quiescent state is the following:

Provider A signs the zone (except for the DNSKEY, CDS and CDNSKEY RRsets) with
ZSK_A. Provider B signs with ZSK_B. Provider A's DNSKEY RRset contains at 
least KSK_A, ZSK_A, and ZSK_B. And is signed by at least KSK_A. Similarly,
provider B's DNSKEY RRset contains at least KSK_B, ZSK_A, and ZSK_B. This
RRset is signed by KSK_B. Both providers have an NS RRset that consists of
NS_A and NS_B. At the parent the DS records refer to KSK_A and KSK_B.

### ZSK rolls

RFC 6781 describes two ZSK rolls, a Pre-Publish ZSK roll and a
Double-Signature ZSK roll. In a multi-signer environment, the Double-Signature
ZSK roll has fewer synchronization requirements, but is generally undesirable
because it leads to more signatures. We will first look at the Double-Signature
ZSK roll.

We assume that the zone of provide A is currently signed with ZSK_A and
that provider wants to roll to ZSK_An. Provider A starts by signing the
zone with ZSK_An in addition to ZSK_A. This is a local operation of
provider A. Provider A waits until all nameservers of provider A serve the
new zone and any cache have expired. 

Now provider A changes its DNSKEY RRset to consist of KSK_A, ZSK_An and ZSK_B
and informs provider B to replace ZSK_A with ZSK_An. Provider A monitors
the propagation of its own zone. Provider A also need to get confirmation
from provider B that ZSK_An is fully propagated.

Once that state is reached, provider A stops signing with ZSK_A and the key
roll is complete.

For the Pre-Publish ZSK roll, provider A first adds ZSK_An to its DNSKEY RRset
and informs provider B that is now has two ZSKs: ZSK_A and ZSK_An.
Provider A monitors the propagation of its own zone and waits for confirmation
from provider B that the new DNSKEY RRset has fully propagated.

Then provider A starts signing with ZSK_An and monitors propagation. This is
a local operation.

Finally, provider A removes ZSK_A from the DNSKEY RRset and informs provider B
that it should only include ZSK_An in its DNSKEY RRset. The key roll is now
complete though provider A SHOULD monitor the propagation of its DNSKEY
RRset and ask provider B to report on propagation as well.

### KSK rolls

RFC 6781 defines two KSK rolls: the Double-Signature KSK roll and the
Double-DS KSK roll. The Double-Signature KSK roll is easiest and we will
look that first.

We assume provider A wants to roll to KSK_An. First provider A adds
KSK_An to its DNSKEY RRset. Provider A monitors the propagation of its
DNSKEY RRset.

When that has happened, provider A and B need to agree on new CDS/CDNSKEY
RRsets. Provider A needs the new CDS/CDNSKEY RRset to contain or refer to
KSK_An. When agreement has been reached, provider A publishes the CDS/CDNSKEY
RRset and assumes that provider B will do so as well. 

Provider A checks that the parent has a DS record for KSK_An and tries to 
monitor to propagation of the DS RRset.

When the DS RRset has fully propagated, provider A removes KSK_A from its
DNSKEY RRset, and tries to reach agreement to remove the CDS/CDNSKEY RRset.

The key roll is now complete but provider A SHOULD monitor the propagation
of its new DNSKEY RRset.

The Double-DS KSK roll starts with introducing the new KSK_An in the parent's
DS RRset. Provider A and B need to agree on a new CDS/CDNSKEY RRsets that
include both KSK_A and KSK_An. When agreement has been reached, provider A
publishes the CDS/CDNSKEY RRset and assumes that provider B will do so as
well.

Provider A checks that the parent has a DS record for both KSK_A and KSK_An
and tries to monitor to propagation of the DS RRset.

When the DS RRset has fully propagated, provider A removes KSK_A from its
DNSKEY RRset and adds KSK_An. Provider A monitors the propagation of the
new DNSKEY RRset.

When the DNSKEY RRset has fully propagated, provider A tries to agree with
provider B on new CDS/CDNSKEY RRsets that include KSK_An (and no longer KSK_A).
When agreement has been reached, provider A
publishes the CDS/CDNSKEY RRset and assumes that provider B will do so as
well.

Provider A checks that the parent has a DS record for KSK_An. Then
provider A tries to reach agreement with provider B to remove the CDS/CDNSKEY
RRset.

(Alternatively: both provider A and B monitor the DS RRset set the parent.
When the DS RRset matches the agreed upon set of keys, each provider
independently removes to CDS and CDNSKEY RRsets. Is there a risk?)

### CSK rolls

RFC 6781 recognizes Straightforward Rollover in a Single-Type Signing Scheme
and Double-DS Rollover in a Single-Type Signing Scheme

The Straightforward Rollover combines the elements of the Double-Signature
ZSK roll with those of the Double-Signature KSK roll. No new requirements
should be necessary.

The Double-DS Rollover combines the Pre-Publish ZSK roll with the
Double-DS KSK roll. Again no new requirements should be necessary.

### Algorithm rolls

RFC 6781 defines the conservative roll and somewhat describes the liberal
roll. The conservative roll combines the Double-Signature ZSK roll with
the Double-Signature KSK roll where the steps in the ZSK and KSK rolls are
separated. No new requirements are expected.

### Updating the NS RRset at the parent

When provider A wants to change its NS RRset to NS_An, it can just update the
NS RRset at the apex of the zone locally. Provider A then tries to agree with
provider B on a new NS RRset. When agreement is reached provider A adds a
CSYNC record to the zone.

Provider A monitors the parent NS RRset. When provider A notices the parent
NS RRset contains NS_An it can try to agree with provider B to remove the
CSYNC record.

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
   the zone and serve the zone. That A and B take full control over the 
   NS RRset and that parent zone scans for CDS/CDNSKEY/CSYNC.

2) The zone owner provides both providers with an unsigned copy of the zone.
   The zone SHOULD NOT include the apex CDS, CDNSKEY, CSYNC, DNSKEY, and
   NS RRsets. 

   Open question: does the SOA version have any meaning beyond transferring 
   the zone to the providers?

3) There is a mechanism that allows a provider to inform another provider of
   its current set of ZSKs.

4) There is a mechanism that allows a provider to wait until another provider
   has fully propagated the first provider's ZSKs.

5) There is a mechanism that allows a provider to propose changes to the 
   CDS/CDNSKEY RRsets and get agreement on the contents of the set.

6) There is a mechanism that allows a provider to propose changes to the 
   NS RRset and get agreement on the contents of the set.

# Security Considerations

The integrity of the data that is used to modify the zone (i.e. the
data synchronized between multiple providers, like the NS RRset and,
in a multi-signer context, the DNSKEY RRset) is of the utmost
importance. Therefore, all such data must either be directly
verifiable or received through secure and authenticated communication.

The requirements above are designed to fulfill this need.

# IANA Considerations

None.
