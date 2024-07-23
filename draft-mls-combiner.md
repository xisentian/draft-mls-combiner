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
This document describes a protocol for combining a standard MLS session with a post-quantum MLS session to achieve flexible and efficient hybrid post-quantum security. Specifically, we describe how to use the exporter secret of a PQ MLS session, i.e. an MLS session using a PQ ciphersuits to seed PQ guarantees into an MLS session using traditional ciphersuites. By supporting on-demand traditional-only key updates (a.k.a. PARTIAL updates) or hybrid-PQC key updates (a.k.a. FULL updates), we can reduce the bandwidth and computational overhead associated with meeting the frequent  key rotations while still providing PQ security.  
<!--[**TODO**: *Consider adding a statement to say how this combiner generalizes combining of two (or more?) arbitrary MLS sessions*]: Ans: Write out the doc first as PQ combiner and then we can talk about abstracting this as a general combiner-->
--- middle 

# Introduction

A fully capable quantum adversary has the ability to break fundamental underlying cryptographic assumptions of traditional Key Encapsulation Mechanisms (KEMs) and Digital Signature Algorithms (DSAs). This has led to the development of post quantum (PQ) cryptographically secure KEMs and DSAs by the cryptographic research community which have been formally adopted by the National Institute of Standards and Technology (NIST), including the Module Lattice KEM (ML-KEM) and Module Lattice DSA (ML-DSA) algorithms. While these provide PQ security, ML-KEM and ML-DSA have significantly worse overhead in terms of public key size, signature sizes, ciphertext size, and CPU time than their traditional counterparts. Moreover, research arms on side-channel attacks, etc., have motivated uses of hybrid-PQ combiners that draw security from both the underlying PQ and underlying traditional components. A variety of hybrid security treatments have arisen across IETF working groups to bridge the gap between performance and security to encourage the adoption of PQ security in existing protocols, including MLS protocol [RFC9420]. 

Within the MLS working group, there are several topic areas that make use of post-quantum security extensions: 
[Copied from draft-mahy-mls-xwing]
1.  A straightforward MLS cipher suite that replaces a traditional KEM with a hybrid post-quantum/traditional KEM.  Such a cipher suite could be implemented as a drop-in replacement in many MLS libraries without changes to any other part of the MLS stack. The aim is for implementations to have a single KEM which would be performant and work for the vast majority of implementations. It addresses the harvest-now / decrypt-later threat model using the simplest, and most practicable solution available.

2. Versions of existing cipher suites that use post-quantum signatures; and specific guidelines on the construction, use, and validation of hybrid signatures.

3. One or more mechanisms which reduce the bandwidth or storage requirements, or improve performance when using post-quantum algorithms (for example by updating post-quantum keys less frequently than traditional keys, or by sharing portions of post-quantum keys across a large number of clients or groups.)

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

**Traditional MLS Session:** An MLS session that uses a Diffie-Hellman (DH) based KEM as described in RFC9180. 

**Key Derivation Function (KDF):** A Hashed Message Authentication Code (HMAC)-based expand-and-extract key derivation function (HKDF) as described in RFC5869. 

**Key Encapsulation Mechanism (KEM):**  A key transport protocol that allows two parties to obtain a shared secret based on the receiver's public key. 

**Post Quantum (PQ) MLS Session:** An MLS session that uses a PQ-KEM construction, such as described by FIPS 203 from NIST. 



# Protocol Execution 

The combiner protocol runs two MLS sessions in parallel synchronizing their group memberships. The two sessions are combined by exporting a secret from the post quantum session and importing it as a PSK in the traditional session. This combination process is mandatory for commits to adds and removals to maintain synchronization between the sessions but is optional for other commits (e.g. to allow for cheap traditional PCS key rotations). Due to the higher computational costs and output sizes of PQ KEM (and signature) operations, it may be desirable to issue PQ updates less frequently than the traditional updates. The combiner protocol design treats both sessions may be treated as black-box interfaces so we only highlight operations requiring synchronizations in this document.

## Commit Flow
<!-- [**TODO**: Full (pair of commits which are *combined* - do the PSK exporter dance) vs Partial (traditional only) commits, then talk about rules for proposals (adds/removes and then everything else)]-->

Commits to proposals MAY be *PARTIAL* or *FULL*. For a PARTIAL commit, only the traditional session's epoch is updated following the proposal-commit sequence from Section 12 of RFC9420. For a FULL commit, a commit is first applied to the PQ session and another commit is applied to the traditional session using a PSK derived from the `exporter_secret` of the PQ session. To ensure the correct PSK is used, the sender includes information about the PSK in a PreSharedKey proposal for in the traditional session's commit chain of proposals (8.4, 8.5 RFC9420). Receivers process the PQ commit and the traditional commit (which also includes a PSK proposal) to derive the new epochs in both sessions.  


[**TODO**: Change this to show full commits ]

                                                  Group
      A                   B                      Channel
    |                     |                        |
    |                     | Upd'(B)                |
    |                     | Upd(B, f)              |
    |                     |----------------------->|
    |                     |                        |
    |                     |                Upd'(B) |
    |                     |              Upd(B, f) |
    |<---------------------------------------------+
    |                     |<-----------------------+
    |                     |                        |
    | Commit'(Upd')       |                        |
    | PreSharedKeyID'()   |                        |
    | Commit(Upd, PSKID)  |                        |
    |--------------------------------------------->|
    |                     |                        |
    |                     |          Commit'(Upd') |
    |                     |      PreSharedKeyID'() |
    |                     |     Commit(Upd, PSKID) |
    |<---------------------------------------------+
    |                     |<-----------------------+
    Fig 1. Full Commit on Update proposal from Client B. 
        Messages with ' come from the PQ session. 

**Remark**: Fig 1 shows Client A accepting the update proposals from Client B as a FULL commit. The flag `f` in the classical update proposal `Upd(B, f)` indicates B's intention for a FULL commit to whomever commits to its proposal. [**Comment**: I think this is better than letting a client choose to full commit or partial commit on a set of proposals it receives b/c we let the originator of the proposals decide on the partial/fullness of the commit.]

## Welcome session validation 
<!--[**TODO**: The welcome messages should come from the same session through some kind of indicator of a dual session in the PQ Welcome. Group Context Extension to include the groupID of the other session? ] -- XT: See the `gid` value added to the welcome message parameters.-->

Since a client must join two sessions, the Welcome messages it receives to each session must indicate that it's not sufficient to join only one or the other. Therefore, a Group Context Extension value `gid` indicating the GroupID and ciphersuites of the two sessions is added to the Welcome message in order to validate joining the combined sessions. 
<!--## Adding and Removing Users
Adding and removing users is done per [RFC9420], except that the joiner is added into two groups: the PQ group and the traditional group. [TODO: add indicator that they are joining the hybrid session.]. 
When the joiner issues its first update, it MUST perform a FULL update, applying both a PQ and traditional update as described above, using the exporter_secret and PSK proposal options.-->


### Adding a User

User leaf nodes are first added to the PQ session following the sequence described in Section 3 of RFC9420 except using PQ algorithms where HPKE algorithms exist. For example, a PQ KeyPackage one containing a PQ public key signed using a PQ DSA, must first be published to the Delivery Service (DS). Then the associated Add Proposal, Commit, and Welcome messages will be sent and processed in the PQ session according to Section 12 of RFC9420. The same sequence is repeated in the standard session except following the FULL Commit combining sequence where a PreSharedKeyID proposal is additionally committed. It's worth mentioninging here that the joiner MUST issue a FULL commit as soon as possible to acheive PCS. 
[**XT**: Pick up edits here]

                                                          Group
    A                         B          Directory            Channel
    |                         |              |                   |
    | KeyPackageB, KeyPackageB'              |                   |
    |<---------------------------------------+                   |
    |                         |              |                   |
    | Add'(A->B)              |              |                   |
    | Commit'(Add')           |              |                   |
    +----------------------------------------------------------->|
    |                         |              |                   |
    | Welcome'(B, gid)        |              |                   | 
    +------------------------>|              |                   |
    |                         |              |                   |
    |                         |              |        Add'(A->B) |
    |                         |              |     Commit'(Add') |
    |<-----------------------------------------------------------+
    |                         |<---------------------------------+
    |                         |              |                   |
    | Add(A->B)               |              |                   |
    | PresharedKeyId(A)       |              |                   |
    | Commit(Add, PSKid)      |              |                   |
    +----------------------------------------------------------->|
    |                         |              |                   |
    | Welcome(B, PSKid, gid)  |              |                   |
    +------------------------>|              |                   |
    |                         |              |         Add(A->B) |
    |                         |              | PresharedKeyId(A) |
    |                         |              |Commit(Add, PSKid) |
    |<-----------------------------------------------------------+
    |                         |<---------------------------------+
    
      Figure 2: 
      Client A creates a group with client B.
      Messages with ' come from the PQ session. 

<!-- Add
new epoch, then two welcome packages to add member (one pq one traditional), joiner will full update when they come online as their first update

Remove
new epoch, commit sequence in both sessions, 

invitee invites joiner to two seperate groups - (certain extension, wire-format, or opaque value - two session ids, indicator bit specifying hybrid, to specify this) 
-->

### External Joins

External joins are used by members who join a group without being explicitly added (via a add-commit sequence) by another existing member. The external user MUST join both the PQ session and the traditional session using the appropriate GroupInfo object to create an external Commit. As stated previously, the GroupInfo used to create the external commit MUST contain the HPQMLS flag [**TODO: Decide on flag/value**]. Then, the new member MUST issue a full hybrid update as described in [Updates](#updates).

### Removing a Group Member

User removals MUST be done in both PQ and traditional sessions followed by a full hybrid update as as described in [Updates](#updates). 


# Application Messages

The HPQMLS combiner serves only to provide hybrid PQ security to a classical MLS session. Application messages are therefore only sent using  the `encryption_secret` provided by the key schedule of the classical session according to Section 15 of RFC9420. 

## TODO? Epoch Agreement (Fork Resiliency) 


# Security Considerations
[TODO:] Remark on PQ KEM vs PQ Signatures and PQ Conf/Auth guarentees we get. 
[TODO:] PQ Session with only PQ KEM (Conf) not PQ Sigs (Auth) - we need to flag this as a Hybrid Conf Combiner or Hybrid Conf+Auth combiner 
[TODO:] Tighter windows for post compromise and FS windows. 
[**TODO** book-keeping operations (for fork resiliency?)]. 
[TODO: Information leakage with the `gid` value being added to welcome messages]

## Transport Security 
Recommendations for preventing denial of service (DoS) attacks, or restricting transmitted messages are inherited from MLS. Furthermore, message integrity and confidentiality is, as for MLS, protected. 

# Extension Requirements to MLS


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
