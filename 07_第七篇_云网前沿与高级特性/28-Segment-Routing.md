# 第28章 Segment Routing (SR-MPLS)

> 新一代极简承载网转发架构：Segment Routing 彻底颠覆了传统 IP“逐跳转发”与 MPLS TE“全网维护软状态信令”的落后模式，树立“源路由（Source Routing）”与“中间节点无状态”的全新哲学；全面拆解 SID 体系（Prefix-SID / Node-SID / Adjacency-SID / Binding SID）与 SRGB 标签空间计算；深度对比 SR-BE（无缝替代 LDP）与 SR-TE（基于 CSPF 与控制器的高级流量工程）；揭秘 SR Policy `<Headend, Color, Endpoint>` 三元组与自动化引流；推导保护算法从 LFA、RLFA（P/Q/PQ 空间）到 TI-LFA（100% 任意拓扑覆盖，<50ms 修复列表）的数学演进与防微环实践。

| 项 | 内容 |
|---|---|
| **适用层级** | HCIE 核心云网与运营商级架构（理论硬核、实验必考、面试核心必杀技） |
| **建议学习位次** | 全书第 28 章，先修第 05 章/06（OSPF/IS-IS 路由协议与扩展 TLV）、第 22 章/23（MPLS 标签转发表与 L3VPN）、第 27 章（EVPN 控制平面）；学完本章后建议后修第 29 章（SRv6 新一代承载） |
| **技术维度** | 协议原理深度剖析 · 生产级VRP配置 · 故障诊断排错 · 架构设计决策 |

---

## 一、总起

在过去二十年间，MPLS 凭借标签交换在运营商骨干网与大型企业专网中占据着统治地位。然而，随着云时代超大带宽、海量端点、微秒级时延敏感业务的爆发，传统 MPLS 架构的两大死穴暴露无遗：
1. **LDP + IGP 双控制面不同步**：LDP 本身没有算路能力，完全依附于 IGP。当链路故障恢复时，IGP 路由先收敛，而 LDP 标签分发滞后，导致在毫秒到秒级的时间窗口内产生**黑洞丢包**；
2. **MPLS TE（RSVP-TE）状态臃肿无法规模化**：RSVP-TE 采用软状态机制（Soft State），为了维持一条隧道，沿途所有路由器都必须周期性发送信令报文并维护该隧道的控制状态。网络中如果有数千条隧道，设备 CPU 和内存瞬间被海量 RSVP 状态拖垮，根本无法在云化多数据中心环境下横向扩展。

**Segment Routing（SR，分段路由）** 由 IETF（RFC 8402）提出，成为划时代的革命性转发架构。**SR 不是一种新的路由协议，而是一种全新的转发哲学**——它基于**源路由（Source Routing）**理念：
- **网络头端编排一切**：在数据包进入 SR 网络的入口（Headend），根据业务对带宽、时延、亲和性的 SLA 诉求，将计算好的转发路径编码成一个**有序的段列表（Segment List，在数据面即为 MPLS 标签栈）**并压入包头；
- **中间节点绝对无状态（Stateless）**：骨干网中间节点（Transit）完全不需要维护任何业务隧道状态，仅按栈顶标签无脑执行最基本的转发或弹出指令，出栈后直接移交下一跳；
- **控制平面极致简化**：彻底废黜 LDP 和 RSVP-TE 信令，直接对现有的 IGP（OSPF / IS-IS）和 BGP 进行扩展（TLV / Opaque LSA），实现单协议即完成拓扑计算与标签分发！

本章从以下核心维度系统拆解 SR-MPLS 的全景图谱：
1. **架构本源与三大 SID**：确立源路由理念与业务驱动网络逻辑；掌握 Prefix-SID、Node-SID、Adjacency-SID 的作用域与生成机制；推导 SRGB（全局标签块）与绝对标签的换算公式；
2. **IGP 协议扩展与 SR-BE**：详解 OSPFv2（Type-10 Opaque LSA）与 IS-IS（TLV 135/22 子 TLV）如何向全网扩散 SID；实战配置 SR-BE 替换传统 LDP 承载 EVPN / L3VPN 业务；
3. **SR-TE 流量工程与粘连标签**：对比传统分布式 RSVP-TE 与集中式控制器算路；掌握 Binding SID（绑定段）如何破解硬件标签栈过深瓶颈并实现跨域隧道拼接；
4. **BGP EPE 跨域工程**：详解 Egress Peer Engineering 的 Peer-Node、Peer-Adj、Peer-Set SID，手搓端到端跨域流量工程；
5. **SR Policy 架构与自动化引流**：告别传统繁杂的 Tunnel 虚接口，掌握由 `<Headend, Color, Endpoint>` 构成的 SR Policy 模型与基于 BGP Color 属性的无感引流；
6. **保护倒换算法数学演进（LFA $\rightarrow$ RLFA $\rightarrow$ TI-LFA）**：从 LFA 三角不等式失效场景切入，系统推导 RLFA 的 P 空间、Q 空间与 PQ 节点交集；攻克 **TI-LFA（拓扑无关 LFA）** 修复路径（Repair List）计算，实现 100% 任意拓扑保护与防微环（Micro-loop）延时回切。

**适用人群**：
- **初学 / 转行人员**：摒弃“每台路由器都要学整条路”的固有思维，建立“源头买好通票，中间沿途打卡”的高级抽象模型；
- **备考工程师（HCIE 核心攻坚）**：彻底攻克 HCIE 笔试与面试中难度最高的“SRGB 与 SRLB 区别”、“TI-LFA 算路过程与 Repair List 生成”、“SR Policy 候选路径优先级选路”；
- **一线工程实践者**：掌握骨干网从传统 LDP/RSVP-TE 向 SR-BE/SR-TE 平滑割接的最佳工程规范、多层标签栈硬件深度规避技巧，以及毫秒级 SBFD 联动部署。

### 本章技术架构与核心脉络

| 模块序号 | 核心技术议题 | 架构要点与设计精髓 |
|:---|:---|:---|
| **§28.1** | Segment Routing简介1 | SR架构本源、传统BE/TE缺陷(LDP不同步/RSVP状态臃肿)、源路由理念、业务驱动网络、Node/Adj-SID、CSPF算法概览、SRGB与标签计算 |
| **§28.2** | Segment Routing简介2 | 邻接SID特性与本地有效性、OSPF Opaque LSA/IS-IS TLV协议扩展、OSPF/IS-IS SR-BE隧道实战、SR承载EVPN/L3VPN、隧道策略匹配 |
| **§28.3** | Segment Routing简介3 | SR-TE流量工程、SR Policy雏形、粘连标签(Binding SID)破解标签栈深度、三种SID组合选路、分布式vs集中式控制器算路、双隧道手工配置 |
| **§28.1** | Segment Routing简介4 | SR跨域E2E TE架构、Binding SID跨域拼接、BGP EPE(Peer-Node/Adj/Set SID)、端到端跨域隧道实战与静态路由引流、BGP-LS跨域思路 |
| **§28.2** | Segment Routing简介5 | 传统Tunnel接口引流缺陷、SR Policy三元组模型(Headend/Color/Endpoint)、候选路径与权重、基于Color自动化引流、LFA/RLFA保护机制总览 |
| **§28.3** | Segment Routing简介6 | RLFA数学推导(P/Q/PQ空间与公式)、TI-LFA修复列表(Repair List)100%覆盖、Anycast FRR、SBFD极速联动、防微环(正切/回切/远端)机制 |

---

## 二、分述

### 2.1 源路由（Source Routing）哲学与架构革新

#### 传统网络 vs Segment Routing

```
传统 IP / MPLS (逐跳转发 / 软状态)               Segment Routing (源路由 / 无状态)
    [源端] ──> [R1] ──> [R2] ──> [目的端]            [源端] ──> [R1] ──> [R2] ──> [目的端]
       │         │        │                            │         │        │
       ▼         ▼        ▼                            ▼         ▼        ▼
  每台路由器独立算路, 维护海量状态             源端贴好标签栈 [Label-A, Label-B]
  (LDP/RSVP 信令满天飞, 协议臃肿)               中间节点无脑执行 POP/SWAP, 零业务状态！
```

【形象工程类比】：“传统网络转发就像以前你出门旅游，每到一个路口都得停下来问路人‘火车站怎么走’，路人查本地地图告诉你去下一个路口；如果修路改道，大家乱作一团。而 Segment Routing 就像现代的高铁出行，你在出发买票时，票面上已经清晰打印了整条行程单：‘第一段坐 G101 到南京南，第二段换乘 G202 到上海虹桥’。沿途列车员只要看一眼顶上的当前行程，到站把你放下来换乘即可，中间车站完全不需要认识你这个人是谁！”

---

### 2.2 SID 体系架构与 SRGB 标签空间计算

Segment Identifier（SID，段标识符）是指引网络执行转发行为的核心指令。在 SR-MPLS 数据平面中，SID 直接映射为标准的 20 位 MPLS 标签。

```
                                  [SID 体系大家族]
           ┌─────────────────────────────┼─────────────────────────────┐
      Prefix-SID (前缀段)          Adjacency-SID (邻接段)         Binding SID (绑定段)
    (全局唯一, 基于最短期望算路)         (本地有效, 指定出接口下一跳)      (折叠标签栈, 跨域拼接)
           │
       Node-SID (节点段)
    (特化 Prefix-SID, 指向 Loopback)
```

#### 1. Prefix-SID 与 Node-SID
- **Prefix-SID（前缀段）**：用于标识网络中的一个特定 IP 地址前缀。由 IGP 扩展通告，具有**全局唯一性**；
- **Node-SID（节点段）**：特化的 Prefix-SID，配置在路由器的 Loopback0 接口上，用于唯一标识该路由器节点本身；
- **算法特征**：数据包沿着针对该 Node-SID 计算的**最短路径（ECMP）**转发，本质上执行标准的最短路径优先（SPF）算路。

#### 2. Adjacency-SID（邻接段）
- **定义**：用于精确标识网络中的某条单向物理链路（出接口与下一跳）；
- **范围特征**：由运行 IGP 协议的路由器动态随机分配，或者手动静态配置。**全网泛洪可见，但仅在本地有效**；
- **算法特征**：指示设备必须**无条件将报文从该特定出接口发往对应邻居**，不经过任何 SPF 算路计算（强行指路）。

#### 3. SRGB（Segment Routing Global Block，全局标签块）与标签换算
为了防止不同厂商、不同节点本地标签冲突，SR 规划了专门的标签空间：
- **SRGB**：用户为 SR-MPLS 预留的全局标签范围（华为默认通常为 `16000 ~ 23999`）；
- **Prefix-SID 索引化设计**：为了跨节点灵活配置，设备在发布 Prefix-SID 时，通告的是一个相对偏移索引（Index），而非绝对标签！
- **绝对标签计算公式**：
  $$\text{本地 MPLS 绝对标签} = \text{本地 SRGB 起始基值 (Base)} + \text{Prefix-SID 索引 (Index)}$$

> **工程计算范例**：
> 设全网路由器均将 Node-4 的 Index 规划为 `4`。
> - 若 Router-A 的 SRGB 为 `[16000, 23999]`，则 Router-A 为 Node-4 分配的出标签为：$16000 + 4 = 16004$；
> - 若跨厂商对接的 Router-B 其 SRGB 为 `[20000, 27999]`，则 Router-B 为 Node-4 分配的出标签为：$20000 + 4 = 20004$。
> 两台设备标签基值虽异，但在各自节点均能基于统一的 Index 精准映射，彻底实现异构互通！

---

### 2.3 IGP 协议扩展与 SR-BE 极简承载

SR-MPLS 无需任何新的信令协议，只需对底层 IGP 进行字段扩充即可实现标签分发：
- **OSPFv2 协议扩展**：利用 RFC 7684 定义的 **Type 10 Opaque LSA**，新增了 Extended Prefix TLV（携带 Prefix-SID）与 Extended Link TLV（携带 Adjacency-SID）；
- **IS-IS 协议扩展**：利用 RFC 7794 定义的 **TLV 135 / TLV 235** 挂载 Prefix-SID 子 TLV；利用 **TLV 22 / TLV 222** 挂载 Adj-SID 子 TLV。

#### SR-BE（Best-Effort）替代传统 LDP

```
传统 LDP 转发: 需要 IGP 路由建立 ──> LDP 建立会话 ──> 分发分配标签 ──> 建立 LSP 隧道
SR-BE 转发:    IGP 邻居建立 ──> 直接携带 Node-SID ──> 自动生成基于 Node-SID 的 SR-BE 隧道！
```

在 SR-BE 模式下，整条端到端路径仅需压入**一个单一的终点 Node-SID 标签**。沿途节点根据 IGP 最短路径查表执行标签替换（SWAP），倒数第二跳执行次末跳弹出（PHP），完美实现与传统 LDP 性能一致但协议复杂度降低 80% 的极简承载。

#### 华为 VRP 标准配置模板（以 IS-IS 部署 SR-BE 为例）

```shell
# ==================== [PE-1 部署 IS-IS SR-BE] ====================
sysname PE1
# 1. 全局使能 MPLS
mpls
 quit

# 2. 配置 IS-IS 进程并开启 SR 功能
isis 1
 is-level level-2
 cost-style wide                   # 必须使用宽开销模式才能携带 SR TLV
 network-entity 49.0001.0000.0000.0001.00
 # 使能 Segment Routing MPLS
 segment-routing mpls
 # 配置本地 SRGB 范围
 segment-routing global-block 16000 23999
quit

# 3. 在 Loopback0 接口配置 Node-SID
interface LoopBack0
 ip address 1.1.1.1 255.255.255.255
 isis enable 1
 # 为 Loopback0 分配 Node-SID 索引 1 (绝对标签 16001)
 isis prefix-sid index 1
quit

# 4. 在物理接口使能 MPLS 与 IS-IS
interface GigabitEthernet0/0/1
 isis enable 1
 mpls
 quit
```

---

### 2.4 SR-TE 流量工程与 Binding SID（粘连标签）

当业务提出苛刻的 SLA 要求（如：必须绕开拥塞链路、必须走低时延线路、或者必须经过特定的防火墙设备）时，SR-BE 的最短路径无法满足要求，必须引入 **SR-TE（Traffic Engineering）**。

#### 三种 SID 组合编排方式

```
1. 纯 Node-SID 编排: [Node-B, Node-D] ──> 在 Node 之间走 IGP 最短路径，节点间具备 ECMP 负载能力
2. 纯 Adj-SID 编排:  [Adj-12, Adj-23, Adj-34] ──> 严格限定逐跳链路，路径绝对精准固化，但失去 ECMP 能力
3. 混合编排 (推荐):  [Node-C, Adj-35, Node-E] ──> 先最短路到达 C，强行走 35 链路，再最短路到达 E！
```

#### Binding SID（绑定段 / 粘连标签）的救赎

现代网络硬件芯片（如商用交换芯片）能够支持的**最大标签栈深度（MSD，Maximum SID Depth）**是有限的（通常为 4~8 层）。如果一条跨国复杂链路需要连续压入 10 个 Adj-SID，硬件芯片将无法处理。
- **Binding SID 的机制**：在中间的粘连节点（Stitching Node）上创建一条子隧道，并为其分配一个本地/全局唯一的 **Binding SID（如 9000）**；
- 9000 这个标签在逻辑上**直接代表了后续 5 个标签的有序集合**；
- 源端路由器只需要在栈底压入一个 9000，报文到达粘连节点后，粘连节点剥离 9000 并将后续展开的标签栈动态推入！**以单标签折叠展开，完美解决标签栈深度过大的硬件硬伤**。

---

### 2.5 跨域流量工程：BGP EPE（Egress Peer Engineering）

在跨自治系统（跨 AS）的场景中，域内 IGP 无法穿透外部网络，无法获取对端 AS 的链路 SID。**BGP EPE（RFC 9086）** 扩展了 BGP 协议，使得集中控制器能够分配并收集 ASBR 之间的对等体标签：

```
      [AS 100]                                     [AS 200]
     ┌─────────┐                                 ┌─────────┐
     │  ASBR1  ├─── Peer-Adj SID 1001 ──────────>│  ASBR2  │
     │         │                                 │         │
     │         ├─── Peer-Adj SID 1002 ──────────>│  ASBR3  │
     └─────────┘<── Peer-Node SID 2001 (ASBR2) ──┴─────────┘
                <── Peer-Set SID  3001 (负载组)  ───
```

1. **Peer-Node SID**：用于指示出向流量发送至特定的对端 BGP 邻居路由器（无论经过哪条物理链路）；
2. **Peer-Adj SID**：用于精确指定出向流量从哪一条具体的物理互联链路上送达对端；
3. **Peer-Set SID**：将连接到相同或不同对端 ASBR 的多条链路编成一个集合，报文压入该 SID 时，由本地 ASBR 自动在集合内执行 **ECMP 负载分担**！

---

### 2.6 SR Policy 架构与基于 Color 的自动化引流

在传统网络中，流量引入隧道依赖手写复杂的静态路由（`ip route-static`）或策略路由（PBR），运维极其脆弱。新一代架构全面废弃 Tunnel 虚接口，转向 **SR Policy**。

#### SR Policy 三元组模型

每个 SR Policy 在全网由不可分割的**三元组**唯一确定：
$$\text{SR Policy} = \langle \text{Headend（头端）},\; \text{Color（业务色彩）},\; \text{Endpoint（目的端）} \rangle$$

- **Headend**：SR Policy 生效的源节点 IP；
- **Color（32位扩展团体属性）**：代表业务 SLA 意图的标记（如：Color 100 = 金牌超低时延通道，Color 200 = 银牌大带宽通道）；
- **Endpoint**：业务流量的目的端 IP。

```
[业务接入 (VPN 实例)] ──> BGP 路由携带 Extended Community: Color 100
                               │ (自动匹配)
                               ▼
     [SR Policy]: <Headend=PE1, Color=100, Endpoint=PE2>
                               │ (查得段列表)
                               ▼
        标签栈: [Node-B, Adj-25, Node-PE2] ──> 极速自动化引流！
```

网络管理员或控制器只需在发布 BGP 业务路由时打上对应的 **Color 团体属性**，头端路由器捕获后，自动在本地路由转发表中将下一跳迭代至拥有相同 Color 和 Endpoint 的 SR Policy 段列表中，**实现全流程零手工干预的秒级业务自动引流**！

---

### 2.7 保护倒换演进史：从 LFA、RLFA 到 TI-LFA

网络链路故障时，传统的 IGP 重新收敛需要经历“故障检测 $\rightarrow$ 泛洪 LSA $\rightarrow$ SPF 重算 $\rightarrow$ 下发 FIB”，耗时通常在数百毫秒到秒级，造成语音通话卡顿或金融交易丢包。必须依赖**快速重路由（FRR）**实现硬件级瞬时倒换。

```
                       [保护倒换算法演进路线]
           LFA (Loop-Free Alternate)
               │ (三角不等式严苛, 保护覆盖率仅 50%~70%)
               ▼
           RLFA (Remote LFA)
               │ (引入 P/Q 空间, 需借助 PQ 节点建立 LDP 隧道, 覆盖率 80%~90%)
               ▼
           TI-LFA (Topology-Independent LFA)
                 (无视拓扑形状, 纯靠 SR 修复列表, 100% 任意拓扑覆盖, <50ms 切换！)
```

#### 1. 传统 LFA 的局限性
LFA 核心条件（无环路径不等式）：
$$\text{Cost}(N, D) < \text{Cost}(N, S) + \text{Cost}(S, D)$$
即邻居节点 $N$ 到达目的端 $D$ 的开销，必须严格小于 $N$ 倒灌回源端 $S$ 再到 $D$ 的开销。在常见的环形网、扁平网或不对称链路中，该不等式极易失效，导致**有备用路径却无法使用**。

#### 2. RLFA 与 P/Q/PQ 空间定义
为了突破直连邻居的限制，RLFA 提出了深度的空间推导：
- **P 空间（P-Space）**：以源节点 $S$ 为根，**在不经过故障链路的前提下**，通过最短路径能够到达的所有节点集合；
- **扩展 P 空间（Extended P-Space）**：以源节点 $S$ 的所有直连直连邻居为根建立 SPF 树，不经过故障链路可达的节点合集（华为默认计算扩展 P 空间）；
- **Q 空间（Q-Space）**：以目的节点 $D$ 为根，建立反向最短路径树，**所有到达 $D$ 的路径中不经过故障链路**的节点集合；
- **PQ 节点**：**P 空间与 Q 空间的交集节点（$\text{PQ} = P \cap Q$）**。

【出门靠朋友经典类比】：源节点 $S$ 就像你遇到了困难，P 空间是你的直连朋友能帮你送到的安全区域；Q 空间是本身距离终点很近、绝不会走回头路的安全区域。PQ 节点就是既能被朋友安全送达、又能自主无环直达终点的“逃出生天关键庇护所”。RLFA 通过向 PQ 节点建立一条 LDP/GRE 隧道实现流量绕行。但当 **$P \cap Q = \emptyset$（无交集）**时，RLFA 彻底失效。

#### 3. TI-LFA（拓扑无关无环重路由）的终极解法

**TI-LFA** 是 Segment Routing 最震撼的技术杀手锏。它针对网络中任何一条链路故障，无论物理拓扑多么畸形：
1. **预先计算故障后收敛路径**：基于收敛后的拓扑计算出备份 SPF 路径；
2. **段列表修复路径（Repair List）**：若收敛路径上存在微环风险，TI-LFA 利用 SR 的源路由能力，**精准计算出一组由 Node-SID 或 Adj-SID 构成的 Repair List（修复标签栈）**；
3. **硬件级备用表项下发**：平时在芯片 FIB 中预先下发为主路径的备份下一跳；
4. **$<50\text{ ms}$ 极速倒换**：链路一旦震荡被物理光口或 BFD 察觉，**硬件转发芯片微秒级直接将流量打入 Repair List 备份表项**，彻底实现 **100% 任意网络拓扑全覆盖保护**！

---

#### 4. TI-LFA 的两大核心工程特质：去中心化算路与防微环（Micro-loop）机制
- **完全去中心化算路**：TI-LFA 的备份路径与 Repair List **无需 SDN 控制器参与**，完全由本地路由器在控制平面依据 IGP LSDB 预先自主算好并下发到 FIB。这赋予了全网在控制器失联或离线场景下的极致生存能力；
- **防微环（Micro-loop Avoidance）时延回切**：
  - **微环形成根源**：主路径故障后，源节点 $ 毫秒级切入备份路径发给下一跳 $；但若 $ 尚未完成 IGP 收敛，仍认为最短路径要经过 $，便会将报文重新丢回 $，形成  \leftrightarrow N$ 之间的临时微环路；
  - **SR 破环机制**：TI-LFA 在 Repair List 中压入显式 Node/Adj-SID，强制流量绕过未收敛的节点；同时配合**防微环延时定时器（Delayed Convergence Timer）**，等待下游节点全部完成收敛后再解除标签约束，实现平滑无缝回切。

### 2.8 排错排查与典型诊断命令

```shell
# 1. 查看本地 SR 全局标签块 (SRGB) 规划与冲突检查
display segment-routing mpls srgb

# 2. 查看 IS-IS / OSPF 分发的 Prefix-SID 及转发表项
display isis segment-routing prefix-sid
display ospf routing segment-routing-prefix

# 3. 查看本地动态分配的邻接标签 (Adjacency-SID)
display isis segment-routing adjacency-sid
# 确认出接口与下一跳 IP，状态是否为 Active

# 4. 查看针对特定前缀计算的 TI-LFA 备份路径与 Repair List
display isis ti-lfa 1.1.1.1 32 verbose
# 重点关注：Repair-List 字段是否包含了显式保护段标签，保护覆盖率是否为 100%

# 5. 查看本地激活的 SR Policy 运行状态与候选路径
display segment-routing policy state
# 验证：Headend、Color、Endpoint 三元组，以及绑定的 Segment-List 标签栈深度
```

---

## 三、总结十带走

1. **源路由本质**：源头定义路径，中间节点无状态，彻底终结网络逐跳转发与全网状态膨胀的旧时代；
2. **协议极致简化**：废除 LDP 与 RSVP-TE 臃肿信令，底层单一 IGP（OSPF/IS-IS）扩展即兼顾算路与标签分发；
3. **Prefix-SID 逻辑**：全局唯一，代表目的网段，沿 IGP 最短路径转发并具备天然 ECMP 负载能力；
4. **Adjacency-SID 逻辑**：本地有效，代表特定链路与下一跳，强制指路，不经过任何 SPF 最短路径计算；
5. **SRGB 标签换算**：绝对标签 = 本地 SRGB 基值 + Prefix-SID 索引；解耦异构设备，支持跨厂商混跑；
6. **SR-BE vs SR-TE**：SR-BE 单标签走最短路（替代 LDP）；SR-TE 多层标签栈按 SLA 约束精准选路（替代 RSVP-TE）；
7. **Binding SID 折叠神技**：以单个标签代表一整条子路径或域内隧道，击碎芯片标签栈深度（MSD）物理瓶颈；
8. **BGP EPE 跨域工程**：利用 Peer-Node/Adj/Set SID 掌控 ASBR 出向选路，实现骨干网跨 AS 端到端流量编排；
9. **SR Policy 现代引流**：抛弃繁复脆弱的 Tunnel 虚接口，依据 `<Headend, Color, Endpoint>` 与 BGP Color 属性自动纳管流量；
10. **TI-LFA 终极护航**：打破 LFA/RLFA 拓扑限制，纯靠 SR 修复段列表实现 **100% 任意拓扑全覆盖保护**与 **$<50\text{ ms}$** 硬件无损倒换。

---

## 附：本章术语速查表

| 术语（缩写） | 英文全称 | 中文标准名称 | 核心作用与定义 |
|:---|:---|:---|:---|
| **SR** | Segment Routing | 分段路由 | 基于源路由理念的新一代无状态数据包转发网络架构 |
| **SID** | Segment Identifier | 段标识符 | 指导网络节点执行特定转发行为的指令，在 SR-MPLS 中映射为 MPLS 标签 |
| **SRGB** | Segment Routing Global Block | 全局段路由标签块 | 用户为 SR 全局段（如 Prefix/Node-SID）预留的专用全局标签范围 |
| **Prefix-SID** | Prefix Segment Identifier | 前缀段标识符 | 全局唯一标识一个 IP 前缀的段，指导报文沿 IGP 最短路径转发 |
| **Node-SID** | Node Segment Identifier | 节点段标识符 | 特殊的前缀段，通常绑定在 Loopback 接口上唯一标识一台路由器设备 |
| **Adj-SID** | Adjacency Segment Identifier | 邻接段标识符 | 本地有效，用于精确指示从某个物理接口向特定邻居转发的单跳段 |
| **Binding SID** | Binding Segment Identifier | 绑定段标识符 | 绑定一条特定 SR-TE 路径或隧道的本地段，用于折叠标签栈与跨域拼接 |
| **SR-BE** | Segment Routing Best-Effort | 尽力而为分段路由 | 基于 IGP 最短路径计算单段转发的模式，直接替代传统 LDP 隧道 |
| **SR-TE** | Segment Routing Traffic Engineering | 流量工程分段路由 | 基于多段有序标签栈编排显式约束路径（带宽/时延）的高级流量工程 |
| **EPE** | Egress Peer Engineering | 出向对等体工程 | BGP 扩展机制，用于为跨 AS 对等体分配 Peer-Node/Adj/Set SID 实现跨域选路 |
| **TI-LFA** | Topology-Independent LFA | 拓扑无关无环重路由 | 利用 SR 显式修复段列表提供 100% 任意物理拓扑覆盖的 50ms 极速保护算法 |
| **MSD** | Maximum SID Depth | 最大段深度 | 硬件转发芯片在数据包中一次性能够推入或处理的最大标签栈层数 |
