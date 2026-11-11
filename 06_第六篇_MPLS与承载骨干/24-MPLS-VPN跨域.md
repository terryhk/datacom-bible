# 第24章 MPLS VPN跨域

> 当 MPLS VPN 需要跨越多个运营商 AS 时，三种 Option 方案各有取舍：OptionA 背对背最简单但 ASBR 负担重（"一学就会一练就废"），OptionB ASBR 交换标签但需维护私网路由，OptionC ASBR 零私网路由负担——私网标签由 PE 一通到底。本章四讲完整覆盖 A→B→C1→C2 四种方案的控制平面与转发平面，核心思路："控制平面花样多，转发平面一层不变——三层标签还是两层标签就这么过。"

| 项 | 内容 |
|---|---|
| **适用层级** | HCIE MPLS 跨域（IE 实验考试随机抽 C1 或 C2） |
| **架构定位与先修** | 全书第 24 章，先修第 23 章（L3VPN 原理与部署）；本章是 MPLS VPN 体系的天花板 |
| **技术维度** | 协议底层机理剖析 · 生产级部署规范 · 故障排错实践 · 考题与面试标准 |

## 一、总起

跨域 MPLS VPN 的核心挑战：两个不同 AS 的运营商如何传递 VPNv4 路由和转发私网报文？关键在于 ASBR 之间如何衔接——每种方案在"ASBR 维护多少私网信息"和"标签层数"上做不同取舍。OptionA 最原始：ASBR 之间纯 IPv4 背对背，VPNv4→IPv4→VPNv4 转换，简单但配置膨胀；OptionB 进化：ASBR 之间交换 VPNv4 标签路由（一层标签），减少配置但仍需维护私网路由；OptionC 终极：ASBR 之间只建 BGP LSP（传 PE 环回口路由的标签），VPNv4 路由由 RR 间 MP-EBGP 传递且下一跳不变——ASBR 零私网路由负担，私网标签由 PE 一通到底。OptionC 分 C1（BGP LSP 贯穿全路径，域内三层标签）和 C2（BGP LSP 仅 ASBR 间存在，域内 LDP 接管，全程两层标签）。本章四讲递进：第一讲 OptionA 背对背，第二讲 OptionB 标签交换，第三讲 OptionC1 三层标签与 RR 优化，第四讲 OptionC2 两层标签衔接。

**适用人群**：
- **初学 / 转行人员（HCIE 进阶）**：理解跨越多个运营商 AS 传递 VPN 业务报文的底层挑战，搞懂 ASBR 边界衔接三种设计思路；
- **备考工程师（HCIE 核心）**：攻克跨域 Option A（背对背）、Option B（单层标签交换）、Option C1（BGP LSP 全贯穿/三层标签）与 Option C2（域内 LDP 接管/全程两层标签）的控制平面与转发平面深度差异，掌握机试/LAB必考命令；
- **一线工程实践者**：掌握大型跨国企业跨多运营商互联架构选型矩阵、ASBR 性能瓶颈规避、以及 RR 路由反射器端到端标签路径排错。

**本章技术脉络与核心架构**：

| 模块节次 | 核心技术要点与工程主题 |
|---|---|
| 24-A-01 | OptionA 背对背：ASBR 间 VPNv4→IPv4→VPNv4，纯 IP 无标签，配置膨胀 |
| 24-A-02 | OptionB：ASBR 间交换 VPNv4 标签路由（一层标签），私网标签重新生成 |
| 24-A-03 | OptionC1：BGP LSP 贯穿全路径（三层标签），RR 优化+下一跳不变策略，ASBR 零私网路由 |
| 24-B-01（混合课取跨域部分） | OptionC2：BGP LSP 仅 ASBR 间存在，域内 LDP 接管（两层标签），`bgp-lsp-router` 衔接 |

## 二、分述

### 2.1 OptionA（背对背方案）

**核心思路**：两个 AS 的 ASBR 各自创建 VPN 实例，ASBR 之间传**普通 IPv4**（VPNv4→IPv4→VPNv4 转换）。

**控制平面**：R2(PE) 发 VPNv4 → R4(ASBR) 收到变 IPv4 → ASBR 间 EBGP 传 IPv4 → R5(ASBR) 重新封装变 VPNv4 → 传 R7(PE)。

★**关键规则**：VPNv4 路由只要下一跳改变，私网标签就要重新生成——下一跳即包文转发目的地，该设备必须为新路由重新分配私网标签。

**转发平面**：R7 打私网标签 → R5 弹出查 VPN 实例 → ASBR 间**纯 IP 无标签** → R4 查 VPN 实例重新打私网标签 → 单域转发到 R2。

**优缺点**：两运营商 RD/RT 独立规划；ASBR 需维护大量私网 IPv4 路由，客户增加配置膨胀——"一学就会，一练就废"。

### 2.2 OptionB（标签交换方案）

**核心思路**：ASBR 之间交换 VPNv4 标签路由（一层标签），ASBR 不再需要为每个客户创建 VPN 实例。

**控制平面**：ASBR 间 MP-EBGP 传 VPNv4 路由 + 私网标签。ASBR 收到 VPNv4 路由后为下一跳重新生成私网标签（因为下一跳改变了）。

**转发平面**：ASBR 间一层标签（VPNv4 私网标签），不同于 OptionA 的纯 IP 和 OptionC 的两层标签。

**优缺点**：比 OptionA 减少了 VPN 实例和子接口配置；但 ASBR 仍需维护私网路由并为私网路由重新产生私网标签。

### 2.3 OptionC1（BGP LSP 贯穿方案）

**核心思路**：ASBR 不维护任何 VPNv4 私网路由和私网标签——私网标签由 PE 自己产生，一通到底。BGP LSP 贯穿全路径。

**控制平面两步走**：
1. **构建跨域 BGP LSP**：ASBR 之间 EBGP 传 PE 环回口路由的 IPv4 标签路由（`label-route-capability` + `apply mpls-label`）；对端 ASBR 收到后用 `if-match mpls-label` + `apply mpls-label` 重新产生标签传给 iBGP 邻居。
2. **VPNv4 路由传递**：PE 间通过 RR 反射 MP-BGP VPNv4 路由，RR 间建 MP-EBGP。★**下一跳不变策略**（`next-hop-invariable`）：RR 传路由但下一跳不变，效果与 PE 直传一致——"没有中间商赚差价"。

**转发平面（三层→两层标签）**：
- R6 出发三层标签：私网标签（BGP VPNv4）+ BGP LSP 标签（去往对端 PE）+ LDP 标签（域内）
- 域内 LDP LSP 送包到 ASBR；ASBR 之间两层标签（BGP LSP + 私网标签）；对端域内又是三层
- ASBR 做 BGP LSP 标签 Swap；对端 ASBR 弹出 BGP LSP 标签换乘 LDP LSP

**有 RR 场景关键配置**：
- 两个 RR 之间只建 VPNv4 邻居，**关闭 IPv4 邻居**（`undo peer x.x.x.x`）——否则 IPv4 路由绕回来被 AS 号防环丢掉
- 所有 RR→PE 和 RR→对端 RR 方向都做 `next-hop-invariable`
- 公网 BGP LSP 构建方式与无 RR 完全一致

### 2.4 OptionC2（BGP LSP 仅 ASBR 间方案）

**C2 与 C1 的核心区别（★强调）**：BGP LSP **仅在 ASBR 之间建立**。ASBR 把 EBGP 路由引入到 IGP 协议中，再利用 LDP 协议为引入的 BGP 路由生成 LSP——域内全由 LDP 包办。

**类比**：C1 是整条路用 BGP 隧道穿越（域内 LDP 封装 BGP 标签=三层标签），C2 是"公路—铁路—公路"直接拉平衔接——域内全由 LDP 包办，ASBR 之间才是 BGP LSP。

**ASBR 关键配置（C2 精髓）**：
```bash
# R3（ASBR）→ R4（对端 ASBR）
bgp 100
 peer 10.1.34.4 label-route-capability     # 激活 IPv4 标签传递
 ipv4-family unicast
  network 1.1.1.1 32                        # 发布 PE 环回口
  peer 10.1.34.4 route-policy asbr-permit export

route-policy asbr-permit permit node 10
 apply mpls-label                           # 产生 BGP LSP 标签

# R4（对端 ASBR）—— 两条核心命令
ospf 1
 import-route bgp                            # 引入带标签的 BGP 路由到 IGP
mpls
 bgp-lsp-router                              # 为带 BGP 标签的路由触发 LDP LSP
```

**标签转发平面（★全程两层标签）**：
- 私网标签 + 外层 LDP/BGP 标签
- **关键衔接点（ASBR）**：BGP LSP 的出标签 = LDP LSP 的入标签——LDP LSP 和 BGP LSP 在 ASBR 上直接拼接
- **代理出节点**：R3 不是 6.6.6.6 的直连路由器，但为 6.6.6.6 产生 LDP LSP，角色是"代理出节点"，分配显式标签（不是隐式空标签 3）

**C1 vs C2 标签层数对比**：
| 方案 | 域内 | ASBR 间 | 总层数 |
|---|---|---|---|
| OptionA | 1层(LDP) | 0层(纯IP) | 1层 |
| OptionB | 1层(LDP) | 1层(VPNv4) | 1-2层 |
| OptionC1 | 3层(LDP+BGP+私网) | 2层(BGP+私网) | 2-3层 |
| OptionC2 | 2层(LDP+私网) | 2层(BGP+私网) | **全程2层** |

### 2.5 三种方案对比总结

| 维度 | OptionA | OptionB | OptionC（C1/C2） |
|---|---|---|---|
| ASBR 间 | 纯 IPv4 | VPNv4 标签路由 | BGP LSP（PE 环回口标签） |
| ASBR 维护私网路由 | ✅ 全部 | ✅ 全部 | ❌ 零 |
| ASBR 维护私网标签 | ✅ 重新生成 | ✅ 重新生成 | ❌ 零（PE 一通到底） |
| 标签层数 | 0/1 | 1 | 2-3（C1）/ 2（C2） |
| 配置复杂度 | 低但膨胀 | 中 | 高但可扩展 |
| RR 优化 | 不适用 | 不适用 | ✅ RR 间 MP-EBGP + 下一跳不变 |

【设计本源与技术哲学】："控制平面上的花样是比较多的，但是转发平面花样就是统一的……控制平面是满足不同场景下解决实际问题的，转发平面上却是一层不变的。"

### 2.6 排错方法论与考试策略

**关键节点验证法**：每到一个关键节点验证一个关键现象——在 P 设备上检查能否看到对端 PE 环回口的 BGP LSP，能则公网 BGP LSP 正常。"跟医生治病一样——望闻问切，根据经验判断网络状态。"

**考试策略**：RS IE 实验考试随机抽 C1 或 C2；C2 配置量少很多但两者都需熟练掌握。考试中 90% 以上问题都是自己配置问题——"不是忘了就是漏了"。

**推荐书籍**：《Internet 路由结构》（讲 BGP 网络设计与实现）。

### 2.7 跨域四大方案核心配置与标签验证模板（VRP 实战）

#### 1. Option B：ASBR 间标签路由交换配置

```shell
# ═══ ASBR1 与 ASBR2 直连接口启用 MPLS ═══
[ASBR1] interface GigabitEthernet 0/0/1
[ASBR1-GigabitEthernet0/0/1] ip address 10.1.34.3 24
[ASBR1-GigabitEthernet0/0/1] mpls                       # ★接口只需开启 mpls 转发，绝不能使能 mpls ldp！
[ASBR1-GigabitEthernet0/0/1] quit

# ═══ ASBR 间建立 MP-EBGP VPNv4 对等体并关闭 RT 过滤 ═══
[ASBR1] bgp 100
[ASBR1-bgp] peer 10.1.34.4 as-number 200
[ASBR1-bgp] ipv4-family vpnv4
[ASBR1-bgp-af-vpnv4] undo policy vpn-target             # ★ASBR 没有配置 VRF，必须关闭 RT 匹配才能接收并转发 VPNv4
[ASBR1-bgp-af-vpnv4] peer 10.1.34.4 enable
[ASBR1-bgp-af-vpnv4] peer 10.1.34.4 next-hop-local      # 向内部 IBGP 通告时将下一跳改为自己（触发重新分配私网标签）
```

#### 2. Option C1 与 C2 核心配置差异对照

```shell
# ═══ Option C1: ASBR 间建立带标签的 EBGP（BGP LSP 全贯穿） ═══
# ASBR1 向 ASBR2 通告带标签的 PE 环回口路由
[ASBR1-bgp] peer 10.1.34.4 label-route-capability       # 开启传递带 MPLS 标签的 IPv4 路由能力
[ASBR1-route-policy] route-policy SET_LABEL permit node 10
[ASBR1-route-policy-node10] apply mpls-label            # 为通告的路由分配 BGP 标签
[ASBR1-bgp] ipv4-family unicast
[ASBR1-bgp-af-ipv4] peer 10.1.34.4 route-policy SET_LABEL export

# ═══ Option C2: ASBR 将 BGP 引入 IGP 并通过 LDP 接管 ═══
# ASBR2 上将对端 BGP 标签路由引入 OSPF
[ASBR2] ospf 1
[ASBR2-ospf-1] import-route bgp
[ASBR2-ospf-1] quit
# MPLS 视图下使能 bgp-lsp-router 触发 LDP 为该路由分配标签（全程两层标签）
[ASBR2] mpls
[ASBR2-mpls] bgp-lsp-router                             # ★C2 灵魂命令：由 LDP 为引入的 BGP 路由产生 LSP
[ASBR2-mpls] quit

# ═══ RR 反射器间下一跳不变策略（C1/C2 通用） ═══
[RR1-bgp-af-vpnv4] peer 5.5.5.5 next-hop-invariable     # ★向对端 RR 通告 VPNv4 时保持原始 PE 下一跳不变！
```

#### 3. 标签路径验证与排错命令

```shell
# 查看端到端公网 BGP LSP 与 LDP LSP 建立情况
<PE1> display mpls lsp
# 查看 VPNv4 路由及其分配的内层私网标签
<PE1> display bgp vpnv4 all routing-table
# 检查 FIB 表项中的双层/三层标签压栈深度
<PE1> display fib vpn-instance VPNA 192.168.2.1
```


### 2.6 跨运营商标准互联界面：NNI（Network-to-Network Interface）

在电信运营商骨干网跨域专线落地中，**NNI** 是连接不同运营商 ASBR 的标准物理与逻辑接口：
1. **网络层承载界面**：NNI 接口直接承载 Option B（单跳 MP-EBGP 传递标签路由）或 Option C（直通公网 LSP）的跨域数据流；
2. **SLA 与责任边界**：NNI 接口作为双方商业合同的分界点，借助以太网 OAM（802.1ag / 802.3ah）持续检测时延与误码率，明确故障定界与服务水平。

## 三、总结

**核心重点十带走**：
1. 跨域三种方案核心区分：ASBR 之间传什么（IPv4/VPNv4标签/BGP LSP）和 ASBR 维护多少私网信息（全部/全部/零）。
2. OptionA 背对背：VPNv4→IPv4→VPNv4 转换，ASBR 间纯 IP 无标签，每增客户需增 VPN 实例+子接口——"一学就会一练就废"。
3. OptionB：ASBR 间一层 VPNv4 标签，仍需维护私网路由并重新生成私网标签。
4. OptionC（C1/C2）：ASBR 零私网路由零私网标签——私网标签由 PE 产生一通到底，ASBR 只维护 BGP LSP（PE 环回口标签）。
5. C1：BGP LSP 贯穿全路径（域内逐跳 IPv4 BGP 产生标签），转发三层标签（LDP+BGP+私网）。
6. C2：BGP LSP 仅 ASBR 间存在，域内 LDP 接管（`import-route bgp` + `bgp-lsp-router`），全程两层标签。
7. C2 ASBR 衔接：BGP LSP 出标签 = LDP LSP 入标签，直接拼接而非封装穿越；代理出节点标签不为 3（非直连路由触发 LDP LSP）。
8. 有 RR 场景：RR 间 MP-EBGP 传 VPNv4，`next-hop-invariable` 保证下一跳不变——"没有中间商赚差价"。
9. 两个 RR 之间只建 VPNv4 邻居关闭 IPv4 邻居（`undo peer`），避免 IPv4 路由绕回被 AS 号防环丢掉。
10. VPNv4 路由只要下一跳改变私网标签就要重新生成——这是跨域所有方案的通用规则。

**落地注意事项**：C2 的 `bgp-lsp-router` 命令在 MPLS 视图下配置（不是 BGP 视图）；RR 间 `next-hop-invariable` 所有方向都要做（R2→R5、R5→R2、R5→R6、R2→R1）；iBGP 侧有了重新产生标签的策略后 `next-hop-unchanged` 不再需要；LDP session 没起来检查环回口是否加入了 LDP；RR 间邻居建不起来检查 `ebgp-max-hop` 是否配了。

**学习建议**：跨域的核心在标签层数——在纸上画出每种方案从源 PE 到目的 PE 的完整标签变化链（每跳标注标签操作：PUSH/SWAP/POP），标完即理解。C1 和 C2 的区别只在于"BGP LSP 在哪里终止"——C1 在 PE 终止（域内三层），C2 在 ASBR 终止（域内两层），画出两条标签链对比即毕业。实验按 A→B→C1→C2 顺序走，每种方案先验证公网 LSP 通（`display mpls lsp` 看到对端 PE 环回口），再验证私网 VPNv4 路由传递，最后 ping 验证端到端。C2 的 `bgp-lsp-router` 是核心命令——配完后在 ASBR 上 `display mpls lsp` 应看到 PE 环回口的 LDP LSP（不是 BGP LSP），这是 C2 与 C1 的关键区别。学完本章，MPLS VPN 体系（第 22 章-24）全部闭环。

## 附：本章术语速查表

| 术语缩写 | 英文全称 | 中文含义 | 关键作用 |
|---|---|---|---|
| **Option A** | Inter-AS Option A (Back-to-Back VRF) | 背对背跨域方案 | ASBR 之间通过物理口/子接口绑定 VRF 互联，纯 IPv4 转发，配置量随客户线性膨胀 |
| **Option B** | Inter-AS Option B (Single-Hop MP-EBGP) | 单跳标签交换跨域方案 | ASBR 之间运行 MP-EBGP 交换 VPNv4 标签路由，ASBR 间仅一层私网标签，免除 VRF 配置 |
| **Option C1** | Inter-AS Option C (End-to-End BGP LSP) | 端到端 BGP LSP 跨域方案 | ASBR 之间传 PE 标签路由形成 BGP LSP，域内三层标签，ASBR 零私网路由负担 |
| **Option C2** | Inter-AS Option C (LDP-over-BGP Stitching) | LDP 拼接 BGP LSP 跨域方案 | ASBR 将 BGP 引入 IGP 并由 LDP 触发分配标签，全网保持两层标签，架构极度优美 |
| **next-hop-invariable** | Next Hop Invariable | 下一跳保持不变策略 | RR 向对端 RR 反射 VPNv4 路由时维持源 PE 的下一跳不变，避免重新生成私网标签 |
