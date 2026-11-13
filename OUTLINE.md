# 《数通工程师全栈学习笔记》技术大纲（Outline）

本大纲整理了全套体系所涵盖的核心技术协议、RFC 标准、华为 VRP 关键配置与排障命令，供快速对照与知识检索。

---

## 篇章大纲与核心技术矩阵

| 篇章 | 包含卷号 | 核心技术 / 协议 | 关键机制与命令 | 认证对应 |
|:---|:---|:---|:---|:---:|
| **第一篇：数通网络通用基石** | 卷01 ~ 04 | OSI 7层、TCP/IP、以太网、IPv4、VLSM、ICMP、ARP、静态路由 | ARP广播学习、免费ARP、PMTU发现、最长匹配原则(LPM)、浮动路由容灾 | HCIA |
| **第二篇：动态路由与控制策略** | 卷05 ~ 10 | OSPFv2、IS-IS、BGP-4、Route-Policy、IP-Prefix、PBR | LSA 1~7类、特殊区域(Stub/NSSA)、DIS选举、13条BGP选路准则、四色打标防环、Transit/Peering | HCIP / HCIE |
| **第三篇：二层交换与园区高可用** | 卷11 ~ 14 | VLAN、802.1Q、STP、RSTP、MSTP、Eth-Trunk、LACP、iStack、CSS、VRRP、BFD | Access/Trunk/Hybrid Tag剥除压入、P/A快速协商、MSTI多实例、跨框聚合免STP、BFD毫秒级故障感知 | HCIP |
| **第四篇：基础网络服务与安全** | 卷15 ~ 18 | DHCP、WLAN (Wi-Fi 6)、NAT、PPP、状态检测防火墙、VRF、IPsec VPN、Zero Trust、SASE | DHCP四步时序/中继、OFDMA/MU-MIMO、Easy-IP、NAT Server、安全区域与会话表、多租户切片、ZTNA | HCIP |
| **第五篇：IPv6 新一代演进** | 卷19 ~ 21 | IPv6 编址、NDP、SLAAC、OSPFv3、IPv6 IS-IS、BGP4+、双栈、IPv6隧道、NAT64 | 定长40B报头、被请求节点组播、RS/RA无状态配置、多拓扑路由、6to4隧道、DNS64/NAT64转换 | HCIP / HCIE |
| **第六篇：MPLS 与承载骨干** | 卷22 ~ 24 | MPLS、LDP、BGP/MPLS IP VPN、跨域 Option A/B/C、NNI 接口 | 32位标签头、PHP次末跳弹出、VRF/RD/RT体系、双层标签流转、MP-EBGP跨域交换、NNI标准划界 | HCIE |
| **第七篇：云网前沿与高级特性** | 卷25 ~ 32 | QoS、VXLAN、BGP EVPN、Segment Routing (SR-MPLS)、SRv6、Fabric、业务随行、SD-WAN | CBQ队列(LLQ/EF/AF/BE)、WRED防全局同步、srTCM/trTCM令牌桶、MAC-in-UDP、Type 1~5路由协同、TI-LFA 100%保护、128位SID、PnP开局、ZTP邮件开局、SLA智能选路 | HCIE 核心 |
| **第八篇：网络自动化与机试** | 卷33 ~ 35 | Python 3、Paramiko、SFTP、NETCONF、YANG、ncclient、ThreadPoolExecutor | 命令文本流式解耦、`**kwargs`字典解包、面向对象类建模、针对性异常隔离、线程池高并发、SSH交互终端、XML/RPC事务回滚 | NetDevOps / HCIE机试 |
| **第九篇：系统级 PRD 工程设计** | 4 份系统 PRD | SD-WAN 解决方案、自动化运维脚本系统、大中型虚拟化园区网、全栈总纲 PRD | 遵循工业级 7 大硬性章节架构，包含详细功能需求清单 (FR) 与非功能需求 (NFR) | 架构师 / 售前专家 |
| **第十篇：附录与实战军火库** | 5 份实战附录 | 数通高频 100 问、组播技术 (IGMP/PIM)、全球互联 (Transit/Peering/NNI/SASE)、华为云网融合与算力网络 (IPv6+/SRv6/无损以太)、一页速查表 | 100 原题问答、Gao-Rexford 商业准则、RoCEv2 零丢包、8大高密协议卡片与TOP15排障命令 | 全场景速查 / 现场突击 |

---

## 核心配置与实战实验清单

1. **二层与局域网高可用**：
   - 华为 VRP Hybrid 接口多 VLAN 隔离互通实战；
   - MSTP 多实例 + VRRP 双网关负载分担联动配置；
   - LACP 动态链路聚合与 iStack 堆叠配置。
2. **企业动态路由与复杂控制**：
   - OSPF 多区域划分、虚链路（Virtual-Link）与特殊区域汇总；
   - OSPF 与 IS-IS 双向引入的“四色打标”防环工程模板；
   - BGP 路由反射器（RR）部署与路径属性（Local-Pref / AS-Path / MED）精细化调优。
3. **MPLS 骨干网与跨域专线**：
   - MPLS LDP 基础标签转发配置与 L3VPN 双层标签抓包验证；
   - MPLS VPN 跨域 Option A（背靠背）、Option B（单跳 MP-EBGP）与 Option C 配置实操。
4. **云网 Overlay 与新一代承载**：
   - 静态与动态 EVPN VXLAN 集中式/分布式网关手搓开局；
   - IS-IS for SR-MPLS 开启、Node-SID 分配与 TI-LFA 备份路径验证；
   - SRv6-BE 极简承载与 EVPN over SRv6 综合实验；
   - 业务随行安全组划分、策略矩阵下发与 VXLAN 报头 Group ID 抓包。
   - 智算中心 AIDC 无损以太网（RoCEv2 / SPFC / AFR）与 IPv6+ 前沿特性（APN6 / IFIT / SFC 代理 / vBNG CU 转控分离）工程架构演进。
5. **Python 自动化与 NetDevOps**：
   - 外部命令文本解耦读取与批量交换机并发巡检脚本；
   - 交换机配置文件每日定时 SFTP 归档归档；
   - 基于 `ncclient` 的 NETCONF 自动化时钟同步与 Syslog 配置下发。
6. **全栈协议与工程配置速查**：
   - 物理/链路/VLAN/STP/LACP 参数速查表；
   - OSPF / IS-IS / BGP 三大动态路由全维对照与 13 条选路口诀；
   - ACL / Prefix / Route-Policy / Community 路由控制矩阵；
   - DHCP / NAT / VRRP / BFD / IPsec 基础网络服务配置清单；
   - MPLS / EVPN / SRv6 / RoCEv2 协议栈与报头封装速查；
   - 华为 VRP TOP15 黄金排障命令与协议卡死状态一键救援指南（详见 [附录E](10_第十篇_附录与数通实战军火库/附录E_数通全栈协议与工程配置一页速查表.md) 与 [CHEATSHEET.md](CHEATSHEET.md)）。
