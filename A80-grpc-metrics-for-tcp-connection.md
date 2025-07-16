## A80: gRPC Metrics for TCP connection

*   Author(s): Yash Tibrewal (@yashykt), Nana Pang (@nanahpang), Yousuk Seung
    (@yousukseung)
*   Approver: Craig Tiller (@ctiller), Mark Roth (@markdroth)
*   Status: In Review
*   Implemented in:
*   Last updated: 2025-06-27
*   Discussion at: https://groups.google.com/g/grpc-io/c/AyT0LVgoqFs

## Abstract

This document proposes adding new TCP connection metrics to gRPC for improved
network analysis and debugging.

## Background

On Linux systems, TCP socket state can be retrieved using the `getsockopt()`
system call with `level` set to `IPPROTO_TCP` and `optname` set to `TCP_INFO`.
The state is returned in a `struct tcp_info` which gives details about the TCP
connection. At present, the machinery to collect such information is available
only on Linux 2.6 or later kernels.

[A79] provides a framework for adding non-per-call metrics in gRPC. This
document uses that framework to propose per-connection TCP metrics.

### Related Proposals:

*   [A66]: OpenTelemetry Metrics
*   [A78]: gRPC OTel Metrics for WRR, Pick First, and XdsClient
*   [A79]: gRPC Non-Per-Call Metrics Framework
*   [A94]: OTel metrics for Subchannels

[A66]: A66-otel-stats.md
[A78]: A78-grpc-metrics-wrr-pf-xds.md
[A79]: A79-non-per-call-metrics-architecture.md
[A94]: https://github.com/grpc/proposal/pull/485

## Proposal

This document proposes adding per-connection TCP metrics in gRPC to improve the
network debugging capabilities for gRPC users.

Name                                    | Type                       | Unit     | Labels (disposition)                                                                    | Description
--------------------------------------- | -------------------------- | -------- | --------------------------------------------------------------------------------------- | -----------
grpc.tcp.min_rtt                        | Histogram (floating-point) | s        | grpc.target (required), grpc.lb.locality (optional), grpc.lb.backend_service (optional) | TCP's current estimate of minimum round trip time (RTT). Corresponds to `tcpi_min_rtt` from `struct tcp_info`. Recommended histogram bucket boundaries are 0, 0.000001, 0.0000025, 0.000005, 0.0000075, 0.00001, 0.000025, 0.00005, 0.000075, 0.0001, 0.00025, 0.0005, 0.00075, 0.001, 0.0025, 0.005, 0.0075, 0.01, 0.025, 0.05, 0.075, 0.1, 0.25, 0.5, 0.75, 1, 2.5, 5, 7.5, 10, 25, 50, 75, 100.
grpc.tcp.delivery_rate                  | Histogram (integer)        | By/s     | grpc.target (required), grpc.lb.locality (optional), grpc.lb.backend_service (optional) | Goodput measured of the TCP connection. Corresponds to `tcpi_delivery_rate` from `struct tcp_info`. Recommended histogram bucket boundaries are 0, 1024, 2048, 4096, 8192, 16384, 32768, 65536, 131072, 262144, 524288, 1048576, 2097152, 4194304, 8388608, 16777216, 33554432, 67108864, 134217728, 268435456, 536870912, 1073741824, 2147483648, 4294967296, 8589934592, 17179869184, 34359738368, 68719476736, 137438953472, 274877906944, 549755813888, 1099511627776, 2199023255552, 4398046511104.
grpc.tcp.packets_sent                   | Counter (integer)          | {packet} | grpc.target (required), grpc.lb.locality (optional), grpc.lb.backend_service (optional) | TCP packets sent. Corresponds to `tcpi_data_segs_out` from `struct tcp_info`.
grpc.tcp.packets_retransmitted          | Counter (integer)          | {packet} | grpc.target (required), grpc.lb.locality (optional), grpc.lb.backend_service (optional) | TCP packets retransmitted. Corresponds to `tcpi_total_retrans` from `struct tcp_info`.
grpc.tcp.packets_spurious_retransmitted | Counter (integer)          | {packet} | grpc.target (required), grpc.lb.locality (optional), grpc.lb.backend_service (optional) | TCP packets spuriously retransmitted packets. Corresponds to `tcpi_dsack_dups` from `struct tcp_info`.

Label Name              | Disposition | Description
----------------------- | ----------- | -----------
grpc.target             | Required    | For client-side sockets, indicates the target of the gRPC channel (defined in [A66].) Use "#server" for server-side sockets.
grpc.lb.backend_service | Optional    | For client-side sockets, indicates the backend service to which an RPC is routed (defined in [A89].) Use "#server" for server-side sockets.
grpc.lb.locality        | Optional    | For client-side sockets, indicates the locality to which the traffic is being sent. This will be set to the resolver attribute passed down from the weighted_target policy, or the empty string if the resolver attribute is unset (defined in [A78].) Use "#server" for server-side sockets.

[A94] documents proposes the labels `grpc.lb.backend_service` and
`grpc.lb.locality` to be propagated to individual subchannels. These attributes
need to be further plumbed down to the layer where the TCP metrics are
available.

Suggested algorithm to collect metrics -

*   Set TCP_CONNECTION_METRICS_RECORD_INTERVAL to a default of 5 minutes.
    Implementations can choose to make it configurable.
*   For each new connected TCP socket, set an initial alarm of 10% to 110%
    (randomly selected) of TCP_CONNECTION_METRICS_RECORD_INTERVAL.
*   When the alarm fires -
    *   Use `getsockopt()` or equivalent method to retrieve and record
        connection metrics.
    *   Re-arm the alarm with TCP_CONNECTION_METRICS_RECORD_INTERVAL and repeat.
*   Before the socket is closed, cancel the alarm set above, and retrieve and
    record connection metrics, providing observability for short-lived
    connections as well.

These metrics will only available on systems running Linux 2.6 or later kernels
because the necessary machinery is only available on those platforms.

### Metric Stability

All metrics added in this proposal will start as experimental. The long term
goal will be to de-experimentalize them and have them be on by default, but the
exact criteria for that change are TBD.

## Rationale

This section is intended to help further understand how these metrics can be
used.

### Min RTT

`grpc.tcp.min_rtt` (`tcpi_min_rtt` from `struct tcp_info`) reports TCP's current
estimate of minimum round trip time (RTT), typically used as an indication of
the network health between two endpoints. It reflects the minimum transmission
time experienced by the application, which includes:

*   The physical length of the path between sender and receiver.
*   Queueing time caused by throttling and load.
*   Other transport-level events, such as delayed acknowledgement.

A step-change in min_rtt values means that traffic is being throttled, is
experiencing congestion, or has been re-routed through a different path.

### Delivery Rate

`grpc.tcp.delivery_rate` (`tcpi_delivery_rate` from `struct tcp_info`) records
the most recent "non-app-limited" throughput. The term "non-app-limited" means
that the link is saturated by the application. In other words, the delivery_rate
is calculated when the application has enqueued enough data to cause a backlog
on the socket. In the opposite case when the connection is app-limited, TCP is
unable to reliably compute throughput, and so collecting this metric would not
be meaningful.

### Transmission Metrics

`grpc.tcp.packets_sent` (`tcpi_data_segs_out` from `struct tcp_info`) reports
the TCP packets sent. This includes retransmissions and spurious
retransmissions.

`grpc.tcp.packets_retransmitted` (`tcpi_total_retrans` from `struct tcp_info`)
reports the TCP packets sent except those sent for the sent time. A packet may
be retransmitted multiple times and counted multiple times as retransmitted.
These counts include spurious retransmissions.

`grpc.tcp.packets_spurious_retransmitted` (`tcpi_dsack_dups` from `struct
tcp_info`) reports TCP packets acknowledged for the second time or more. These
are retransmissions that TCP later discovered unnecessary. A packet may be
counted as spuriously retransmitted multiple times. TCP can under-count spurious
retransmissions if the acknowledgements to reflect the spurious retransmissions
are lost. This may occur when the return path is highly congested.

#### Packet Loss Rate

TCP packet loss is the ratio of total packets that the TCP sender did not
receive an acknowledgement for out of all packets sent. It can be calculated as
(`grpc.tcp.packets_retransmitted` - `grpc.tcp.packets_spurious_retransmitted`) /
`grpc.tcp.packets_sent`.

TCP packet loss also includes loss NOT in the network. It is what the TCP sender
sees as “lost” from the transport layer’s perspective and includes loss from
various stages such as:

*   Acknowledgements lost on the return path
*   Drops on the sender below the transport layer (e.g. drops on the NIC)
*   Drops on the receiver NIC
*   Drops on the receiver due to out-of-memory conditions

TCP can overestimate the packet loss rate if acknowledgements for any
duplicate/spurious retransmissions are lost. Also note that the loss rate in TCP
generally covers only the data (and SYN) packets. In TCP, transmission of
acknowledgements is not reliable (i.e. not retransmitted when lost) and the true
acknowledgement loss rate is not available.

#### Spurious Retransmission Rate

Spurious retransmission rate can be calculated as
`grpc.tcp.packets_spurious_retransmitted` / `grpc.tcp.packets_retransmitted`.

TCP can underestimate the spurious retransmission rate. TCP can under-count
spurious retransmissions if the acknowledgements for the duplicate/spurious
retransmissions are lost.

### Temporary environment variable protection

This proposal does not include any features enabled via external I/O, so it does
not need environment variable protection.

## Implementation

Will be implemented in C-core, Java and Go.
