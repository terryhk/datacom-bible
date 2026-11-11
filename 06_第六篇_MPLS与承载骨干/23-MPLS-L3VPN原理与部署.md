# 第23章 MPLS L3VPN原理与部署

> MPLS VPN 的核心魔法：公网不维护私网路由，却能为私网提供转发服务。PE 用 VRF 隔离客户、RD 区分路由、RT 控制收发；MP-BGP 传 VPNv4 路由自动迭代 LSP 隧道；双层标签（外层 LDP 公网+内层 BGP 私网）让报文穿越公网时 P 设备零感知。本章五讲从"为什么需要"到"怎么部署"到"怎么排错"——Sham Link 解决后门链路一类 LSA 无法跨骨干传递、Hub-Spoke 星型 RT 规划、DN Bit/Route Tag 双保险防环、TTL 管道模式隐藏公网、跨域 OptionA 背对背接力。

| 项 | 内容 |
|---|---|
| **适用层级** | HCIE MPLS L3VPN 核心（IE 实验考试绝对重点） |
| **架构定位与先修** | 全书第 23 章，先修第 17 章（VRF）、第 22 章（MPLS+LDP）；本章是第 24 章（跨域）的直接前置 |
| **技术维度** | 协议底层机理剖析 · 生产级部署规范 · 故障排错实践 · 考题与面试标准 |

## 一、总起

MPLS L3VPN 把三个独立技术组合成一套完整的私网互联方案：VRF（第 17 章 已学，接口+路由表+路由协议三隔离）解决本地多客户共存；MP-BGP VPNv4（RD 区分+RT 控制收发）解决私网路由跨公网传递；LDP 双层标签（外层公网隧道+内层私网标识）解决报文穿越 P 设备时实例归属识别。三者缺一不可。本章五讲递进：第一讲立架构（PE/P/CE 角色、单公司到多公司、VPN 实例+RD+RT 三件套），第二讲钻转发（双层标签机制、私网标签分配、RR 反射器、OSPF 扩展属性 Domain-ID/Route Type/Router ID、双归防环 DN Bit+Route Tag），第三讲搭双归实验（PE-CE 协议选择、关键节点验证法），第四讲攻难点（后门链路五类/三类/一类场景、Sham Link 伪连接、Hub-Spoke RT 规划），第五讲收尾部署（Hub-Spoke 实战、TTL 统一/管道模式、Traceroute 移花接木、跨域 OptionA 背对背）。

**适用人群**：
- **初学 / 转行人员（HCIP/IE 进阶）**：理解运营商公网不维护私网路由却能转发私网流量的本质，掌握 PE/P/CE 角色模型与 VRF+RD+RT 三件套；
- **备考工程师（HCIE 核心）**：攻克外层 LDP 隧道与内层 MP-BGP 私网双层标签转发机制、Domain-ID 影响 LSA 转换、DN Bit 与 Route Tag 双归防环、以及 Sham Link 伪连接；
- **一线工程实践者**：掌握大型企业广域网 Hub-Spoke 星型组网、双 PE 冗余接入、以及 TTL 管道模式隐藏运营商骨干拓扑的端到端交付标准。

**本章技术脉络与核心架构**：

| 模块节次 | 核心技术要点与工程主题 |
|---|---|
| 23-A-01 | VPN 本质=隧道、PE/P/CE 架构、单公司到多公司（VRF+RD+RT 三件套）、VPNv4 路由传递流程 |
| 23-A-02 | 双层标签转发机制、私网标签分配（基于路由/基于实例）、RR+`undo policy vpn-target`、OSPF 扩展属性、Domain-ID 对 LSA 类型影响、DN Bit+Route Tag 双归防环 |
| 23-A-03 | PE-CE 协议选择（不能用 IBGP）、双归实验拓扑搭建、关键节点验证法、考试策略 |
| 23-B-01 | 后门链路五类/三类/一类场景与开销控制、Sham Link 伪连接原理与配置、隐藏机制（建立后比开销不比优先级）、Hub-Spoke RT 规划 |
| 23-B-02 | Hub-Spoke 实战（双实例 in/out）、BGP 优先级 255 天然防环、MCE `vpn-instance-capability simple`、TTL 统一/管道模式、Traceroute 移花接木、跨域 OptionA 背对背 |

## 二、分述

### 2.1 VPN 本质与 MPLS VPN 架构

**VPN = 隧道技术**：对私网报文重新封装，利用新的包头或标签头转发。【设计本源与技术哲学】："万变不离其中，别跟我说什么 SR、MPLS VPN、IPSec VPN、L2TP VPN，基本上都是这种套路去实现的。"

**GRE 痛点 → LSP 优势**：GRE 静态隧道站点多了手动维护海量隧道口；LSP 纯天然动态隧道，IGP+LDP 配好自动建立。

**MPLS VPN 三层架构**：
| 角色 | 职责 |
|---|---|
| PE（Provider Edge） | 业务接入：VPN 实例、MP-BGP、PE-CE 路由协议 |
| P（Provider） | 纯标签转发，不维护私网路由 |
| CE（Customer Edge） | 用户出口路由器，跑普通 IGP/BGP |

**公网三件套**：IGP 做通 → LDP 建 LSP → MP-BGP 传私网路由。环回口必须 32 位主机路由（LDP 默认 host 触发策略）。

### 2.2 VRF + RD + RT 三件套（★核心）

**问题 1：本地路由冲突** → **VRF**：VPN 实例独立地址空间/路由协议/路由表。`ip binding vpn-instance A` 绑定接口（绑定即清 IP 要重配）。

**问题 2：BGP 中无法区分** → **RD**：附加到 IPv4 前缀组成 VPNv4 地址族。RD 规划两种方式：每站点全局唯一（最安全）或同 VPN 同 RD。★RD 只区分不同 VPN 的路由，不解决同一 VPN 内部地址冲突。

**问题 3：VPNv4 路由该放入哪个实例** → **RT**：BGP 团体属性。Export RT 随路由发出，Import RT 做接收判断（或逻辑，一个共同值即可）。RT 可配多个灵活组网：全互联用同一 RT；Hub-Spoke 各自不同 RT。

**VPNv4 路由传递流程**：
- 发送侧：IGP 路由→引入/network 进 VPN 实例 IPv4 BGP 表→自动变 VPNv4（加 RD）→MP-BGP 传出
- 接收侧：收到 VPNv4→匹配 Import RT→自动放入对应实例 BGP 表→引入 IGP→传 CE

【设计本源与技术哲学】："很多新手刚开始学 MPLS VPN 在这一块都很模糊，当年我也是很模糊，现在你们不会模糊了，一步到位。"

### 2.3 双层标签转发机制（★精华）

**问题**：报文封装进公网 LSP 到达对端 PE，PE 弹出公网标签后收到纯 IP 报文——**不知道属于哪个 VPN 实例**（IP 报文不携带 RD）。

**解决方案**：MP-BGP 传递 VPNv4 路由时，发送方 PE 为每条路由分配**私网标签**（内层）。报文打两层标签：外层（LDP 公网标签，建 LSP 隧道）+ 内层（BGP 私网标签，标识 VPN 实例）。

**转发流程**（以 R4→R1 为例）：
1. R4 查 VPN 实例 FIB → 私网标签 1027 + 公网标签 1024
2. 先打私网标签再打公网标签，从接口发出
3. P 设备只 Swap 外层公网标签（不看内层）
4. R1（Egress）弹出公网标签→查私网标签 1027→查 BGP LSP→弹出→查 A1 路由表→纯 IP 转发

★**每台路由器只根据顶部标签做转发决策**——"不关心第二第三标签是什么"。

**私网标签分配方式**：
| 方式 | 行为 |
|---|---|
| 基于路由（默认） | 每条私网路由一个标签，一对一 |
| 基于实例（`label-space per-instance`） | 一个 VPN 实例所有路由共享一个标签（省标签，精度降低） |

**TTL 处理**：打两层标签时 TTL 从 IP 复制到内层再到外层（向上繁殖）。

### 2.4 路由反射器（RR）与 `undo policy vpn-target`

**RR 作用**：站点多时 PE 间全互联 TCP 会话太多，用 RR 简化。

**★关键命令 `undo policy vpn-target`**（BGP-VPNv4 地址族下）：
| 场景 | 行为 |
|---|---|
| RR 自身无 VPN 实例 | 默认不收任何 VPNv4 路由→必须 `undo` 无条件接收 |
| PE 有 VPN 实例 | 默认仅接收 RT 匹配的路由→节省 PE 内存 |

### 2.5 OSPF 在 MPLS VPN 中的扩展

三个扩展团体属性随 VPNv4 路由传递：
| 属性 | 含义 |
|---|---|
| **Domain-ID** | 默认 0.0.0.0，相同=同一 AS，不同=不同 OSPF 网络 |
| **Route Type** | 区域号+LSA 类型（1/2/3/5/7）+开销类型 |
| **Router ID** | PE 在 OSPF 进程下的 Router ID |

**Domain-ID 对 LSA 类型的影响**：
| 原始 LSA | Domain-ID 相同 | Domain-ID 不同 |
|---|---|---|
| 1类/2类 | 变3类 | 变5类 |
| 3类 | 仍3类 | 变5类 |
| 5类 | 仍5类 | 仍5类 |
| 7类 | 变5类 | 变5类 |

**开销传递**：OSPF 开销转 VPNv4 时 MED=原始开销+1，对端引入 OSPF 继承 MED。

### 2.6 双归防环机制

**环路场景**：CE 双归连两台 PE，OSPF↔BGP 双点双向重引入。

| 机制 | 适用 | 原理 |
|---|---|---|
| **DN Bit** | 3类/5类/7类 LSA | PE 将 VPNv4 路由转 LSA 时 DN=1，收到 DN=1 的 PE 不计算路由 |
| **Route Tag** | 5类/7类 LSA | 默认 Tag=0xD000+AS 号，收到 Tag 与本地相同的 PE 不计算 |

5/7 类有双重防环，DN 被 disable 则 Route Tag 作后手。

★**ABR 清零 DN Bit 的坑**：ABR 将区域 0 的 3 类 LSA 转到非骨干区时 DN Bit 被清零→双归场景下骨干不连续→**做虚链路解决**。

### 2.7 PE-CE 协议选择

四种可用：OSPF、EBGP、IS-IS、思科私有协议。静态路由不在内（无路由传递能力）。

★**不能用 IBGP**：VPNv4 路由进入 VPN 实例 BGP 表后仍是 IBGP 路由，IBGP 水平分割导致无法继续传给 PE 的 IBGP 邻居。

### 2.8 后门链路与 Sham Link（★本章最大难点）

**后门链路**：连接两站点的低带宽备份链路，也在 OSPF Area 0。希望流量优先走 MPLS VPN 骨干网。

**三类场景与解法**：
| LSA 类型 | 骨干传来 | 后门泛洪 | 改开销有效？ |
|---|---|---|---|
| 5类 | 5类 | 5类 | ✅ 比开销即可 |
| 3类 | 3类（Domain-ID 相同） | 3类 | ✅ 比开销 |
| **1类** | 变3类 | 1类 | ❌ 一类优于三类，不比开销 |

**一类 LSA 的核心困难**：BGP 无法将一类 LSA（链路状态信息）携带到 VPNv4 再还原——"改开销没有办法让他去走骨干"。

**Sham Link（伪连接）**：在两台 PE 的 VPN 实例中各建一个 Loopback（绑定 VPN 实例，通过 BGP 宣告**不进 OSPF**），利用公网 LSP 隧道建立 OSPF 邻居。建立后一/二/三/四/五类 LSA 均可通过它泛洪——相当于在 PE 间多了一条"物理链路"。

**四步配置**：
1. 后门链路开销改大
2. 两台 PE 各新建 Loopback 绑 VPN 实例，BGP 宣告（不进 OSPF）
3. `sham-link <源> <目标>` 建立（VPN 实例 OSPF 区域下）
4. 纯 OSPF 场景无需再 import BGP 到 OSPF

★**关键约束**：Sham Link 地址**绝对不能加入 OSPF 进程**否则邻居震荡；必须用 /32 主机地址。

★**隐藏机制**：Sham Link 建立且开销小于后门→不再比 BGP/OSPF 优先级，直接比开销→之前改的优先级命令可删除恢复默认 255。

### 2.9 Hub-Spoke 星型组网

**RT 规划**：
- 分支 Export RT 各不同（1:2、1:3…）；总部 `in` 实例 Import 全部分支 RT
- 总部 `out` 实例 Export 统一 RT（1:10）；分支 Import 1:10
- 效果：分支↔总部互通，分支间不互通

**总部双实例架构**：`in` 实例收分支路由→绕经 CE→`out` 实例重新发布（统一 Export RT）。AS 号防环：`peer allow-as-loop`。

**BGP 优先级 255 的天然防环**：自己站点发出又从总部绕回的路由，BGP 优先级 255 不如本地 OSPF 优先级 10→不会路由震荡。"BGP 为什么优先级默认 255 最高？就是在这些场景下天然帮你解决很多麻烦。"

**MCE 场景**：`vpn-instance-capability simple` 一键解决 DN Bit+Route Tag 双保险限制。

### 2.10 TTL 处理模式与 Traceroute

| 模式 | 行为 | 用途 |
|---|---|---|
| **统一（Uniform，默认）** | IP TTL 复制到标签 TTL，标签转发减标签 TTL | 用户可探测公网路径 |
| **管道（Pipe）** | `undo ttl propagate`，标签 TTL 固定 255 与 IP 分离 | 运营商隐藏公网结构 |

**Traceroute 移花接木**：TTL 超时发生在 P 设备，P 无私网路由→两类回应：①有发送者路由直接 IP 回应；②无发送者路由沿 LSP 继续到对端 PE，由 PE 查 VPN 实例绕回。"移花接木——先烈死了牺牲了，咱们继承先烈的遗志继续发。"

### 2.11 跨域 OptionA（背对背）

**核心思路**：两个 AS 的 ASBR 各自创建 VPN 实例，ASBR 之间传**普通 IPv4**（VPNv4→IPv4→VPNv4 转换）。

**控制平面**：R2 发 VPNv4→R4(ASBR) 收到变 IPv4→ASBR 间 EBGP 传 IPv4→R5(ASBR) 重新封装变 VPNv4→传 R7。

★**关键规则**：VPNv4 路由只要下一跳改变，私网标签就要重新生成。

**转发平面**：R7 打私网标签→R5 弹出查 VPN 实例→ASBR 间**纯 IP 无标签**→R4 查 VPN 实例重新打私网标签→单域转发。

**优缺点**：两运营商 RD/RT 独立规划；ASBR 需维护大量私网 IPv4 路由，客户增加配置膨胀——"一学就会，一练就废"。

### 2.12 配置命令速查

```bash
# VPN 实例 + RD/RT
ip vpn-instance A
 route-distinguisher 1:1
 vpn-target 1:100 both

# PE-CE OSPF（绑定 VPN 实例）
ospf 1 router-id 1.1.1.1 vpn-instance A
 area 0
  network 10.1.14.1 0.0.0.0

# MP-BGP VPNv4
bgp 100
 peer 4.4.4.4 as-number 100
 peer 4.4.4.4 connect-interface LoopBack0
 ipv4-family vpnv4
  peer 4.4.4.4 enable
  undo policy vpn-target              # RR 上必须配
 ipv4-family vpn-instance A
  import-route ospf 1                 # 或 network

# RR
 ipv4-family vpnv4
  peer 2.2.2.2 reflect-client
  peer 3.3.3.3 reflect-client

# Sham Link
interface LoopBack10
 ip binding vpn-instance A
 ip address 10.10.10.10 32
bgp 100
 ipv4-family vpn-instance A
  network 10.10.10.10 32
ospf 1 vpn-instance A
 area 0
  sham-link 10.10.10.10 30.30.30.30

# Hub-Spoke AS 号防环
 ipv4-family vpn-instance out
  peer 10.1.21.1 allow-as-loop

# MCE 一键解决 DN+Route Tag
ospf 1 vpn-instance C
 vpn-instance-capability simple

# TTL 管道模式
undo ttl propagate

```shell
# ═══ 1. PE 节点经典完整配置模板 ═══
# Step 1: 创建 VRF 并配置 RD 与 RT
[PE1] ip vpn-instance VPNA
[PE1-vpn-instance-VPNA] route-distinguisher 100:1       # RD 区分符（将 IPv4 扩展为 VPNv4）
[PE1-vpn-instance-VPNA] vpn-target 100:1 both           # Export 与 Import RT 均为 100:1
[PE1-vpn-instance-VPNA] quit

# Step 2: 接入客户 CE 的物理接口绑定 VRF
[PE1] interface GigabitEthernet 0/0/2
[PE1-GigabitEthernet0/0/2] ip binding vpn-instance VPNA # ★绑定瞬间清空已有 IP，必须重新配置！
[PE1-GigabitEthernet0/0/2] ip address 10.1.12.1 24
[PE1-GigabitEthernet0/0/2] quit

# Step 3: 建立 PE 间 MP-IBGP VPNv4 对等体
[PE1] bgp 100
[PE1-bgp] peer 3.3.3.3 as-number 100                    # 对端 PE 的 LoopBack0
[PE1-bgp] peer 3.3.3.3 connect-interface LoopBack0
[PE1-bgp] ipv4-family vpnv4                             # 进入 VPNv4 地址族
[PE1-bgp-af-vpnv4] peer 3.3.3.3 enable                 # 激活对端 PE 的 VPNv4 能力
[PE1-bgp-af-vpnv4] quit

# Step 4: PE-CE 间运行 OSPF 并与 MP-BGP 双向引入
[PE1] ospf 2 vpn-instance VPNA                          # 运行在 VPN 实例下的 OSPF 多实例
[PE1-ospf-2] import-route bgp                           # 将 BGP 学到的远端私网路由引入 OSPF 传给本地 CE
[PE1-ospf-2] area 0
[PE1-ospf-2-area-0.0.0.0] network 10.1.12.0 0.0.0.255
[PE1-ospf-2-area-0.0.0.0] quit
[PE1-ospf-2] quit
[PE1] bgp 100
[PE1-bgp] ipv4-family vpn-instance VPNA
[PE1-bgp-VPNA] import-route ospf 2                      # 将本地 CE 的 OSPF 路由引入 BGP 形成 VPNv4 路由通告远端

# ═══ 2. 状态验证与双层标签排错 ═══
<PE1> display bgp vpnv4 all routing-table               # 查看所有 VPNv4 路由、RD、RT 及分配的内层私网标签
<PE1> display ip routing-table vpn-instance VPNA        # 查看特定客户实例的私网单播路由表
<PE1> display fib vpn-instance VPNA 192.168.2.1         # 查看数据转发时压入的双层标签（外层 LSP + 内层 VPN）
```

## 三、总结

**核心重点十带走**：
1. MPLS VPN = VRF（本地隔离）+ RD（区分路由）+ RT（控制收发）+ MP-BGP（传 VPNv4）+ LDP 双层标签（公网隧道+私网标识），五件套缺一不可。
2. 双层标签：外层 LDP 公网标签建 LSP 隧道，内层 BGP 私网标签标识 VPN 实例——P 设备只看外层 Swap，PE 弹外层后查内层确定实例。
3. VPNv4 路由自动迭代 LSP 隧道；传统 IPv4 BGP 需 `tunnel-policy` + `ip route recursive-lookup tunnel`。
4. RR 必须 `undo policy vpn-target` 无条件接收 VPNv4 路由；PE 默认按 RT 过滤节省内存。
5. OSPF 扩展属性 Domain-ID 决定 LSA 类型转换：相同=1/2 类变 3 类，不同=全部变 5 类；直连路由转 VPNv4 不带扩展属性变 5 类（华为特点）。
6. 双归防环：DN Bit（3/5/7 类 LSA，PE 引入时置 1）+ Route Tag（5/7 类，0xD000+AS 号）双保险；ABR 清零 DN Bit 的坑用虚链路解决。
7. PE-CE 不能用 IBGP（水平分割）；四种可用：OSPF/EBGP/IS-IS/思科私有。
8. 后门链路：5类/3类改开销即可控制，1类改开销无效（BGP 无法携带一类 LSA）→ Sham Link 伪连接在 PE 间建 OSPF 邻居传所有 LSA。
9. Sham Link 地址绝不进 OSPF（否则邻居震荡），用 /32 主机地址；建立后比开销不比优先级（隐藏机制）→改的优先级命令可删除。
10. Hub-Spoke：总部双实例 in/out，分支间不互通（RT 过滤），BGP 优先级 255 天然防环；MCE 用 `vpn-instance-capability simple` 一键解双保险；OptionA 跨域 ASBR 间纯 IPv4，下一跳变则私网标签重生成。

**落地注意事项**：环回口必须 32 位主机路由（LDP host 触发策略）；`ipv4-family vpnv4` 下 `peer enable` 双方都要配；VPN 场景禁止汇总环回口路由（FEC 变化 LSP 断裂）；Sham Link 地址必须用 /32 且不进 OSPF；Hub-Spoke 的 `in` 实例不设 export RT（否则无意义路由消耗 RR 内存）；`out` 实例不设 import RT（只发不收）；ping 时必须带 VPN 实例名 `ping -vpn-instance xxx -a x.x.x.x`；`undo ttl propagate` 需重置 LSP 生效。

**学习建议**：MPLS L3VPN 的学习核心在"控制平面每台设备的角色和操作必须能在脑中盘熟"——R2 是 PE 负责 VPNv4 生成、R4 负责转换、R5 负责重新封装……每一步在脑中走通，排错就有方向。实验按"单公司→多公司（VRF+RD+RT）→RR→双归防环→后门+Sham Link→Hub-Spoke→跨域 OptionA"顺序走，每步验证关键现象（FIB Tunnel ID、BGP VPNv4 表、MPLS LSP、OSPF LSDB）。Sham Link 是本章最大难点——在纸上画出"R6→R3→Sham Link→R1→R4→45.0"的完整路径（每跳标注查哪张表/哪个实例），画通即理解。OptionA 跨域在纸上画出"VPNv4→IPv4→VPNv4"的标签变化链（每跳标注私网标签重生成），画通即毕业。学完本章即可进第 24 章（跨域 OptionB/C）。

## 附：本章术语速查表

| 术语缩写 | 英文全称 | 中文含义 | 关键作用 |
|---|---|---|---|
| **RD** | Route Distinguisher | 路由区分符 | 8 字节标识符，与 32 位 IPv4 地址拼接生成全局唯一的 96 位 VPNv4 路由 |
| **RT** | Route Target | 路由目标（VPN Target） | BGP 扩展团体属性，分为 Export 与 Import，控制 VPN 路由的发布与接收过滤 |
| **MP-BGP** | Multi-Protocol BGP | 多协议 BGP | 扩展支持传递 IPv4/IPv6 单播以外多种地址族（如 VPNv4、EVPN）路由的 BGP 协议 |
| **Sham Link** | Sham Link | 伪连接 | 在两台 PE 的 VPN 实例之间建立的逻辑链路，使跨骨干私网流量呈现为 OSPF 区域内路由 |
| **MCE** | Multi-VPN-Instance CE | 多实例客户边缘设备 | 允许单一 CE 设备逻辑切分多个 VRF 接入不同业务 VPN，支持 `vpn-instance-capability simple` |
