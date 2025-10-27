# TCP Pacing Implementation

## Overview

TCP pacing spreads packet transmissions evenly over time based on a congestion controller's specified rate, preventing bursts that can overflow bottleneck buffers. This implementation enables rate-based congestion control algorithms like BBR to function correctly in smoltcp.

## Design

**Core Mechanism**

A per-socket timer (`pacing_next_send_at`) determines when the next packet can be transmitted. After sending a packet, the timer is set based on:

```
delay = packet_size / pacing_rate
next_send_time = current_time + delay
```

**Controller Interface**

Congestion controllers expose their pacing rate via:

```rust
trait Controller {
    fn pacing_rate(&self) -> u64;  // Returns bytes/second, 0 = disabled
}
```

**Transmission Control**

The socket's dispatch logic checks the pacing timer before sending:
1. If timer is unset (None), send immediately
2. If current time < next_send_time, defer transmission
3. Otherwise, send packet and update timer

### Implementation

**Socket State**
```rust
struct Socket {
    // Existing fields...
    pacing_next_send_at: Option<Instant>,
}
```

**Timer Update (after transmission)**
```rust
let pacing_rate = self.congestion_controller.inner_mut().pacing_rate();
if pacing_rate > 0 {
    let delay_micros = (packet_size * 1_000_000) / pacing_rate;
    let delay = Duration::from_micros(delay_micros);
    self.pacing_next_send_at = Some(now + delay);
} else {
    self.pacing_next_send_at = None;
}
```

**Transmission Check (before dispatch)**
```rust
if let Some(next_send_at) = self.pacing_next_send_at {
    if now < next_send_at && self.seq_to_transmit() {
        return Ok(());  // Defer transmission
    }
}
```

**Poll Integration**
```rust
if self.seq_to_transmit() {
    match self.pacing_next_send_at {
        Some(next_send_at) if now < next_send_at => PollAt::Time(next_send_at),
        _ => PollAt::Now,
    }
}
```

### Controller Implementations

**BBR (rate-based)**
```rust
fn pacing_rate(&self) -> u64 {
    self.pacing_rate  // Calculated based on bandwidth estimate
}
```

**Reno/Cubic/NoControl (window-based)**
```rust
fn pacing_rate(&self) -> u64 {
    0  // Default implementation, pacing disabled
}
```

## Behavior by Algorithm

**BBR**
- Pacing active with calculated rate (typically 1-1000 Mbps)
- Packets sent at precise intervals matching bandwidth estimate
- Essential for correct BBR operation

**Reno/Cubic**
- Pacing disabled (rate = 0)
- Traditional window-based transmission with ACK clocking
- Zero performance impact from pacing infrastructure

**NoControl**
- Pacing disabled (rate = 0)
- Unlimited transmission rate
- Zero performance impact

## Example

10 Mbps pacing rate with 1500 byte packets:

```
delay = (1500 bytes × 1,000,000 μs) / 10,000,000 bytes/s = 150 μs
```

Packets transmitted at 150 microsecond intervals, achieving 10 Mbps average rate.

## Comparison with Linux

**Linux Approach**
- FQ (Fair Queue) scheduler at qdisc layer
- System-wide pacing across all flows
- Requires kernel modifications

**smoltcp Approach**
- Per-socket pacing in TCP layer
- No external scheduler required
- Portable across environments (bare-metal, userspace)
