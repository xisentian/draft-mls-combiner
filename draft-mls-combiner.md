---
title: Flexible Hybrid PQ MLS Combiner
abbrev: HPQMLS
docname: draft-hale-mls-combiner-00
category: info

ipr: trust200902
area: Security
keyword: 
  - security
  - authenticated key exchange
  - PCS
  - Post-Quantum

stand_alone: yes
pi: [toc, sortrefs, symrefs]

#number: 
#date: 2023-12-11
#consensus: true
#v: 1
workgroup: MLS

#venue:
#  group: MLS
#  type: "Working Group"
#  mail: WG@example.com
#  arch: https://example.com/WG
#  github: TODO
#  latest: https://example.com/LATEST
author:
  - ins: "J. Alwen"
    name: "Joël Alwen"
    organization: "AWS"
    email: alwenjo@amazon.com
  - ins: "B. Hale"
    name: "Britta Hale"
    organization: "Naval Postgraduate School"
    email: britta.hale@nps.edu
  - ins: "M. Mularczyk"
    name: "Marta Mularczyk" 
    organization: "AWS" 
    email: mulmarta@amazon.ch
  - ins: "X. Tian"
    name: "Xisen Tian"
    organization: "Naval Postgraduate School"
    email: xisen.tian1@nps.edu



--- abstract 
This document describes a protocol for combining a standard MLS session with a post-quantum MLS session to achieve flexible and efficient hybrid post-quantum security. Specifically, we describe how to use the exporter secret of a PQ MLS session, i.e. an MLS session using a PQ KEM and PQ signature algorithm, to seed PQ confidentiality and authentication guarantees into an MLS session using traditional KEM and signatures algorithms. By providing flexible support for on-demand traditional-only key updates or hybrid-PQC key updates, we can reduce the bandwidth and computational overhead associated with maintaining a PQC-only MLS session by providing flexibility as to how frequently they occur while maintaining a tighter traditional post-compromise security epoch length. 

[**TODO**: *Consider adding a statement to say how this combiner generalizes combining of two (or more?) arbitrary MLS sessions*]. 

--- middle 

# Introduction

A fully capable quantum adversary has the ability to break fundamental underlying cryptographic assumptions of classical Key Exchange Mechanisms (KEMs) and Digital Signature Algorithms (DSAs). This has led to the development of post quantum cryptographically secure KEMs and DSAs by the cryptographic research community which have been formally adopted by the National Institute of Standards and Technology (NIST) under the category of Module Lattice KEM (ML-KEM) and Module Lattice DSA (ML-DSA) algorithms. While they provide PQ security, ML-KEM and ML-DSA have significantly worse overhead in terms of keyshares size, signature sizes, and CPU time  than their classical counterparts. A variety of hybrid security treatments have risen across IETF working groups to bridge the gap between performance and security to encourage the adoption of PQ security in existing protocols, including MLS protocol [RFC9420]. 

Within the MLS working group, there are several topic areas requiring the use of post-quantum security extensions: 
[Copied from draft-mahy-mls-xwing]
1.  A straightforward MLS cipher suite that replaces a classical KEM with a hybrid post-quantum/traditional KEM.  Such a cipher suite could be implemented as a drop-in replacement in many MLS libraries without changes to any other part of the MLS stack. The aim is for implementations to have a single KEM which would be performant and work for the vast majority of implementations. It addresses the the harvest-now / decrypt-later threat model using the simplest, and most practicable solution available.

2. Versions of existing cipher suites that use post-quantum signatures; and specific guidelines on the construction, use, and validation of hybrid signatures.

3. One or more mechanisms which reduce the bandwidth or storage requirements, or improve performance when using post-quantum algorithms (for example by updating post-quantum keys less frequently than classical keys, or by sharing portions of post-quantum keys across a large number of clients or groups.)

This document addresses the third topic of theses work items. 

# About This Document

This note is to be removed before publishing as an RFC.

Status information for this document may be found at *[Todo]*.

Discussion of this document takes place on the MLS Working Group mailing list (mailto:mls@ietf.org), which is archived at https://mailarchive.ietf.org/arch/browse/mls/.  Subscribe at https://www.ietf.org/mailman/listinfo/mls/.

Source for this draft and an issue tracker can be found at https://github.com/PairedMLS/draft-pairedMLS.

# Status of this Memo 
This Internet-Draft is submitted in full conformance with the provisions of BCP 78 and BCP 79.

Internet-Drafts are working documents of the Internet Engineering Task Force (IETF).  Note that other groups may also distribute  working documents as Internet-Drafts.  The list of current Internet-Drafts is at https://datatracker.ietf.org/drafts/current/.

Internet-Drafts are draft documents valid for a maximum of six months and may be updated, replaced, or obsoleted by other documents at any time.  It is inappropriate to use Internet-Drafts as reference material or to cite them other than as "work in progress."

This Internet-Draft will expire on XX May 2024.

# Copyright Notice 

Copyright (c) 2023 IETF Trust and the persons identified as the document authors.  All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (https://trustee.ietf.org/license-info) in effect on the date of publication of this document. Please review these documents carefully, as they describe your rights and restrictions with respect to this document.  Code Components extracted from this document must include Revised BSD License text as described in Section 4.e of the Trust Legal Provisions and are provided without warranty as described in the Revised BSD License.


# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14, [RFC2119], and [RFC8174] when, and only when, they appear in all capitals, as shown here.

The terms MLS client, MLS member, MLS group, Leaf Node, GroupContext, KeyPackage, Signature Key, Handshake Message, Private Message, Public Message, and RequiredCapabilities have the same meanings as in the [MLS protocol] <https://www.rfc-editor.org/rfc/rfc9420.html>.

# Notation 

**Classical MLS Session:** An MLS session that uses Diffie Hellman (DH) based KEM as described in RFC9180. 

**Key Derivation Function (KDF):** A Hashed Message Authentication Code (HMAC)-based expand-and-extract key derivation function (HKDF) as described in RFC5869. 

**Key Encapsulation Mechanism (KEM):** 

**Post Quantum (PQ) MLS Session:** An MLS session that uses Modular Lattice (ML)-KEM as described by FIPS 203 from NIST. 

**Session Combiner:** 


# Protocol Execution 

The combiner protocol runs two MLS sessions in parallel, performing synchronizations from the PQ session to the classical session [**TODO** and book-keeping operations (for fork resiliency?)]. Both sessions may be treated as black-box interfaces. The combiner protocol adds mandatory synchronization operations that exports state information from the PQ to the classical session for group operations. This synchronization process is mandatory for adds and removals but is optional for updates to allow for flexibility. Due to the higher computational and output sizes of PQ KEM (and signature) operations, it may be desirable to issue PQ updates less frequently than the classical updates. 

## Updates

Updates MAY be *partial* or *full*. For a partial-update, only the classical session's epoch is updated following the proposal-commit sequence from Section 12 of RFC9420. For a full-update, the PQ session update seeds the update for the classical session. Specifically, the sender updates the PQ session with an empty commit and derives a PreShared Key (PSK) from the `exporter_secret` of the new epoch. Then, the same sender updates its standard session's group secret with the PQ PSK injected into the key schedule and commits the update with a PreSharedKey proposal (8.4, 8.5 RFC9420). Receivers process the PQ commit and the standard commit to derive the new epochs in both sessions. 

<This process brings entropy from the PQ session into the standard session.>

[Insert diagram of a full update]

## Adding and Removing Users
Adding and removing users are done sequentially, first in the PQ session and then in the classical session following the spirit of a full-update whereby entropy from the PQ session is injected into the standard session. 


### Adding a User

User leaf nodes are first added to the PQ session with an Add proposal. The associated Commit and Welcome messages will be sent and processed in the PQ session according to Section 12 of RFC9420. Similar to the full-update, the sender of the Add proposal will update its standard session's key schedule using the PSK generated by the `exporter_secret` of the new epoch in the PQ session. Then the sender generates an Add proposal in its standard session for the same user leaf nodes and includes a PreSharedKey proposal its Commit and PreSharedKeyID in its Welcome message.

[Diagram of Adding a User]
### External Joins

### Removing a Group Member

[**TODO:** Add a example execution]

## Updates 

## Message Sending
Messages are sent only in the 

## Epoch Agreement (Fork Resiliency) 



                                                                    Group
    A            B              G1  ...    Gn         Directory     Channel
    |Update(B,f) |              |          |              |           |
    |Commit(Upd) |              |          |              |           |
    +----------------------------------------------------------------->
    |            |              |          |              |           |
    |            |              |          |             Commit(Upd,f)|
    |            |              |          |             Update(B,f)  |
    <-----------------------------------------------------------------+
    |            |              <----------+--------------+-----------+
    |            |              |          <--------------+-----------+
    |            |              |          |            Commit(Upd, f)|
    |            |              |          |              |Notify(B)  |
    |            <--------------+----------+--------------+-----------+
    |            |              |          |              |           |

**Figure 1** Figure caption here




# Security Considerations

## Transport Security 
Recommendations for preventing denial of service (DoS) attacks, or restricting transmitted messages are inherited from MLS. Furthermore, message integrity and confidentiality is, as for MLS, protected. 


## Visibility to the Group 

## Visability to Delivery Service

# Extension Requirements to MLS

## Leaf Node Contents

# IANA Considerations 
**[TODO]** Determine an extension code to use

# References

[I-D.ietf-mls-protocol]
Barnes, R., Beurdouche, B., Robert, R., Millican, J., Omara, E., and K. Cohn-Gordon. "The Messaging Layer Security (MLS) Protocol". Work in Progress, Internet-Draft, draft-ietf-mls-protocol-20, 27 March 2023. <https://datatracker.ietf.org/doc/html/draft-ietf-mls-protocol-20>


## Normative References (i.e. RFCs)
[1] <https://www.rfc-editor.org/info/rfc9420> "MLS RFC"
[2] <https://www.rfc-editor.org/info/rfc5246> "TLS RFC"


## Informational References 


<!--# Appendices -->


# Acknowledgments
{:numbered="false"}
## Contributors 
## Authors 
