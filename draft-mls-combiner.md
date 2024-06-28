---
title: PQ MLS Combiner
abbrev: PQCMLS
docname: draft-hale-mls-combiner-00
category: info

ipr: trust200902
area: Security
keyword: 
  - security
  - authenticated key exchange
  - PCS

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
  - ins: "M. Mularczyk"
    name: "Marta Mularczyk" 
    organization: "AWS" 
    email: 
  - ins: "B. Hale"
    name: "Britta Hale"
    organization: "Naval Postgraduate School"
    email: britta.hale@nps.edu
  - ins: "X. Tian"
    name: "Xisen Tian"
    organization: "Naval Postgraduate School"
    email: xisen.tian1@nps.edu



--- abstract 
This document describes a protocol for combining a standard MLS session with a postquantum MLS session to achieve hybrid security. 

--- middle 

# Introduction


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




# Extension Execution 


### Issuing an Update


                                                                    Group
    A            B              G1  ...    Gn         Directory     Channel
    |Update(B,f) |              |          |              |           |
    |Commit(Upd) |              |          |              |           |
    |Notify(B)   |              |          |              |           |
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
