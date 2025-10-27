# BBR Congestion Control Implementation

## Overview

This document describes the BBR (Bottleneck Bandwidth and RTT) congestion control implementation for smoltcp, based on Quinn's implementation and the Linux kernel implementation (tcp_bbr.c).

BBR is a rate-based congestion control algorithm that estimates the bottleneck bandwidth and round-trip time to determine the optimal sending rate, rather than reacting to packet loss like traditional algorithms.

## Design Details

**State Machine**
- Startup: Exponential growth to find bandwidth
- Drain: Drain queue created during startup
- ProbeBW: Steady state with periodic bandwidth probing
- ProbeRTT: Periodic RTT measurement

**Bandwidth Estimation**
- Windowed maximum filter tracking delivery rate over 10 RTT cycles
- Accounts for app-limited periods to avoid underestimation
- Uses delivered bytes and elapsed time for accurate measurement

**RTT Tracking**
- Minimum RTT window of 10 seconds
- Filters out queuing delay to find true propagation delay
- Triggers ProbeRTT when minimum expires

**Pacing Rate Calculation**
```
pacing_rate = bandwidth × pacing_gain × (1 - margin)
```
- Pacing gain varies by state (2.77 in Startup, 0.75-1.25 in ProbeBW)
- 1% margin prevents buffer buildup

**Congestion Window**
```
cwnd = max(bandwidth × min_rtt × cwnd_gain, 4×MSS)
```
- Used as a safety net, not primary control mechanism
- Adapts based on bandwidth-delay product

**Packet Conservation**

During loss recovery, BBR implements packet conservation to reduce loss amplification:
- First round: Send P packets per P packets acknowledged
- Subsequent rounds: Allow up to 2P growth (slow-start)
- After recovery: Restore previous congestion window

**ACK Aggregation**

Compensates for bursty ACK patterns caused by:
- ACK compression at bottleneck
- Delayed ACKs
- TSO/GRO coalescing

Uses a windowed maximum filter to track excess acknowledged bytes over 5-10 RTT cycles. Adds compensating cwnd increment when aggregation is detected.

**Idle Restart**

After idle periods, BBR:
- Skips ProbeRTT entry even if min_rtt expired
- Resets ACK aggregation epoch tracking
- Prevents unnecessary throughput drops

## Implementation Details

### Key Data Structures

```rust
struct Bbr {
    // State
    mode: Mode,
    round_count: u64,

    // Bandwidth estimation
    max_bandwidth: BandwidthEstimation,

    // RTT tracking
    min_rtt: Duration,
    min_rtt_timestamp: Instant,

    // Pacing
    pacing_rate: u64,
    pacing_gain: u32,

    // Congestion window
    cwnd: usize,
    cwnd_gain: u32,

    // Packet conservation
    packet_conservation: bool,
    prev_in_recovery: bool,
    round_start: bool,

    // ACK aggregation
    ack_aggregation: AckAggregationState,

    // Idle restart
    idle_restart: bool,
}
```

### Control Flow

**On ACK**
1. Update bandwidth estimate with delivered bytes
2. Update minimum RTT from RTT estimator
3. Detect round start (for state transitions)
4. Update recovery state and packet conservation
5. Update ACK aggregation tracking
6. Execute state machine (startup, drain, probe_bw, probe_rtt)
7. Recalculate pacing rate and congestion window

**On Send**
- Track app-limited state
- Detect idle restart conditions
- Increment sent packet counter

**On Loss**
- Enter recovery state
- Apply packet conservation during first round
- Deduct lost bytes from cwnd

## Differences from Linux BBR

**Simplified Recovery States**
- Linux tracks full TCP CA state (Open, Disorder, CWR, Recovery, Loss)
- Our implementation uses boolean flag (in_recovery)
- Sufficient for detecting recovery entry/exit transitions

**Byte-based vs Packet-based**
- Linux operates on packet counts
- Our implementation uses bytes (consistent with smoltcp design)
- Maintains equivalent behavior

**No TSO**
- Linux BBR interacts with TCP Segmentation Offload
- Not applicable in smoltcp

## Testing

All tests pass with BBR enabled:
```bash
cargo test --features socket-tcp-bbr
```

## References

- BBR Paper: "BBR: Congestion-Based Congestion Control", Cardwell et al., 2016
- Linux Implementation: net/ipv4/tcp_bbr.c
