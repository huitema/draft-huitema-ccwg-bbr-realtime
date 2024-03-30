---
title: "BBR Improvements for Realtime connections"
abbrev: "BBR-Realtime"
category: info

docname: draft-huitema-ccwg-bbr-realtime-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Congestion Control Working Group"
keyword:
 - BBR
 - Realtime Communication
 - Media over QUIC
venue:
  group: "Congestion Control Working Group"
  type: "Working Group"
  mail: "ccwg@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/ccwg/"
  github: "huitema/draft-huitema-ccwg-bbr-realtime"
  latest: "https://huitema.github.io/draft-huitema-ccwg-bbr-realtime/draft-huitema-ccwg-bbr-realtime.html"

author:
 -
   ins: C. Huitema
   name: Christian Huitema
   org: Private Octopus Inc.
   email: huitema@huitema.net

normative:

informative:
   QUIC-TRANSPORT: rfc9000
   I-D.cardwell-iccrg-bbr-congestion-control:
   BBRv3-Slides:
    target: https://datatracker.ietf.org/meeting/117/materials/slides-117-ccwg-bbrv3-algorithm-bug-fixes-and-public-internet-deployment-00
    title: "BBRv3: Algorithm Bug Fixes and Public Internet Deployment"
    date: "Jul 26, 2023"
    seriesinfo: "Materials for IETF 117 Meeting"
    author:
    -
      ins: N. Cardwell
    -
      ins: Y. Cheng
    -
      ins: K. Yang
    -
      ins: D. Morley
    -
      ins: S. Hassas Yeganeh
    -
      ins: P. Jha
    -
      ins: Y. Seung
    -
      ins: V. Jacobson
    -
      ins: I. Swett
    -
      ins: B. Wu
    -
      ins: V. Vasiliev






--- abstract

BBR is great, BBR for realtime could be better. here is how.


--- middle

# Introduction

Motivation: avoid building queues, support "layered" media encoding
or "simulcast" transmission.

Assume that the realtime application carries media over QUIC in a series of
QUIC streams. Each of these streams is marked with a scheduling priority. For
example, in a "simulcast" service, audio packets may be sent as QUIC datagrams
scheduled at a high priority, then a low definition version of the video
stream in a QUIC stream marked at the next highest priority, then medium
definition video in another stream, then high definition video in the lowest
priority stream. At any give time, the sender could schedule data according
to the connection's capacity as evaluated by the congestion control algorithm.
If the path has a high capacity the receiver will receive all QUIC streams and
enjoy a high definition experience. If the capacity is lower, the higher
definition streams will be delayed but the receiver will reliably obtain
the medium or low definition version of the media.

This realtime retransmission strategy relies on timely assessment of the
path capacity by the congestion control algorithm. If the assessment is delayed,
the scheduling algorithm will make wrong decisions, such as wrongly believing that
the path does not have the capacity to send high definition media, or in contrast
sending high definition media and causing queues and maybe packet losses
because the lowering of the path capacity has not yet been detected.

Other realtime applications may use different categories of traffic than low
or high definition video, but they will follow the genral principle of trying
to schedule just the right amount of transmission to obtain a good experience
without creating queues.

In our experience, we see that BBR generally works well for these applications.
We have experimented with BBR V3, define by the BBRv2 IETF 
Draft {{I-D.cardwell-iccrg-bbr-congestion-control}} and by
complementary data in a presentation to the IRTF {{BBRv3-Slides}}.
However, we see problems
in the early stage of the connection, when the path capacity is not yet
assessed, and during sudden transitions, such as experienced in Wi-Fi networks.


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
