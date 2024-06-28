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
 -
   ins: S. Nandakumar
   name: Suhas Nandakumar
   organization: Cisco
   email: snandaku@cisco.com
 -
   ins: C. Jennings
   name: Cullen Jennings
   organization: Cisco
   email: fluffy@iii.ca

normative:
informative:
   RFC9000:
   I-D.cardwell-iccrg-bbr-congestion-control:
   I-D.ietf-quic-ack-frequency:
   I-D.ietf-moq-transport:
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

   Wi-Fi-Suspension-Blog:
    target: https://www.privateoctopus.com/2023/05/18/the-weird-case-of-wifi-latency-spikes.html
    title: "The weird case of the wifi latency spikes"
    date: "May 18, 2023"
    seriesinfo: "Christian Huitema's blog"
    author:
    -
      ins: C. Huitema



--- abstract

We are studying the transmission of real-time Media over QUIC. There are
two priorities: maintain low transmission delays by avoiding building queues, and
use the available network capacity to provide the best possible
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
experience that the current transmission capacity allows. We elected
to use the version 3 of the BBR congestion control algorithm
(see {{I-D.cardwell-iccrg-bbr-congestion-control}} and
{{BBRv3-Slides}}).
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

Real-time transmissions have a variable data rate. Different media codecs have
different behavior, but a common pattern is to periodically send "fully encoded"
frames, followed by series of "differentially encoded" frame. The fully encoded
frames serve as possible synchronization points at which a recipient can start
rendering data, while the differentially encoded frames can only be processed
if the previous frames have been received. This structure create periodic peaks
of traffic, during which the connection might experience some for of congestion,
followed by long periods of relatively low bandwidth demand, which the congestion
control algorithm may treat as "application limited". Of course, if the bandwidth
is too low, even the differentially encode frame may create some congestion. In the
simulcast structure, the application would react to congestion by delaying the higher definition
video channels.

In our experience, we see that BBR generally works well for these applications.
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

## Extra delays during initial startup {#startup-queues}

At the beginning of the connection, BBR is initially in Startup state.
In Startup BBR sets BBR.pacing_gain to BBRStartupPacingGain (2.77) and BBR.cwnd_gain to
BBRStartupCwndGain (2). BBR exits the end of the startup phase when the estimated bandwidth
does grow significantly (by at least 25%) for 3 consecutive rounds, or if packet losses
are detected. This results in
an algorithm that robustly discovers the bottleneck bandwidth. Inconveniently for
us, is also results in building significant queues.

Towards the end of the Startup phase, we reach a point when both the bottleneck bandwidth
and the min RTT have been correctly estimated. After that point, per specification, the
pacing rate will be set to 2.77 times the bottleneck bandwidth, and the CWND to
twice the product of estimated bandwidth and min RTT. Since the pacing rate is significantly
larger than the bottleneck capacity, packets will be queued at the bottleneck. If the
bottleneck has sufficient buffers, the queue
size will increase until the amount of bytes in transit matches the CWND value. The RTT
will increase to twice the min RTT, and will remain at that level for 3 roundtrips, which
means 6 times the minimum RTT.

The extra buffers and the extra delays do affect the performance of real time application.
The application will end up sending more "low priority" data than necessary, resulting in
a form of "priority inversion" as high priority data are queued and low priority data are
delivered during the "draining" phase that follows startup.

## Sensitivity to early bandwidth estimation {#early-sensitivity}

After Startup and Drain, BBR will enter the ProbeBW state. BBRv2 organizes those as
series of epochs, each composed of a ProbeBW-Down rounds, a variable number of ProbeBW-Cruise
rounds, a ProbeBW-Refill round, and finally a series of ProbeBW-Up phases. The number of
ProbeBW-Cruise rounds is computed so that BBR competes fairly with other congestion
control algorithms like Reno or Cubic. In some conditions, there can be up to 60 rounds
of ProbeBW-Cruise, during which the pacing rate will never exceed the bandwidth
estimated previously.

If BBR exited the Startup state too soon for any reason, the large number of ProbeBW-Cruise
causes the connection to retain a very low pacing rate for a long time. In real time
applications, this could mean using a much lower video definition than actual bandwidth permits.

## Wi-Fi suspension {#suspension}

We observed that Wi-Fi networks often go into "suspension" for brief intervals
(see {{Wi-Fi-Suspension-Blog}}, typically of 100 to 200ms. During the transmission,
the Wi-Fi radio appears to be turned off, or maybe switched to a different channel.
Packets sent by the application during the interval are kept in a queue for the Wi-Fi driver.
Packets sent by remote nodes are at the Wi-Fi router. After the suspension, queued
packets are sent. There is generally no packet loss, merely an added delay. We have
observed this behavior on multiple Wi-Fi networks, using different brands of Wi-Fi
equipment and different operating systems. We can speculate about the cause of the
suspension, such as exploring alternative Wi-Fi channels or saving energy.

When Wi-Fi suspension happens, the sender, using BBR, keeps sending at the computed
pacing rate as long as the congestion window allows -- these packets will be queued.
The suspension often lasts longer than 1 PTO, at which point no new data will be sent
but some queued packets maybe repeated. Then, at the end of the suspension,
the sender starts receiving ACK from the peer that were queued at the router.
BBR refreshes the congestion parameters, and start polling the application.
New data will be sent in order of priority, ending with the 1080p frames.

We can describe this process as a succession of phases:

* Normal transmission, at the application "nominal" rate when all quality
  levels are sent, with no queue.
* Undetected suspension, during which queues are building up with a mix of
  data of various priority levels
* Detected suspension, during which no more data is sent and new audio or
  video frames are kept in application queues.
* End of suspension, during which the Wi-Fi driver sent the queued packets,
* Resumption, during which the sender using BBR progressively ramps up the
  sending rate and dequeues the frames stuck in application queues, in order of priority
* and back to normal state, when the application queues are emptied rapidly.

The driver queue fills during the "undetected suspension" state, which currently lasts one PTO,
i.e., a bit more than one RTT. So we get at least one RTT worth of audio, 360p, 720p and 1080p
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

## Spotty or Bad Wi-Fi Transmission

Wi-Fi bandwidth can drop suddenly due to small changes in the position or orientation
of the device, or if a mobile obstacle like a human body moves into the path of
transmission. We see small events causing a drastic reduction in bandwidth, and
sometimes a marked increase in packet loss rates. We expect that the congestion
control algorithm will quickly detect the bandwidth reduction, and inform the
application, but in practice the detection always lag the event.

As in the Wi-Fi suspension scenario, the delay in detecting the bandwidth reduction
causes priority inversions. The application will continue sending data at the previously
allowed rate for some time, which will cause less important packets (say 1080p)
to be enqueued for transmission. These packets will likely be queued in front of
the Wi-Fi driver, and will be transmitted before any more urgent data can be sent.

BBRv3 will reduce the CWND quickly if packet losses are detected, but these losses
will typically only be detected after a couple of RTT -- or maybe more if the
packet queues are allowed to grow to large sizes. The effect is amplified if the
pacing rate or the CWND was over estimated before the event, maybe because the
connection was application limited.

The bandwidth will be restored when the position or orientation of the device
changes again, or if the obstacles are removed. When that happens, we expect
that the congestion control algorithm will find out rapidly and let the application
resume full transmission, but this will only happen after when BBR reaches the
ProbeBW UP state, which may take a large number of RTTs with BBR v3.

## Downward drift

In addition to suspension, we also observe that the congestion control API tends to gradually drop the packing rate over the long run.

It seems that the application never fully uses the available rate, and that successive queuing events cause the bandwidth to go down progressively. Maybe we have something systematic here? The application always stays within the limits posed by congestion control, which means that the measured rate will always be lower than the allowed rate. BBR sometimes move to a "probe BW UP" state, but the application does not appear to push much more transmission during that state, so the measured rate does not really increase.

The hypothesis is that the application never sends faster than pacing permits, so there is some kind of negative feedback loop.

The downward spiral may be compounded by the practice of resetting streams that have fallen behind.
A new stream will be restarted for the next block of frames,
starting with the next I-Frame, using the "stream per group" approach defined in
{{I-D.ietf-moq-transport}}.
The application will try to send more data at that point, but the
probing rate will stay low until until the next "probe BW UP" phase, which may be a few seconds away.
When BBR probes for more data, the high speed stream may have already been reset, and the offered
bandwidth will not really "push up" the data rate.

## Lingering in Probe BW UP state {#lingering}

The offered rate of real-time application alternates between rare peaks during the
transmission of I-Frames, and a lower data rate between these frames. In a typical
setup, BBR will operate in "application limited" mode most of the time, except
maybe during the transmission of I-Frames, or if the network becomes congested.

If BBR reaches the Probe BW UP state while application limited, it will only exit
that state when the next I-Frames are sent, which may be seconds away, or if congestion
happens. When congestion or Wi-Fi suspension happens in that state, both the
pacing rate and the congestion window will be larger than the "cruise" value.
In the case of congestion, the connection will remain in
Probe BW UP state during three rounds. This
will exacerbate the building of queues and the occurrence of priority inversion
during congestion events.

# Proposed improvements {#improv}

We implemented several improvements to BBR for handling real time applications:

* implement an "early exit" from startup.

* add an option for rapid start of ProbeBW-Up,

* add explicit handling of "suspension" to BBR,

* add detection of feedback loss to minimize building of queues during suspension,

* exiting Probe BW UP on delay increase,

* entering Probe BW UP after new streams are started

## Exit startup early upon RTT increase

BBR exits start up if packet losses are observed, or if the estimated bottleneck bandwidth
does not increase for 3 rounds. We added a third exit condition, exit the startup state if the RTT increases too much.

We defined too much as "the RTT measurement is at least 25% larger than the min RTT. However, there
can be significant jitter in RTT measurements, and we do not want to exit startup based
solely on an event caused by random circumstances. Instead, we exit Startup only if
7 consecutive measurements of the RTT are significantly larger than the Min RTT.

Even with the requirement of 7 measurements, there is still a risk that spurious events cause
early exit of startup while the bandwidth is not properly assessed. We mitigate these risks
by requiring an early entry in ProbeBW-Up state during the first ProbeBW epoch after StartUp.

## Rapid entry into ProbeBW-Up

We add to the BBR state a flag "bw_probe_quickly", to signal a desire to enter the ProbeBW-Up
state at the first opportunity. When that flag is set, instead of moving to the ProbeBW-Cruise
state at the end of the ProbeBW-Down round, BBR moves directly to ProbeBW-Refill and then ProbeBW-UP.
The flag is cleared after starting ProbeBW-UP.

During each round of ProbeBW-Up, BBR will increase the pacing rate to 25% more than the measured
value. The combination of exiting Startup after a delay test and then starting ProbeBW-UP quickly
is very similar to the two phases of Hystart++ {{?RFC9406}}.

## Explicit handling of suspension

During Wi-Fi suspension, the BBR endpoint will not receive any acknowledgement. This
will cause the PTO and then RTO timers to expire. We handle that issue by adding a
"PTO recovery" flag to the BBR state. The flag is set if a PTO event occurs, at which
point BBR enters "packet conservation" mode, in which only a single packet is sent
after each PTO or RTO event.

The first acknowledgement receiver after setting the PTO recovery flag causes the flag
to be cleared, and BBR to reenter the "Startup" state.

## Detection of feedback loss.

Detecting suspension by waiting for a PTO works, but still allows transmission of a full
CWND worth of packets between the start of the suspension and the PTO timer. We mitigate
that by adding a "feedback lost" event.

In normal operation, successive acknowledgements are received at short intervals.
These intervals can be predicted if the QUIC implementation supports the "ACK Frequency"
extension {{I-D.ietf-quic-ack-frequency}}, which directs the peer to acknowledge
packets before a maximum delay. The "feedback lost" event is set if expected
packet acknowledgements do not arrive after twice the predicted interval.

Our modified BBR code reacts to this event by resetting the CWND to the current "in-flight" value,
which effectively prevents adding new packets until a next acknowledgement is received.
The "feedback lost" event typically arrives sooner than the PTO event, and thus helps
avoiding queue building
during the "undetected" phase of suspension defined in {{suspension}}.

## Exit ProbeBW-Up on delay increase.

We added a test of delay increase when in ProbeBW-Up state. This helps avoiding build
excessive queues in ProbeBW-up state, and it also avoids lingering in ProbeBW-up state
(see {{lingering}}).

## Entering ProbeBW-UP on the creation of new streams

The bandwidth drift issue happens largely because there is no synchronization between
the increase of bandwidth demand and the probing cycle of BBR. We are considering adding
a mechanism to synchronize bandwidth probing with period of higher bandwidth demand,
characterized for example with the beginning of new streams.

At the time of this writing, this synchronization mechanism is not yet implemented.

# Failed experiments

In {{improv}}, we provide a list of improvements to BBR to better serve real-time applications.
We validated these improvements with a mix of simulations and real-life testing. We also considered
other improvements, which appeared logical at first sight but were not validated by simulations or
experience: pushing the bandwidth limit by injecting fake traffic; and,

## Push limits with fake traffic

We were concerned that BBR did not exploit the full bandwidth of the connection. Real-time
applications often appear "bandwidth limited", and the bandwidth discovered by BBR is never
greater than the highest bandwidth previously used by the application. This can slow down the
transmission of data during peaks,
for example when the video codec decides to produce a "self referenced" frame (sometimes called I-Frame). To solve that, we tried an option to add "filler" traffic during the "probe BW UP" phases,
instructing the QUIC code to schedule redundant copies of the previously transmitted packets in order
to "fill the pipe".

This did not quite pan out. The various simulation did not show any particular improvement, and
in fact some test cases show worse results. Pushing the bandwidth limits increases the risk
of filling bottleneck buffers and causing queues or losses, and discovering a high bandwidth did not really compensate for that.

The theoretical case for injecting fake traffic during just a few segments of the connection is
also weak. Suppose that multiple connections competing for the same bottleneck do that. If they
probe the bandwidth at different times, they may all find that there is a lot of capacity available.
But if all the connection try to use that bandwidth, they will experience congestion.
Similarly, if a real time connection competes with an elastic connection, the elastic
connection may back off a little when the real-time connection is probing, but it will
quickly discover and consume the available bandwidth after that. If the application wanted
to "reserve" a higher bandwidth, it would probably need to inject filler traffic all the time, to
"defend" its share of bandwidth. We were not sure that this would be a good idea.

## Flow control low priority streams

We tested adding fewer packets in transit for low priority flows so there will be fewer "priority inversions" during events like Wifi Suspension. We implemented that using QUIC's flow control
functions. The results did not really confirm our hypothesis. The change appears to create two
competing effects:

* having fewer packets in transit does limit the priority inversions, and some tests to show a (very
small) improvement,

* but when the low priority flows are rate limited, their load gets spread overtime, instead of
being dealt with immediately when plenty of bandwidth is available. This means more packets
lingering over time and thus more priority inversions during network events.

The measurements also show that any random change in scheduling does create random changes in
performance, which means that observing small effects can be very misleading. Given the lack of
compelling results, we did not push the idea further.

# Security Considerations

We do not believe that changes in the BBR implementation introduce new security issues.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
