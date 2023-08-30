---
title: "QUIC in Space"
abbrev: "QUIC in Space"
category: info

docname: draft-huitema-quic-in-space-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Transport"
workgroup: "QUIC"
keyword:
 - quic
 - delay tolerant
 - space communication
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "huitema/quic-in-space"
  latest: "https://huitema.github.io/quic-in-space/draft-huitema-quic-in-space.html"

author:
 -
    fullname: Christian Huitema
    organization: Private Octopus Inc.
    email: huitema@huitema.net
 -
    fullname: Marc Blanchet
    organization: Viagenie
    email: marc.blanchet@viagenie.ca

normative:

informative:

    DTN-ARCH: rfc4838
    QUIC-TRANSPORT: rfc9000
    QUIC-RECOVERY: rfc9002

--- abstract
This document discusses the challenges of running the QUIC transport over deep space links,
where delays are in order of minutes and communications are based on scheduled time windows.
Using the experience of various testbeds, it provides guidance to implementations to support
this use case. This document may apply to other use cases that have similar characteristics,
such as IoT in disconnected and distant settings.

--- middle

# Introduction

QUIC is a new transport bringing very interesting features that could enable
its use in space, where TCP is not good, as assessed in {{DTN-ARCH}}.
However, QUIC was designed for
terrestrial Internet, which brings assumptions on typical delays and
connectivity. In (deep) space, delays are much larger, in order of minutes
(4-20 minutes to Mars), and long disruptions, such as because of orbital
dynamics, in order of minutes or hours or days.

It may be possible to modify the base behavior of QUIC stacks to satisfy
these requirements. For example, several assumptions, such as initial RTT,
are just static constants in the code that could be externalized so they
could better start the QUIC machinery in the context of space.

The purpose of this document is to provide guidance for supporting space
communications in QUIC implementations. It should be noted that it may also apply
to other use cases that have similar characteristics, such as IoT in disconnected
and far away settings, but these are not considered specifically in this document.

Various other considerations such as how to store packets during disruptions or
network management considerations are out of scope of this document.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Timer Constants in QUIC

QUIC implementations typically use a number of time related variables,
with different characteristics. Some may be constants suggested by the
QUIC specification or chosen by the stack developers, some may
be negotiated between peers, and some may be discovered as part of running
the protocol. Preliminary tests showed that setting the constant values
appropriately seemed to make QUIC usable in those deep space scenarios. Therefore,
this document discusses them in the following sections.

## Probe Timeout and Initial RTT (#initial-rtt)

As defined in {{QUIC-RECOVERY}}, QUIC uses two mechanisms to detect packet losses.
The acknowledgement based method detect losses if a packet is not yet
acknowledged while packets sent later have already been. That method works
well even if the transmission delay is long, but cannot detect the loss
of the "last packet". For that, QUIC uses the probe timeout defined
in {{Section 6.2 of QUIC-RECOVERY}}. If the last packet is not yet acknowledged
after the probe timeout, the endpoint sends a "probe" to trigger an acknwledgement.
The probe timer is initially set as a function of the measured RTT and RTTVAR. It
is then increased exponentially as the number of unacknowledged repetitions increases.

The mechanism works well if the transmission delay is long after the RTT has
been evaluated at least once. But before that, the probe timeout is set as a function of
the Initial RTT, whose recommended value is 333 milliseconds per {{Section 6.2.2 of
QUIC-RECOVERY}}. Many implementations use smaller values
because waiting too long results in longer connection delays when losses occur. The
recommended initial value of 333ms results in a PTO of 1 second, but the shorter values
used by some implementations can result in a PTO of 200 or 250ms. On a long delay link,
we will probably see the following:

1. Initial transmission
2. First repeat after timer (e.g. 1 second)
3. 2nd repeat after timer (e.g., 1 second again), after which the timer is doubled
4. 3rd repeat after the increased (e.g., 2 seconds), after which the timer is doubled
again
5. etc.

If we let the process go long enough, a succession of doubling will probably match
the required value, probably after a dozen repeats if the delay is about 20 minutes. In
that case, one of the dozen repeats will most likely be successful, but of course
a lot of extra energy will have been expanded. But the connection establishment will fail
if the process is interrupted too soon, either because the maximum number of repeats has
been reached, or because the "idle timeouty" has been exceeded.

## Idle Timeout

The idle timeout is defined in {{Section 10.1 of QUIC-TRANSPORT}}. Each peer
proposes a "max_idle_timeout" value, and commits to close the connection
if no activity happens during that timeout. The "max_idle_timeout" value
is often set as a constant by either the stack or the application using it,
with typical values ranging from a few seconds to a few minutes.

The specification anticipated the usage of long delay links somewhat, and states
that "To avoid excessively small idle timeout periods, endpoints MUST increase the idle
timeout period to be at least three times the current Probe Timeout (PTO)." This
will prevent interference between the idle timeout once the PTO has been properly
assessed. However, this proper assessment of the PTO requires properly assessing the
RTT, which is requires a full RTT.

If the Initial Idle Timeout, Initial RTT and Initial PTO are set values too small, the
connection attempt will be interrupted by the Idle Timeout and will fail.

# Flow control and congestion control

QUIC congestion control and flow control processes both control how fast an endpoint
can send, either using local measurements of network capacity in the case of
congestion control, or abiding to transmission limits set by the peer in the case
of congestion control.

In both cases, if the transmission limits are smaller than
the "bandwidth delay product" (BDP),
transmission will throttled. Because of long delays, the BDP required for spatial
communications can be quite large, as we discuss in the next sections.

# Flow Control

Flow control in QUIC allow an endpoint to limit how many many bytes the peer
can send on the opened streams, how many bytes the peer can send on specific streams,
and how many streams the peer can open. Different stacks follow different strategies,
balancing two risks:

* if the flow control is too loose, the peer could send data faster than
the local application can process and create a form of local congestion.
* if the flow control is too restrictive, the peer will be blocked and
will have to wait for the next flow control update before sending data or
opening streams.

The peer will not be blocked if three conditions are met:

* the "max streams" credits allow for a number of stream at least as large as
expected to be open in an RTT.     
* the global flow control (MAX DATA) allows transmission of a full
bandwidth-delay product (BDP) worth of data,
* for each stream, the stream flow control allows transmission of either a full
BDP worth of data, or the full size of the data stream.

Setting these three limits correctly requires anticipating the RTT and bandwidth
of the connection. Implementations relying on adaptive algorithms are at risk here,
especially if they use a low default value until RTT and bandwidth have been
measured.

# Congestion control and Slow Start

QUIC implementation use congestion control to ensure that they are not sending
data faster than the network path can forward, which would cause network queues
and packet drops. Congestion control will typically set a congestion window size
(CWIN) to limit the amount of bytes in transit. Transmission will be slowed down
CWIN is lower than the BDP.

The main congestion control issues for long delay space connections are observed
at the beginning of the connection. The congestion control algorithms typically
start with conservative values of CWIN, which they ramp up progressively, typically
not faster than one doubling per RTT. On a long delay space connection, this
ramping up will take a very long time, leading to very inefficient use of the
connection.

Implementations have tried to palliate this issue in many ways:

* measure the transmission speed of short trains of packets to rapidly estimate
  the bandwidth of the connection,
* remember the delays and bandwidths of past connections and set the initial
  parameters of new connections accordingly,
* obtain estimates of delays and bandwidth through network management interfaces
  and use them to set appropriate parameters.  

If initial parameters are set correctly, connections can still be unnecessarily
throttled if they fail to adapt to changing conditions. For example, after the
initial "slow start", the classic RENO algorithm moves to a "congestion avoidance"
phase in which the CWIN increases by at most one packet per RTT -- which would be
completely inadequate with RTTs of several minutes.

# Packet losses

Packet losses will likely occur in space communications like in other media. The
loss recovery mechanisms spedified in {{QUIC-RECOVERY}} correct losses after
at best one RTT, with repeated losses requiring further RTT sized delays. In space
communications, the long delays for packet loss recovery will affect the
responsiveness of the application. In some cases, the loss recovery delays may
extend the duration of the transfer past the time of availability of the
transmission links.

## Packet Losses During Handshake

Packet losses during the initial handshake may prevent the endpoints from obtaining
a proper estimate of the RTT. The implementations in these situations can either
repeat the packets "too soon" if they underestimate the RTT, or "too late" if they
set a large value. Expereince show that of those too pitfalls, "too soon" is much
better, because it simply leads to repeating too many copies of the handshake
packets. This may look wasteful, but it does ensure that the handshake completes
rapidly even if some packets are actually lost.

## Reordering Buffers

If packets carrying stream data are lost, the other packets received on that stream
are supposed to be buffered until the packet loss is corrected. Receivers have to be
ready to buffer a full BDP worth of data. This means not only having large amount of
receive buffers, but also make sure that reordering of data is done efficiently,
risking otherwise some significant performance bottlenecks.

Reordering will also interfere with per-stream and per-connection flow control. In
{{flow-control}}, we wrote that the amount of credits should be at least one full BDP.
But if we assume that a full BDP worth of buffers can be consumed by reordering
after losses, then the required credits are actually two BDPs, not just one.

# Implementation Guidance

TODO: pay attention to delays.

## Setting the initial RTT

## Setting the initial BDP

(Cite existing work for geo satellites)

## Using Forward Error Correction 

(Consider ananlogy with requirements for Media over QUIC)



# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
