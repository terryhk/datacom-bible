# 数通工程师全栈学习笔记 (Datacom Bible)

> **从物理连通到云网智联——用协议第一性原理与 NetDevOps 全栈实战打造顶级数通架构专家**

[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)
[![Volumes: 35 Volumes](https://img.shields.io/badge/Volumes-35%20Volumes%20--%20530k%2B%20Words-blue.svg)](SUMMARY.md)
[![PRDs: 4 Industry PRDs](https://img.shields.io/badge/PRDs-4%20Industry%20PRDs-orange.svg)](09_第九篇_系统级PRD工程设计/数通工程师全栈学习笔记体系_PRD.md)
[![Status: Complete](https://img.shields.io/badge/Status-Complete-success.svg)](00_总览与知识地图.md)
[![Cheatsheet: 1-Page QuickRef](https://img.shields.io/badge/Cheatsheet-1--Page%20QuickRef-success.svg)](CHEATSHEET.md)
[![Huawei: HCIA/HCIP/HCIE](https://img.shields.io/badge/Huawei-HCIA%20%7C%20HCIP%20%7C%20HCIE-red.svg)](OUTLINE.md)
[![NetDevOps: Python & NETCONF](https://img.shields.io/badge/NetDevOps-Python%20%26%20NETCONF%2FYANG-brightgreen.svg)](08_第八篇_网络自动化与机试/35-Python网络自动化.md)

---

## 关于本书

很多从事网络工程、IT 运维或者初入数通行业的工程师，在职业成长中常常陷入四大经典困局：
1. **“死记硬背命令行，不懂底层协议原理”**：只会照搬拓扑敲配置，一旦网络发生微小的震荡丢包或协议卡在协商状态（如 OSPF 卡在 ExStart、BGP 卡在 Active、EVPN 双归属泛洪回灌），完全抓瞎无从排查；
2. **“局限于传统二三层局域网，缺乏云网广域骨干视野”**：熟悉静态路由与传统 VLAN，但面对现代数据中心大二层（VXLAN）、骨干网大一统控制平面（EVPN）、新一代源路由（SR-MPLS / SRv6）与混合多链路（SD-WAN）时望而生畏；
3. **“只会肉眼敲 CLI，被海量设备巡检拖垮”**：面对成百上千台交换机，依然依靠人工登录敲 `display` 巡检，缺乏面向对象（OOP）、并发线程池（ThreadPoolExecutor）与模型驱动（NETCONF/YANG）的现代自动化武器；
4. **“考过证书却缺乏系统级工程设计能力”**：手握 HCIE 证书，面对实际企业大中型园区网规划、跨域专线架构或需求规格说明书（PRD）撰写时，依然不知道如何将技术指标转化为商业工程闭环。

本书依托**华为数通官方技术体系（HCIA / HCIP / HCIE Datacom）、主流企业级工程规范及海量大型园区网与广域骨干网真实项目实践**深度萃取重构而成。

全书彻底破除“命令行搬运工”的肤浅认知，贯穿一条由第一性原理驱动的核心价值方程式：

$$\text{协议底层原理 (Protocol)} \times \text{现网工程架构 (Architecture)} \times \text{自动化编程 (NetDevOps)} = \text{数通全栈专家 (Datacom Architect)}$$

数通工程师不是单纯的“配网工具人”，而是企业数字化基座的**连接总设计师、高可用守护者与效率解放者**。你所交付的不仅是一行行 VRP 配置，更是**企业业务毫秒级稳健运行、跨域全球智能互联的坚实底座**。

---

## 适合谁读

- **网络初学 / 运维工程师转型**：打破碎片化自学盲区，建立从物理介质、以太网帧到三层路由的扎实理论坐标系；
- **HCIP / HCIE Datacom 认证攻坚考生**：全面吃透必考高难核心技术（BGP 13条选路、MPLS 跨域 Option A/B/C、QoS 队列与令牌桶、EVPN 5类路由时序、SRv6 128位 SID 指令集、TI-LFA 算法及 Python 编程机试真题）；
- **资深网络工程师 / 解决方案架构师**：精通大型虚拟化园区网（Fabric / 业务随行 / PnP 开局）、企业级 SD-WAN 混合链路改造与多租户安全切片规划；
- **NetDevOps 自动化转型开发者**：掌握 Python 面向对象建模、命令解耦读取、定时 SFTP 配置灾备归档与 NETCONF/YANG 现代结构化可编程。

---

## 多轨高效学习路线

本书提供多轨针对性学习路线：

- 🚀 **新锐极速通关路线**：[00 总览与知识地图](00_总览与知识地图.md) $\to$ [第一篇：数通基石（卷01~04）](01_第一篇_数通网络通用基石/01-网络基础与网络参考模型.md) $\to$ [第三篇：交换与高可用（卷11/14）](03_第三篇_二层交换与园区高可用/11-VLAN、VLAN间通信与ACL.md) $\to$ [第四篇：服务与安全（卷15/16）](04_第四篇_基础网络服务与安全/15-DHCP与无线接入.md) $\to$ [附录A：数通核心100问](10_第十篇_附录与数通实战军火库/附录A_数通高频核心100问与面试突击宝典.md)；
- 🎯 **HCIP / HCIE 核心攻坚路线**：[第二篇：动态路由（卷05~10）](02_第二篇_动态路由与控制策略/05-OSPF路由协议.md) $\to$ [第六篇：MPLS承载（卷22~24）](06_第六篇_MPLS与承载骨干/22-MPLS原理与LDP.md) $\to$ [第七篇：QoS与Overlay（卷25~27）](07_第七篇_云网前沿与高级特性/25-QoS服务质量.md) $\to$ [第八篇：网络自动化与机试（卷33~35）](08_第八篇_网络自动化与机试/33-Python编程基础.md)；
- ☁️ **云网前沿与骨干架构路线**：[卷26：VXLAN技术](07_第七篇_云网前沿与高级特性/26-VXLAN技术.md) $\to$ [卷27：EVPN原理](07_第七篇_云网前沿与高级特性/27-EVPN原理与配置.md) $\to$ [卷28：Segment Routing](07_第七篇_云网前沿与高级特性/28-Segment-Routing.md) $\to$ [卷29：SRv6原理](07_第七篇_云网前沿与高级特性/29-SRv6原理与配置.md) $\to$ [卷30~32：园区虚拟化与SD-WAN](07_第七篇_云网前沿与高级特性/30-企业园区网全景与业务随行.md) $\to$ [附录C：全球互联与SASE](10_第十篇_附录与数通实战军火库/附录C_全球广域网互联与前沿专题_Transit_Peering_NNI_SASE.md) $\to$ [附录D：云网融合与算力网络前沿](10_第十篇_附录与数通实战军火库/附录D_华为云网融合与算力网络前沿实战_IPv6Plus_SRv6_无损以太_CU转控分离.md)；
- 🤖 **NetDevOps 自动化实战路线**：[卷33：Python编程基础](08_第八篇_网络自动化与机试/33-Python编程基础.md) $\to$ [卷34：面向对象与并发线程池](08_第八篇_网络自动化与机试/34-Python面向对象与进阶.md) $\to$ [卷35：Paramiko与NETCONF](08_第八篇_网络自动化与机试/35-Python网络自动化.md) $\to$ [PRD：自动化运维脚本系统](09_第九篇_系统级PRD工程设计/企业园区网自动化运维脚本系统_PRD.md)。

---

## 全书知识全景与技术拓扑地图

```mermaid
flowchart LR
    subgraph S1["第一篇：通用基石"]
        direction TB
        V1["01-网络参考模型"]
        V2["02-VRP设备管理"]
        V3["03-IP编址与ARP"]
        V4["04-IP路由基础"]
    end

    subgraph S2["第二篇：动态路由"]
        direction TB
        V5["05-OSPF路由协议"]
        V6["06-IS-IS骨干路由"]
        V7["07-路由策略控制"]
        V8["08-BGP基础选路"]
        V9["09-BGP高级特性"]
        V10["10-IGP高级特性"]
    end

    subgraph S3["第三篇：二层交换与高可用"]
        direction TB
        V11["11-VLAN与ACL"]
        V12["12-生成树协议STP"]
        V13["13-链路聚合与堆叠"]
        V14["14-VRRP与BFD"]
    end

    subgraph S4["第四篇：服务与安全"]
        direction TB
        V15["15-DHCP与WLAN"]
        V16["16-NAT与PPP"]
        V17["17-防火墙与VRF"]
        V18["18-网络安全技术"]
    end

    subgraph S5["第五篇：IPv6演进"]
        direction TB
        V19["19-IPv6基础编址"]
        V20["20-IPv6路由协议"]
        V21["21-IPv6过渡技术"]
    end

    subgraph S6["第六篇：MPLS承载"]
        direction TB
        V22["22-MPLS与LDP"]
        V23["23-MPLS L3VPN"]
        V24["24-MPLS跨域VPN"]
    end

    subgraph S7["第七篇：云网前沿与高级特性"]
        direction TB
        V25["25-QoS服务质量"]
        V26["26-VXLAN数据中心"]
        V27["27-BGP EVPN控制面"]
        V28["28-Segment Routing"]
        V29["29-SRv6新一代承载"]
        V30["30-园区业务随行"]
        V31["31-虚拟化园区Fabric"]
        V32["32-SD-WAN解决方案"]
    end

    subgraph S8["第八篇：NetDevOps自动化"]
        direction TB
        V33["33-Python编程基础"]
        V34["34-面向对象与并发"]
        V35["35-网络自动化机试"]
    end

    subgraph S9["第九篇：系统级PRD工程设计"]
        PRD1["全栈体系PRD"]
        PRD2["SD-WAN项目PRD"]
        PRD3["自动化运维脚本PRD"]
        PRD4["虚拟化园区网PRD"]
    end

    subgraph S10["第十篇：附录与军火库"]
        AP1["附录A: 高频100问速查"]
        AP2["附录B: 组播技术深度解析"]
        AP3["附录C: 全球互联与SASE"]
    end

    S1 ==> S2 ==> S3 ==> S4 ==> S5 ==> S6 ==> S7 ==> S8
    S7 -.-> S9
    S8 -.-> S9
    S2 -.-> S10
    S6 -.-> S10
    S7 -.-> S10
```

---

## 篇章导读与章节速查

### [全书总览与知识地图](00_总览与知识地图.md)
* 数通宏观技术坐标系、三大第一性原理心法、全流程排障四步法。

### [第一篇：数通网络通用基石（第 01 ~ 04 卷）](01_第一篇_数通网络通用基石/01-网络基础与网络参考模型.md)
* **[卷01：网络基础与网络参考模型](01_第一篇_数通网络通用基石/01-网络基础与网络参考模型.md)**：OSI 七层与 TCP/IP 四层架构、数据封装解封装、以太网双绞线与光纤介质；
* **[卷02：VRP系统与设备管理](01_第一篇_数通网络通用基石/02-VRP系统与设备管理.md)**：华为 VRP 命令行体系、用户级别与权限、文件存储系统、配置保存与备份初始化；
* **[卷03：网络层协议与IP编址](01_第一篇_数通网络通用基石/03-网络层协议与IP编址.md)**：IPv4 报头字段、子网划分与 VLSM、ICMP 控制报文机制与 ARP 动态映射；
* **[卷04：IP路由基础](01_第一篇_数通网络通用基石/04-IP路由基础.md)**：IP 路由查表原理、最长匹配原则（LPM）、路由优先级与开销、浮动静态路由主备容灾。

### [第二篇：动态路由与控制策略（第 05 ~ 10 卷）](02_第二篇_动态路由与控制策略/05-OSPF路由协议.md)
* **[卷05：OSPF路由协议](02_第二篇_动态路由与控制策略/05-OSPF路由协议.md)**：SPF 链路状态算法、八大状态机跃迁、LSA 1~7类报文全貌、DR/BDR 选举与 Stub/NSSA 特殊区域；
* **[卷06：IS-IS路由协议](02_第二篇_动态路由与控制策略/06-IS-IS路由协议.md)**：电信级骨干路由首选、TLV 灵活扩展、Level-1/2 区域层次设计与 DIS 抢占机制；
* **[卷07：路由策略与路由控制](02_第二篇_动态路由与控制策略/07-路由策略与路由控制.md)**：Route-Policy 规则匹配、IP-Prefix 掩码范围过滤、双向重分发四色打标防环与 PBR 策略路由；
* **[卷08：BGP基础与路径优选](02_第二篇_动态路由与控制策略/08-BGP基础与路径优选.md)**：外部网关协议外交官、13 条路径选优规则、IBGP 水平分割打破与 RR 路由反射器、Transit 与 Peering 商业流量模型；
* **[卷09：BGP高级特性](02_第二篇_动态路由与控制策略/09-BGP高级特性.md)**：BGP 手工与自动路由聚合、四大团体属性（No_Export/No_Advertise）高级流控、ORF 与安全防御；
* **[卷10：IGP高级特性](02_第二篇_动态路由与控制策略/10-IGP高级特性.md)**：OSPF/IS-IS 快速收敛技术（I-SPF / PRC）、智能定时器、LFA 快速重路由与链路安全加固。

### [第三篇：二层交换与园区高可用（第 11 ~ 14 卷）](03_第三篇_二层交换与园区高可用/11-VLAN、VLAN间通信与ACL.md)
* **[卷11：VLAN、VLAN间通信与ACL](03_第三篇_二层交换与园区高可用/11-VLAN、VLAN间通信与ACL.md)**：Access / Trunk / Hybrid 接口 Tag 处理机制、单臂路由与三层交换（VLANIF）、基本/高级 ACL；
* **[卷12：生成树协议家族](03_第三篇_二层交换与园区高可用/12-生成树协议家族.md)**：STP 端口阻塞破环、RSTP P/A 快速协商机制、边缘端口与 MSTP 多实例二层负载均衡；
* **[卷13：链路聚合与设备堆叠](03_第三篇_二层交换与园区高可用/13-链路聚合与设备堆叠.md)**：手工聚合 vs LACP 动态协商、iStack 交换机堆叠与 CSS 集群架构、跨设备聚合免 STP 实践；
* **[卷14：VRRP与BFD](03_第三篇_二层交换与园区高可用/14-VRRP与BFD.md)**：网关冗余保护、VRRP 状态机与抢占、BFD 硬件级毫秒探针联动极速倒换。

### [第四篇：基础网络服务与安全（第 15 ~ 18 卷）](04_第四篇_基础网络服务与安全/15-DHCP与无线接入.md)
* **[卷15：DHCP与无线接入](04_第四篇_基础网络服务与安全/15-DHCP与无线接入.md)**：DHCP 四步交互与中继、WLAN 组网架构、AC+AP 隧道建立、CAPWAP 协议与 Wi-Fi 6 关键特性；
* **[卷16：NAT与PPP](04_第四篇_基础网络服务与安全/16-NAT与PPP.md)**：静态/动态 NAT、NAPT、Easy-IP、NAT Server 端口映射、域内双向 NAT 与广域网 PPP 协议；
* **[卷17：防火墙与VRF](04_第四篇_基础网络服务与安全/17-防火墙与VRF.md)**：状态检测防火墙安全区域、状态会话表生成、VRF 虚拟路由转发与多租户切片硬隔离；
* **[卷18：网络安全技术](04_第四篇_基础网络服务与安全/18-网络安全技术.md)**：端口安全、ARP 欺骗与泛洪防范、DHCP Snooping、IPsec VPN 安全隧道与零信任/SASE 演进。

### [第五篇：IPv6 新一代演进（第 19 ~ 21 卷）](05_第五篇_IPv6新一代演进/19-IPv6基础与编址.md)
* **[卷19：IPv6基础与编址](05_第五篇_IPv6新一代演进/19-IPv6基础与编址.md)**：定长 40 字节报头、128 位地址分类（GUA/ULA/LLA）、NDP 邻居发现（终结 ARP 广播）与 SLAAC 自动配置；
* **[卷20：IPv6路由协议](05_第五篇_IPv6新一代演进/20-IPv6路由协议.md)**：OSPFv3 协议与 LSA 演进、IPv6 IS-IS 多拓扑（Multi-Topology）与 BGP4+ 组网；
* **[卷21：IPv6过渡技术](05_第五篇_IPv6新一代演进/21-IPv6过渡技术.md)**：双栈技术（Dual-Stack）、IPv6 over IPv4 隧道技术、NAT64 与 DNS64 跨协议互通。

### [第六篇：MPLS 与承载骨干（第 22 ~ 24 卷）](06_第六篇_MPLS与承载骨干/22-MPLS原理与LDP.md)
* **[卷22：MPLS原理与LDP](06_第六篇_MPLS与承载骨干/22-MPLS原理与LDP.md)**：2.5 层 32 位标签头结构、标签操作（PUSH/SWAP/POP）、PHP 次末跳弹出与 LDP 会话分发；
* **[卷23：MPLS L3VPN原理与部署](06_第六篇_MPLS与承载骨干/23-MPLS-L3VPN原理与部署.md)**：VRF 路由隔离、RD 区分符与 RT 目标、MP-BGP 私网标签分发与双层标签端到端转发；
* **[卷24：MPLS VPN跨域](06_第六篇_MPLS与承载骨干/24-MPLS-VPN跨域.md)**：Option A 背靠背 VRF、Option B 单跳 MP-EBGP、Option C 端到端跨域与 NNI 国际专线标准接口。

### [第七篇：云网前沿与高级特性（第 25 ~ 32 卷）](07_第七篇_云网前沿与高级特性/25-QoS服务质量.md)
* **[卷25：QoS服务质量](07_第七篇_云网前沿与高级特性/25-QoS服务质量.md)**：DiffServ 区分服务模型、CBQ 队列调度（LLQ/EF/AF/BE）、WRED 防 TCP 全局同步、CAR 监管与 GTS 令牌桶算法；
* **[卷26：VXLAN技术](07_第七篇_云网前沿与高级特性/26-VXLAN技术.md)**：破除传统二层瓶颈、MAC-in-UDP 50 字节封装与 24 位 VNI、集中式网关 vs 分布式网关架构；
* **[卷27：EVPN原理与配置](07_第七篇_云网前沿与高级特性/27-EVPN原理与配置.md)**：大一统控制平面、Type 1~5 路由协同、三大私网标签（ESI/BUM/单播）、多归属双活水平分割防环与对称 IRB；
* **[卷28：Segment Routing](07_第七篇_云网前沿与高级特性/28-Segment-Routing.md)**：源路由颠覆性架构、SID 体系（Prefix/Node/Adj-SID）、SRGB 标签空间计算、SR Policy 与 TI-LFA 100% 拓扑保护；
* **[卷29：SRv6原理与配置](07_第七篇_云网前沿与高级特性/29-SRv6原理与配置.md)**：原生 IPv6 承载、128 位 SID 指令系统（End/End.X/End.DT4）、SRH 扩展头与 G-SRv6 报头压缩；
* **[卷30：企业园区网全景与业务随行](07_第七篇_云网前沿与高级特性/30-企业园区网全景与业务随行.md)**：大型园区网分层拓扑设计、准入控制三剑客（802.1X/Portal/MAC）、业务随行权限与 IP 解耦（安全组与策略矩阵）；
* **[卷31：大中型虚拟化园区网设计](07_第七篇_云网前沿与高级特性/31-大中型虚拟化园区网设计.md)**：CloudCampus Fabric 组网、Border/Edge 节点矩阵、Underlay PnP 自动化即插即用开局与多 VN 切片；
* **[卷32：SD-WAN解决方案](07_第七篇_云网前沿与高级特性/32-SD-WAN解决方案.md)**：Hybrid WAN 混合链路降本增效、ZTP 邮件开局 Token 鉴权、DPI 应用深度识别与基于 SLA 的智能选路。

### [第八篇：网络自动化与机试（第 33 ~ 35 卷）](08_第八篇_网络自动化与机试/33-Python编程基础.md)
* **[卷33：Python编程基础](08_第八篇_网络自动化与机试/33-Python编程基础.md)**：Python 3 环境、数通核心数据结构、CLI 文本清洗、`with open` 命令解耦与 `*args/**kwargs` 参数解包；
* **[卷34：Python面向对象与进阶](08_第八篇_网络自动化与机试/34-Python面向对象与进阶.md)**：面向对象（OOP）交换机建模、多厂商类继承与多态、防御性异常处理（`try...except`）、`ThreadPoolExecutor` 线程池高并发巡检；
* **[卷35：Python网络自动化](08_第八篇_网络自动化与机试/35-Python网络自动化.md)**：Paramiko SSH 交互式通道（`invoke_shell`）、SFTP 每日配置灾备归档、NETCONF/YANG 模型驱动开发与 HCIE Datacom 编程机试大题满分复盘。

### [第九篇：系统级 PRD 工程设计（4 份系统规范）](09_第九篇_系统级PRD工程设计/数通工程师全栈学习笔记体系_PRD.md)
* **[数通工程师全栈学习笔记体系 PRD](09_第九篇_系统级PRD工程设计/数通工程师全栈学习笔记体系_PRD.md)**：全栈知识库产品需求规格说明书；
* **[SD-WAN 企业分支互联解决方案 PRD](09_第九篇_系统级PRD工程设计/SD-WAN企业分支互联解决方案_PRD.md)**：大型企业 SD-WAN 混合广域网改造项目工程立项规范；
* **[企业园区网自动化运维脚本系统 PRD](09_第九篇_系统级PRD工程设计/企业园区网自动化运维脚本系统_PRD.md)**：Python 自动化巡检与 24 小时灾备归档系统研发需求说明书；
* **[大中型虚拟化园区网络架构设计 PRD](09_第九篇_系统级PRD工程设计/大中型虚拟化园区网络架构设计_PRD.md)**：华为 CloudCampus 虚拟化 Fabric 组网与业务随行设计方案。

### [第十篇：附录与数通实战军火库（5 份高价值附录）](10_第十篇_附录与数通实战军火库/附录A_数通高频核心100问与面试突击宝典.md)
* **[附录A：数通高频核心 100 问与面试突击宝典](10_第十篇_附录与数通实战军火库/附录A_数通高频核心100问与面试突击宝典.md)**：提炼 100 个经典高频考核问答（覆盖面试高频与现网排错全场景），面试/突击必备；
* **[附录B：组播技术深度解析（IGMP与PIM）](10_第十篇_附录与数通实战军火库/附录B_组播技术深度解析_IGMP与PIM.md)**：组播 D 类地址体系、MAC 映射碰撞、IGMPv1/v2/v3 演进与 PIM-SM 汇聚点切换；
* **[附录C：全球广域网互联与前沿专题（Transit/Peering/NNI/SASE）](10_第十篇_附录与数通实战军火库/附录C_全球广域网互联与前沿专题_Transit_Peering_NNI_SASE.md)**：全球骨干 Transit/Peering 商业博弈、IXP 机制、NNI 专线对接界面与 SASE 云安全；
* **[附录D：华为云网融合与算力网络前沿实战（IPv6+/SRv6/无损以太/CU转控分离）](10_第十篇_附录与数通实战军火库/附录D_华为云网融合与算力网络前沿实战_IPv6Plus_SRv6_无损以太_CU转控分离.md)**：算网大脑、东数西算、AIDC 无损以太网（RoCEv2/SPFC/AFR）、IPv6+（APN6/IFIT/SFC）、TI-LFA 100% 收敛与 vBNG CU 转控分离；
* **[附录E：数通全栈协议与工程配置一页速查表](10_第十篇_附录与数通实战军火库/附录E_数通全栈协议与工程配置一页速查表.md)**（亦见根目录 **[CHEATSHEET.md](CHEATSHEET.md)**）：高密度 8 大核心卡片，覆盖二层/OSPF/IS-IS/BGP对齐矩阵/13条选路口诀/路由控制/服务安全/IPv6/MPLS/EVPN/SRv6/无损RoCEv2与华为 VRP TOP15 排障黄金命令。

---

## 开源协议与版权声明

- 本项目知识库正文与代码遵循 **[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)（知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议）**。
- 您可以自由地分享、复制和修改本书内容，但必须给出适当的署名，且**严禁用于任何形式的商业盈利目的**。基于本书衍生的作品必须采用相同的许可协议分发。
- 欢迎在 GitHub 提交 Issue 与 Pull Request 共同完善知识库！⭐ 如果本项目对您的数通学习与职业进阶有所帮助，请为本项目点亮一颗 **Star**！
