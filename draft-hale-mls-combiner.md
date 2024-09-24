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
#  github: https://github.com/xisentian/draft-mls-combiner/blob/main/draft-hale-mls-combiner.md
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
This document describes a protocol for combining a traditional MLS session with a post-quantum (PQ) MLS session to achieve flexible and efficient hybrid PQ security that amortizes the computational cost of PQ Key Encapsulation Mechanisms and Digital Signature Algorithms. Specifically, we describe how to use the exporter secret of a PQ MLS session, i.e. an MLS session using a PQ ciphersuite, to seed PQ guarantees into an MLS session using a traditional ciphersuite. By supporting on-demand traditional-only key updates (a.k.a. PARTIAL updates) or hybrid-PQ key updates (a.k.a. FULL updates), we can reduce the bandwidth and computational overhead associated with PQ operations while meeting the requirement of frequent key rotations.  
--- middle 

# Introduction

A fully capable quantum adversary has the ability to break fundamental underlying cryptographic assumptions of traditional Key Encapsulation Mechanisms (KEMs) and Digital Signature Algorithms (DSAs). This has led to the development of post-quantum (PQ) cryptographically secure KEMs and DSAs by the cryptographic research community which have been formally adopted by the National Institute of Standards and Technology (NIST), including the Module Lattice KEM (ML-KEM) and Module Lattice DSA (ML-DSA) algorithms. While these provide PQ security, ML-KEM and ML-DSA have significantly worse overhead in terms of public key size, signature size, ciphertext size, and CPU time than their traditional counterparts. Moreover, research arms on side-channel attacks, etc., have motivated uses of hybrid-PQ combiners that draw security from both the underlying PQ and underlying traditional components. A variety of hybrid security treatments have arisen across IETF working groups to bridge the gap between performance and security to encourage the adoption of PQ security in existing protocols, including the MLS protocol [RFC9420]. 

Within the MLS working group, there are several topic areas that make use of PQ security extensions: 

1.  A single MLS ciphersuite for a hybrid post-quantum/traditional KEM.  The ciphersuite can act as a drop-in replacement for the KEM, focusing on hybrid confidentiality but not authenticity, and does not incur changes elsewhere in the MLS stack. As a confidentiality focus, it addresses the the harvest-now / decrypt-later threat model. However, every key epoch incurs a PQ overhead cost. 

2. Hybrid PQ signature ciphersuites that address hybrid authenticity, including construction and security considerations of hybrid signatures.

3. Mechanisms that leverage hybridization as a means to not only address the security balance between PQ and traditional components and achieve resistance to harvest-now / decrypt-later attacks, but also use it as a means to improve performance of PQ use.

This document addresses the third topic of these work items. 

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

We use terms from from MLS [RFC9420]. Below, we have restated relevant terms and define new ones: 

**Application Message:** A PrivateMessage carrying application data.

**Handshake Message:** A PublicMessage or PrivateMessage carrying an MLS Proposal or Commit object, as opposed to application data.

**Key Derivation Function (KDF):** A Hashed Message Authentication Code (HMAC)-based expand-and-extract key derivation function (HKDF) as described in RFC5869. 

**Key Encapsulation Mechanism (KEM):**  A key transport protocol that allows two parties to obtain a shared secret based on the receiver's public key. 

**Post-Quantum (PQ) MLS Session:** An MLS session that uses a PQ-KEM construction, such as described by FIPS 203 from NIST. It may optionally also use a PQ-DSA construction, such as described by FIPS 204 from NIST.

**Traditional MLS Session:** An MLS session that uses a Diffie-Hellman (DH) based KEM as described in RFC9180. 


<!---## PQ Confidentiality + Hybrid Authenticity
The highest and most computationally costly mode of operation is to use 
-->
# The Combiner Protocol Execution 

The hybrid PQ MLS (HPQMLS) combiner protocol runs two MLS sessions in parallel, synchronizing their group memberships. The two sessions are combined by exporting a secret from the PQ session and importing it as a Pre-Shared Key (PSK) into the traditional session. This combination process is mandatory for Commits of Add and Remove proposals in order to maintain synchronization between the sessions. However, it is optional for any other Commits (e.g. to allow for less computationally expensive traditional key rotations). Due to the higher computational costs and output sizes of PQ KEM (and signature) operations, it may be desirable to issue PQ combined (a.k.a. FULL) Commits less frequently than the traditional-only (a.k.a. PARTIAL) Commits. Since FULL Commits introduce PQ security into the MLS key schedule, the overall key schedule remains PQ-secure even when PARTIAL Commits are used. The FULL Commit rate establishes the post-quantum Post-Compromise Security (PCS) window, while the PARTIAL Commit rate can tighten the traditional PCS window even while maintaining PQ security more generally. The combiner protocol design treats both sessions as black-box interfaces so we only highlight operations requiring synchronizations in this document.

The default way to start a HPQMLS combined session is to create a PQ MLS session and then start a traditional MLS session with the exported PSK from the PQ session, as previously mentioned. Alternatively, a combined session can also be created after a traditional MLS session has already been running. This is done through creating a PQ MLS session with the same group members, sending a Welcome message containing the HPQMLSInfo struct in the GroupContext, and then making a FULL Commit as described in in the [Commit Flow](#commit-flow) section.

## Commit Flow

Commits to proposals MAY be *PARTIAL* or *FULL*. For a PARTIAL Commit, only the traditional session's epoch is updated following the Propose-Commit sequence from Section 12 of RFC9420. For a FULL Commit, a Commit is first applied to the PQ session and another Commit is applied to the traditional session using a PSK derived from the PQ session using the `hpqmls_psk` label (see [Key Schedule](#key-schedule)). To ensure the correct PSK is used, the sender includes information about the PSK in a PreSharedKey proposal for the traditional session's Commit list of proposals (8.4, 8.5 RFC9420). Receivers process the PQ Commit to derive a new epoch in the PQ session and then the traditional Commit (which also includes the PSK proposal) to derive the new epoch in the traditional session.  

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
        [TODO: It is not clear from the figure/caption in 1a and 1b from what PQ epoch the PSK is derived from as it looks like the commits are simultaneous]

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

**Remark**: Fig 1b shows Client A accepting the update proposals from Client B as a FULL Commit. The flag `f` in the classical update proposal `Upd(B, f)` indicates B's intention for a FULL Commit to whomever Commits to its proposal. 

## Adding a User

User leaf nodes are first added to the PQ session following the sequence described in Section 3 of RFC9420 except using PQ algorithms where HPKE algorithms exist. For example, a PQ-DSA signed PQ KeyPackage, i.e. containing a PQ public key, must first be published via the Authentication Service (AS). Then the associated Add Proposal, Commit and Welcome messages will be sent and processed in the PQ session according to Section 12 of RFC9420. The same sequence is repeated in the standard session except following the FULL Commit combining sequence where a PreSharedKeyID proposal is additionally committed. The joiner MUST issue a FULL Commit as soon as possible to acheive PCS. 


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
      [TODO: same comment as on other figure]



### Welcome Message Validation 


Since a client must join two sessions, the Welcome messages it receives to each session MUST indicate that it is not sufficient to join only one or the other. Therefore, the HPQMLSInfo struct indicating the GroupID and ciphersuites of the two sessions MUST be included in the Welcome message via serialization as a GroupContext Extension in order to validate joining the combined sessions. All members MUST verify group membership is consistent in both sessions after a join and the new member MUST issue a FULL Commit as described in Fig 1b. 


### External Joins

External joins are used by members who join a group without being explicitly added (via an Add-Commit sequence) by another existing member. The external user MUST join both the PQ session and the traditional session. As stated previously, the GroupInfo used to create the External Commit MUST contain the HPQMLSInfo struct. After joining, the new member MUST issue a FULL Commit as described in Fig 1b. 

## Removing a Group Member

User removals MUST be done in both PQ and traditional sessions followed by a full update as as described in Fig 1b. Members MUST verify group membership is consistent in both sessions after a removal. 


## Application Messages

HPQMLS combiner provides PQ security the traditional MLS session. Application messages are therefore only sent in the traditional session using the `encryption_secret` provided by the key schedule of the traditional session according to Section 15 of RFC9420. 

# Modes of Operation

Security needs vary by organizations and use-case specific risk tolerance and/or constraints. While this combiner protocol targets combining a PQ session and a traditional session the degree of PQ security may be tuned depending on the use-case: PQ confidentiality only or PQ confidentiality and authenticity. By PQ Confidentiality, we refer to the security provided by PQ KEM protected handshake messages in MLS. By PQ authenticity, we refer to non-repudiation guarantees provided by PQ signatures for handshake and application messages. 

The modes of operation are specified by the `mode` flag in HPQMLSInfo struct and are listed below ordered by least amount of PQ security to most. 



## PQ Confidentiality Only

The default mode of operation is in PQ Confidentiality Only mode. Since a cryptographically relevant quantum computer (CRQC) has not been publicly revealed, the harvest-now-decrypt-later attack suffices as the threat model for the HPQMLS combiner. Under this lense, PQ confidentiality with traditional authenticity are appropriate minimum security goals. 

By using a PQ KEM to protect the key schedule of the PQ session, we can extend the PQ confidentiality to the standard session's key schedule. Recall, this is done via the inject of the exporter key from the PQ session as a pre-shared key in the traditional session's key schedule  which generates the symmetric group keys used for AEAD and MAC calculations. Even if an adversary is successful in breaking the KEM used in the traditional session, they would be unsuccessful in calculating the group secret without also knowing the PSK value derived from the out-of-band PQ KEM. Therefore, in this mode, the PQ session can be defined as using PQ KEM and traditional signatures for handshake messages while the traditional session uses traditional KEM and signature algorithms for application and handshake messages. 


## PQ Confidentiality + Authenticity 

The elevated mode of operation is the PQ Confidentiality and Authenticity mode. If a CRQC exists, then the threat model used in the default mode would be too weak. Thus, the default mode would be insufficient to guarantee authenticity of messages from an external CRQC adversary. Recall that authenticity in MLS refers to two types of guarantees: 1) that messages were sent by a member of the group provided by the computed symmetric group key used in AEAD and 2) that a message was sent by a particular user (i.e. non-repudiation) provided by digital signatures. While the symmetric group key used for AEAD in the traditional session remains protected from a CRQC adversary through the PSK from the PQ session, signatures would not be secure against forgery without using a PQ DSA to sign handshake messages to guarantee non-repudiation against a CRQC adversary. Therefore, in this mode the PQ session MUST use a PQ DSA in addition to PQ KEM ciphersuites for handshake messages (the traditional session remains unchanged). 



# Extension Requirements to MLS

The HPQMLSInfo struct contains characterizing information to signal to users that they are participating in a combined session. This is necessary both functionally to allow for group synchronization and as a security measure to prevent downgrading attacks to coax users into parcipating in just one of the two sessions. The `group_id`, `cipher_suite`, and `epoch` from both sessions (`t` for traditional and `pq` for the Post Quantum session) are used as bookkeeping values to validate and synchronize group operations. The `mode` is a boolean value: `0` for the default PQ Confidentiality Only mode and `1` for the PQ Confidentiality and Authenticity mode. 

The HPQMLSInfo struct conforms to the Safe Extensions API (see [I-D.ietf-mls-extensions]). Recall that an extension is called *safe* if it does not modify base MLS protocol or other MLS extensions beyond using components of the Safe Extension API. This allows security analysis of our HPQMLS Combiner protocol in isolation of the security guarantees of the base MLS protocol to enable composability of guarantees. The HPMLSInfo extension struct SHALL be in the following format: 

[BH: What's in the clear here? Trad session vs T session notation]

      struct{
          ExtensionType HPQMLS;
          opaque extension_data<V>; 
          } ExtensionContent; 

      struct{
          opaque t_session_group_id<V>; 
          opaque PQ_session_group_id<V>; 
          bool mode; 
          CipherSuite t_cipher_suite; 
          CipherSuite pq_cipher_suite; 
          uint64 t_epoch; 
          uint64 pq_epoch;   
      } HPQMLSInfo


## Key Schedule

The `hpqmls_psk` exporter key derived in the PQ session MUST be derived in accordance with the Safe Extensions API guidance (see 2.1.5 Exporting Secrets in [I-D.ietf-mls-extensions]). In particular, it SHALL NOT use the `extension_secret` and MUST be derived from only the `epoch_secret` from the key schedule in [[RFC9420]](https://www.rfc-editor.org/rfc/rfc9420.html). This is to ensure forward secrecy guarantees (see [Security Considerations](#security-considerations)). 

Even though the `hpqmls_psk` is not sent over the wire, members of the HPQMLS session must agree on the value of of which PSK to use. In alignment with the Safe Extensions API policy for PSKs, HPQMLS PSKs used SHALL set `PSKType = 3` and `extension_type = HPQMLS` (see Section 2.1.6 Pre-Shared Keys in [I-D.ietf-mls-extensions]). 
        
      
      PQ Session                       Traditional Session
      ----------                       -------------------  

        [...] 
    DeriveExtensionSecret(epoch_secret, 
          |            "hpqmls_export")    
          | = hpqmls_psk                      [...]
          |                               joiner_secret
          |                                     |
          |                                     |
          |                                     V
          +----------> <psk_secret (or 0)> --> KDF.Extract
        [...]                                   |
                                                |
                                                +--> DeriveSecret(., "welcome")
                                                |    = welcome_secret
                                                |
                                                V
                                        ExpandWithLabel(., "epoch", GroupContext_[n], KDF.Nh)
                                                |
                                                |
                                                V
                                          epoch_secret
                                                |
                                                |
                                                +--> DeriveSecret(., <label>)
                                                |    = <secret>
                                              [...]
    Fig 3: The hpqmls_psk of the PQ session is injected into the key schedule of the 
    traditional session using the safe extensions API DeriveExtensionSecret. 



# Security Considerations

## Full Commit Frequency 

So long as the FULL Commit flow is followed for group administration actions, PQ security is extended to the traditional session. Therefore, FULL Commits can occur as frequently or infrequently as desired by any given security policy. This results in a flexible and efficient use of compute,  storage, and bandwidth resources for the host by mainly calling partial updates on the traditional MLS session, given that the group membership is stable. Thus, our protocol provides PQ security and can maintain a tighter forward secrecy and post-compromise security window with lower overhead when compared to running a single MLS session that only uses hybrid signatures or only PQ KEM/DSAs. 

## Attacks on Authentication
While message integrity is protected by the symmetric key, non-repudiation attacks (e.g. forgery, impersonation, masquerading) on the digital signature of messages may be possible by a CQRC adversary. Such an attack can cause confusion in group state agreement leading to denial of service or entity authentication of the sender. If this is a concern, we recommend using hybrid PQ DSAs in the traditional session to sign messages. Note, this would negate much of the efficiency gains from using this protocol. 

Digital signatures in the traditional session in the PQ Confidentiality Only mode are susceptible to forgery attacks by a CRQC adversary. However, in terms of group key agreement, this is insufficient to mount anything more than a denial of service attack (e.g. via group state desynchronization). In terms of application messages, while a traditional DSA signature may be forged by an external CRQC adversary, the content (including sender information) is still protected by AEAD which uses the symmetric group key. Thus, only an insider CRQC adversary could actually mount masquerading or forgery attacks which is beyond the scope of this protocol.  

## Forward Secrecy (FS)
Recall that one of the ways MLS achieves FS is by deleting security sensitive values after they are consumed (e.g. to encrypt or derive other keys/nonces). For example, values such as the `init_secret` or `epoch_secret` are deleted at the *start* of a new epoch. If the MLS `exporter_secret` or the `extension_secret` from the PQ session is used directly as a PSK for the traditional session, then there is a scenario in which an adversary can break FS because these keys are derived *during* an epoch and are not deleted. Therefore, the `hpqmls_psk` must be derived from the `epoch_secret` of the PQ session to ensure FS (see Figure 3). 

## Transport Security 
Recommendations for preventing denial of service (DoS) attacks, or restricting transmitted messages are inherited from MLS. Furthermore, message integrity and confidentiality is, as for MLS, protected. 


# IANA Considerations 
The MLS sessions combined by this protocol conform to the IANA registries listed for MLS RFC9420. 

<!---
## MLS Exporter Label
The MLS Exporter Label 


|Label   | Recommended | Reference |
|:---    |:---       |:----|
|hpqmls_export| N| This Document
|hpqmls_psk| N | This Document 
--->

# References

## Normative References (i.e. RFCs)

[I-D.ietf-mls-extensions] Robert, R., "The Messaging Layer Security (MLS) Extensions", Work in Progress, Internet-Draft, draft-ietf-mls-extensions-04, 24 April 2024,  <https://datatracker.ietf.org/doc/html/draft-ietf-mls-extensions-04>

[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, DOI 10.17487/RFC2119, March 1997, <https://www.rfc-editor.org/rfc/rfc2119>.

 [RFC8174]  Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174, May 2017, <https://www.rfc-editor.org/rfc/rfc8174>.

[RFC9420] Barnes, R., Beurdouche, B., Robert, R., Millican, J., Omara, E., and K. Cohn-Gordon, "The Messaging Layer Security (MLS) Protocol", RFC 9420, DOI 10.17487/RFC9420, July 2023 <https://www.rfc-editor.org/rfc/rfc9420>. 


## Informational References 
TODO

<!--# Appendices -->


# Acknowledgments
{:numbered="false"}
## Contributors 

## Authors 
