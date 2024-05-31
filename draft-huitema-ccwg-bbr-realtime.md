---
title: "BBR Improvements for Real-Time connections"
abbrev: "BBR-Realtime"
category: info

docname: draft-huitema-ccwg-bbr-realtime-latest
submissiontype: IETF
number:
date:
consensus: true
ipr: trust200902
area: "Web and Internet Transport"
keyword:
 - BBR
 - Realtime Communication
 - Media over QUIC

author:
 -
   ins: C. Huitema
   name: Christian Huitema
   org: Private Octopus Inc.
   email: huitema@huitema.net

normative:
informative:
   RFC9000:
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

We are studying the transmission of real-time Media over QUIC. There are
two priorities: maintaining low transmission delays by avoid queues, and
using the available network capacity to provide the best possible
experience. We found through experiments that while the BBR algorithm
generally allow us to correctly and timely assess network capacity,
we still see "glitches" in specific conditions, in particular when
using wireless networks. We analyze these issues and
propose small changes in the BBR algorithm that could improve the quality
of experience delivered by the real-time media application.


--- middle

# Introduction

When designing real-time communication applications over QUIC, we want to
maintaining low transmission delays but provide the best possible
experience that the current transmission capacity allows.
We found through experiments that while the BBR algorithm
generally allow us to correctly and timely assess network capacity,
we still see "glitches" in specific conditions, during which the
BBR algorithm either allows the building of unnecessary queues or
unnecessarily restrict the transmission rate of the application.

The real-time application carries media over QUIC in a series of
QUIC streams {{RFC9000}}. Each of these streams is marked with a scheduling priority.
For example, in a "simulcast" service, audio packets may be sent as QUIC datagrams
scheduled at a high priority, then a low definition version of the video
stream in a QUIC stream marked at the next highest priority, then medium
definition video in another stream, then high definition video in the lowest
priority stream. At any given time, the sender would schedule data according
to the connection's capacity as evaluated by the congestion control algorithm.
If the path has a high capacity the receiver will receive all QUIC streams and
enjoy a high definition experience. If the capacity is lower, the higher
definition streams will be delayed but the receiver will reliably obtain
the medium or low definition version of the media.

This real-time retransmission strategy relies on timely assessment of the
path capacity by the congestion control algorithm. If the assessment is delayed,
the scheduling algorithm will make wrong decisions, such as wrongly believing that
the path does not have the capacity to send high definition media, or in contrast
sending high definition media and causing queues and maybe packet losses
because the lowering of the path capacity has not yet been detected.

Other real-time applications may use different categories of traffic than low
or high definition video, but they will follow the general principle of trying
to schedule just the right amount of transmission to obtain a good experience
without creating queues.

Real-time transmissions have a variable data rate. Different media codec have
different behavior, but a common pattern is to periodically send "fully encoded"
frames, followed by series of "differentially encoded" frame. The fully encoded
frames serve as possible synchronization point at which a recipient can start
rendering data, while the differentially encoded frames can only be processed
if the previous frames have been received. This structure create periodic peaks
of traffic, during which the connection might experience some for of congestion,
followed by long periods of relatively low bandwidth demand, which the congestion
control algorithm may treat as "application limited". Of course, if the bandwidth
is too low, even the differentially encode frame may create some congestion. In the
simulcast structure, that would react to that by delaying the higher definition
video channels.

In our experience, we see that BBR generally works well for these applications.
We have experimented with BBR V3, define by the BBRv2 IETF
Draft {{I-D.cardwell-iccrg-bbr-congestion-control}} and by
complementary data in a presentation to the IRTF {{BBRv3-Slides}}.
However, we see problems
in the early stage of the connection, when the path capacity is not yet
assessed, and during sudden transitions, such as experienced in Wi-Fi networks.
There are also details of the algorithm that work poorly when the traffic
alternate between periodic congestion and application limited periods.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Problem statement

With BBRv3, we see less than ideal "real-time" performance during the initial phase
of the connection, during Wi-Fi suspension events, or when the quality of the
wireless transmission changes suddenly. We also see anomalous behavior that may
be tied to the "application limited" nature of the application.

These issues are developed in the following sections.

## Extra delays during initial startup

## Wi-Fi suspension

The current behavior is that the sender, using BBR, detects the Wi-Fi suspension about 1 RTT after it happens, at which point it stops polling for new data. Then, maybe 60 to 100ms later depending on conditions, it starts receiving ACK from the peer that were queued at the router during the suspension. Picoquic and BBR refresh the congestion parameters, and start polling the application. The queued data will be sent in order of priority, ending with the 1080p frames.

We can describe this process as a succession of phases:

* Normal transmission, at the application "nominal" rate when all quality
  levels are sent, with no queue.
* Undetected suspension, during which queues are building up with a mix of
  data of various priority levels
* Detected suspension, during which no more data is sent
* Resumption, during which the sender using BBR progressively ramps up the
  sending rate and dequeues the frames stuck in applcation queues, in order of priority
* and back to normal state, when the application queues are emptied rapidly.

The queuing happens in the "undetected suspension" state, which currently lasts one PTO,
i.e., a bit more than one RTT. So we get one RTT worth of audio, 360p, 720p and 1080p
frames in the pipe-line, typically queued in front of the Wi-Fi driver. These frames
will be delivered by the network before any other frame sent during the "resumption".
We will observe some kind of "priority inversion".

If new suspension happens shortly after the first one interval, it may catch the system
during the "resumption" state. During that state, the sender may have ramped up the data
rate to match the underlying network rate (maybe 100 Mbps) and empty the application
queues quickly. The transmission rate may be several times the normal rate. The amount
of frames queued in front of the Wi-Fi driver will be several times more than during
the first suspension. The "priority inversion" effect will be much larger. Depending
on random events, we may see the 360p video freezing while the 1080p video is still animating.

## Bad Wifi Spot

## Downward drift

In addition to suspension, we also observe that the congestion control API tends to gradually drop the packing rate over the long run.

It seems that the application never fully uses the available rate, and that successive queuing events cause the bandwidth to go down progressively. Maybe we have something systematic here? The application always stays within the limits posed by congestion control, which means that the measured rate will always be lower than the allowed rate.. BBR sometimes move to a "probe BW UP" state, but the application does not appear to push much more transmission during that state, so the measured rate does not really increase.

The hypothesis is that the application never sends faster than pacing permits, so there is some kind of negative feedback loop.

The downward spiral may be compounded by the practice of resetting streams that have fallen behind.
This is line with discussions in MoQ. A new stream will be restarted for the next block of frames,
starting with the next I-Frame. The application will try to send more data at that point, but the
probing rate will stay low until until the next "probe BW UP" phase, which may be a few seconds away.
When BBR probes for more data, the high speed stream may have already been reset, and the offered
bandwidth will not really "push up" the data rate.

## Lingering in Probe BW UP state

# Proposed improvements

We decided three short term actions:

* Revise the "suspension" tests in the picoquic test suite to verify the behavior of the 'quality' API.

* Add a "max pacing rate" API to picoquic, so the application can limit how fast picoquic is willing
to send data. This will most likely apply during the "resumption" described above,
limiting how fast the application is willing to empty the application queues.
It should limit the amount of data queued during the "undetected suspension"
phase, and thus the impact of "priority inversion" during these phases.

* Study an additional "No Feedback" event passed by the stack to the CC algorithm.
Normally, picoquic receives trains of acknowledgements at regular intervals,
much shorter that the RTT. In case of suspension, the stream of ACKs stops.
The "No Feedback" API would detect that stoppage much faster than waiting for a PTO.
The stack could thus temporarily stop polling for new data until new ACKs are received,
which would limit the amount of data queued in front of the Wi-Fi drivers, and thus also limit the effect of "priority inversion".

BBR only increases the bandwidth during the "Probe BW UP" state. We want to trigger that "UP" state when the application has new data to send, for example when new streams are being opened.

We discussed that before. I think we should try an option to add "filler" traffic during the "probe BW UP" phases. Maybe send redundant copies of the previously transmitted packets, maybe send padded packets. This may well backfire, so the emphasis is "try". Probably add a configuration option to control the behavior, and also set a limit, such as "add probing traffic if measured rate is lower than 20 Mbps".


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
