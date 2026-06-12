# ComputerNetwork
# Low Latency in Multiplayer Games
### Measuring, Analyzing, and Optimizing Network Latency in an Unreal Engine 5 Multiplayer Environment

> **Course:** Computer Networks | **Team:** Group 9 : 고은서, 김현서, 박은혁, 박재형
> **Stack:** Unreal Engine 5 · tshark / Wireshark · Python (pyshark, pandas) · Blueprint

## 1. Motivation

Latency is not a uniform problem — its impact varies dramatically depending on the **interaction type** a game demands from its players.

Shafiee Sabet et al. (2020) formalize this through two dimensions:

| Dimension | Definition |
|---|---|
| **Temporal Accuracy (TA)** | The time window available for a player to react and perform an action |
| **Spatial Accuracy (SA)** | The precision required to complete an interaction successfully |

Based on a decision-tree classifier achieving **90% accuracy** in latency-sensitivity classification, genres can be ranked as follows:

```
Latency Sensitivity
──────────────────────────────────────────────────►  HIGH
Card Game    RTS    Fighting    [Our Shooter]
```

Our target — a third-person shooter built on UE5 — sits at the **highest latency-sensitivity class**: players fire projectiles at fast-moving targets, where even a few milliseconds of desynchronization can mean the difference between a hit and a miss.

This makes it an ideal subject for rigorous latency measurement and optimization.

---

## 2. Background — What Does Ping Actually Measure?

A common misconception is that the in-game ping value directly reflects network-level round-trip time. In reality, `GetPingInMilliseconds()` in UE5's `PlayerState` returns an **Application-Level (L7) RTT** — a value that bundles both network cost and engine processing overhead.

```
Client                                          Server
  │── packet out ──►│ prop │ tx │ queue │ proc │── response ──►│
                     └─────────────────────────┘
                           L4 RTT (pure network)
  │◄────────────────────── L7 Ping (engine RTT) ──────────────►│
                                      ▲
                              app_delay included
```

The relationship is:

```
app_delay = L7_ping − L4_rtt
```

**In our LAN baseline:**

```
app_delay = 12.964 ms (L7) − 7.20 ms (L4) = 5.76 ms
```

This ~5.76 ms is pure engine processing overhead — invisible if only L7 ping is monitored, yet it accounts for **44% of the total observed latency**. Conflating the two layers leads to misdiagnosis of performance bottlenecks.

---

## 3. Protocol Landscape: TCP vs UDP vs QUIC

Before diving into measurements, it is worth establishing why the protocol choice matters.

| Feature | TCP | UDP (UE5) | QUIC |
|---|---|---|---|
| Reliability | Ordered, guaranteed | Custom Reliable/Unreliable layer | Per-stream reliable |
| HOL Blocking | Present | None | None (stream-isolated) |
| Connection Setup | 1-RTT (3-way handshake) | Connectionless | 0-RTT |
| Encryption | None (TLS separate) | None | TLS 1.3 built-in |
| Game Suitability | ❌ Not suitable | ✅ Standard | ⚠️ Web-focused |

**Why UE5 uses UDP with a custom layer:**  
TCP's head-of-line blocking is fatal for real-time games — a dropped position packet stalls all subsequent data in the stream. UE5 implements its own **Reliable** and **Unreliable** channel separation on top of UDP, applying delivery guarantees only where gameplay correctness demands it (e.g., hit registration) while letting position updates flow unreliably at high frequency.

**Why not QUIC?**  
QUIC's 0-RTT connection and stream-isolated HOL blocking look attractive, but the protocol is web-oriented and would require a full rewrite of UE5's network driver. Its TLS 1.3 encryption also introduces per-packet crypto overhead that is unnecessary for LAN/competitive environments. At present, UE5's UDP custom layer remains better optimized for game-specific traffic patterns.

---

## 4. Measurement System Design

To cleanly separate network cost from engine cost, we built a **dual-layer measurement architecture**.

```
┌─────────────────────────────────────────────────────────────┐
│                    MEASUREMENT SYSTEM                        │
│                                                              │
│  L7 — Engine Layer              L4 — OS Layer               │
│  ┌─────────────────────┐        ┌──────────────────────┐    │
│  │  Blueprint Ping HUD │        │  tshark / Wireshark  │    │
│  │                     │        │                      │    │
│  │ Event Tick →        │        │ libpcap nanosecond   │    │
│  │ GetPingInMs()       │        │ precision timestamps │    │
│  │ CSV: epoch_ms,      │        │ Filter: udp port 7777│    │
│  │       ping_ms       │        │ Ground truth RTT     │    │
│  └─────────────────────┘        └──────────────────────┘    │
│           │                              │                   │
│           └──────────┬───────────────────┘                  │
│                      ▼                                       │
│           app_delay = L7_ping − L4_rtt                      │
└─────────────────────────────────────────────────────────────┘
```

**Why tshark over a custom implementation?**

- Validated in 5G research, security, and penetration testing contexts
- Hardware-level timestamps — no reimplementation risk
- Passive measurement — zero additional traffic injected into the test environment

The dual-layer approach is the methodological core of this project: without it, we would be unable to distinguish whether an observed latency spike originated from the network path or from engine-side replication processing.

---

## 5. Experiment Design

Four scenarios were designed to isolate specific replication events:

| Scenario | Load Level | Description |
|---|---|---|
| **Idle** | Baseline | Character stationary. Establishes baseline packet rate and RTT. |
| **Movement** | Medium | Continuous position, rotation, and jump state synchronization. |
| **Box Interaction** | Medium | Interaction with 3 and 18 boxes. Tests object-count hypothesis. |
| **Shooting** | High | Projectile Actor spawned on server, replicated to all clients. |

**Variables:** Players: 2 and 3 · Boxes: 3 vs 18 · Metrics: packet count, total bytes, average/P95/max packet size, RTT

### Hypothesis Revision

Our initial hypothesis was: *more objects → more packets → higher network load.*

The box interaction scenario disproved this directly — 3 boxes and 18 boxes produced **identical packet counts**. Only boxes that underwent a state change generated replication events.

This finding reshaped our approach: the true driver of network cost is **replication events**, not object count. The Shooting scenario was added specifically to explore high-frequency, high-payload replication.

---

## 6. Results & Analysis

### 6.1 Packet Capture Summary (2-Player, L4)

| Scenario | Packets | BW (KB/s) | Avg Pkt Size | RTT Avg | BW Δ | P95 Pkt Size | Max Pkt Size |
|---|---|---|---|---|---|---|---|
| Idle | 1,361 | 7.07 | 56.1 B | 7.20 ms | baseline | 70 B | 96 B |
| Movement | 1,334 | 7.87 | 58.2 B | 7.23 ms | +11.2% | 95 B | 145 B |
| Jump | 1,299 | 7.54 | 57.0 B | 7.38 ms | +6.6% | — | — |
| Box Interaction | 1,247 | 7.88 | 61.4 B | 7.62 ms | +11.3% | — | — |
| **Shooting** | **903** | **12.48** | **141.5 B** | — | **+76.5%** | **271 B** | **319 B** |

### 6.2 Key Observations

**Movement — High frequency, low payload:**  
Position, rotation, and jump state are replicated every tick, producing the highest packet count (1,334). However, each packet is small (avg 58.2 B), so total byte burden remains modest. Bandwidth increase is only +11.2% over idle.

**Box Interaction — Object count ≠ network load:**  
The packet count with 18 boxes was indistinguishable from 3 boxes. UE5 replication is event-driven: only state-changed actors generate traffic. This disproved the initial hypothesis and is a critical insight for game developers who assume large worlds automatically mean high network cost.

**Shooting — The real bottleneck:**  
Despite the fewest packets (903), Shooting produced the largest average packet size (141.5 B — 2.4× Movement) and the highest bandwidth (+76.5%). The Projectile Actor carries spawn metadata, movement vectors, and collision state, inflating each packet significantly. Lag was **directly observed during testing**, confirming the real-world impact.

```
Bandwidth Increase Over Idle
──────────────────────────────────────────────
Movement      ████░░░░░░░░░░░░░░░░░  +11.2%
Jump          ██░░░░░░░░░░░░░░░░░░░   +6.6%
Box           ████░░░░░░░░░░░░░░░░░  +11.3%
Shooting      ████████████████████  +76.5%  ← bottleneck
```

---

## 7. Optimization

The Shooting scenario was selected for optimization. The target: reduce per-packet payload without sacrificing gameplay correctness.

### BP_Bullet Replication Settings

| Setting | Before | After | Rationale |
|---|---|---|---|
| Replicates | `true` | `true` | Actor must remain server-authoritative |
| **Replicate Movement** | **`true`** | **`false`** | Removes per-frame position sync — the primary payload driver |
| Life Span | 5.0 s | 2–3 s | Reduces the window during which the actor exists and can replicate |
| Net Update Frequency | 100 | 80 | Lowers max replication rate; marginal benefit vs. the above |

**Test conditions:** 50 shots over 10 seconds (5/s) · Worst-case latency scenario

### Optimization Strategy Rationale

Disabling Replicate Movement shifts the rendering responsibility to the client: each client simulates bullet trajectory locally using the initial velocity and direction received at spawn time. The server retains authority over **hit detection** — only the collision result is synchronized, not the intermediate positions.

This is a well-established pattern in competitive multiplayer games (sometimes called *client-side prediction with server reconciliation*), where visual smoothness and network efficiency are balanced by accepting minor positional divergence between clients.

---

## 8. Optimization Results

### 8.1 Measured Data

| Metric | P2 Before | P2 After | P2 Δ | P3 Before | P3 After | P3 Δ |
|---|---|---|---|---|---|---|
| Packet Count | 1,095 | 1,119 | +2.2% | 920 | 925 | +0.5% |
| **Total Bytes** | 106,398 | 66,559 | **−37.4%** | 104,549 | 60,150 | **−42.5%** |
| **Avg Packet Size** | 97.2 B | 59.5 B | **−38.8%** | 113.6 B | 65.0 B | **−42.8%** |
| **P95 Packet Size** | 158 B | 74 B | **−53.2%** | 185 B | 87 B | **−53.0%** |
| **Max Packet Size** | 203 B | 96 B | **−52.7%** | 230 B | 101 B | **−56.1%** |

### 8.2 Interpretation

```
Packet Count        ██████████████████████  +2.2%   (nearly unchanged)
Total Bytes         ████████████░░░░░░░░░░  −37.4%  ✓
Avg Packet Size     ████████████░░░░░░░░░░  −38.8%  ✓
P95 Packet Size     ██████████░░░░░░░░░░░░  −53.2%  ✓✓
Max Packet Size     ██████████░░░░░░░░░░░░  −52.7%  ✓✓
```

Packet count is essentially unchanged (+2%), confirming that the overhead was never in the number of packets — it was in the **content of each packet**. Disabling Replicate Movement surgically removed the movement vector payload while keeping the actor replication skeleton intact.

The P95 and max packet size reductions (~53%) are particularly significant: these large outlier packets are precisely what causes **burst congestion** and perceived lag spikes during heavy shooting sequences.

---

## 9. Discussion: Trade-offs & Limitations

### 9.1 Gameplay Quality Trade-offs

Disabling Replicate Movement is not cost-free. Three areas warrant scrutiny:

| Concern | Impact | Mitigation |
|---|---|---|
| **Bullet positional accuracy** | Clients simulate locally; minor divergence between clients is possible | Acceptable for projectile-based gameplay; critical for precision scenarios |
| **Ping registration** | High-latency players may observe hits on their screen that the server rejects | Requires lag compensation (rewind-based hit detection) to be fair |
| **Fairness** | Without lag compensation, high-ping players are structurally disadvantaged | Server-side lag compensation is the standard industry solution |

> **Note:** Our experiment was conducted on a LAN environment (RTT ~7 ms), where ping differences between players were negligible. The trade-offs above become meaningful in WAN conditions with heterogeneous player latencies.

### 9.2 Hitscan vs. Projectile

Our Shooting scenario used **Projectile Actor replication** — a server-spawned actor whose existence and movement are replicated to clients. An alternative implementation is **hitscan**, where the server performs an instant raycast at fire time and no actor is spawned or replicated.

Hitscan eliminates the Replicate Movement problem entirely, as there is no actor to replicate. However, it introduces different challenges:
- Visual feedback must be generated client-side from the raycast result
- Lag compensation becomes more critical for fair hit registration at range

A direct performance comparison between projectile and hitscan implementations was not conducted in this study and remains an avenue for future work.

### 9.3 LAN vs. WAN Generalizability

All measurements were taken on a LAN with sub-10 ms RTT. In WAN conditions:
- Propagation delay dominates L4 RTT, reducing the relative proportion of app_delay
- Packet loss introduces UDP retransmission overhead on Reliable channels
- The absolute bandwidth savings from optimization remain valid, but their impact on perceived latency increases

---

## 10. Conclusion

| Finding | Detail |
|---|---|
| **Dual-layer measurement validated** | `app_delay = L7_ping − L4_rtt` isolates engine processing overhead. LAN baseline: ~5.76 ms (44% of total L7 ping). |
| **Object count ≠ network load** | 3 vs. 18 boxes produced no measurable packet difference. Only state-changed objects generate replication events. |
| **Replication events are the bottleneck** | Shooting had the fewest packets but the largest total bytes and packet size (+76.5% BW). |
| **Content, not count** | Disabling Replicate Movement left packet count unchanged (+2%) while cutting total bytes by 37–43% and P95 packet size by 53%. |

**Core principle:**

> *Low latency optimization is not about eliminating packets — it is about synchronizing only what gameplay requires, at the right frequency, in the smallest possible payload.*

---

## 11. References

- Shafiee Sabet, S., Schmidt, S., Zadtootaghaj, S., Griwodz, C., & Möller, S. (2020). *Delay Sensitivity Classification of Cloud Gaming Content.* arXiv:2004.05609. https://arxiv.org/abs/2004.05609
- Unreal Engine 5.2 API – APlayerState::GetPingInMilliseconds. https://docs.unrealengine.com/5.2/en-US/API/Runtime/Engine/GameFramework/APlayerState/GetPingInMilliseconds/
- Unreal Engine Networking Architecture (UDK). https://docs.unrealengine.com/udk/Three/NetworkingOverview.html
- Unreal Engine 4.27 – Networking Insights Overview. https://docs.unrealengine.com/4.27/en-US/TestingAndOptimization/PerformanceAndProfiling/UnrealInsights/NetworkingInsights/
- HAProxy Blog – Choosing the Right Transport Protocol: TCP vs. UDP vs. QUIC. https://www.haproxy.com/blog/choosing-the-right-transport-protocol-tcp-vs-udp-vs-quic

---

## 12.Presentation video
https://www.youtube.com/watch?v=Jcogrbha-co

---

<details>
<summary>Appendix: Tools & Environment</summary>

| Component | Detail |
|---|---|
| Game Engine | Unreal Engine 5 (Blueprint + C++ PlayerState) |
| Packet Capture | tshark (libpcap, nanosecond precision) |
| Capture Filter | `udp port 7777` |
| Analysis | Python — pyshark, pandas |
| Logging | CSV via Blueprint Tick (epoch_ms, ping_ms) |
| Network Environment | LAN (Ethernet) |
| Player Configurations | 2-player and 3-player sessions |

</details>
