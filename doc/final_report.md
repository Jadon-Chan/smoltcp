# smoltcp BBR 拥塞控制实现报告

## 1. 引言

### 1.1 项目动机

本项目旨在为 smoltcp TCP/IP 协议栈实现 BBR（Bottleneck Bandwidth and RTT）拥塞控制算法。BBR 是 Google 开发的现代拥塞控制算法，相比传统的基于丢包的算法（如 Reno、Cubic），BBR 通过主动估计瓶颈带宽和往返时延来确定最优发送速率，能够在高带宽、高延迟网络中实现更高的吞吐量和更低的延迟。

smoltcp 作为一个面向裸机和嵌入式系统的独立 TCP/IP 协议栈，原本仅支持 Reno 和 Cubic 等传统算法。引入 BBR 可以使 smoltcp 在更广泛的网络环境中获得更好的性能表现。

### 1.2 实现成果

本项目完成了以下工作：

1. **BBR 核心算法实现**：基于 Quinn QUIC 实现和 Linux 内核 `tcp_bbr.c` 参考，实现了完整的 BBR 状态机，包括 Startup、Drain、ProbeBW 和 ProbeRTT 四个状态。

2. **TCP Pacing 实现**：实现了 BBR 所需的分组发送机制，使数据包按照计算出的速率均匀发送，避免突发造成的缓冲区溢出。

3. **完整的测试框架**：
   - 网络仿真脚本（使用 tc netem 和 tbf）
   - 性能测试服务器和客户端
   - 自动化测试脚本

### 1.3 当前状态与问题

**性能表现**：从测试结果来看，BBR 的吞吐量性能与 Cubic 相当，在某些场景下略有优势。然而，预期中 BBR 在延迟和队列管理方面的显著优势并未完全体现。

**问题**：
- 实现中可能存在一些尚未发现的问题，导致 BBR 的优势未能充分发挥

**关键收获**：本项目最重要的成果是建立了一套完整的网络性能测试方法论，包括：
- 使用 Linux tc 工具模拟真实网络环境（延迟、带宽、缓冲区）
- 带宽延迟积（BDP）的计算和缓冲区配置策略
- 如何设计测试场景来对比不同拥塞控制算法的特性

这套测试框架对于理解和调试拥塞控制算法至关重要，也为后续的优化工作奠定了基础。

---

## 2. BBR 拥塞控制算法

### 2.1 算法原理

BBR 是一种**基于速率**的拥塞控制算法，核心思想是：
- 估计瓶颈链路的带宽（Bottleneck Bandwidth）
- 估计网络的往返时延（Round-Trip Time）
- 根据这两个参数计算最优发送速率，而不是等待丢包信号

传统算法（Reno、Cubic）通过"填满缓冲区直到丢包，然后退避"的方式工作，这导致：
- 高延迟（缓冲区膨胀 bufferbloat）
- 吞吐量不稳定（锯齿状模式）

BBR 通过主动探测避免了这些问题。

### 2.2 状态机

BBR 包含四个状态：

1. **Startup（启动）**
   - 目标：快速找到可用带宽
   - 行为：指数增长发送速率（pacing_gain = 2.77）
   - 退出条件：连续三轮带宽增长低于 25%

2. **Drain（排空）**
   - 目标：排空启动阶段在队列中积累的数据
   - 行为：降低发送速率（pacing_gain = 0.75）
   - 退出条件：在途数据量降至 BDP 以下

3. **ProbeBW（带宽探测）**
   - 目标：稳态运行，同时周期性探测更多带宽
   - 行为：循环使用不同的 pacing_gain（1.25, 0.75, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0）
   - 特点：8 轮循环，1 轮探测、1 轮排空、6 轮巡航

4. **ProbeRTT（RTT 探测）**
   - 目标：测量真实的传播延迟
   - 行为：将拥塞窗口降至最小（4 MSS）
   - 触发条件：min_rtt 超过 10 秒未更新

### 2.3 关键机制

**带宽估计**
- 使用滑动窗口最大值滤波器，跟踪 10 个 RTT 周期内的最大传输速率
- 计算方式：已确认字节数 / 已耗时间
- 过滤应用层限速期，避免低估

**RTT 跟踪**
- 维护 10 秒窗口内的最小 RTT
- 过滤掉队列延迟，只保留传播延迟
- 用于计算 BDP 和触发 ProbeRTT

**速率计算**
```
pacing_rate = bandwidth × pacing_gain × (1 - margin)
```
- `pacing_gain`：根据状态变化（Startup: 2.77, ProbeBW: 0.75-1.25）
- `margin`：1% 的安全余量，防止缓冲区堆积

### 2.5 实现细节

**与 Linux BBR 的差异**

1. **恢复状态简化**
   - Linux：完整的 CA 状态机（Open, Disorder, CWR, Recovery, Loss）
   - smoltcp：使用布尔标志（`in_recovery`）
   - 足以检测恢复的进入/退出

2. **字节 vs 数据包**
   - Linux：基于数据包计数
   - smoltcp：基于字节（与 smoltcp 设计一致）
   - 行为等效

3. **无 TSO**
   - Linux BBR 与 TCP Segmentation Offload 交互
   - smoltcp 中不适用

**控制流程**

收到 ACK 时：
1. 用已确认字节更新带宽估计
2. 从 RTT 估计器更新最小 RTT
3. 检测轮次开始（用于状态转换）
4. 执行状态机逻辑
5. 重新计算速率和拥塞窗口

发送数据时：
- 跟踪应用层限速状态
- 检测空闲重启条件
- 增加已发送数据包计数

发生丢包时：
- 进入恢复状态
- 从拥塞窗口中扣除丢失字节

---

## 3. TCP Pacing 实现

### 3.1 实现背景：一段意外的旅程

在开始这个项目时，我的目标很明确：为 smoltcp 实现 BBR 拥塞控制算法。我研究了 BBR 的核心机制——带宽估计、RTT 跟踪、状态机转换，并完成了代码实现。

然而，当我经过仔细调试和阅读 Linux 内核代码后，我发现了问题的根源：**BBR 依赖 TCP Pacing 才能正常工作**。

BBR 是一个**基于速率**的拥塞控制算法，它计算出一个精确的发送速率（例如 100 Mbps），期望数据包按照这个速率均匀发送。但是，如果没有 Pacing 机制：
- 数据包会在拥塞窗口允许的情况下突发发送
- 突发流量会瞬间填满瓶颈链路的缓冲区
- 造成丢包和延迟增加
- BBR 的速率控制形同虚设

而 smoltcp 原本只支持传统的**基于窗口**的算法（Reno、Cubic），这些算法依靠 ACK 时钟自然地调节发送速率，不需要额外的 Pacing 机制。

因此，我意识到：**要让 BBR 工作，必须先实现 TCP Pacing**。这是一个意外但必要的步骤。

### 3.2 Pacing 原理

TCP Pacing 的核心思想是：根据拥塞控制器指定的速率，将数据包均匀地分散在时间上发送，而不是突发发送。

**基本机制**

每个 socket 维护一个定时器（`pacing_next_send_at`），记录下一个数据包允许发送的时间。发送数据包后，根据以下公式更新定时器：

```
delay = packet_size / pacing_rate
next_send_time = current_time + delay
```

**示例**

假设 pacing_rate = 10 Mbps，数据包大小 = 1500 字节：

```
delay = (1500 bytes × 8 bits/byte) / 10,000,000 bits/s = 1.2 ms
```

每个数据包间隔 1.2 毫秒发送，平均速率为 10 Mbps。

### 3.3 实现细节

**控制器接口**

拥塞控制器通过以下接口暴露速率：

```rust
trait Controller {
    fn pacing_rate(&self) -> u64;  // 返回 bytes/second，0 表示禁用
}
```

**Socket 状态**

```rust
struct Socket {
    // ... 其他字段
    pacing_next_send_at: Option<Instant>,
}
```

**发送前检查**

```rust
if let Some(next_send_at) = self.pacing_next_send_at {
    if now < next_send_at && self.seq_to_transmit() {
        return Ok(());  // 延迟发送
    }
}
```

**发送后更新**

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

### 3.4 不同算法的行为

**BBR**
- Pacing 启用，速率由带宽估计计算（通常 1-1000 Mbps）
- 数据包按精确间隔发送
- 对 BBR 正常运行至关重要

**Reno / Cubic**
- Pacing 禁用（`pacing_rate()` 返回 0）
- 传统的基于窗口的传输，ACK 时钟调节
- Pacing 基础设施零性能影响

**NoControl**
- Pacing 禁用
- 无限传输速率
- 零性能影响

### 3.5 与 Linux 的对比

**Linux 方案**
- 使用 FQ (Fair Queue) 调度器在 qdisc 层实现
- 系统级 Pacing，跨所有流
- 需要内核修改

**smoltcp 方案**
- 在 TCP 层实现，每个 socket 独立 Pacing
- 不需要外部调度器
- 可移植到各种环境（裸机、用户空间）

---

## 4. 测试框架

### 4.1 测试架构

测试框架由以下组件构成：

**网络拓扑**

```
┌─────────────┐         ┌──────────┐         ┌─────────────┐
│             │  tap0   │          │  tap1   │             │
│  perf_server├─────────┤  bridge  ├─────────┤ perf_client │
│ 192.168.69.1│         │   (br0)  │         │192.168.69.2 │
└─────────────┘         └──────────┘         └─────────────┘
```

通过桥接两个 TAP 接口，并在每个接口上应用流量控制，模拟真实网络环境。

**流量控制**

每个 TAP 接口使用两层 qdisc 层次结构：
1. **netem**：添加延迟和管理队列限制
2. **tbf (Token Bucket Filter)**：控制带宽和实现尾丢弃缓冲

### 4.2 快速开始

**1. 构建测试程序**

```bash
cargo build --release --example perf_server --features="std medium-ethernet medium-ip phy-tuntap_interface proto-ipv4 socket-tcp socket-tcp-cubic socket-tcp-bbr"
cargo build --release --example perf_client --features="std medium-ethernet medium-ip phy-tuntap_interface proto-ipv4 socket-tcp socket-tcp-cubic socket-tcp-bbr"
```

**2. 设置网络仿真**

```bash
./scripts/emulate_setup.sh
```

创建：
- `tap0`、`tap1`：服务器和客户端接口
- `br0`：连接两个接口的桥
- 流量控制规则

默认参数：
- 延迟：10ms × 2 = 20ms RTT
- 带宽：400 Mbit/s
- 缓冲区：4000 数据包（~6MB）

**3. 运行服务器**

```bash
./scripts/perf_server_run.sh bbr  # 或 cubic/reno/none
```

服务器会发送 3GB 数据并报告吞吐量。

**4. 运行客户端**

```bash
./scripts/perf_client_run.sh bbr  # 或 cubic/reno/none
```

客户端接收数据并测量吞吐量和延迟。

**5. 清理**

```bash
./scripts/cleanup.sh
```

### 4.3 网络参数配置

**默认配置分析**

编辑 `scripts/emulate_setup.sh` 可以自定义：

```bash
DELAY="10ms"           # 单向延迟（RTT = 2 × 这个值）
BANDWIDTH="400mbit"     # 链路带宽
BUFFER_PACKETS="4000"   # 路由器队列大小
MTU=1500               # 最大传输单元
```

**为什么这样配置？**

计算 BDP：
- Bandwidth × RTT = 400 Mbit/s × 0.02s = 8 Mbits = 1 MB
- Buffer size = 4000 packets = ~6 MB = **6× BDP**

**设计理由**

1. **测试对比 vs 基于丢包的算法**
   - **Cubic/Reno**：填满缓冲区直到丢包，然后退避（锯齿状）
   - **BBR**：应该在高吞吐量的同时保持低队列占用和低延迟

2. **浅缓冲区场景**
   - 减小 `BUFFER_PACKETS` 到 ~1 BDP（例如 700）
   - BBR 应该仍能良好工作
   - Cubic/Reno 会遇到持续丢包

### 4.4 流量控制命令详解

`emulate_setup.sh` 脚本使用 Linux Traffic Control (`tc`) 工具创建真实网络条件。下面详细解释这些命令的工作原理：

#### Qdisc 层次结构

流量控制在每个 TAP 接口上使用两层 qdisc（排队规则）层次结构：

```bash
# 第一层：netem（网络模拟器）
sudo tc qdisc add dev tap0 root handle 1: netem delay $DELAY limit $NETEM_LIMIT

# 第二层：tbf（令牌桶过滤器）
sudo tc qdisc add dev tap0 parent 1:1 handle 10: tbf rate $BANDWIDTH burst $BURST limit $BUFFER_BYTES
```

**层次结构可视化：**

```
tap0
 └── root (1:)          ← netem qdisc
      └── 1:1 (10:)     ← tbf qdisc
```

#### netem（网络模拟器）命令

```bash
sudo tc qdisc add dev tap0 root handle 1: netem delay 10ms limit 4000
```

**参数说明：**
- `dev tap0`：应用到 tap0 接口
- `root`：这是根 qdisc（层次结构中的第一层）
- `handle 1:`：该 qdisc 的标识符（用于附加子 qdisc）
- `netem`：网络模拟 qdisc
- `delay 10ms`：为所有数据包添加 10ms 固定延迟
- `limit 4000`：最大队列长度（数据包数），必须 ≥ BDP 以避免提前丢包

**作用：**
- 添加传播延迟以模拟距离
- 在转发数据包前将其保持在队列中
- 如果队列超过 `limit`，数据包被丢弃（队头丢弃）

**为什么需要大的 limit？**
- 设置 limit=4000 确保数据包不会在到达 tbf 缓冲区前被丢弃
- netem 的职责只是添加延迟，而不是限制吞吐量

#### tbf（令牌桶过滤器）命令

```bash
sudo tc qdisc add dev tap0 parent 1:1 handle 10: tbf rate 400mbit burst $BURST limit $BUFFER_BYTES
```

**参数说明：**
- `parent 1:1`：netem qdisc 的子节点（附加到类 1:1）
- `handle 10:`：该 qdisc 的标识符
- `tbf`：令牌桶过滤器 qdisc
- `rate 400mbit`：最大带宽（令牌生成速率）
- `burst`：最大突发大小（字节），通常为 rate/10
- `limit`：缓冲区大小（字节）—— **这就是我们要测试的路由器队列！**

**令牌桶算法：**

1. **令牌累积**：以 `rate` 速率累积（例如 400 Mbit/s = 50 MB/s = 每毫秒 50,000 个令牌）
2. **发送数据包**：消耗等于数据包大小的令牌
3. **突发（Burst）**：允许临时突发最多 `burst` 字节
4. **缓冲区（limit）**：当令牌不可用时，数据包在这里等待
5. **尾丢弃**：当缓冲区满时，新数据包被丢弃

**计算 limit（缓冲区大小）：**
```bash
BUFFER_BYTES=$((BUFFER_PACKETS * MTU))  # 4000 × 1500 = 6,000,000 字节
```
- 这是**路由器队列**，拥塞控制算法与之交互
- 大缓冲区：Cubic/Reno 填满它，造成缓冲区膨胀
- 小缓冲区：频繁丢包，测试拥塞响应
- BBR 应该无论缓冲区大小都不填满它

#### 验证命令

检查 qdisc 配置：

```bash
# 显示 tap0 上的所有 qdisc
tc qdisc show dev tap0

# 预期输出：
# qdisc netem 1: root refcnt 2 limit 4000 delay 10ms
# qdisc tbf 10: parent 1:1 rate 400Mbit burst 15000b lat 120ms

# 显示详细统计
tc -s qdisc show dev tap0

# 预期输出包括：
# - 已发送/丢弃的数据包计数
# - Backlog（当前队列深度）
# - Overlimits（达到速率/缓冲区限制的次数）
```

### 4.5 性能指标

**吞吐量指标**

服务器和客户端都报告：
- 传输字节数（MB）
- 耗时（秒）
- 吞吐量（Gbps）

公式：`Throughput = (bytes × 8) / time / 1e9`

**延迟指标**

客户端测量单向延迟（使用数据中嵌入的时间戳）：
- **Samples**：测量样本数
- **Min**：最小延迟
- **Mean**：平均延迟
- **p50**：中位数
- **p95**：95 百分位
- **p99**：99 百分位
- **Max**：最大延迟

### 4.5 测试场景

**对比拥塞控制算法**

```bash
# 测试 BBR
./scripts/emulate_setup.sh
./scripts/perf_server_run.sh bbr
./scripts/perf_client_run.sh bbr

# 测试 Cubic
./scripts/cleanup.sh
./scripts/emulate_setup.sh
./scripts/perf_server_run.sh cubic
./scripts/perf_client_run.sh cubic
```

### 4.6 实际测试结果

在默认配置下（400 Mbit/s, 20ms RTT, 6MB 缓冲区）进行了 Cubic 和 BBR 的对比测试，传输 3GB 数据。

#### Cubic 测试结果

```
Performance Client (Cubic)
==========================
Congestion Control: Cubic
Buffer size: 6291456 bytes

Progress:
237.64 MB received | 0.380 Gbps | Avg Latency: 120.434 ms
478.72 MB received | 0.383 Gbps | Avg Latency: 120.434 ms
719.48 MB received | 0.384 Gbps | Avg Latency: 120.505 ms
...
2886.47 MB received | 0.385 Gbps | Avg Latency: 120.566 ms
3127.55 MB received | 0.385 Gbps | Avg Latency: 120.557 ms

Final Results:
==============
Total received: 3221.23 MB
Time: 66.96 seconds
Throughput: 0.385 Gbps

Latency Statistics:
  Samples: 2108
  Min:     119.939 ms
  Mean:    120.552 ms
  p50:     120.435 ms
  p95:     120.458 ms
  p99:     125.502 ms
  Max:     129.421 ms
```

#### BBR 测试结果

```
Performance Client (BBR)
========================
Congestion Control: Bbr
Buffer size: 6291456 bytes

Progress:
226.39 MB received | 0.362 Gbps | Avg Latency: 122.064 ms
467.41 MB received | 0.374 Gbps | Avg Latency: 121.247 ms
708.49 MB received | 0.378 Gbps | Avg Latency: 120.984 ms
...
2878.18 MB received | 0.384 Gbps | Avg Latency: 120.585 ms
3119.25 MB received | 0.384 Gbps | Avg Latency: 120.577 ms

Final Results:
==============
Total received: 3221.23 MB
Time: 67.14 seconds
Throughput: 0.384 Gbps

Latency Statistics:
  Samples: 1969
  Min:     119.861 ms
  Mean:    120.571 ms
  p50:     120.449 ms
  p95:     120.488 ms
  p99:     120.895 ms
  Max:     279.506 ms  ← 偶发峰值
```

#### 结果分析

1. **吞吐量相当**: 两种算法都能充分利用可用带宽（~96%），在大缓冲区环境下表现接近。

2. **延迟接近**: 平均延迟和 p50/p95 几乎相同，约 120 ms（基准延迟 20 ms + 约 100 ms 队列延迟）。

3. **缓冲区膨胀严重**: 两种算法的延迟都显示约 100 ms 的额外队列延迟，远超基准 RTT（20 ms），说明在 6× BDP 的大缓冲区环境下，两者都未能有效避免缓冲区填充。

4. **BBR 未达预期**: 理论上 BBR 应保持低队列占用（~2 BDP），但实际测试显示队列延迟与 Cubic 相当。这表明实现中可能存在以下问题：
   - 带宽估计可能不够准确
   - Pacing 可能未完全生效
   - 状态转换逻辑需要优化
   - ProbeRTT 可能未正确清空队列

5. **BBR 的偶发峰值**: Max 延迟 279 ms 表明存在异常情况，可能是：
   - ProbeRTT 状态转换时的测量误差
   - ProbeBW 探测过度时的短暂队列堆积
   - 或是测量本身的异常值

**总体评价：**

当前的 BBR 实现在吞吐量上与 Cubic 持平，在 p99 延迟上略有优势，但并未展现理论上应有的显著队列控制能力。

后续优化方向应聚焦于：
- 确保 ProbeRTT 能有效清空队列

---

## 5. 总结

### 5.1 项目成果

本项目成功为 smoltcp 实现了 BBR 拥塞控制算法及其所需的 TCP Pacing 机制。虽然 BBR 的性能表现与 Cubic 相当，预期中的显著优势尚未完全体现，但项目取得了以下重要成果：

**技术实现**
1. 完整的 BBR 状态机（Startup、Drain、ProbeBW、ProbeRTT）
2. 带宽估计和 RTT 跟踪机制
4. 通用的 TCP Pacing 框架

**测试方法**
1. 基于 Linux tc 的网络仿真框架
2. BDP 计算和缓冲区配置策略
3. 吞吐量和延迟测量工具
4. 可重复的测试流程

### 5.2 经验教训

**实现的复杂性**

BBR 不仅仅是一个拥塞控制算法，它是一个需要多个协同机制支持的系统：
- Pacing 是必要前提，而非可选项
- 带宽估计需要准确的时间戳和传输追踪

**测试环境的重要性**

没有好的测试环境，就无法理解算法的行为：
- 真实网络太不可控
- 仿真环境必须精确配置延迟、带宽、缓冲区
- BDP 是关键参数，必须正确计算

### 5.3 关键收获

尽管 BBR 的性能优化尚未完成，但本项目最宝贵的成果是**建立了一套完整的测试和调试方法**：

1. **如何搭建网络仿真环境**
   - 使用 TAP 接口和桥接
   - 使用 tc netem 和 tbf 控制网络参数
   - 理解 BDP 并配置合适的缓冲区

2. **如何理解拥塞控制**
   - 基于丢包 vs 基于速率
   - 窗口控制 vs Pacing

这些知识和工具对于任何网络性能相关的工作都至关重要，其价值远超过单个算法的实现。

### 5.4 结语

网络拥塞控制是一个复杂而精妙的领域。从传统的 Reno、Cubic 到现代的 BBR，每一代算法都试图在吞吐量、延迟、公平性之间找到更好的平衡点。

本项目虽然在性能上还有优化空间，但通过实践深入理解了：
- 拥塞控制的核心机制
- 算法设计与实现的权衡
- 测试和调试的方法论

这些经验将为后续的优化和改进提供坚实的基础。

---

## 参考文献

1. Cardwell, N., et al. (2016). "BBR: Congestion-Based Congestion Control", ACM Queue: https://dl.acm.org/doi/10.1145/3009824`
2. Linux Kernel Source: `net/ipv4/tcp_bbr.c`: https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/tree/net/ipv4/tcp_bbr.c
3. Quinn QUIC Implementation: https://github.com/quinn-rs/quinn
4. smoltcp Documentation: https://docs.rs/smoltcp/
