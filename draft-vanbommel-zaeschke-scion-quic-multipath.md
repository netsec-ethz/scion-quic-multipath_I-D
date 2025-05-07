---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "QUIC Multipath over SCION"
abbrev: "SCION-QUIC-MP"
category: info

docname: draft-vanbommel-zaeschke-scion-quic-mp-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date: 2025-05-07
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: netsec-ethz/scion-quic-multipath_I-D
  latest: https://example.com/LATEST

author:
 -
    fullname: Jelte van Bommel
    organization: ETH Zurich - NetSec Group
    email: your.email@example.com
 -
    fullname: Tilmann Zäschke
    organization: ETH Zurich - NetSec Group
    email: tilmann.zaeschke@inf.ethz.ch
normative:

informative:


--- abstract

Using Multipath Extension for QUIC [xxx] with SCION provides unique
opportunities application but also for for congestion control, 
path selection and related algorithms.
This document discusses some opportunities and general recommendations.

--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
