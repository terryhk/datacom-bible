# 第08章 BGP基础与路径优选

> 互联网的路由外交官：AS 之间的路径矢量协议——TCP 179 上的应用层设计、IBGP 水平分割与全互联、五大队列属性的传递与修改、RR 反射器、11 条选路规则的实战运用与负载分担全景。

| 项 | 内容 |
|---|---|
| **适用层级** | HCIP（IP 阶段压轴大项，IE 的 BGP 高级特性以其为基础） |
| **架构定位与先修** | 全书第 8 章，需先修《路由策略与路由控制》（route-policy/if-match/apply 语法全程复用） |
| **技术维度** | 协议底层机理剖析 · 生产级部署规范 · 故障排错实践 · 考题与面试标准 |

## 一、总起

IGP 管好一个 AS 的内部，BGP 负责 AS 之间"国与国"的路由外交。本章按课程节奏展开：先讲清为什么 IGP 干不了 AS 间的活（路由量、跨组织管理、路径控制），再建立 BGP 的全套装载（邻居、报文、状态机、路由表）；随后进入 BGP 的灵魂——**路径属性**：人为设计的"路由 KPI"，逐个拆 NextHop/AS_Path/Origin/Local_Pref/Community/MED/Preferred-Value 的传递规则与修改手法；用路由反射器解决 IBGP 全互联的扩展性瓶颈；最后把 11 条选路规则练成"闭眼预判"的实战能力，附一整场六路由器三 AS 的综合调优实验与负载分担命令矩阵。

**适用人群**：
- **初学 / 转行人员（HCIP 核心）**：理解 BGP 作为互联网外交官的应用层设计定位，掌握 EBGP 与 IBGP 对等体建邻与报文交互；
- **备考工程师（HCIP / HCIE 级）**：攻克 IBGP 水平分割与路由黑洞、路由反射器 RR（Originator-ID / Cluster-List 防环）、11 步选路规则闭眼预判与属性调优；
- **一线工程实践者**：掌握大型企业多出口选路控制、BGP 负载分担、双线机房防穿越与流量牵引的标准工业配置模板。

**本章技术脉络与核心架构**：

| 模块节次 | 核心技术要点与工程主题 |
|---|---|
| BGP基础1 | 协议背景、AS 号、TCP/应用层定位、EBGP/IBGP 建邻、五报文与 Open 协商 |
| BGP基础2 | Update/撤销、七状态机排错、BGP 表符号、Network/Import、通告四原则 |
| BGP路径属性与路由反射器1 | 五大传递规则、下一跳三规则与第三方优化、黑洞三解法、同步规则、属性分类 |
| BGP路径属性与路由反射器2 | AS_Path 防环与修改、四张数据库与策略引擎、Origin、Local_Pref 与 default 命令 |
| BGP路径属性与路由反射器3 | Community 公认四件套与挤牙膏、MED 规律、原子聚合/聚合者、协议首选值 |
| BGP路由优选1 | RR 三角色与反射规则、起源者 ID/簇列表防环、三架构、11 条选路规则 |
| BGP路由优选2 | 综合实验八需求（首选值/环回开销/次序坑/起源/MED/AS_Path 叠加）、负载分担全景 |

## 二、分述

### 2.1 BGP 的诞生与协议定位

**IGP 为什么干不了 AS 间的活**：①选路只看 cost，路径控制手段贫乏；②路由容量有限——OSPF 引一万条外部路由就一万条五类 LSA，设备扛不住；③用"引入+共管路由器"打通 AS 不现实——跨组织/运营商的设备共管天天扯皮，且双向引入会把一个 AS 的路由震荡传染全网。BGP 从 EGP 演化而来，专司 AS 间路由传递与策略控制。

**AS（自治系统）**：同一组织管理下、统一选路策略的设备集合。AS 号由 IANA 统一分配，经典 2 字节 1~65535（私有 64512~65534），后扩展 4 字节。

**协议定位（★）**：**应用层路由协议，基于 TCP 179 端口**。两大红利：
- TCP 自带可靠/流控/重传，BGP 不必重复造轮子（类比 20GB 种子走 FTP——TCP 的传输能力 BGP 全部继承）；
- 应用层扩展性强（类比 HTTP 从传文本进化到视频/游戏/小程序——功能扩充不必动底层包头）。
- **触发式更新、无周期刷新**：互联网几十万条路由，周期更新会让全网永远在震荡——有变化才发（★考点）。
- **只传路由不传拓扑**：可精细控制传什么、不传什么。

**路径属性**：**人为设计**的路由 KPI（形象工程类比：KPI 是公司为管理需要发明的，回家就不用打卡）——三大用途：防环、路由管理（选路/过滤）、能力扩展（IPv6/VPNv4/EVPN）。

### 2.2 邻居关系与配置套路

- **EBGP**（跨 AS）/ **IBGP**（同 AS）——判定只看 AS 号，与是否直连无关。
- BGP 基于 TCP 单播，**邻居必须手配**（不像 OSPF 组播自动发现）；但**不要求直连**——只要路由可达、179 通即可建。
- **建邻两铁律**（缺一不可）：①对方报文的**源地址**=我配的邻居地址（交叉匹配）；②对方 Open 里的 **AS 号**=我配的邻居 AS 号。

**三种基本款**：

| 场景 | 配置 | 要点 |
|---|---|---|
| EBGP 物理口 | `peer 10.1.12.2 as-number 200` | 最简单；源地址默认取查路由表的出接口 |
| EBGP 环回口 | 加 `connect-interface LoopBack0` + **`ebgp-max-hop 255`** | EBGP 默认 TTL=1，跨链路必改；多物理链路共用一个 TCP 会话，任一链路断邻居不中断 |
| IBGP 环回口 | 加 `connect-interface LoopBack0` | 默认 TTL=255 无需改；IGP 先打通环回可达 |

**Open 报文四参数（★面试题）**：版本号、AS 号、Holdtime、Router-ID。Holdtime 协商取**较小值**；Keepalive 默认为协商 Holdtime 的 1/3，可单独配置（小于 1/3 按配置，大于 1/3 按 1/3；两端可以不一样）。华为特性：**Router-ID 大的一方发起 TCP 握手**，小的发起会被 RST。能力协商默认携带 IPv4 单播/路由刷新/4 字节 AS 三能力——**至少一个地址族匹配邻居才能建立**。

**五报文**：Open（协商）/ Update（传+撤）/ Notification（报错即断：错误代码+子代码，"医生说感冒了，再说是病毒性感冒"）/ Keepalive（保活，仅公共头）/ **Route-Refresh**（请求邻居重发全部路由——触发更新体制下"属性改了要重传"的补救通道）。

### 2.3 状态机与排错（★只有 Established 是正常的）

```
Idle → Connect → Active → OpenSent → OpenConfirm → Established
不发起/没路由  发起无回应   有回应握手失败   已发Open    收到Open协商过   互收Keepalive
```

| 卡住状态 | 首查方向 |
|---|---|
| Idle | 没有去往邻居的路由；对方握手源地址与我配的邻居地址不匹配 |
| Connect | 对方没有回我路由（对端无路可回），5s 重传循环 |
| Active | 地址可达但 TCP 握手失败——十之八九源目地址交叉不匹配（配置错） |

【设计本源与技术哲学】："实验里 Idle/Connect/Active 三种状态下不去，十之八九配置问题。" 建立完成后先做一次**全量路由交换**，之后只增量；无路由时不出现 Update——"建交了但没有货物贸易"。面试按单边视角描述状态变迁即可。

### 2.4 路由的产生与 BGP 表

**BGP 没有路由计算机制**——【工程类比】"农夫山泉：我们不生产水，只是大自然的搬运工；BGP 只是路由的搬运工，管理员是指挥员。"四种产生方式（常用前两种）：
1. **Network**：精确发布路由表已有路由，网络号+掩码必须一致——"自己要有这条路由才能发，否则就是不负责"（不存在就发=无中生有，不会生效）；
2. **Import-Route**：从 IGP/静态/直连批量引入（可挂 route-policy 加工）；
3/4. 手动/自动汇总（后续展开）。

**BGP 路由表符号学（★）**：`*`=下一跳可达（有效）；`>`=最优；`0.0.0.0` 下一跳≈本地产生；Local/Best/Selected/Active 各表身份。★**下一跳可达性判断不认缺省路由**——必须明细可达。

**四张数据库与策略引擎**（核心架构补充）：
```
邻居 → Adj-RIB-In（原材料）→ [入策略引擎] → Local-RIB（display bgp routing-table）
     → 选最优 → [出策略引擎] → Adj-RIB-Out（成品）→ 发出
```
★**防环检测（AS_Path 含本 AS + 下一跳可达）发生在进入 Adj-RIB-In 之前**——所以想在入方向用 route-policy 删掉自己的 AS 号绕过防环是徒劳的，只能 `allow-as-loop` 放宽。

**Update 拆解**：路径属性与 NLRI **分开存放**（同为 TLV）——多路由共享属性时可合并传输提效；撤销路由只带前缀不带属性（"人都离职了 KPI 对他没意义了"）。

### 2.5 传递规则、IBGP 水平分割与路由黑洞

**五大传递规则（★一切分析的总纲）**：
1. **只传最优路由**（带 `>` 的）；
2. 从 EBGP 邻居学到的 → 传给**所有**邻居；
3. 自身产生的 → 也传给所有邻居（关键补充规则）；
4. **IBGP 水平分割**：从 IBGP 学到的不传 IBGP（AS 内防环，**不可关闭**）；
5. 从 IBGP 学到的，下一跳可达即可传 EBGP。

**水平分割 vs 全互联（一石双鸟）**：水平分割让路由传不动 → 必须全互联补路；全互联有了路又会环 → 必须水平分割防环。【设计本源与技术哲学】："全互联+水平分割，以不变应万变的固定框架。"全互联的代价是 n(n-1)/2 条会话——引出 2.8 的路由反射器。

**路由黑洞**：AS 内某台路由器没跑 BGP——报文到它那里"既没有去目的的路由，也没有回源的路由"，无声丢包。三种解法：
1. **IBGP 全互联**（主流）：让每台都有路由；
2. ~~BGP 引入 IGP~~（淘汰）：BGP 大路由量将击穿 IGP 稳定性，路由震荡跨协议传染，排错责任扯不清——"BGP 的问题找 BGP，OSPF 的问题找 OSPF，别互相躺枪"；
3. 隧道（GRE/MPLS，实验指南案例）。

**BGP 同步规则（历史名词）**：IBGP 学到的路由须 IGP 里也有才传 EBGP——**它是黑洞的检测机制，不是解决机制**；华为默认关闭且**无法开启**（考点：面试要答"华为不支持开启同步"）。

### 2.6 路径属性（一）：NextHop、AS_Path、Origin

**属性分类框架（★面试答题专用）**：公认必遵（Origin、AS_Path、NextHop——"打死都要记住就这三个"）/ 公认任意（Local_Pref、Atomic_Aggregate）/ 可选过渡（Community）/ 可选非过渡。公认=所有 BGP 设备都懂；可选=厂家扩展。理念：**分类只用于面试，实际学习逐个属性具体学**。

**下一跳 NextHop 三条默认规则**：
1. 自身产生 → 下一跳=向该邻居发更新的源 IP（与 B 建邻居用地址 A、与 C 用 M，则分别宣告不同下一跳）；
2. **EBGP→IBGP：保持不变**（酿成 R4 看到 12.1 不可达的经典实验）；
3. IBGP→EBGP、自身→EBGP：**自动改为自己的更新源 IP**。

不可达三解法：静态路由（维护噩梦）、AS 间链路入 IGP（不应该）、**`peer x.x.x.x next-hop-local`（标准答案，仅对 IBGP 生效）**——IBGP 邻居配置三件套：`peer + connect-interface + next-hop-local`。

**第三方下一跳优化（★细节考点）**：R1 向 EBGP 邻居 R3 传路由时，若发现该路由下一跳与 R3 **同网段**，则下一跳**保持不变**——让 R3、R2 直连互通省去绕行 R1。仅 EBGP 方向生效、环回建邻居不触发；决策在 R1。route-policy 的 `apply ip-address next-hop` 出入方向都可改下一跳，但 **EBGP 有隐藏检查：下一跳须与更新源一致/直连**——非直连下一跳要让对方收，需 `ebgp-max-hop` 放行。【深度延伸】"你发什么不重要，我想收什么我自己说了算——自治系统的独立性。"

**AS_Path（公认必遵）**：传给 **EBGP 邻居时把自己 AS 号加在最前**；IBGP 间传递**不添加**。两大用途：**AS 间防环**（收到含本 AS 号的路由整条丢弃——检测在 Adj-RIB-In 之前）+ **选路**（越短越优）。类型：有序 AS_Sequence（常规）/ 无序 AS_Set（汇总时大括号继承明细 AS 号防环，**比长度时整包算 1 个**）。放宽：`peer x.x.x.x allow-as-loop [次数]`（默认容忍 1 次）。route-policy 三种修改：`additive` 追加 / `overwrite` 覆盖 / `none overwrite` 清空（华为 IBGP/EBGP 出入方向都能改，强于思科）。★叠加顺序：**先走 route-policy，再叠加默认"加自身 AS"**——export 策略加的号排在自己 AS 号后面。

**Origin（公认必遵）**：路由怎么变成 BGP 路由的——`i`（network）> `e`（EGP 引入，已断代）> `?`（import 引入）。产生后传递中不变，除非人为改；同前缀两种方式同时产生时 i 优胜。

### 2.7 路径属性（二）：Local_Pref、Community、MED、Preferred-Value

**Local_Preference（公认任意，默认 100，越大越优）**：
- **只在 IBGP 间传递**，EBGP Update 不携带；所以 EBGP 邻居**只能入方向改**（出方向改了也传不出去），IBGP 出入皆可。
- **显示值规律（易错）**：本地产生/EBGP 学来 → 表中**显示空**（按默认，或按 `bgp default local-preference` 的新默认）；IBGP 学来 → 显示携带值。同一前缀在 R2 显示空、R3 显示 100 是正常现象。
- **应用=控制本 AS 出口**：在出口路由器对 EBGP 邻居**入方向**改 LocalPref → 全 AS 统一出向选路。
- `bgp default local-preference <值>` 的生效边界（★四句结论）：只对**本地产生/EBGP 收到/手动汇总**三类路由生效；**对 IBGP 传来无效**（已携值）；route-policy 优先级更高。

**Community（可选过渡，BGP 的 Tag）**：一条路由可携带**多个**团体（"讲师/父亲/武汉人"多重身份；省市区三级管理各取所需）。公认四件套控制传递范围：
| 团体 | 范围 | 类比 |
|---|---|---|
| Internet(0) | 不限（**所有 BGP 路由天生属于它**） | 成年人随便浪 |
| No_Export | 只在 AS 内传 | 别出小区 |
| No_Export_Subconfed | 只在联盟成员 AS 内 | 别出学校 |
| No_Advertise | 不传任何邻居 | 只在家玩 |
- ★No_Export 的语义是"**携带它的路由**不传 EBGP"，不是属性本身不能传。
- **挤牙膏原则（★）**：团体属性默认**不随路由传递**——想让谁看到就要对谁敲 `peer x.x.x.x advertise-community`。
- 两种加法效果不同：Network/Import 时加=Local-RIB 直接生效（可能直接限制传出）；Peer export 方向加=本地照传、属性在**下一台**设备生效——经典场景"双线机房防中转"（对联通路由打 No_Export，防数据中心沦为运营商中转 AS）。

**MED（多出口鉴别器，"BGP 的开销"，越小越优）**：
- 默认值规律：直连/静态引入=0；IGP 路由引入=**继承 IGP 开销**（把 AS 内真实代价透给邻居，让对端从更近的口进来）。
- 传递规则（华为）：自产 → 全传；EBGP 学来 → 只传 IBGP；IBGP 学来 → 传 EBGP **不带**（人为 route-policy 可强制打破）。一句话：MED 只影响**邻居 AS** 的选路，不传染邻居的邻居。
- 【面试标准答案】**Local_Pref 控制本 AS 怎么出去；MED 控制邻居 AS 从哪进来**。
- 默认只比**同 AS** 来源的 MED；跨 AS 比较须全 AS 统一 `compare-different-as-med`（类比参考带宽——全网必须一致）。`default med` 只对 import/汇总路由、且仅发 EBGP 时生效。

**Atomate_Aggregate / Aggregator**：汇总并抑制明细后，前者是"丢了明细属性"的警告牌，后者记录"谁做的汇总"（Router-ID）——公网 IPv4 汇总出了问题找得到责任人（IPv6 分配规整后此类问题锐减）。

**Preferred-Value（协议首选值，思科叫 Weight）**：**本地有效不传递、默认 0、越大越优、选路第 1 条**。设计意图：Local_Pref 影响全 AS（"美国式辐射影响"），Preferred-Value 给**单台路由器**独立自主（"大国自主决定不受摆布"）。只能在**入方向** route-policy 改（`apply preferred-value`），或 `peer x.x.x.x preferred-value` 按邻居批量。

### 2.8 路由反射器（RR）

**动机**：IBGP 全互联 n² 扩展性枯竭——RR 把网状改星状（逻辑同 DR/BDR）。

**三角色与四句邻居规则**：客户机只需与 RR 建邻；RR 与 RR 全互联；非客户机之间全互联；RR 与非客户机全互联。【配置秘诀】**客户机不知道自己是客户机**（"鱼不知道自己在鱼塘里，养鱼的人知道"）——两端配置完全一样，只有 RR 上敲 `peer x.x.x.x reflect-client` 圈定客户机。

**反射的定义（★字面即考点）**：【设计本源与技术哲学】："反射的路由=**打破了 IBGP 水平分割**而被传递的路由；没打破的按常规规则走，不叫反射。"三条传递规则：非客户机→发给所有客户机（反射）；客户机→发给所有非客户机及其他客户机（反射）；EBGP→发给所有（**常规**，本来就传）。RR 只反射**最优**路由。

**防环两属性（由第一个 RR 添加）**：
- **Originator ID**（始发客户机 Router-ID，加了不再变）→ 防簇内环；
- **Cluster List**（反射时把自己的簇 ID 加在最前）→ 防簇间环（逻辑同 AS_Path）。
- 防环规则一句话：**从 IBGP 邻居收到的路由，这两个属性里只要出现自己的 Router-ID，一律拒收**——不分客户机/非客户机/RR。
- 两者不传出 AS/联盟外。属性修改权限：入方向一律可改；出方向**反射路由默认不可改**（"照镜子不能变"，`reflect-change-path-attribute` 可解锁）。

**三种架构**：备份 RR（同簇 ID 双机，城域网/数中心）/ 同级 RR（RR 互联互为非客户机，省级）/ 分级 RR（下级 RR 兼任上级客户机——"你在某些人面前是爸爸，在另一些面前是孙子"）。★RR 只反射不转发（控制面/转发面分离）——可拿表项容量大、转发要求低的设备专职 RR。

### 2.9 选路 11 规则与综合调优实战

**BGP 路由优选 11 条（华本次序）**：
1. 丢弃下一跳不可达（实为忽略）；
2. **Preferred-Value 越大越优**；
3. 本地始发优于邻居学得（手动汇总>自动汇总>network>import>学到）；
4. **AS_Path 越短越优**；
5. Origin：i > e > ?；
6. **MED 越小越优**（默认同 AS 才比）；
7. **EBGP 优于 IBGP**（"外来的和尚会念经"——直达出口优于绕 AS 内转发，避免次优）；
8. 下一跳 IGP 开销小者优；
9. 簇列表短者优（**空=未反射 优于 有**）；
10. Originator-ID（或无 RR 场景的 Router-ID）小者优；
11. 邻居地址小者优。
★华为 9/10 与思科**互反**（思科先 Originator-ID 后簇列表）——IA/IP 经典考点。
**控路实用原则**：1/2/4/5/6 是工作主力（规则 7 以后"听天由命"——IGP 开销改动有连带、簇列表/Router-ID 不许动）。
**分析方法论**：先在脑中预判目标路由器 BGP 表里各来源路由的属性值 → 按序逐条比对 →锁定胜者 → 决定改哪个属性实现意图。**"闭着眼睛都想得出它在干啥，这才叫熟练。"**

**综合实验八需求精华（六路由器三 AS，R6 为 RR）**：
- **首选值锁路两法**：按邻居批量（`peer preferred-value 100`）vs 前缀列表+route-policy 单条改——**个性策略覆盖共性策略**；首选值是第 1 规则，会**屏蔽后面所有属性比较**（AS_Path 调优前必须先把首选值"打平"为 0）。
- **保约束调选路**：要 R6 走 R3 又不许动"去 R1 环回口"的路径——不调链路开销（会连带改路），而是**改 R1 环回口的开销值**：开销数值劣化但路径不变，规则 8 立即翻转。**改开销值 ≠ 改路径**。
- **反射器入方向改属性的次序坑**：先给 R1 侧改 LocalPref 160 → 160>100 立即优选并反射 → R3 原路由被顶掉 → 后配的 360 无从到达——**同路由双邻居场景，入方向策略的配置次序影响结果**；正序（先 360 后 160）才对。
- **MED 绕路**：R6 访问 5.0 偏走 R4、访问 4.0 偏走 R5——先推演两路属性（自产默认 0、IBGP→EBGP 不带 MED），在对端邻居入方向把"近路"的 MED 改大（555/44）即成。
- **AS_Path 叠加**：`apply as-path 2 additive` 加**自己**的 AS 号（不触发对端防环，无需 allow-as-loop；加对端 AS 号则被拒收）。

### 2.10 BGP 负载分担

- **默认不做负载分担**——选路一路比到第 11 条，"死活要选出一条最优"。
- **启用条件**：`maximum load-balancing <n>`，要求**前八条规则全同**（含 AS_Path **长度与内容完全一致**）。
- **表项真相**：BGP 表里**仍只有一条最优**（"你大爷还是你大爷"——通告邻居的仍是最优那条）；负载分担体现在 **IP 路由表**（多条带 Selected/Active 下发）。
- **EBGP 与 IBGP 不混担**：只能在"最优所在的那一类"里分担——最优是 EBGP 就在 EBGP 里选，IBGP 落选者无资格；`maximum load-balancing ebgp N / ibgp N` 细分（两命令互斥）。
- **递归分担（免配置）**：BGP 下一跳本身在 IGP/静态里就是 ECMP → BGP **自动**沿下一跳迭代做负载分担（同一下一跳、多个出接口）。
- **命令矩阵（★压轴考点）**：
  - `maximum load-balancing N` 单用：AS_Path 既参与选路又是分担条件；
  - 加 `load-balancing as-path-ignore`：AS_Path 仍参与**选路**，但**不再是分担条件**（七条件即可分担，数据中心常用）；
  - 换 `bestroute as-path-ignore`：AS_Path **退出选路**（出现"短的没赢、比到 Router-ID"的反直觉现象，排错线索）但分担时仍要求 AS_Path 一致；
  - 两个 ignore 参数**互斥**。
- 【思维升华】【设计本源与技术哲学】："**不要把规则当圣经**"——BGP 是应用层协议（HTTP 早已超越传文本的初衷），MPLS VPN 场景 EBGP+IBGP 都能混担；负载分担本质只是"本地找多个出口同时用"。

### 2.11 BGP 核心配置模板与排错命令集（VRP 实战）

#### 经典建邻与选路调优模板

```shell
# ═══ 1. IBGP / EBGP 邻居建立三件套 ═══
[R1] bgp 100                                            # 本地 AS 号为 100
[R1-bgp] router-id 1.1.1.1
# IBGP 环回口建邻：指定源地址 + 下一跳本地化
[R1-bgp] peer 2.2.2.2 as-number 100
[R1-bgp] peer 2.2.2.2 connect-interface LoopBack0       # ★更新源设为 LoopBack0（防单物理链路中断断邻居）
[R1-bgp] peer 2.2.2.2 next-hop-local                    # ★向 IBGP 邻居通告路由时，将下一跳强行改为自己！

# EBGP 直连/多跳建邻
[R1-bgp] peer 10.1.13.3 as-number 200                   # 对端 AS 为 200
# 若 EBGP 用环回口建邻，必须配置多跳放宽 TTL（默认 TTL=1）：
[R1-bgp] peer 3.3.3.3 ebgp-max-hop 2

# 路由发布与团体属性发送
[R1-bgp] network 192.168.1.0 255.255.255.0              # 精确宣告路由（本地路由表中必须精确存在此路由）
[R1-bgp] peer 2.2.2.2 advertise-community               # ★华为默认不传 Community，必须显式开启“挤牙膏”

# ═══ 2. BGP 核心选路属性调优实例 ═══
# 方法 A: 修改 Preferred-Value（华为私有第一规则，越大越优，仅本地有效）
[R1-bgp] peer 10.1.13.3 preferred-value 100             # 该邻居发来的所有路由首选值设为 100

# 方法 B: 通过 Route-Policy 调整 Local-Preference（公认遵从，越大越优，控本 AS 全部出向流量）
[R1-route-policy] route-policy SET_LP permit node 10
[R1-route-policy-node10] apply local-preference 200
[R1-bgp] peer 10.1.13.3 route-policy SET_LP import

# 方法 C: AS_Path Prepend 延长路径（控外部流量进入本 AS 的路径，越短越优）
[R1-route-policy] route-policy AS_PREPEND permit node 10
[R1-route-policy-node10] apply as-path 100 100 additive  # 重复填充本 AS 号，使外部设备因 AS_Path 过长而不选本出口
[R1-bgp] peer 10.1.13.3 route-policy AS_PREPEND export
```

#### 状态查看与排错回显分析

```shell
# 1. 检查 BGP 对等体状态（排错第一步：确认 State 是否为 Established）
<R1> display bgp peer
 BGP local router ID : 1.1.1.1
 Local AS number : 100
 Total number of peers : 2                 Peers in established state : 2
  Peer            V    AS  MsgRcvd  MsgSent  OutQ  Up/Down       State PrefRcv
  2.2.2.2         4   100       12       14     0 00:09:32 Established       5
  10.1.13.3       4   200       10       11     0 00:08:15 Established       3

# 2. 检查 BGP 路由表符号标记
<R1> display bgp routing-table
 Total Number of Routes: 2
      Network            NextHop        MED        LocPrf    PrefVal Path/Ogn
 *>   192.168.1.0        0.0.0.0         0                     0      i
 * i  192.168.2.0        2.2.2.2         0          100        0      i
# ★ 关键符号诊断：
#   *  : 有效路由（Valid，下一跳在 IGP 中可达且通过检验；无 * 说明下一跳不可达，绝不进全局路由表）
#   >  : 最优路由（Best，经过 11 条选路规则 PK 胜出的唯一路由，只有带 > 的路由才会被下发 FIB 并通告给邻居）
#   i  : 内部路由（从 IBGP 邻居学到的路由，受 IBGP 水平分割限制，不会再转发给其他 IBGP 邻居）
```


### 2.6 全球互联网商业模型：Transit、Peering 与 Gao-Rexford 准则

在真实全球互联网（The Internet）中，BGP 路由不仅受技术指标控制，更由**商业经济收益**绝对主导：
1. **Transit（付费转接）**：下游小型网络向 Tier-1 骨干运营商付费购买全球全表可达性；
2. **Peering（对等互联）**：两个规模相当的网络在 IXP（互联网交换中心）免费互通，但遵循互免结算原则，仅向对方通告自身及客户路由，绝不充当免费过境通道；
3. **Gao-Rexford 商业选路准则**：
   4405\text{选路优先级}: \text{客户路由 (Customer)} > \text{对等路由 (Peer)} > \text{转接路由 (Provider)}4405
   在网络配置中，通常通过给客户路由打上最高 Local-Preference = 200，对等路由打上 Local-Preference = 150，付费转接路由打上最低 Local-Preference = 100 来实现该商业策略。

## 三、总结

**核心重点十带走**：
1. BGP=TCP 179 上的应用层路径矢量协议：触发更新、只传路由不传拓扑、属性人为设计（防环/管理/扩展三用途）。
2. 建邻两铁律=源地址交叉匹配+AS 号匹配；EBGP 环回建邻必须 ebgp-max-hop；Open 四参数（版本/AS/Holdtime/Router-ID），Holdtime 取小、Keepalive 为其 1/3。
3. 状态机只有 Established 正常；Idle=没路由、Active=握手失败（十之八九源目不匹配）。
4. BGP 无路由计算机制——Network（要精确存在）/Import 搬运路由；`*`=可达、`>`=最优、0.0.0.0=自产；下一跳判断不认缺省。
5. 五传递规则+IBGP 水平分割（不可关）→ 全互联；黑洞三解法取全互联/RR；同步=检测非解决（华为不可开）。
6. 属性三分：NextHop（EBGP→IBGP 不变是实验常客、第三方同网段优化）、AS_Path（EBGP 加号/IBGP 不加、防环在进 Adj-RIB-In 前、Set 算 1 个）、Origin（i>e>?）。
7. Local_Pref 只传 IBGP（EBGP 只能入方向改）控全 AS 出口；Community 挤牙膏（advertise-community 才传）、No_Export 控范围；MED 继承 IGP 开销、只影响邻居 AS、默认同 AS 才比；Preferred-Value 本地第一规则、个性压共性。
8. RR：反射=打破水平分割的传递；Originator-ID/Cluster-List 防环（含自己 ID 即拒收）；客户机无感知、只有 RR 圈人；RR 只反射不转发。
9. 11 条规则中华为 9/10 与思科互反；控路用 2/4/5/6 条；首选值会屏蔽后续比较；改环回开销=改值不改路。
10. 负载分担：BGP 表单最优、路由表多路径；EBGP/IBGP 不混担；下一跳 ECMP 自动递归；两个 as-path-ignore 语义相反且互斥。

**落地注意事项**：IBGP 邻居三件套（peer/connect-interface/next-hop-local）是架构级配置模板；改任何属性前先想它对既有策略的**叠加影响**（首选值屏蔽、次序坑）；route-policy 一定留 permit 兜底节点（否则默认 deny 全灭）；团体属性记得对每个需要看到的邻居挤牙膏；跨 AS 比 MED 与参考带宽同理——全网统一敲；策略命名要有体系（重名冲突现场难查）。

**学习建议**：本章练习核心是"沙盘推演"：拿到拓扑先闭眼预判每台设备 BGP 表的属性与最优标记，再上 eNSP 验证——预判错了就是知识漏洞所在。综合实验八需求建议完整复刻一遍（含故意触发次序坑与打平首选值），能独立解释"为什么 160 先配会顶掉 100"才算过关。负载分担的三个命令版本各做一次实验并记录路由表差异。修完本章，IP 阶段知识地图闭环，可进第 09 章《BGP高级特性》或按需求切入 IE 专题。

## 附：本章术语速查表

| 术语缩写 | 英文全称 | 中文含义 | 关键作用 |
|---|---|---|---|
| **AS** | Autonomous System | 自治系统 | 拥有统一选路策略与管理机构的网络设备集合 |
| **BGP** | Border Gateway Protocol | 边界网关协议 | 运行于 TCP 179 的外部网关协议，用于 AS 间的大规模路由外交与策略控制 |
| **NLRI** | Network Layer Reachability Information | 网络层可达性信息 | BGP Update 报文中携带的具体路由前缀与掩码 |
| **RR** | Route Reflector | 路由反射器 | 打破 IBGP 水平分割全互联要求的核心角色，将客户机路由反射给非客户机 |
| **Local_Pref** | Local Preference | 本地优先级 | 公认遵从属性，仅在 IBGP 内传递，控制本 AS 流量出外部 AS 的统一出口 |
| **MED** | Multi-Exit Discriminator | 多出口鉴别器 | 可选非过渡属性，向外部 AS 宣告，影响邻居 AS 进入本 AS 的流量入口 |
| **Preferred-Value** | Preferred Value | 协议首选值 | 华为私有属性，仅本地有效不向邻居传递，BGP 选路 11 规则的第一顺位 |
| **Community** | Community Attribute | 团体属性 | 可选过渡属性，一组 4 字节标识符，用于对海量路由打标以简化集中策略控制 |
