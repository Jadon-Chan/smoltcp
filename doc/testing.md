# Performance Testing Guide

This guide describes the performance testing framework for smoltcp's TCP implementation, including congestion control algorithms (BBR, Cubic, Reno).

## Overview

The testing framework consists of:
- **Network Emulation Scripts**: Set up realistic network conditions (delay, bandwidth, packet loss)
- **Performance Server** (`perf_server`): Sends 3GB of data with embedded timestamps
- **Performance Client** (`perf_client`): Receives data and measures throughput and latency
- **Helper Scripts**: Automate setup, testing, and cleanup

## Architecture

### Network Topology

```
┌─────────────┐         ┌──────────┐         ┌─────────────┐
│             │  tap0   │          │  tap1   │             │
│  perf_server├─────────┤  bridge  ├─────────┤ perf_client │
│ 192.168.69.1│         │   (br0)  │         │192.168.69.2 │
└─────────────┘         └──────────┘         └─────────────┘
```

The bridge connects two TAP interfaces, each with traffic control applied for realistic network emulation.

### Traffic Control Setup

Each TAP interface has a two-layer qdisc hierarchy:
1. **netem**: Adds delay and manages queue limits
2. **tbf** (Token Bucket Filter): Controls bandwidth and implements tail-drop buffering

## Prerequisites

### Build Requirements

Build the performance examples in release mode for accurate measurements:

```bash
cargo build --release --example perf_server --features="std medium-ethernet medium-ip phy-tuntap_interface proto-ipv4 socket-tcp socket-tcp-cubic socket-tcp-bbr"
cargo build --release --example perf_client --features="std medium-ethernet medium-ip phy-tuntap_interface proto-ipv4 socket-tcp socket-tcp-cubic socket-tcp-bbr"
```

### System Permissions

The setup scripts require sudo privileges to:
- Create TAP interfaces
- Configure network bridges
- Apply traffic control rules

## Quick Start

### 1. Set Up Network Emulation

```bash
./scripts/emulate_setup.sh
```

This creates:
- `tap0`: Server-side interface (192.168.69.1/24)
- `tap1`: Client-side interface (192.168.69.2/24)
- `br0`: Bridge connecting the interfaces
- Traffic control with configurable delay, bandwidth, and buffer size

**Default Parameters:**
- Delay: 10ms per direction (20ms RTT)
- Bandwidth: 400 Mbit/s
- Buffer: 4000 packets (~6MB with MTU=1500)
- MTU: 1500 bytes

### 2. Run the Server

In one terminal:

```bash
./scripts/perf_server_run.sh [bbr|cubic|reno|none]
```

Example:
```bash
./scripts/perf_server_run.sh bbr
```

The server will:
- Listen on port 8000
- Wait for client connection
- Send 3GB of data with timestamps
- Report progress every 5 seconds
- Display final throughput statistics

### 3. Run the Client

In another terminal:

```bash
./scripts/perf_client_run.sh [bbr|cubic|reno|none]
```

Example:
```bash
./scripts/perf_client_run.sh bbr
```

The client will:
- Connect to server at 192.168.69.1:8000
- Receive data and measure performance
- Report progress every 5 seconds
- Display throughput and latency statistics

### 4. Clean Up

After testing:

```bash
./scripts/cleanup.sh
```

This removes the TAP interfaces and bridge.

## Configuration

### Network Emulation Parameters

Edit `scripts/emulate_setup.sh` to customize network conditions:

```bash
DELAY="10ms"           # One-way delay (RTT = 2x this)
BANDWIDTH="400mbit"     # Link bandwidth
BUFFER_PACKETS="4000"   # Router queue size
MTU=1500               # Maximum transmission unit
NETEM_LIMIT="4000"    # Netem queue limit
```

**Important Considerations:**

1. **Bandwidth-Delay Product (BDP)**
   - BDP = Bandwidth × RTT
   - Buffer should be ≥ BDP to avoid losses at full utilization
   - The script calculates and displays BDP automatically

2. **Buffer Sizing**
   - Too small: Packet loss even with proper congestion control
   - Too large: Bufferbloat, increased latency
   - Rule of thumb: serveral times of BDP for testing

3. **Netem Limit**
   - Must be large enough to handle BDP
   - Should match or exceed BUFFER_PACKETS
   - Too small causes early packet drops

#### Why This Buffer Configuration for BBR Testing

The default configuration (400 Mbit/s, 20ms RTT, 4000 packet buffer) is specifically designed to test BBR effectively:

**Calculating BDP:**
- Bandwidth × RTT = 400 Mbit/s × 0.02s = 8 Mbits = 1 MB
- Buffer size = 4000 packets = ~6 MB = **6× BDP**

**Rationale:**

1. **Shallow Buffer Scenario**: To test BBR's advantage in shallow-buffer networks (where loss-based algorithms struggle), reduce `BUFFER_PACKETS` to ~1 BDP (e.g., 700 packets). BBR should still perform well, while Cubic/Reno will experience continuous losses.

2. **Testing vs. Loss-Based Algorithms**:
   - **Cubic/Reno**: Fill buffers until loss occurs, then back off. Large buffers allow them to ramp up fully and demonstrate their sawtooth pattern.
   - **BBR**: Should maintain high throughput with lower queue occupancy and latency, even with large buffers available. This is the key differentiator.

### Understanding the Traffic Control Commands

The `emulate_setup.sh` script uses Linux Traffic Control (`tc`) to create realistic network conditions. Here's a detailed breakdown of the commands:

#### Qdisc Hierarchy Structure

Traffic control uses a two-layer qdisc (queuing discipline) hierarchy on each TAP interface:

```bash
# Layer 1: netem (Network Emulator)
sudo tc qdisc add dev tap0 root handle 1: netem delay $DELAY limit $NETEM_LIMIT

# Layer 2: tbf (Token Bucket Filter)
sudo tc qdisc add dev tap0 parent 1:1 handle 10: tbf rate $BANDWIDTH burst $BURST limit $BUFFER_BYTES
```

**Hierarchy Visualization:**

```
tap0
 └── root (1:)          ← netem qdisc
      └── 1:1 (10:)     ← tbf qdisc
```

#### netem (Network Emulator) Command

```bash
sudo tc qdisc add dev tap0 root handle 1: netem delay 10ms limit 4000
```

**Parameters:**
- `dev tap0`: Apply to tap0 interface
- `root`: This is the root qdisc (first in hierarchy)
- `handle 1:`: Identifier for this qdisc (used for attaching child qdiscs)
- `netem`: Network emulation qdisc
- `delay 10ms`: Add 10ms constant delay to all packets
- `limit 4000`: Maximum queue length in packets (must be ≥ BDP to avoid early drops)

**What it does:**
- Adds propagation delay to simulate distance (e.g., 10ms = ~3000 km of fiber)
- Holds packets in a queue before forwarding them
- If queue exceeds `limit`, packets are dropped (head-of-line drop)

**Why large limit?**
- For 400 Mbit/s × 20ms RTT, BDP ≈ 667 packets
- Setting limit=4000 ensures we don't drop packets before they reach the tbf buffer
- netem's job is only to add delay, not to limit throughput

#### tbf (Token Bucket Filter) Command

```bash
sudo tc qdisc add dev tap0 parent 1:1 handle 10: tbf rate 400mbit burst $BURST limit $BUFFER_BYTES
```

**Parameters:**
- `parent 1:1`: Child of netem qdisc (attached to class 1:1)
- `handle 10:`: Identifier for this qdisc
- `tbf`: Token Bucket Filter qdisc
- `rate 400mbit`: Maximum bandwidth (token generation rate)
- `burst`: Maximum burst size in bytes (rate/10 typically)
- `limit`: Buffer size in bytes (this is the router queue we're testing!)

**Token Bucket Algorithm:**

1. **Tokens accumulate** at `rate` (e.g., 400 Mbit/s = 50 MB/s = 50,000 tokens/ms)
2. **Sending a packet** consumes tokens equal to packet size
3. **Burst** allows temporary bursts up to `burst` bytes
4. **Buffer (`limit`)**: Packets wait here when tokens unavailable
5. **Tail drop**: When buffer fills, new packets are dropped

**Calculating limit (buffer size):**
```bash
BUFFER_BYTES=$((BUFFER_PACKETS * MTU))  # 4000 × 1500 = 6,000,000 bytes
```
- This is the **router queue** that congestion control algorithms interact with
- Large buffer: Cubic/Reno fill it, causing bufferbloat
- Small buffer: Frequent drops, tests congestion response
- BBR should avoid filling it regardless of size

#### Verification Commands

Check the qdisc configuration:

```bash
# Show all qdiscs on tap0
tc qdisc show dev tap0

# Expected output:
# qdisc netem 1: root refcnt 2 limit 4000 delay 10ms
# qdisc tbf 10: parent 1:1 rate 400Mbit burst 15000b lat 120ms

# Show detailed statistics
tc -s qdisc show dev tap0

# Expected output includes:
# - Sent/dropped packet counts
# - Backlog (current queue depth)
# - Overlimits (times rate/buffer limit was hit)
```

### Application Parameters

Both server and client use:
- **Buffer Size**: 6MB socket buffers (defined in code as `BUFFER_SIZE`)
- **Data Size**: 3GB total transfer (defined in `perf_server.rs` as `DATA_SIZE`)

## Manual Testing

For more control, run examples directly:

### Server

```bash
SMOLTCP_IFACE_MAX_ADDR_COUNT=3 ./target/release/examples/perf_server \
  --tap tap0 \
  --congestion bbr \
  --port 8000
```

### Client

```bash
SMOLTCP_IFACE_MAX_ADDR_COUNT=3 ./target/release/examples/perf_client \
  --tap tap1 \
  --congestion bbr \
  --server 192.168.69.1 \
  --port 8000
```

### Command-Line Options

**Common Options:**
- `--tap DEVICE`: TAP interface to use
- `-c, --congestion ALGO`: Congestion control (bbr/cubic/reno/none)

**Server Options:**
- `-p, --port PORT`: Port to listen on (default: 8000)

**Client Options:**
- `-s, --server ADDRESS`: Server IP address (default: 192.168.69.1)
- `-p, --port PORT`: Server port (default: 8000)

## Metrics and Output

### Throughput Metrics

Both client and server report:
- **Bytes Transferred**: Total MB sent/received
- **Time**: Elapsed time in seconds
- **Throughput**: Calculated in Gbps (Gigabits per second)

Formula: `Throughput = (bytes × 8) / time / 1e9`

### Latency Metrics

The client measures one-way delay using timestamps embedded in the data:
- **Samples**: Number of latency measurements
- **Min**: Minimum observed latency
- **Mean**: Average latency
- **p50 (Median)**: 50th percentile
- **p95**: 95th percentile
- **p99**: 99th percentile
- **Max**: Maximum observed latency

**Note**: Latency sampling occurs every 100,000 8-byte chunks to minimize measurement overhead.

## Interpreting Results

### Good Performance Indicators

1. **Throughput near bandwidth limit**: Efficient link utilization
2. **Low latency variance**: Stable queue management
3. **Low p99 latency**: Minimal bufferbloat
