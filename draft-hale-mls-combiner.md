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
This document describes a protocol for combining a traditional MLS session with a post-quantum (PQ) MLS session to achieve flexible and efficient hybrid post-quantum security. Specifically, we describe how to use the exporter secret of a PQ MLS session, i.e. an MLS session using a PQ ciphersuite, to seed PQ guarantees into an MLS session using a traditional ciphersuite. By supporting on-demand traditional-only key updates (a.k.a. PARTIAL updates) or hybrid-PQ key updates (a.k.a. FULL updates), we can reduce the bandwidth and computational overhead associated with meeting the requirement of frequent key rotations while still providing PQ security.  
--- middle 

# Introduction

A fully capable quantum adversary has the ability to break fundamental underlying cryptographic assumptions of traditional Key Encapsulation Mechanisms (KEMs) and Digital Signature Algorithms (DSAs). This has led to the development of post-quantum (PQ) cryptographically secure KEMs and DSAs by the cryptographic research community which have been formally adopted by the National Institute of Standards and Technology (NIST), including the Module Lattice KEM (ML-KEM) and Module Lattice DSA (ML-DSA) algorithms. While these provide PQ security, ML-KEM and ML-DSA have significantly worse overhead in terms of public key size, signature size, ciphertext size, and CPU time than their traditional counterparts. Moreover, research arms on side-channel attacks, etc., have motivated uses of hybrid-PQ combiners that draw security from both the underlying PQ and underlying traditional components. A variety of hybrid security treatments have arisen across IETF working groups to bridge the gap between performance and security to encourage the adoption of PQ security in existing protocols, including the MLS protocol [RFC9420]. 

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

# Copyright Notice 

Copyright (c) 2024 IETF Trust and the persons identified as the document authors.  All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (https://trustee.ietf.org/license-info) in effect on the date of publication of this document. Please review these documents carefully, as they describe your rights and restrictions with respect to this document.  Code Components extracted from this document must include Revised BSD License text as described in Section 4.e of the Trust Legal Provisions and are provided without warranty as described in the Revised BSD License.


# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14, [RFC2119], and [RFC8174] when, and only when, they appear in all capitals, as shown here.

The terms MLS client, MLS member, MLS group, Leaf Node, GroupContext, KeyPackage, Signature Key, Handshake Message, Private Message, Public Message, and RequiredCapabilities have the same meanings as in the [MLS protocol] <https://www.rfc-editor.org/rfc/rfc9420.html>.

# Notation 

We use terms from from MLS [RFC9420]. Below, we've restated relevant terms and define new ones: 

**Application Message:** A PrivateMessage carrying application data.

**Handshake Message:** A PublicMessage or PrivateMessage carrying an MLS Proposal or Commit object, as opposed to application data.

**Key Derivation Function (KDF):** A Hashed Message Authentication Code (HMAC)-based expand-and-extract key derivation function (HKDF) as described in RFC5869. 

**Key Encapsulation Mechanism (KEM):**  A key transport protocol that allows two parties to obtain a shared secret based on the receiver's public key. 

**Post Quantum (PQ) MLS Session:** An MLS session that uses a PQ-KEM construction, such as described by FIPS 203 from NIST. 

**Traditional MLS Session:** An MLS session that uses a Diffie-Hellman (DH) based KEM as described in RFC9180. 



# Modes of Operation

Security needs vary by organizations and use-case specific risk tolerance and/or constraints. While this combiner protocol targets combining a PQ session and a traditional session the degree of PQ security may be tuned depending on the use-case: PQ confidentiality only or PQ confidentiality and authenticity. By PQ Confidentiality, we refer to the security provided by PQ KEM protected handshake messages in MLS. By PQ authenticity, we refer to non-repudiation guarantees provided by PQ signatures for handshake and application messages. 

The modes of operation are specified by the [**TODO**]`mode` flag in the HPQMLS extension and are listed ordered by least amount of PQ security to most. 


<!--Note, PQ authenticity alone (without confidentiality) does not make sense for MLS group key agreement and the use of this protocol as a generic combiner protocol for two MLS sessions - both are out of scope for this document. -->
[**TODO**: Extension flag or code for this? Or leave it to be interpreted by the ciphersuites?]

## PQ Confidentiality Only

The default mode of operation is in PQ Confidentiality Only mode. Since a cryptographically relevant quantum computer (CRQC) has not been publicly revealed, the harvest-now-decrypt-later attack suffices as the threat model for the HPQMLS combiner. Under this lense, PQ confidentiality with traditional authenticity are appropriate minimum security goals. 

By using a PQ KEM to protect the key schedule of the PQ session, we can extend the PQ confidentiality to the standard session's key schedule. Recall, this is done via the inject of the exporter key from the PQ session as a pre-shared key in the traditional session's key schedule  which generates the symmetric group keys used for AEAD and MAC calculations. Even if an adversary is successful in breaking the KEM used in the traditional session, they would be unsuccessful in calculating the group secret without also knowing the PSK value derived from the out-of-band PQ KEM. Therefore, in this mode, the PQ session can be defined as using PQ KEM and traditional signatures for handshake messages while the traditional session uses traditional KEM and signature algorithms for application and handshake messages. 


## PQ Confidentiality + Authenticity 

The elevated mode of operation is the PQ Confidentiality and Authenticity mode. If a CRQC exists, then the threat model used in the default mode would be too weak. Thus, the default mode would be insufficient to guarantee authenticity of messages from an external CRQC adversary. Recall that authenticity in MLS refers to two types of guarantees: 1) that messages were sent by a member of the group provided by the computed symmetric group key used in AEAD and 2) that a message was sent by a particular user (i.e. non-repudiation) provided by digital signatures. By using a PQ DSA to sign handshake messages non-repudiation is guaranteed against a CRQC adversary. Therefore, in this mode the PQ session MUST use a PQ DSA in addition to PQ KEM ciphersuites for handshake messages (the traditional session remains unchanged). 

[**TODO:** Move this to security considerations?] 
If the traditional session remains unchanged from the default mode, a CRQC adversary may forge signatures associated with messages sent in traditional session. However, in terms of group key agreement, this is insufficient to mount anything more than a denial of service attack (e.g. via group state desynchronization). In terms of application messages, while the signature may be forged by an external CRQC adversary, the content (including sender information) is still protected by AEAD which uses the symmetric group key. Thus, only an insider CRQC adversary could actually mount masquerading or forgery attacks which is beyond the scope of this protocol.  


<!---## PQ Confidentiality + Hybrid Authenticity
The highest and most computationally costly mode of operation is to use 
-->
# The Combiner Protocol Execution 

The combiner protocol runs two MLS sessions in parallel, synchronizing their group memberships. The two sessions are combined by exporting a secret from the post quantum session and importing it as a Pre-Shared Key (PSK) in the traditional session. This combination process is mandatory for commits to add and remove proposals, in order to maintain synchronization between the sessions. However, it is optional for any other commits (e.g. to allow for cheap traditional PCS key rotations). Due to the higher computational costs and output sizes of PQ KEM (and signature) operations, it may be desirable to issue PQ combined commits less frequently than the traditional-only commits. The combiner protocol design treats both sessions as black-box interfaces so we only highlight operations requiring synchronizations in this document.

## Commit Flow

Commits to proposals MAY be *PARTIAL* or *FULL*. For a PARTIAL commit, only the traditional session's epoch is updated following the propose-commit sequence from Section 12 of RFC9420. For a FULL commit, a commit is first applied to the PQ session and another commit is applied to the traditional session using a PSK derived from the `exporter_secret` of the PQ session. To ensure the correct PSK is used, the sender includes information about the PSK in a PreSharedKey proposal for in the traditional session's commit list of proposals (8.4, 8.5 RFC9420). Receivers process the PQ commit and the traditional commit (which also includes a PSK proposal) to derive the new epochs in both sessions.  

                                                         Group
      A                       B                         Channel
    |                         |                            |
    | Commit'()               |                            |
    | Commit(PreSharedKeyID)  |                            |
    |----------------------------------------------------->|
    |                         |                            |
    |                         |                 Commit'()  |
    |                         |    Commit(PreSharedKeyID)  |
    |<-----------------------------------------------------+
    |                         |<---------------------------+
    Fig 1a. FULL Commit to an empty proposal list.
        Messages with ' are sent in the the PQ session. 
        PreSharedKeyID identifies a PSK exported from the PQ
        session and is included in the commit in the classical
        session.

                                                                 Group
      A                           B                             Channel
    |                             |                                |
    |                             | Upd'(B)                        |
    |                             | Upd(B, f)                      |
    |                             |------------------------------->|
    |                             |                                |
    |                             |                        Upd'(B) |
    |                             |                      Upd(B, f) |
    |<-------------------------------------------------------------+
    |                             |<-------------------------------+
    |                             |                                |
    | Commit'(Upd')               |                                |
    | Commit(Upd, PreSharedKeyID) |                                |
    |------------------------------------------------------------->|
    |                             |                                |
    |                             |                  Commit'(Upd') |
    |                             |    Commit(Upd, PreSharedKeyID) |
    |<-------------------------------------------------------------+
    |                             |<-------------------------------+
    Fig 1b. FULL Commit to an Update proposal from Client B. 
        Messages with ' are sent in the the PQ session.

**Remark**: Fig 1b shows Client A accepting the update proposals from Client B as a FULL commit. The flag `f` in the classical update proposal `Upd(B, f)` indicates B's intention for a FULL commit to whomever commits to its proposal. 

## Adding a User

User leaf nodes are first added to the PQ session following the sequence described in Section 3 of RFC9420 except using PQ algorithms where HPKE algorithms exist. For example, a PQ KeyPackage one containing a PQ public key signed using a PQ DSA, must first be published to the Delivery Service (DS). Then the associated Add Proposal, Commit, and Welcome messages will be sent and processed in the PQ session according to Section 12 of RFC9420. The same sequence is repeated in the standard session except following the FULL Commit combining sequence where a PreSharedKeyID proposal is additionally committed. The joiner MUST issue a FULL commit as soon as possible to acheive PCS. 


                                                         Key Package                                    Group
    A                                          B          Directory                                    Channel
    |                                          |              |                                           |
    |                                          | KeyPackageB' |                                           |
    |                                          |  KeyPackageB |                                           |
    |<--------------------------------------------------------+                                           |
    |                                          |              |                                           |
    | Commit'(Add'(KeyPackageB'))              |              |                                           |
    | Commit(Add(KeyPackageB), PreSharedKeyID) |              |                                           |
    +---------------------------------------------------------------------------------------------------->|
    |                                          |              |                                           |
    | Welcome'                                 |              |                                           | 
    | Welcome(PreSharedKeyID)                  |              |                                           | 
    +----------------------------------------->|              |                                           |
    |                                          |              |                                           |
    |                                          |              |  Commit'(Add'(KeyPackageB'))              |
    |                                          |              |  Commit(Add(KeyPackageB), PreSharedKeyID) |
    |<----------------------------------------------------------------------------------------------------+
    
      Figure 2: 
      Client A adds client B to the group.
      Messages with ' come from the PQ session. Processing Welcome and Commit in the traditional
      sessio requires the PSK exported exported from the PQ session.



### Welcome Message Validation 


Since a client must join two sessions, the Welcome messages it receives to each session must indicate that it is not sufficient to join only one or the other. Therefore, a HPQMLS Group Context Extension value indicating the GroupID and ciphersuites of the two sessions MUST be included in the Welcome message in order to validate joining the combined sessions. After joining, the new member MUST issue a FULL update commit as described in Fig 1b. 


### External Joins

External joins are used by members who join a group without being explicitly added (via a add-commit sequence) by another existing member. The external user MUST join both the PQ session and the traditional session. As stated previously, the GroupInfo used to create the external commit MUST contain the HPQMLS Group Context Extension value. After joining, the new member MUST issue a FULL update commit as described in Fig 1b. 

## Removing a Group Member

User removals MUST be done in both PQ and traditional sessions followed by a full update as as described in Fig 1b. 

## Application Messages

The HPQMLS combiner serves only to provide hybrid PQ security to a classical MLS session. Application messages are therefore only sent in the traditional session using the `encryption_secret` provided by the key schedule of the classical session according to Section 15 of RFC9420. 


# Security Considerations

## Full Commit Frequency 

So long as the full commit flow is followed for group administration actions, PQ security is extended to the traditional session. Therefore, full commits can occur as frequently or infrequently as desired by any given security policy. This results in a flexible and efficient use of compute,  storage, and bandwidth resources for the host by mainly calling partial updates on the traditional MLS session, given that the group membership is stable. Thus, our protocol provides PQ security and can maintain a tighter forward secrecy and post-compromise security window with lower overhead when compared to running a single MLS session that only uses hybrid signatures or only PQ KEM/DSAs. 

## Attacks on Authentication
While message integrity is protected by the symmetric key, non-repudiation attacks (e.g. forgery, impersonation, masquerading) on the digital signature of messages may be possible by a CQRC adversary. Such an attack can cause confusion in group state agreement leading to denial of service or entity authentication of the sender. If this is a concern, we recommend using hybrid PQ DSAs in the traditional session to sign messages. Note, this would negate much of the efficiency gains from using this protocol. 

## Transport Security 
Recommendations for preventing denial of service (DoS) attacks, or restricting transmitted messages are inherited from MLS. Furthermore, message integrity and confidentiality is, as for MLS, protected. 

# Extension Requirements to MLS
Group Context Extension for HPQMLS SHALL be in the following format: 

[BH: What's in the clear here? Trad session vs T session notation]

      struct {
        ProtocolVersion version = mls10;
        CipherSuite cipher_suite;
        opaque group_id<V>;
        uint64 epoch;
        opaque tree_hash<V>;
        opaque confirmed_transcript_hash<V>;
        Extension extensions<V>;
      } GroupContext;

      Extension hpqmls{
          opaque t_session_group_id<V>; 
          opaque PQ_session_group_id<V>; 
          bool mode; 
          CipherSuite t_cipher_suite; 
          CipherSuite pq_cipher_suite; 
          uint64 t_epoch; 
          uint64 pq_epoch;   
      }

# IANA Considerations 
**[TODO]** Determine an extension code to use

# References



## Normative References (i.e. RFCs)
[1] <https://www.rfc-editor.org/info/rfc9420> "MLS RFC"
[2] <https://www.rfc-editor.org/info/rfc5246> "TLS RFC"


## Informational References 
J. Alwen, M. Mularczyk, Y. Tselekounis. "Fork-Resilient Continuous Group Key Agreement", 2023. <https://eprint.iacr.org/2023/394>

<!--# Appendices -->


# Acknowledgments
{:numbered="false"}
## Contributors 
## Authors 
