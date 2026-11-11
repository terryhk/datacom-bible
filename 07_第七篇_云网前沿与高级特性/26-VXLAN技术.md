# 第26章 VXLAN技术与数据中心Overlay

> 突破传统二层桎梏的云网互联底座：VXLAN 从数据中心服务器虚拟化的大二层诉求与跨三层热迁移出发，深度拆解“MAC-in-UDP”的封装机制（UDP 4789 与 24 位 VNI）；系统剖析 Underlay 与 Overlay 架构解耦、NVE 接口、VTEP 隧道端点与 BD（Bridge Domain）虚拟交换机模型；完整呈现静态头端复制（Head-end Replication）同子网已知单播与 BUM 泛洪转发流程；对比集中式网关与分布式网关模型；最终手搓落地跨租户 VPN 隔离、虚拟防火墙引流与多业务融合实战。

| 项 | 内容 |
|---|---|
| **适用层级** | HCIE 核心云网前沿特性（实验必备、面试连环追问、数据中心与虚拟化园区网基石） |
| **建议学习位次** | 全书第 26 章，先修第 05 章（OSPF 底层路由）、第 11 章（VLAN 与子接口）、第 13 章（链路聚合）、第 17 章（VRF 虚拟路由转发）；学完本章后建议后修第 27 章（EVPN 动态控制平面） |
| **技术维度** | 协议原理深度剖析 · 生产级VRP配置 · 故障诊断排错 · 架构设计决策 |

---

## 一、总起

随着云计算与数据中心服务器虚拟化（VMware ESXi、KVM、容器）的普及，业务架构发生了革命性变革。传统的“接入-汇聚-核心”二层网络面临着三大无法逾越的物理瓶颈：
1. **虚拟机动态迁移（vMotion）受限于二层边界**：虚拟机迁移时必须保持 IP 地址和 TCP 会话不中断，传统二层网络跨三层网关即失效，无法实现跨机房、跨数据中心跨三层迁移；
2. **STP 破环机制阻断链路资源**：传统生成树协议（STP/RSTP/MSTP）为了防环必须阻塞冗余端口，带宽利用率仅有 50%，且在大规模拓扑中收敛极慢、极易震荡震塌全网；
3. **VLAN 4096 规格上限无法满足多租户隔离**：IEEE 802.1Q 的 VLAN ID 仅占 12 位，最多 4094 个可用 VLAN，在公有云及大型私有云成千上万个租户面前捉襟见肘。

**VXLAN（Virtual Extensible Local Area Network，虚拟扩展局域网）**应运而生。它采用 **MAC-in-UDP** 的报文封装技术，将二层以太网帧封装在标准的 IPv4/IPv6 UDP 报文中，在现有的三层 IP 网络（Underlay）之上构建起无视物理拓扑限制的逻辑大二层网络（Overlay）。

本章遵循从“单点原理到多业务综合手搓实战”的完整逻辑链：
1. **技术本源与封装体系**：拆解大二层通信核心、VXLAN 报文格式（外层 MAC、外层 IP、UDP 4789、VXLAN 报头 24 位 VNI）、BD 本地桥接域与 NVE 逻辑接口定义；
2. **同子网转发与静态隧道**：详解三类子接口模式（Dot1q/Untag/Default）的 VLAN Tag 剥离与添加动作；手写 NVE 静态头端复制（Head-end Replication）列表；推演 BUM 报文泛洪抑制与二层 MAC 表动态学习转发；
3. **网关架构与跨子网通信**：对比**集中式网关**（Spine 作为集中网关，配置直观，但流量次优绕行，类似武汉物流集散中心）与**分布式网关**（Leaf 本地作为 VBDIF 网关，Symmetric IRB 本地直转发）；
4. **多租户与多业务手搓实战**：脱离 SDN 控制器，纯手工在华为 CloudEngine（CE12800）交换机上完成 Underlay OSPF 路由打通、VXLAN 业务映射、跨 VPN 实例隔离与路由渗透、结合虚拟防火墙（VSystem）实现公网引流与跨租户安全防护。

**适用人群**：
- **初学 / 转行人员**：摆脱“大二层就是网络扁平化”的误区，建立“IP 可达之地，二层即刻可达”的 Overlay 空间想象力；
- **备考工程师（HCIP/HCIE）**：彻底吃透 VXLAN 报文结构 50 字节开销与 MTU 计算、BD/VNI/VBDIF/NVE 绑定关系、BUM 泛洪流程，以及 CE12800 交换机候选配置库（commit 机制）；
- **一线工程实践者**：掌握数据中心与中大型虚拟化园区网在无控制器环境下的手工开局模板、跨 VRF 引流策略设计，以及防范隧道丢包与 MTU 超标排错技能。

### 本章技术架构与核心脉络

| 模块序号 | 核心技术议题 | 架构要点与设计精髓 |
|:---|:---|:---|
| **§26.1** | Vxlan技术1 | VXLAN背景诉求、静态手动 vs EVPN动态、CE12800候选配置库、BD域与VNI映射、三种子接口封装（Dot1q/Untag/Default）、同网段二层转发与BUM头端复制 |
| **§26.2** | Vxlan技术2 | 干道(Trunk)+VLAN虚接口组网、同子网互访、大二层定义与类比、集中式网关选型(Spine)、跨子网VBDIF网关转发、融合组网与多租户VPN隔离 |
| **§26.1** | Vxlan技术3 | 集中式网关综合拓扑实战、Underlay OSPF打通、无线AC/AP三层组网融入VXLAN、跨VPN实例DHCP中继配置与避坑 |
| **§26.2** | Vxlan技术4 | 核心与物理防火墙Eth-Trunk对接、多租户虚墙(VSystem A/B)与根墙(Public)架构、引流表直甩技术、公网DNS/Web访问、双向NAT与跨租户互访 |

---

## 二、分述

### 2.1 VXLAN 核心价值与“大二层”本质

#### 传统网络困境与大二层真谛

在传统三层分层网络架构中，每个二层广播域都被限制在汇聚或接入交换机之下。当虚拟化技术允许一台物理服务器运行数十甚至数百台虚拟机时，两大矛盾爆发：
1. **集群计算资源调度（虚机热迁移）**：业务高峰期，VM 需从物理机 Server-A 动态漂移到 Server-B。为了保证已建立的 TCP 链接不中断，VM 的 IP 地址和 MAC 地址必须保持不变，这意味着 Server-A 与 Server-B 必须同处于一个二层广播域中；
2. **机房物理跨度大**：现代数据中心分布在不同机柜、不同排乃至不同物理机房，中间经过了大量三层路由器和三层交换机。传统二层网络无法跨越三层路由边界。

【设计本源与技术哲学】：“VXLAN 的‘大二层’指的是——**IP 可达的地方二层网络就可达**！不管主机在哪台设备上接入，都可以放到同一个广播域中。中间 IP 网络有多大、跨了多少台路由器都没关系，都能在逻辑上把这些主机放到一个广播域中。这跟你的物理网络扁不扁平没有任何关系，‘大二层’是专门用来形容 VXLAN 这类 Overlay 隧道技术的。”

#### 报文封装架构与 50 字节开销

VXLAN 采用标准的三层封装二层（L2 over L4 / MAC-in-UDP）技术：

```
+---------------------------------------------------------------------------------------+
|  外层以太帧头  | 外层 IP 报头 | 外层 UDP 报头 | VXLAN 报头 |        原始二层以太网帧         |
|   (14 字节)   |  (20 字节)  |   (8 字节)   |  (8 字节)  | (DMAC+SMAC+Type+Payload+FCS)  |
+---------------------------------------------------------------------------------------+
|<--------------------------- 外部封装总开销: 50 字节 ------------------------->|
```

- **外层以太头（Outer Ethernet Header）**：14 字节，源/目 MAC 地址为 Underlay 下一跳设备的物理 MAC 地址；
- **外层 IP 报头（Outer IP Header）**：20 字节，源 IP 为源端 VTEP 地址，目的 IP 为对端 VTEP 地址；协议号为 `17`（UDP）；
- **外层 UDP 报头（Outer UDP Header）**：8 字节，**目的端口固定为 `4789`**（IANA 规定）；源端口由芯片根据原始内层报文的特征（五元组哈希）计算得出，取值范围为临时端口（49152~65535），**其核心价值是允许 Underlay 网络在三层等价多路径（ECMP）时实现平滑的负载均衡**；
- **VXLAN 报头（VXLAN Header）**：8 字节：
  - Flags（8 位）：其中第 3 位为 `I` 标志位（Instance Flag），必须置为 `1`，表示 VNI 字段有效；其余 7 位保留为 0；
  - **VNI（VXLAN Network Identifier）**：**24 位（3 字节）**，取值范围 $1 \sim 2^{24}-1$（超过 1600 万个），从根本上解决了 802.1Q 只有 4094 个 VLAN 的规模上限；
  - 24 位保留字段。

> [!WARNING] MTU 炸弹预警与工程调优
> 由于引入了 50 字节的外部封装（14 字节以太头 + 20 字节 IP 头 + 8 字节 UDP 头 + 8 字节 VXLAN 头），若终端发送标准 1500 字节的 IP 报文，封装后总长达到 1550 字节。如果 Underlay 交换机物理端口 MTU 仍然为默认的 1500，报文将因为无法通过而被静默丢弃（若设置了 DF=1 不分片）或产生大量分片降低性能。
> **一线工程铁律**：部署 VXLAN 时，**必须将 Underlay 网络沿途所有物理交换机端口的 MTU 调大至 1600 以上（推荐配置 Jumbo Frame 9216）**！

---

### 2.2 核心术语与逻辑架构模型

VXLAN 构建在物理网络（Underlay）与逻辑网络（Overlay）的解耦基础之上：

```
             +-----------------------------------------+
             |            Overlay 逻辑网络              |
             |   +--------+               +--------+   |
             |   | PC1/VM |<==== VXLAN ===| PC2/VM |   |
             |   +---+----+    Tunnel     +---+----+   |
             +-------|------------------------|--------+
                     |                        |
         ............|........................|............
         :  +--------+--------+      +--------+--------+  :
         :  |  Leaf1 (VTEP1)  |      |  Leaf2 (VTEP2)  |  :
         :  +--------+--------+      +--------+--------+  :
         :           \                      /             :
         :            \    Underlay IP     /              :
         :             \     Network      /               :
         :            +------------------+                :
         :            |      Spine       |                :
         :            +------------------+                :
         :................................................:
```

1. **Underlay 网络**：底层的物理 IP 承载网。职责极度纯粹——通过 IGP（OSPF 或 IS-IS）保证各设备 Loopback 接口之间三层 IP 高速全连通，无需关注任何租户业务；
2. **Overlay 网络**：运行在 Underlay 之上的虚拟逻辑网络。业务报文在网络边缘被封装为 VXLAN 报文，在 Underlay 中以普通 UDP 报文透明穿行，在目的地解封装还原；
3. **VTEP（VXLAN Tunnel End Point，隧道端点）**：VXLAN 隧道发起和终结的节点，负责 VXLAN 报文的封装与解封装。VTEP 具有唯一的 IP 地址（通常使用稳定可靠的 Loopback 地址）；
4. **NVE（Network Virtualization Edge，网络虚拟化边缘接口）**：VTEP 上创建的逻辑虚拟接口（`interface Nve 1`），用于挂接 VXLAN 隧道与配置源端 IP；
5. **BD（Bridge Domain，桥接域）**：
   - 设备本地有效的一个**二层广播域**，取值范围 $1 \sim 16777215$；
   - 传统交换机用 VLAN 隔离广播域，而在 VXLAN 体系中，设备内部采用 **BD** 隔离广播域；
   - 一个 BD 映射一个全局唯一的 **VNI**，从而将本地广播域延伸至全网；
6. **VBDIF 接口（Virtual BD Interface）**：基于 BD 创建的三层虚拟网关接口（类比传统 VLAN 的 VLANIF 接口），用于实现不同 BD 间或 VXLAN 与外部 IP 网络的跨子网三层路由。

---

### 2.3 业务接入与子接口 Tag 处理机制

终端流量如何接入到 VXLAN BD 域中？华为 CE 系列交换机通过在二层物理端口上创建**二层子接口（Sub-interface）**并绑定 BD 实现：

```
物理接口 10GE 1/0/1
  ├── 10GE 1/0/1.10 mode l2 (绑定 BD 10) ── 封装 dot1q vid 10
  ├── 10GE 1/0/1.20 mode l2 (绑定 BD 20) ── 封装 dot1q vid 20
  └── 10GE 1/0/1.100 mode l2 (绑定 BD 100) ── 封装 untag
```

子接口定义了三种核心封装模式，其针对 VLAN Tag 的处理逻辑严谨明晰：

| 封装类型 | 配置命令 | 匹配入向报文特征 | 进入 BD 域动作（入向） | 离开 BD 域动作（出向） | 典型应用场景 |
|:---|:---|:---|:---|:---|:---|
| **Dot1q** | `encapsulation dot1q vid <id>` | 带指定 VLAN Tag 的数据帧 | **剥离（Strip）**该 VLAN Tag，以内层纯原始帧进入 BD | **压入（Push）**该子接口配置的 VLAN Tag 后发出 | 下游对接 Trunk 干道或 Access 交换机（最常用） |
| **Untag** | `encapsulation untag` | 不带任何 VLAN Tag 的纯净数据帧 | 直接进入 BD，不做 Tag 修改 | 直接剥除可能存在的 Tag，发出纯净无 Tag 帧 | 接口直连物理服务器、PC 或终端主机 |
| **Default** | `encapsulation default` | 匹配接口上所有其他子接口未匹配的报文 | **不剥除**原始 Tag，透传（含 Tag）进入 BD | 原样透传保留原始 Tag 发出 | 透传全量 VLAN 或做二层透传专线 |

【核心设计精髓】：**BD 内部永远只流淌纯净的二层数据帧（无本地 VLAN Tag）**。无论外部接入采用 VLAN 10 还是 VLAN 20，只要映射进入同一个 BD，在 BD 域内就处于同一逻辑广播域；出接口时，由出方向子接口根据自己的定义重新压入对应物理链路所需的 Tag。这种解耦机制使得两端站点无需强制规划相同的物理 VLAN ID，只要 VNI 相同即可互通！

---

### 2.4 数据平面转发机制推演

静态 VXLAN（无 EVPN 控制面）依赖传统以太网的“**数据面泛洪学习**”与“**静态头端复制（Head-end Replication）**”实现通信。

#### 场景一：同子网首次通信（BUM 报文处理与 MAC 学习）

设 Host-A（192.168.10.10, MAC-A）欲访问同网段 Host-B（192.168.10.20, MAC-B），此时 Host-A 发送 ARP 请求广播报文：

```
[Host-A] ──(ARP Req)──> [Leaf-1] ═══════(VXLAN Head-end BUM)═══════> [Leaf-2] ──(ARP Req)──> [Host-B]
  发送广播                 1. 学习 MAC-A 对应入子接口                  1. 解封装，学习 MAC-A 对应 VTEP-1
                         2. 查 BD 10 对应 VNI 10                      2. 本地 BD 10 泛洪广播出接口
                         3. 复制两份报文分别单播送达对端 VTEP
```

1. **源端接入与学习**：
   - Host-A 发出 ARP 请求（广播，DMAC=全 F）；
   - Leaf-1 子接口收到，根据 `encapsulation dot1q vid 10` 剥去 Tag，将原始帧送入 BD 10；
   - **本地 MAC 学习**：Leaf-1 在 BD 10 的 MAC 表中记录条目：`MAC-A -> 10GE 1/0/1.10`；
2. **头端复制（Head-end Replication）**：
   - 报文是广播报文（属于 BUM 范畴：Broadcast, Unknown-unicast, Multicast）；
   - Leaf-1 查找 NVE 接口下配置的 `vni 10 head-end peer-list`；
   - Leaf-1 为列表中的每一个远端 VTEP 单独复制一份报文，封装为单播 UDP 报文（源 IP=VTEP-1, 目的 IP=VTEP-2，UDP 4789，VNI=10），通过 Underlay IP 路由送出；
3. **对端解封装与学习**：
   - Leaf-2 收到 UDP 报文，发现目的端口为 4789，剥离外层 IP/UDP/VXLAN 头；
   - 提取 VNI=10，映射到本地 BD 10；
   - **远端 MAC 学习**：Leaf-2 在 BD 10 的 MAC 表中学习到：`MAC-A -> VTEP-1 (2.2.2.2)`；
   - Leaf-2 将内层原始 ARP 广播帧在本地 BD 10 的所有本地成员接口（除接收隧道外）进行广播；
   - Host-B 收到 ARP 请求。

#### 场景二：已知单播转发（双向已学表）

Host-B 回应 ARP Reply（单播）：
1. Leaf-2 收到单播帧（SMAC=MAC-B, DMAC=MAC-A）；
2. Leaf-2 在 BD 10 学习到本地条目：`MAC-B -> 10GE 1/0/1.10`；
3. Leaf-2 查找目的 MAC-A，命中此前学到的远端表项：`MAC-A -> VTEP-1 (2.2.2.2)`；
4. Leaf-2 直接进行精准的单播 VXLAN 封装（源 IP=VTEP-2, 目的 IP=VTEP-1, VNI=10），经 Underlay 路由送回 Leaf-1；
5. Leaf-1 解封装，查本地 MAC 表将单播送达 Host-A。后续两台主机之间所有通信均为**精准已知单播**。

---

### 2.5 集中式网关 vs 分布式网关

当需要跨子网（不同 VNI）互通或访问外部网络时，必须由三层网关执行路由转发：

```
           集中式网关 (Centralized)                     分布式网关 (Distributed)
               +-------------+                              +-------------+
               | Spine(网关) |                              |    Spine    | (纯Underlay转发)
               +-------------+                              +-------------+
                 ^         |                                    /         \
          VXLAN1 |         | VXLAN2                      VXLAN /           \ VXLAN
                 |         v                              隧道/             \ 隧道
           +-------+     +-------+                    +-------+         +-------+
           | Leaf1 |     | Leaf2 |                    | Leaf1 |         | Leaf2 |
           +-------+     +-------+                    |(含网关)|        |(含网关)|
              |             |                         +-------+         +-------+
            PC-1          PC-2                           |                 |
         (192.168.10.x) (192.168.20.x)                 PC-1              PC-2
```

| 维度 | 集中式网关（Centralized Gateway） | 分布式网关（Distributed Gateway） |
|:---|:---|:---|
| **网关部署位置** | 核心交换机（Spine）统一担任所有子网的 VBDIF 网关 | 每台接入交换机（Leaf）均担任本地接入子网的网关 |
| **同 Leaf 跨子网通信** | 必须封装 VXLAN 上送 Spine，由 Spine 路由后再封装下发（**次优绕行，折返跑**） | 本地 Leaf 的 VBDIF 直接在芯片内部完成三层路由，**零网络绕行** |
| **异 Leaf 跨子网通信** | 流量：Leaf1 $\rightarrow$ Spine $\rightarrow$ Leaf2（经过两次 VXLAN 解封装/封装） | 流量：Leaf1 本地直接路由并封装 VXLAN 直达 Leaf2（**一次封装一次转发**） |
| **网关 ARP 规格压力** | 全网所有主机的 ARP 表项均集中在 Spine 上，**容易击穿 Spine 硬件表项瓶颈** | 每台 Leaf 仅维护本地直连主机的 ARP，表项负载极度分散 |
| **典型应用场景** | 中小型园区网、传统数据中心向 SDN 渐进演进 | 超大型云数据中心、多租户海量虚机场景（通常配合 BGP EVPN） |

【形象工程类比】：“集中式网关就像中国邮政把全国所有的快递全部拉到武汉总集散中心分拣，哪怕是同一个小区的两个邻居寄件，也得先运到武汉再发回来；而分布式网关就像每个小区门口设立了智能自提柜与分拣站，邻居之间互发快递直接在小区门口柜子完成交接，跨小区的包裹直接点对点发往目标小区，效率天差地别！”

---

### 2.6 集中式网关综合实战（手搓配置与全网贯通）

以下为在无控制器环境下，使用华为 CE12800 交换机手动完成集中式网关组网的标准生产级配置模板：

```
[Leaf-1 / CE2]                                      [Spine / CE1 (集中式网关)]
 Loopback0: 2.2.2.2/32                               Loopback0: 1.1.1.1/32
 VBDIF 10/20 网关均部署在 Spine
      │                                                   │
      └────── Eth-Trunk 1 (VLAN 12 Trunk) ───────────────┘
```

#### 步骤 1：Underlay IP 网络打通（OSPF）

```shell
# ==================== [Spine / CE1] ====================
sysname CE1
# 开启 CE 候选配置自动提交（或手工 commit）
# 创建互联 VLAN 并配置 IP
vlan batch 12 13
interface Vlanif12
 ip address 10.1.12.1 255.255.255.0
#
interface LoopBack0
 ip address 1.1.1.1 255.255.255.255
#
interface 10GE1/0/1
 port link-type trunk
 port trunk allow-pass vlan 12
#
ospf 1 router-id 1.1.1.1
 area 0.0.0.0
  network 1.1.1.1 0.0.0.0
  network 10.1.12.0 0.0.0.255
commit

# ==================== [Leaf-1 / CE2] ====================
sysname CE2
vlan batch 12
interface Vlanif12
 ip address 10.1.12.2 255.255.255.0
#
interface LoopBack0
 ip address 2.2.2.2 255.255.255.255
#
interface 10GE1/0/1
 port link-type trunk
 port trunk allow-pass vlan 12
#
ospf 1 router-id 2.2.2.2
 area 0.0.0.0
  network 2.2.2.2 0.0.0.0
  network 10.1.12.0 0.0.0.255
commit
```

#### 步骤 2：业务接入与 BD/VNI 映射

```shell
# ==================== [Leaf-1 / CE2] 业务接入 ====================
# 1. 创建本地桥接域 BD 并绑定全局 VNI
bridge-domain 10
 vxlan vni 10
bridge-domain 20
 vxlan vni 20
#
# 2. 在二层物理接口创建二层子接口接入业务并剥离 Tag
interface 10GE1/0/2.10 mode l2
 encapsulation dot1q vid 10
 bridge-domain 10
#
interface 10GE1/0/2.20 mode l2
 encapsulation dot1q vid 20
 bridge-domain 20
commit
```

#### 步骤 3：建立 NVE 接口与静态头端复制隧道

```shell
# ==================== [Leaf-1 / CE2] NVE 配置 ====================
interface Nve1
 source 2.2.2.2
 # 静态指定集中式网关 Spine(1.1.1.1) 作为头端复制对端
 vni 10 head-end peer-list 1.1.1.1
 vni 20 head-end peer-list 1.1.1.1
commit

# ==================== [Spine / CE1] 集中式网关三层配置 ====================
# 1. 同样创建 BD 与 VNI 映射
bridge-domain 10
 vxlan vni 10
bridge-domain 20
 vxlan vni 20
#
# 2. 配置三层 VBDIF 接口作为租户主机的默认网关
interface Vbdif10
 ip address 192.168.10.1 255.255.255.0
#
interface Vbdif20
 ip address 192.168.20.1 255.255.255.0
#
# 3. 创建 NVE 接口，指定 Leaf-1(2.2.2.2) 与 Leaf-2(3.3.3.3) 为对端
interface Nve1
 source 1.1.1.1
 vni 10 head-end peer-list 2.2.2.2
 vni 20 head-end peer-list 3.3.3.3
commit
```

---

### 2.7 虚拟化园区多租户隔离与安全引流实战

在真实数据中心与大型虚拟化园区网中，不同租户之间需要强制安全隔离，并通过集中物理防火墙的虚拟防火墙（VSystem）实现受控互通与互联网出口引流：

```
                      [物理防火墙 (USG)]
                 ┌────────────────────────┐
                 │ 根墙 (Public VSystem)   │ ===> Internet (出口路由器 R1)
                 │   ▲引流表      ▲引流表  │
                 │   │           │        │
                 │ [虚墙 A]    [虚墙 B]   │
                 └─────┬───────────┬──────┘
       VLAN 100 Trunk  │           │  VLAN 200 Trunk
                 ┌─────┴───────────┴──────┐
                 │ 集中式核心网关 (CE1)    │
                 │ VPN-A          VPN-B   │
                 └─────┬───────────┬──────┘
                       │(VXLAN)    │(VXLAN)
                    租户 A       租户 B
```

#### 关键技术要点解析：

1. **核心交换机多租户隔离（VRF / VPN-Instance）**：
   - 在 CE1 上创建 `ip vpn-instance Tenant-A` 与 `Tenant-B`；
   - 将 VBDIF 10 绑定至 `Tenant-A`，VBDIF 20 绑定至 `Tenant-B`，实现两租户在核心设备上的路由表彻底物理级切片隔离；
2. **三层互联对接（跨域 Option A 思想）**：
   - 核心与防火墙配置 Eth-Trunk 干道（LACP）；
   - 使用不同的 VLAN（如 VLAN 100 对应租户 A，VLAN 200 对应租户 B，VLAN 250 对应根墙 Public）在二层干道上透明隔离互联；
   - 交换机侧 `interface Vlanif100` 绑定 `Tenant-A`，`interface Vlanif200` 绑定 `Tenant-B`；
3. **防火墙引流表直甩技术（Traffic Diversion）**：
   - 传统方案：外网访问内网，根墙必须先查路由、匹配安全策略、做 NAT 转换后再丢给虚墙，不仅消耗根墙 CPU 资源，还会产生巨量并发会话表；
   - **引流表优化机制**：在根防火墙上配置地址范围引流（`vsys-direction`），当公网访问租户 A 的公网映射地址池时，**根墙不建安全会话、不写安全策略，直接甩给虚拟防火墙 A**，由虚墙 A 自主执行 NAT Server 映射与包过滤检查；
4. **跨 VPN 实例 No-NAT 与双向 NAT**：
   - 内网租户 A 主机使用公网域名访问内部服务器时，必须在虚拟防火墙上配置**域内双向 NAT**（源地址转换为防火墙接口地址，目的地址转换为服务器私网地址），防止服务器回包直甩绕过防火墙导致 TCP 握手失败。

---

### 2.8 排错排查与典型故障根因

在 VXLAN 运维与实验排障中，最常见以下五类经典故障：

```
[故障排查矩阵]
1. Underlay 不通  ──> 检查 OSPF/IS-IS 邻居状态、Loopback 路由可达性、物理链路 MTU
2. 隧道未建立     ──> display vxlan tunnel 验证对端 IP、UDP 4789 端口是否被中间 ACL 过滤
3. 无法学到远端MAC ──> display mac-address bridge-domain 验证 VNI 与 BD 映射、头端复制列表
4. 无法跨子网路由 ──> 检查 VBDIF IP 配置、arp 学习状态、VPN-Instance 绑定是否漏配
5. 丢包或性能骤降 ──> 检查全链路 MTU 是否设置 >= 1600，内层报文 DF 置位导致的分片丢包
```

#### 实用诊断命令全集

```shell
# 查看全网 VXLAN VNI 运行状态及映射 BD
display vxlan vni
# 回显重点关注：VNI ID、BD ID、State (UP/DOWN)

# 查看本地生成的 VXLAN 隧道列表
display vxlan tunnel
# 验证：Tunnel ID、Source IP、Destination IP、State (UP)

# 查看指定 BD 域内的 MAC 地址表
display mac-address bridge-domain 10
# 区分：Type 列是 "dynamic"（本地接口学到）还是 "vxlan"（远端 VTEP 学到）

# 查看指定 BD 域内的 ARP 表项
display arp bridge-domain 10

# 验证 CE 交换机候选配置提交状态
display configuration candidate
# 若有星号 "*" 说明存在未 commit 的修改，必须执行 commit 生效！
```

---

## 三、总结十带走

1. **大二层核心真谛**：IP 可达之处，二层网络即刻可达；无视底层三层路由器跨度，将全网终端纳入同一虚拟广播域；
2. **报文封装口诀**：MAC-in-UDP，外加 50 字节头（以太 14 + IP 20 + UDP 8 + VXLAN 8），UDP 目标端口固定为 **4789**；
3. **MTU 铁律**：引入 50 字节开销，物理承载网全链路 MTU 必须调大至 1600 以上（推荐 9216 Jumbo Frame），严防静默分片丢包；
4. **VNI 规模突破**：24 位 VNI 提供高达 1600 万个逻辑隔离网络，彻底终结传统 802.1Q 的 4094 个 VLAN 规格瓶颈；
5. **UDP 源端口妙用**：利用内层五元组哈希动态生成 49152~65535 的临时端口，使得 Underlay 网络无需感知 VXLAN 即可自动实现 ECMP 负载均衡；
6. **BD 域解耦价值**：BD 域是本地广播域，内部只跑纯净无 Tag 帧；通过不同接入子接口剥除/压入 Tag，两端接入站点物理 VLAN ID 可完全独立；
7. **三类接入子接口**：Dot1q 剥除指定 Tag 进入 BD；Untag 接收无 Tag 帧；Default 全量透传保留 Tag；
8. **网关架构对比**：集中式网关部署在 Spine，配置简单但流量次优折返跑、ARP 规格压力大；分布式网关部署在 Leaf，本地直路由，横向扩展能力无限；
9. **头端复制机制**：在无 EVPN 控制面时，BUM 报文依赖 Leaf 在本地复制为多份单播 UDP 报文送达远端对端列表；
10. **CE 交换机避坑**：华为 CloudEngine 交换机默认启用候选配置库机制（`~` 标识），敲完所有命令必须打 **`commit`** 才能真正写入芯片生效！

---

## 附：本章术语速查表

| 术语（缩写） | 英文全称 | 中文标准名称 | 核心作用与定义 |
|:---|:---|:---|:---|
| **VXLAN** | Virtual Extensible Local Area Network | 虚拟扩展局域网 | 基于 MAC-in-UDP 的大二层 Overlay 隧道技术，打破 4096 VLAN 限制 |
| **VTEP** | VXLAN Tunnel End Point | VXLAN 隧道端点 | 负责 VXLAN 报文封装与解封装的网络边缘设备或虚拟交换机节点 |
| **VNI** | VXLAN Network Identifier | VXLAN 网络标识符 | 24 位标识符（取值达 1600 万），用于区分不同的 Overlay 逻辑广播域 |
| **NVE** | Network Virtualization Edge | 网络虚拟化边缘接口 | VTEP 设备上用于建立和维护 VXLAN 隧道的逻辑虚拟接口（`Nve1`） |
| **BD** | Bridge Domain | 桥接域 | 交换机本地划分的一个二层广播域，与 VNI 进行 1:1 映射 |
| **VBDIF** | Virtual Bridge Domain Interface | 虚拟桥接域三层接口 | 基于 BD 创建的逻辑三层路由网关接口，实现跨子网及外网路由互通 |
| **Underlay** | Underlay Network | 底层承载网络 | 提供端到端三层 IP 连通性的物理网络（通常跑 OSPF/IS-IS/BGP） |
| **Overlay** | Overlay Network | 上层叠加网络 | 叠加在 Underlay 之上的虚拟逻辑网络，对上层业务呈现为大二层网络 |
| **BUM** | Broadcast, Unknown-unicast, Multicast | 广播、未知单播、组播 | 以太网中需要泛洪处理的三类数据帧，VXLAN 中采用头端复制或组播树分发 |
| **IRB** | Integrated Routing and Bridging | 综合路由与桥接 | 允许在同一交换节点同时处理二层桥接与三层路由的架构模型 |
| **VSystem** | Virtual System | 虚拟防火墙系统 | 物理防火墙内部切分的逻辑防火墙实例，用于多租户公网引流与独立安全策略 |
| **ECMP** | Equal-Cost Multi-Path | 等价多路径 | 路由层面的多链路负载分担，VXLAN 利用 UDP 源端口哈希天然匹配 ECMP |
