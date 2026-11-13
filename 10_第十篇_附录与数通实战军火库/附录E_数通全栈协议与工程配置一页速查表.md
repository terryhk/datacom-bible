# 附录E 数通全栈协议与工程配置一页速查表

> **【使用说明】**  
> 本速查表为面向数通工程师、网络架构师与 HCIE 考生的**一站式掌中宝（Cheat Sheet）**。  
> 汇聚全书 35 卷与 4 份核心附录中最易混淆、现网排错与面试最高频查阅的**底层报文结构、协议定时器、状态机跃迁、选路口诀与华为 VRP 黄金诊断指令**。建议随身存阅、打印置于案头，在实施交付、割接排障或技术答辩前快速拉齐记忆。

---

## 卡片一：物理层与二层交换高可用

### 1.1 以太网报文与 MTU / MSS 换算
- **标准以太网 II 帧结构**：`DMAC(6B) + SMAC(6B) + Type(2B) + Payload(46~1500B) + FCS(4B)`，整帧长度 $64 \sim 1518\text{ Bytes}$（带 802.1Q Tag 时为 $68 \sim 1522\text{ Bytes}$）；
- **MTU（最大传输单元）**：数据链路层 Payload 上限，默认 **$1500\text{ Bytes}$**；
- **MSS（TCP 最大报文段）**：$\text{MSS} = \text{MTU} - \text{IP Header}(20\text{B}) - \text{TCP Header}(20\text{B}) = \mathbf{1460\text{ Bytes}}$；
- **PPPoE 专线场景**：PPPoE 报头占 $8\text{ Bytes}$ $\implies$ $\text{MTU} = 1492\text{ Bytes}$，$\text{MSS} = \mathbf{1452\text{ Bytes}}$（接口须配置 `tcp adjust-mss 1452`）；
- **VXLAN 叠加封装**：外层 MAC(14B) + IP(20B) + UDP(8B) + VXLAN(8B) = **$50\text{ Bytes}$** $\implies$ 物理 Underlay 链路 MTU 须放大至 **$\ge 1550\text{ Bytes}$**（建议开启 Jumbo Frame 9000~9600）。

### 1.2 华为交换机接口三种链路类型处理规则矩阵
| 接口模式 | 接收未带 Tag 报文（Untagged） | 接收带 Tag 报文（Tagged） | 发送报文出接口处理 | 典型应用场景 |
|:---|:---|:---|:---|:---|
| **Access** | 打上本接口 `PVID`，接收 | 若 Tag == PVID 则接收；若 Tag != PVID 则直接丢弃 | **剥离 Tag（Untagged）** 发送 | 连接终端 PC、服务器、打印机 |
| **Trunk** | 打上本接口 `PVID`；若 PVID 在允许通过列表中则接收，否则丢弃 | 若 Tag 在允许通过列表中则接收；否则直接丢弃 | 若 Tag == PVID 且在列表中，**剥离 Tag 发送**；若 Tag != PVID 则**保留 Tag 透传** | 交换机间互联、透传多 VLAN |
| **Hybrid** | 打上本接口 `PVID`；若 PVID 在 Untagged/Tagged 列表中则接收，否则丢弃 | 若 Tag 在 Untagged 或 Tagged 列表中则接收；否则直接丢弃 | 查列表：若在 `port hybrid vlan untagged` 列表中则**剥离 Tag 发送**；若在 `tagged` 列表中则**保留 Tag 发送** | 灵活单臂路由、跨部门无三层隔离互访 |

### 1.3 生成树家族（STP / RSTP / MSTP）关键参数速查
- **BPDU 关键定时器**：Hello Time = **$2\text{ s}$**，Forward Delay = **$15\text{ s}$**，Max Age = **$20\text{ s}$**；
- **收敛时延对照**：
  - **STP (802.1D)**：根桥直连链路中断需 **$30\text{ s}$**（$2 \times \text{Forward Delay}$）；非直连故障需 **$50\text{ s}$**（Max Age $20\text{ s}$ + $2 \times 15\text{ s}$）；
  - **RSTP (802.1w)**：引入 AP/BP 预备角色、P/A 快速协商机制，秒级（$<1\text{ s}$）极速收敛；
  - **华为 STP 私有特性**：借用 RSTP 次优 BPDU 反杀与 AP/BP 角色，根桥故障在华为设备上实测收敛为 **$30\text{ s}$**（非标准 50s）；
- **端口角色与状态演进**：
  - STP 5 状态：`Disabled` $\to$ `Blocking` $\to$ `Listening` $\to$ `Learning` $\to$ `Forwarding`；
  - RSTP 3 状态：`Discarding`（不学MAC、不转数据）、`Learning`（学MAC、不转数据）、`Forwarding`（学MAC、正常转发）；
- **MSTP 核心口诀**：**“实例是车，VLAN 是乘客”**；同域三要素（域名相同、修订级别相同、VLAN-实例映射表完全一致）；CIST 为全网总树，MSTI 为域内独立多实例树。

---

## 卡片二：动态路由协议核心参数矩阵

### 2.1 常见动态路由协议全景横向对比大表
| 特性维度 | OSPF (开放最短路径优先) | IS-IS (中间系统到中间系统) | BGP (边界网关协议) |
|:---|:---|:---|:---|
| **协议类型** | 内部网关链路状态型 (Link-State) | 内部网关链路状态型 (Link-State) | 外部网关路径矢量型 (Path-Vector) |
| **底层承载** | IP 协议号 **`89`** | 直接封装于链路层 (Ethernet Type `0xFEFE` / LLC) | TCP 端口号 **`179`** |
| **华为默认优先级** | **内部 10 / 外部引入 150** | **15** | **EBGP 255 / IBGP 255 / 本地引入 255** |
| **组播 / 单播地址** | `224.0.0.5` (All SPF) / `224.0.0.6` (DR/BDR) | `01-80-c2-00-00-14` (L1) / `01-80-c2-00-00-15` (L2) | 单播建立 TCP 会话 |
| **邻居保活定时器** | Hello **$10\text{ s}$** / Dead **$40\text{ s}$** (NBMA 30s/120s) | Hello **$10\text{ s}$** / Hold **$30\text{ s}$** (DIS Hello 3.3s/10s) | Keepalive **$60\text{ s}$** / Hold **$180\text{ s}$** |
| **邻居核心状态机** | `Down` $\to$ `Init` $\to$ `2-Way` $\to$ `ExStart` $\to$ `Exchange` $\to$ `Loading` $\to$ **`Full`** | `Down` $\to$ `Initializing` $\to$ **`Up`** | `Idle` $\to$ `Connect` $\to$ `Active` $\to$ `OpenSent` $\to$ `OpenConfirm` $\to$ **`Established`** |
| **区域拓扑设计** | 基于接口划分区域，必须有骨干 Area 0 | 基于整机划分 Level-1 / Level-2 / Level-1-2 | 基于 AS 划分（2 字节 / 4 字节 AS 号） |

### 2.2 OSPF 六大核心 LSA 速查
| LSA 类型 | 名称 | 产生者 (Advertiser) | 泛洪扩散范围 | 承载核心内容 |
|:---|:---|:---|:---|:---|
| **Type 1** | Router LSA | 区域内每一台运行 OSPF 的路由器 | 本区域内泛洪 | 描述本地接口物理拓扑与网段开销 (P2P/TransNet/StubNet/Vlink) |
| **Type 2** | Network LSA | 广播网/NBMA 链路上的 **DR 路由器** | 本区域内泛洪 | 描述伪节点拓扑及当前广播网连接的全部邻居 Router-ID 列表 |
| **Type 3** | Network Summary | 区域边界路由器 **ABR** | 相邻其他区域（区域间穿透） | 描述区域间聚合网段路由与到 ABR 的开销（纯距离矢量特性） |
| **Type 4** | ASBR Summary | 区域边界路由器 **ABR** | 除 ASBR 所在区域外的其他区域 | 描述如何到达 ASBR（通告 ASBR 所在的 Area 0 到该 ASBR 的开销） |
| **Type 5** | AS External LSA | 自治系统边界路由器 **ASBR** | 全网泛洪（除 Stub/NSSA 特殊区域外） | 描述外部引入的非 OSPF 路由，包含 E1/E2 开销与外部路由标记 Tag |
| **Type 7** | NSSA External | NSSA 区域内的 **ASBR** | **仅在 NSSA 区域内泛洪** | 携带外部路由，出 NSSA 区域时由 ABR 执行 **7转5** 转换为 Type 5 扩散 |

### 2.3 BGP 十三条选路准则十六字决
> **口诀**：**优选权值最大，本地优先次之；起源本地汇总，路径最短最亲；起源代码最优，MED值最小；EBGP先于IBGP，下一跳开销最小；集群最短，Router-ID最小，对等体地址最小。**

1. **协议优选值 (PrefVal)**：华为私有属性，仅本地有效，越大越优（默认 0）；
2. **本地优先级 (Local-Preference)**：AS 内部泛洪，越大越优（默认 100）；
3. **本地生成路由优选**：`手工聚合` $>$ `自动聚合` $>$ `network` $>$ `import` $>$ `对等体学来`；
4. **AS-Path 路径长度**：越短越优（`AS_SET` 算作 1 个长度）；
5. **Origin 起源属性**：`IGP (i)` $>$ `EGP (e)` $>$ `Incomplete (?)`；
6. **MED (Multi-Exit Discriminator)**：越小越优（默认仅比对来自同一 AS 的路由）；
7. **对等体类型**：**EBGP 路由** 优于 **IBGP 路由**；
8. **下一跳 IGP Metric**：本地到达 BGP Next-Hop 的内部 IGP 开销越小越优；
9. **BGP 负载分担**：若配置 `maximum load-balancing`，前 8 条完全一致则形成 ECMP 等价分担；
10. **Cluster-List 长度**：反射器场景下越短越优；
11. **Originator-ID / Router-ID**：数值越小越优；
12. **Peer IP 地址**：对等体物理接口或环回口 IP 地址数值越小越优。

---

## 卡片三：路由控制与策略工具箱

### 3.1 访问控制列表（ACL）编号与规则
- **基本 ACL（2000 ~ 2999）**：仅基于**源 IPv4 地址**匹配（例：`rule 5 permit source 192.168.1.0 0.0.0.255`）；
- **高级 ACL（3000 ~ 3999）**：五元组匹配（源/目 IP、协议号、源/目端口号、TCP Flag、DSCP/IP Precedence）；
- **二层 ACL（4000 ~ 4999）**：基于源/目 MAC、VLAN ID、EtherType（如 `0x0800` IPv4、`0x86DD` IPv6、`0x0806` ARP）；
- **反掩码（通配符 Wildcard）速查**：
  - `0` 代表必须严格匹配，`1` 代表任意忽略；
  - 严格匹配单主机：`192.168.1.1 0.0.0.0`（可简写为 `source 192.168.1.1 0`）；
  - 匹配所有奇数网段：`192.168.1.0 0.0.254.0`（二进制最后一位固定为 1）。

### 3.2 IP 前缀列表（IP-Prefix List）计算法则
语法：`ip ip-prefix <name> index <num> [permit|deny] <network>/<mask-len> [greater-equal <ge>] [less-equal <le>]`
- **无 ge / le**：严格精确匹配，掩码必须等于 `mask-len`；
- **带 ge / le 约束公式**：**$\text{mask-len} \le \text{ge-value} \le \text{le-value} \le 32$**；
- **五大经典工程前缀段速查**：
  - 匹配所有缺省路由：`ip ip-prefix P1 permit 0.0.0.0 0`
  - 匹配任意所有主机路由：`ip ip-prefix P2 permit 0.0.0.0 0 greater-equal 32`
  - 匹配全网所有路由（放行所有）：`ip ip-prefix P3 permit 0.0.0.0 0 less-equal 32`
  - 匹配标准 A 类主网路由：`ip ip-prefix P4 permit 0.0.0.0 1 less-equal 8`
  - 匹配私网 10.x.x.x 所有子网：`ip ip-prefix P5 permit 10.0.0.0 8 less-equal 32`

### 3.3 BGP 四大公认团体属性（Community）
| 团体属性名 | 十六进制值 | 控制传播行为与工程作用 |
|:---|:---|:---|
| **Internet** | `0x00000000` | 默认属性，可以向全网任何 BGP 对等体随意通告扩散 |
| **No_Export** | `0xFFFFFF01` | **不能发布给本地 AS 之外的 EBGP 邻居**（但可在联盟子 AS 间传递） |
| **No_Advertise** | `0xFFFFFF02` | **收到此属性的路由器不得将该路由发布给任何其他邻居**（IBGP/EBGP 统统禁发） |
| **No_Export_Subconfed** | `0xFFFFFF03` | 不能发布给任何 AS 外对等体，**也不能在联盟内部的子 AS 间通告** |

---

## 卡片四：基础服务、安全与高可用

### 4.1 DHCP 交互流程与高频 Option 字典
- **四步交互流程（DORA）**：`DHCP Discover (广播/UDP 67)` $\to$ `DHCP Offer (单/广播/UDP 68)` $\to$ `DHCP Request (广播)` $\to$ `DHCP Ack (单/广播)`；
- **续租周期**：$T_1 = 50\%$ 单播 Request 续租；$T_2 = 87.5\%$ 广播 Request 续租；$100\%$ 到期清空重新 Discover；
- **关键 Option 字典**：
  - **Option 43**：向华为无线 AP 宣告无线控制器 **AC 的 IP 地址列表**；
  - **Option 60**：Vendor Class Identifier（厂商标识，AP 据此识别华为设备身份）；
  - **Option 82**：DHCP Relay 代理信息（包含 Circuit ID 接入端口号、Remote ID 交换机 MAC），用于准确定位与防 IP 仿冒；
  - **Option 148**：华为 SD-WAN 与云管理网络中指示 **iMaster NCE 控制器 IP 与端口号**（ZTP 自动上线必备）。

### 4.2 NAT 四大工作模式对比
| NAT 模式 | 转换维度 | 公网 IP 地址池 | 端口复用 (PAT) | 典型应用与配置特征 |
|:---|:---|:---|:---|:---|
| **静态 NAT** | 一对一固定转换 | 私网 IP 与公网 IP 严格 1:1 静态绑定 | 否 | 外网固定访问内网高安全主机/服务器 |
| **动态 NAT (Basic)** | 一对一动态抽取 | 建立公网 IP 地址池（No-PAT） | 否 | 仅转换 IP 不转端口，耗尽即阻塞，极少使用 |
| **NAPT** | 多对多动态复用 | 建立独立公网 IP 地址池 | **是（基于源端口映射）** | 多数企业出海口：`nat outbound 2000 address-group 1` |
| **Easy-IP** | 多对一动态复用 | **直接复用出接口当前公网 IP**（无地址池） | **是（基于源端口映射）** | PPPoE 拨号或小型分支出口：`nat outbound 2000` |
| **NAT Server** | 静态端口映射 | 公网 IP + 指定公网端口 | 是（四层端口级别重定向） | 内网 Web/Mail 发布：`nat server protocol tcp global ...` |

### 4.3 VRRP 协议核心参数与虚 MAC 计算
- **虚拟 MAC 规范**：IPv4 VRRP 虚 MAC 固定为 **`00-00-5E-00-01-XX`**（`XX` 为十六进制 VRID，如 VRID 1 为 `00-00-5E-00-01-01`）；
- **报文发送**：IP 协议号 **`112`**，目的组播地址 **`224.0.0.18`**，通告间隔默认 **$1\text{ s}$**；
- **主备倒换**：Master 挂掉后，Backup 等待倒换时间 = **$3 \times \text{Advertisement\_Interval} + \text{Skew\_Time}$**（约 $3.6\text{ s}$）；配合 **BFD 毫秒联动** 可压至 **$<50\text{ ms}$**。

---

## 卡片五：IPv6 编址体系与 NDP 核心协议

### 5.1 IPv6 地址块分类速查
| 地址类型 | 前缀范围 (Prefix) | 作用域与特性 | 等价 IPv4 对应概念 |
|:---|:---|:---|:---|
| **GUA (全球单播地址)** | **`2000::/3`** (2000:: ~ 3FFF::) | 全球唯一可路由公网单播地址 | IPv4 公网 IP 地址 |
| **ULA (唯一本地地址)** | **`FC00::/7`** (常取 `FD00::/8`) | 本地组织内部自由规划，不可在公网路由 | IPv4 RFC1918 私网地址 (10.0.0.0/8 等) |
| **LLA (链路本地地址)** | **`FE80::/10`** (接口自动生成) | 仅在直连同物理二层链路上有效，路由器不转发 | 自动私有 IP (169.254.0.0/16) |
| **组播地址 (Multicast)** | **`FF00::/8`** | 替代广播，所有节点 `FF02::1`、所有路由器 `FF02::2` | IPv4 组播与全网广播 |
| **被请求节点组播地址** | **`FF02::1:FFxx:xxxx/104`** | 前缀固定，后 24 位取自对应 IPv6 单播地址末尾 24 位 | 用于 IPv6 替代 ARP 的高效监听组播 |
| **未指定地址 / 环回口** | `::/128` (未指定) / `::1/128` (环回) | 用于 DAD 地址冲突检测 / 本机回环测试 | `0.0.0.0` / `127.0.0.1` |

### 5.2 NDP（邻居发现协议）五大 ICMPv6 报文
- **RS（路由器请求，Type 133）**：主机开机主动向 `FF02::2` 广播请求网关下发前缀；
- **RA（路由器通告，Type 134）**：网关周期性或响应 RS 向 `FF02::1` 发送网络前缀、跳数限制及 M/O 标志位（M=0/O=0 无状态 SLAAC，M=1 有状态 DHCPv6）；
- **NS（邻居请求，Type 135）**：代替 ARP Request 发送给目标单播地址对应的**被请求节点组播地址**，亦用于 DAD 冲突检测；
- **NA（邻居通告，Type 136）**：代替 ARP Reply 携带本端 MAC 单播响应 NS；
- **Redirect（重定向，Type 137）**：网关通知主机有更优的同子网下一跳。

---

## 卡片六：MPLS 骨干网与跨域 L3VPN

### 6.1 MPLS 32 位 Shim 报头结构
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Label (20 Bits)                | Exp |S|  TTL  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
- **Label（20 位）**：标签值范围 $0 \sim 2^{20}-1$（$0 \sim 1048575$）；
  - **`0`（IPv4 显式空标签）**：倒数第二跳带 0 转发，Egress 设备保留 EXP 查表；
  - **`3`（隐式空标签，PHP 次末跳弹出）**：**倒数第二跳节点直接弹出最外层公网标签**，Egress 设备仅需执行一次查表；
  - **`16 ~ 1023`**：静态标签保留区间；**$\ge 1024$**：LDP / MP-BGP 动态分配区间；
- **Exp / TC（3 位）**：服务质量等级（QoS CoS 映射）；
- **S（1 位，Bottom of Stack）**：栈底指示位。**`S=1` 表明是内层最底层标签**，`S=0` 表明后续还有标签压栈；
- **TTL（8 位）**：生存时间，防止标签环路。

### 6.2 跨域 MPLS VPN 三大方案核心特征矩阵
| 跨域方案 | ASBR 接口对接方式 | ASBR 控制面状态负担 | 跨域信令与公网标签 | 典型优劣与适用场景 |
|:---|:---|:---|:---|:---|
| **Option A (背靠背)** | 物理子接口划入不同 VRF 直连 | **最高**（ASBR 需维护所有租户的 VRF 实例与路由表） | 无标签跨域，转为纯普通 IPv4 报文转发 | 配置最简，隔离最彻底；但 ASBR 接口与性能扩展性差 |
| **Option B (单跳 MP-EBGP)** | 物理主接口，使能 MPLS | **中等**（ASBR 无需配 VRF，但需维护全量 VPNv4 路由） | 跨域仅有一层标签（ASBR 间替换 VPNv4 标签） | 部署灵活；但所有跨域流量经过 ASBR 标签转发，ASBR 成瓶颈 |
| **Option C (端到端标签)** | 物理主接口，仅跑普通 BGP/IGP | **最低**（ASBR 仅维护骨干网公网路由，完全不感知 VPN） | **三层标签栈架构**（私网 BGP + 跨域 BGP + 底层 IGP/LDP） | 扩展性最强，ASBR 零负担；但跨运营商需暴露对端 PE 环回口 |

---

## 卡片七：云网 Overlay、EVPN 与智算前沿

### 7.1 EVPN 五类核心路由（RFC 7432）速查
| 路由类型 (Route Type) | 路由名称 (Name) | 关键携带字段 | 核心工程功能 |
|:---|:---|:---|:---|
| **Type 1** | Ethernet Auto-Discovery | ESI + ESI 标签 + 别名标志 | ① **水平分割防环**（分配 ESI 标签）；② **Aliasing 别名**（实现双归多活负载分担）；③ **快速收敛**（批量撤销） |
| **Type 2** | MAC/IP Advertisement | ESI + MAC + IP + MPLS1(二层)/MPLS2(三层) | 通告主机 MAC 地址与主机 IP 地址，驱动远端生成二层 MAC 表与 ARP 缓存（代答） |
| **Type 3** | Inclusive Multicast | Originating Router IP + BUM 标签 | 自动建立 BUM 报文的头端复制隧道列表（Head-end Replication） |
| **Type 4** | Ethernet Segment Route | ESI + 发送端 IP | 发现接入同一 ES 的兄弟双归 PE 节点，并通过哈希算法选举指定转发器（DF） |
| **Type 5** | IP Prefix Route | IP 前缀 + 掩码 + IP VPN 实例标签 | 通告网段级路由，在数据中心多租户与跨域场景中实现全网纯三层互通 |

### 7.2 SR-MPLS 与 SRv6 核心换算与指令集
- **SR-MPLS 标签计算**：
  $$\text{出标签值} = \text{对端设备本地 SRGB 起始值} + \text{Prefix-SID 偏移量}$$
  （工程建议全网统一预留规划 SRGB 为 `16000 ~ 23999`）；
- **SRv6 128 位 SID 构成**：`Locator(路由寻址 64~96b) + Function(指令行为 16~32b) + Arguments(参数)`；
- **四大核心 SRv6 行为指令**：
  - **`End`**：更新 Segments Left 指针，查外层 IPv6 路由表转发至下一个 SID；
  - **`End.X`**：更新指针，**强制从绑定的三层特定物理出接口**转发至直连邻居；
  - **`End.DT4`**：剥离外层 IPv6 头与 SRH，直接查绑定的 **IPv4 VPN 实例路由表**转发（替代传统 MPLS L3VPN 内层标签）；
  - **`End.DX2`**：剥离外层头，直接透传二层数据帧至关联的 AC 接口（替代传统二层 VPWS 标签）；
- **智算无损以太网底线准则**：
  - 智算训练中 **$0.1\%$ 丢包直接导致 $90\%$ 吞吐腰斩崩溃**；
  - AIDC 无损网络必须达到 **$\le 10^{-6}$ 丢包率**；采用 **RoCEv2（UDP 端口 4791）+ SPFC 精细化流控 + AFR 大象流动态调度**。

---

## 卡片八：华为 VRP 排障黄金命令行与状态速查字典

### 8.1 经典协议卡死状态与一线秒级定位字典
| 协议 | 异常挂起状态 | 最常见根因排查 | 瞬间定界排查命令 |
|:---|:---|:---|:---|
| **OSPF** | **`ExStart` / `Exchange`** | **两端接口 MTU 不一致**（配了 `mtu-enable` 后协商不过）；或报文过大被安全设备拦截 | `display ospf interface`（核对两端 MTU 与掩码） |
| **OSPF** | **`Init`** | 单向连通性故障（本端收到对端 Hello，但对端的 Hello 报文中没有本端的 Router-ID） | 检查中间交换机 VLAN 放行与 ACL 单向拦截 |
| **BGP** | **`Active`** | **TCP 三次握手建立失败**：无路由到达对端更新源、ACL 阻断 TCP 179、或 `connect-interface` 敲错 | `display ip routing-table <peer-ip>` / `display tcp status` |
| **BGP** | **`Idle`** | 没有去往对端地址的有效 IGP/静态路由，或 EBGP 间缺少 `ebgp-max-hop`（默认 TTL=1 无法非直连建邻） | `display bgp peer` / `ping -a <src> <peer-ip>` |
| **MSTP** | 端口频繁震荡 / 环路 | 边缘端口下挂傻瓜交换机成环；或未开启 BPDU 保护被抢根 | `display stp brief` / `display stp abnormal-port` |
| **EVPN** | BUM 泛洪流量被丢弃 | 双归属场景下非 DF 节点执行了合法丢弃；或 ESI 标签未分配 | `display evpn es-member` / `display evpn routing df-election` |

### 8.2 华为数通工程师 TOP 15 高频神级诊断指令
```shell
# 1. 快速排查接口物理与协议状态、输入输出错包与带宽利用率
display interface brief

# 2. 检索当前激活生效的全局 IP 路由精简总表
display ip routing-table

# 3. 跨协议查看到达特定目的 IP 的最长匹配选路与下一跳详细信息
display ip routing-table 192.168.10.1 verbose

# 4. 查看所有已激活的 VPN 实例（VRF）及绑定的接口矩阵
display ip vpn-instance

# 5. 查看 OSPF 邻居简明状态（重点关注 State 是否为 Full）
display ospf peer brief

# 6. 查看 OSPF 本地链路状态数据库（LSDB）的 LSA 摘要分布
display ospf lsdb

# 7. 查看 IS-IS 邻居邻接关系及链路级别（Level-1/Level-2）
display isis peer

# 8. 查看 BGP 所有对等体状态（State 必须为 Established，查看收发路由数）
display bgp peer

# 9. 查看由 BGP 选优胜出并实际写入本地 IP 路由表的 BGP 路由条目
display bgp routing-table

# 10. 极速查看所有物理端口的生成树角色（RP/DP/AP）与转发状态（FORWARDING/DISCARDING）
display stp brief

# 11. 查看 LACP 动态链路聚合组协商详情（核对 Selected/Unselected 接口及本端/对端优先级）
display eth-trunk

# 12. 查看 VRRP 虚拟路由器双机热备当前角色（Master/Backup）及虚 IP 状态
display vrrp brief

# 13. 查看硬件级毫秒探针（BFD）当前会话状态（必须显示 State: Up）
display bfd session all

# 14. 查看二层 MAC 地址动态转发表，核对接口学习与 VLAN 绑定（排查 MAC 漂移）
display mac-address

# 15. 查看 BGP EVPN 控制面邻居与五类 EVPN 路由的生成与收发详情
display bgp evpn peer
display bgp evpn all routing-table
```
