# 第15章 DHCP与无线接入

> 接入侧双引擎：DHCP 从"三种用法"（接口池/全局池/中继）深入到四报文七步交互、T1/T2 续租、Option 43/82 与"服务器被动响应"的安全本质；WLAN 从 WiFi 标准、BSS/SSID/VAP 映射思维到 AC+FIT AP 架构、CAPWAP 隧道 5246/5247、上线五步、VLAN 池障眼法、四种高可靠方案与漫游暗度陈仓。

| 项 | 内容 |
|---|---|
| **适用层级** | HCIA（DHCP 基础配置）→ HCIP（无线是当前认证重点章节，CAPWAP/VAP 模板为实验主菜） |
| **架构定位与先修** | 全书第 15 章，先修第 11 章（VLAN/Trunk——无线业务全靠 VLAN 映射思维）；第 14 章 的 VRRP 在本章 AC 双机热备中直接复用 |
| **技术维度** | 协议底层机理剖析 · 生产级部署规范 · 故障排错实践 · 考题与面试标准 |

## 一、总起

园区网接入侧的两大工程主题在本章会师。**DHCP** 三讲走完"配置—原理—进阶"：先掌握华为设备三种部署形态（懒人接口池、灵活全局池、跨网段中继），再下钻报文级——八种报文、七步交互、免费 ARP 冲突检测、50%/87.5% 两级续租，以及 Option 机制如何被 AC 发现（43）与运营商标识（82）借用；最后落到"DHCP 服务器完全由客户端驱动"的安全反思。**WLAN** 两讲半从零到园区级：WiFi1→7 演进、BSS/BSSID/SSID/VAP/ESS 五概念（全靠与有线 VLAN 的映射思维）、胖瘦 AP 与敏分三种架构、CAPWAP 双通道、AP 上线五步、三个模板四条命令的业务下发体系，直到 VLAN 池、四种高可靠方案与两层漫游的进阶话题。贯穿全章的底层思维心法：**"把无线变成有线的那一套思维逻辑"**——无线的一切封装最终都落到 VLAN、Trunk、MAC 表这些你已经会的东西上。

**适用人群**：
- **初学 / 转行人员（HCIA 入门）**：理解动态主机配置协议（DHCP）与无线局域网（WLAN）基本框架，掌握接口池/全局池配置与胖/瘦 AP 架构差异；
- **备考工程师（HCIA / HCIP 核心）**：攻克 DHCP 交互七步、Option 43/82 字段格式、CAPWAP 控制通道（UDP 5246）与数据通道（UDP 5247）、AP 上线五步、以及无线三模板（Security/SSID/VAP Profile）标准化配置；
- **一线工程实践者**：掌握大型企业与分支网中“AC + Fit AP”集中管控、VLAN 池与转发模式（直接转发 vs 隧道转发）规划、以及 VRRP+HSB 双机热备工程模板。

**本章技术脉络与核心架构**：

| 模块节次 | 核心技术要点与工程主题 |
|---|---|
| DHCP与BFD（取DHCP） | 接口池/全局池/中继三种用法、全局池自动关联逻辑、VRRP 场景网关指定、中继回程路由排错 |
| DHCP深入 | 八报文七步交互、T1/T2 续租、Option 全家桶、抓包实证 + WLAN 入门（WiFi 演进、五概念、信道） |
| DHCP应用与无线引入 | AP 上线五步、Option 43 三写法、CAPWAP 5246/5247、三模板四命令、隧道 vs 直接转发 |
| 无线配置案例 | VLAN 池两种算法、VRRP 双机热备+HSB、双链路/N+1、漫游二层三层、MSTP 误阻塞排错 |

## 二、分述

### 2.1 DHCP 三种部署形态（华为配置基座）

**接口地址池（懒人专用）**：接口所在网段即地址池，接口 IP 自动当网关——一条命令 `dhcp select interface` 搞定。配套：`dhcp server dns-list` / `excluded-ip-address`（连续段自动合并）/ `static-bind ip-address X mac-address Y`。

**全局地址池（灵活标准型）**：`ip pool <名>` → `network 10.1.20.0 mask 24`（★必须 network+mask 格式）→ `gateway`（★独立指定）→ `dns-list` → 接口下 `dhcp select global` 引用。
- ★**自动关联逻辑**：多个全局池并存时，设备按**接口 IP 落在哪个池的 network 网段**选择地址池——接口 IP 无匹配池=分配失败。
- **网关独立指定的价值**：双网关拆分场景（A 发地址 B 当网关）；★**VRRP 场景必须用全局池**——把 gateway 配成虚拟 IP。接口池的网关=物理接口 IP，VRRP 直接失效。

**DHCP 中继**：客户端与服务器异网段，网关接口 `dhcp select relay` + `dhcp relay server-ip 10.1.2.1`。机制：广播变单播、以中继口 IP 为源、GIAddr 携带客户端网段供服务器选池。★**排错第一问：中继回程路由通不通**——服务器必须有到客户端网段的路由（`ip route-static 10.1.30.0 24 10.1.2.254`），否则回包不可达、DHCP 必死。单臂路由场景 Trunk Tag 帧到路由器物理口不通——子接口或改 PVID 两解。

### 2.2 八种报文与七步交互（协议原理核）

**八种报文**：
| 报文 | 方向/用途 |
|---|---|
| Discover | 客户端广播找服务器（255.255.255.255） |
| Offer | 服务器应答（单播或广播，Flag 位控制） |
| Request | 客户端广播确认选择 |
| ACK | 被选服务器最终确认（含租期） |
| Decline | 客户端检测地址冲突→要求服务器禁用该地址 |
| NAK | 服务器拒绝 Request |
| Release | 客户端主动退租 |
| Inform | 已有 IP 只要参数（类比：汉堡薯条到手再要包番茄酱） |

**关键字段**：OP（1 客户端/2 服务器）、Flag（最高位 0=要单播回应、1=要广播）、**YiAddr**（分给你的 IP）、**SiAddr**（服务器 IP）、**CHAddr**（客户端 MAC）、**GIAddr**（中继口 IP，服务器选池依据）、Hop（中继跳数上限 16）。

**七步交互（★面试级流程）**：
1. 广播 Discover（源 0.0.0.0，Option 55 带参数愿望清单）
2. 所有服务器都回 Offer 并**各自记账**
3. 客户端只认最先到的 Offer（类比：投多家简历，只去一家二面）
4. 广播 Request 携带选中服务器 IP——★**广播的用意是通知落选者销账**（【设计本源与技术哲学】："人家都做好了准备——'我裤子都脱了你就给我看这'，不礼貌"）
5. 被选者回 ACK（补上租期——"Offer 是看房，ACK 才谈租多久"）；落选者清除记录
6. 客户端发**免费 ARP 探冲突**：冲突→Decline→服务器拉黑该地址，客户端重来
7. 续租：**T1=50% 租期单播 Request** 给原服务器；失败则 **T2=87.5% 广播 Request** 再试；再失败用到期退租重来（★T1 时 Request 从广播变单播——经典考点）

【设计本源与技术哲学】："DHCP 服务器的行为完全由客户端驱动，服务器无法控制客户端——协议安全性比较差，服务器是被动响应式服务。"

**分配策略与重温 IP**：优先级=静态绑定>曾用地址（租约记录查 MAC）>空闲最小>超期>冲突过。客户端重启可在 Option 50 里填上次 IP 要回来——服务器查记录有则给、无则不理；支持与否看客户端实现（"应用层协议具体问题具体分析"）。

**Offer 单播/广播面试细节**：Flag 控制；**IP+MAC 静态绑定时 Offer 走单播**。中继场景抓包实证：客户端侧四报文如常，中继到服务器段全单播，Offer 里暴露的是 GIAddr（中继地址）而非服务器真身（真身只藏 Option 54）。

### 2.3 Option 机制：标配与选配

TLV 结构，1-255。**预定义（类比购车标配）**：1=掩码、3=网关、50=请求 IP、51=租期、53=报文类型、54=服务器标识、55=参数请求列表、58/59=T1/T2。**自定义（选配）**：
- **Option 43（★华为私有）**：给 AP 下发 AC 地址——无线上线的钥匙（三种写法：子选项 1 十六进制"没人愿意写"、**子选项 2 IP 直写（推荐）**、子选项 3 ASCII 逗号隔多 AC）；IPv6 场景对应 Option 52
- **Option 82**：中继信息选项（最复杂，255 个子选项——VLAN/端口/中继 MAC），配合软件实现分配限制与计费

### 2.4 WLAN 入门：标准、概念与信道

**WiFi 演进**：802.11(2M)→b(11M)→a/g(54M)→n(WiFi4)→ac(WiFi5，仅 5G)→ax(WiFi6，双频)→WiFi7。双频=2.4G+5G 两根天线；三频（2×5G+2.4G）可拿一条 5G 做 AP 间 Mesh 桥接。【设计本源与技术哲学】："带宽的增加是驱动这个世界发展的最根本原因。"

**五概念映射表（★无线学习钥匙）**：
| 无线术语 | 有线对应 | 说明 |
|---|---|---|
| BSS 基本服务集 | 一个广播域 | 一个 AP 的覆盖范围（全向球形） |
| BSSID | MAC 地址 | AP 的 MAC——**给设备看的** |
| SSID | 域名 | 字符串名——**给人看的** |
| VAP 虚拟 AP | VLAN | 一台 AP 切多个虚拟 AP（最多 16 个）——交换机从单广播域到多 VLAN 的同款进化 |
| ESS 扩展服务集 | VLAN 跨交换机 | 多 AP 同 SSID 连续覆盖，二层漫游不断线 |

**结论**：每个 SSID 的 VAP 映射到有线侧一个 VLAN——无线用户体验与有线二层完全一致。**信道**：2.4G 共 14 信道（中国 1-13，20MHz 宽），重叠信道不能同用（"一把牙刷俩人不能同时刷"），**1-6-11 经典三不重叠**；5G 资源丰富（两段频带，20/40/80/160MHz 可调）；AC 支持自动信道调优。

### 2.5 组网架构与 AP 上线五步

**三种架构**：胖 AP（FAT，独立配置，小办公室）／**AC+瘦 AP（FIT，企业主流）**／敏捷分布式（中心 AP+房间级敏分 AP+AC——酒店医院专属）。

**CAPWAP 隧道**：应用层管理协议（IP/UDP），AC 与 AP 管理地址 IP 可达即可建立。**双通道（★必考）**：**控制通道 5246**（discovery/join/配置下发，Echo 保活）+ **数据通道 5247**（业务报文，Keepalive 保活）——类比 FTP 双通道。防火墙放行 Trust→Untrust 目的端口 5246/5247（状态检测自动放行回程）。

**上线五步**：
1. **获取 IP**：AP 默认发 untag DHCP——**归属 VLAN 由接入交换机接口决定**（Access 或 Trunk PVID）
2. **发现 AC**：二层组网同广播域→广播 discovery；三层组网→**必须 Option 43** 下发 AC 地址才单播 discovery（排错名场面：先拿到地址后配 Option 43 的 AP 还在广播——重启接口重新 DHCP 才吃上 43）
3. **建 CAPWAP 隧道**：`capwap source ip-address` 定源
4. **镜像升级**：版本不一致则传软件重启重连
5. **配置下发**：AC 推业务模板

**AP 认证**：MAC 认证（工作首选）/SN 认证/不认证（"千万别裸奔"）。

### 2.6 业务下发：三模板四命令

```
wlan
 security-profile name SEC
  security wpa2 psk pass-phrase A1234567 aes    # ①安全模板（WPA2+PSK+AES）
 ssid-profile name IT
  ssid IT                                        # ②SSID 模板
 vap-profile name IT
  security-profile SEC                           # ③VAP 模板四件套：
  ssid-profile IT                                #    安全+SSID+
  service-vlan vlan-id 10                        #    业务VLAN+
  forward-mode tunnel                            #    转发模式（默认直接转发）
 ap-group name IT
  vap-profile IT wlan 1 radio all                # ④AP组引用（WLAN ID + 双射频）
```
- WEP 已淘汰不用；WAPI 国标（政府保密场景）；企业主流 WPA/WPA2
- WLAN ID（1-16）AP 本地有效，不同 AP 复用同 ID 不冲突（类比各交换机各自的 VLAN 1）
- 国别代码必配：`regulatory-domain-profile` + `country-code CN`

### 2.7 隧道转发 vs 直接转发（★本章核心对比）

| | 隧道转发 | 直接转发 |
|---|---|---|
| 数据路径 | 无线帧→以太帧→打业务 VLAN Tag→**封装 CAPWAP**（AP 管理地址→AC 管理地址）→AC 解封装查 MAC 表转发 | 无线帧→以太帧→打 Tag→**AP 物理口直接发出** |
| 类比 | AP=无线客户端的有线网卡，**AC=这台主机的接入交换机** | AP 物理口=一个放行所有 VLAN 的干道口 |
| 交换机连 AP 口 | 只放行**管理 VLAN** | 必须放行**管理+业务 VLAN** |
| 客户端 MAC 学习位置 | 核心交换机在 **AC 侧端口**学到 | 在 **AP 侧端口**学到 |
| AC 负担 | 重（所有业务过 AC） | 轻 |
| 增值 | 集中调优/QoS/过滤/认证 | 认证放网关做 |

【设计本源与技术哲学】："AP 把无线变成了有线，无非封装在 IP 报文里送给 AC——你按有线网络的正常思维去分析就完了。"

**万能配置思路（工程经验）**：交换机连 AP 的口 干道放行管理+全部业务 VLAN、PVID=管理 VLAN；AC 与核心互联口同全放——**VAP 转发模式随你怎么切，交换机侧永不改**。

### 2.8 VLAN 池：一个 SSID 的障眼法

**痛点**：单 SSID 单 VLAN，500 用户得 /22 大广播域。
**机制**：多个 VLAN 入池绑同一 SSID，用户连入时池中分配——无线侧一个网络（用户无感），有线侧多广播域隔离，容量=N×254。
**两种算法（★）**：顺序（Round Robin，分得均匀但重上线换 VLAN 换 IP）／哈希（按 MAC，重上线 VLAN 不变但可能不均——"男女比例失调，哈希看运气"）。
```
vlan pool IT
 vlan 10 20
 assign order          # 或 hash
# VAP 模板中：service-vlan pool IT（替换 vlan-id）
```

### 2.9 四种高可靠方案（★选型题）

| 方案 | 机制 | 切换 | 负载分担 | 定位 |
|---|---|---|---|---|
| **VRRP 双机热备** | 双 AC 虚拟一个 VIP，AP 连 VIP；**HSB 通道**同步 AP/用户/CAPWAP 状态 | 最快（直接转发近乎无感） | ✗ | **园区首选** |
| 双链路热备 | AP 同时连主备两 AC（主干活备待命，HSB 同步）；心跳 Echo 25s×3=**75s** 切换 | 有等待 | ✓（两个 AP System Profile 交叉指定主备） | 需负载分担/异地 |
| 双链路冷备 | 同上但**无 HSB** | 中断久 | — | 不常用 |
| N+1 备份 | N 台分支 AC+1 台云端公共备 AC；本地挂了 AP 重新上线到备 AC | 慢（重新建隧道） | ✗ | 分支低成本（"备胎中的备胎"，RAID 热备盘；必须直接转发） |

**VRRP 热备配置链（★HSB 四步）**：
```
# ①AC1: Vlanif100 跑 VRRP（VIP 10.1.100.1，priority 120，抢占延时 120s）
 admin-vrrp vrid 1                        # 指定为管理 VRRP 组
# ②HSB 服务通道
 hsb-service 0
  service-ip 10.1.102.1 peer 10.1.102.2   # AC2 互为镜像
  local-port 10241 peer-port 10241
  service-keepalive retransmit 3 interval 6
# ③HSB 备份组绑定通道并关联 VRRP 定主备
 hsb-group 0
  bind-service 0
  track vrrp vrid 1 interface Vlanif 100
  hsb-service-type access-user hsb-group 0   # 用户信息同步
  hsb-service-type ap hsb-group 0            # AP 信息同步
  hsb enable
```
验证：`display hsb service`（Connect）、`display hsb group 0`（Active/Inactive）、备用 AC 上 `display ap all` 全 Standby。【模拟器坑】HSB 只备份状态不同步配置——两台 AC 业务配置要手动做成一模一样；真机支持同步。
实验排错名场面：AC 间 HSB 直连链路被 **MSTP 按 CIST 误阻塞**（三设备不同域）——最终关生成树了事（实际无环）。【设计本源与技术哲学】："技术你不做实验，没有好的理论，你都不知道怎么分析。"

### 2.10 漫游：Home/Foreign 与暗度陈仓

**前提**：SSID、安全模板、认证参数全部一致。**术语**：Home AC/AP（首次关联）、Foreign AC/AP（漫游目标）。**漫游组**配置极简：成员互写管理地址，自动全互联 CAPWAP。

**二层漫游**：同一广播域内换 AP——模拟器完整支持（AC 内/AC 间、隧道/直接转发均可），无丢包。
**三层漫游**：跨 VLAN 但 **IP 不变**——原理是 Foreign AP 把原始二层帧经 CAPWAP 隧道**送回 Home AC** 再转发（暗度陈仓：换了台设备替你做封装）。模拟器做不出（会断、IP 会变）。
**实践铁律**：能用二层漫游绝不碰三层——三层+802.1X/Portal"指不定掉进哪个厂家的坑里，排错把自己搞疯"。
【升华】【设计本源与技术哲学】："CAPWAP 和 VXLAN 一样，都是 IP 隧道封装以太网帧，具备大二层能力——网络玩来玩去就是隧道。"

## 三、总结

**核心重点十带走**：
1. DHCP 三形态：接口池懒人快（网关=接口 IP，不能改）；全局池灵活（network+mask 定义、gateway 独立指定、**接口 IP 匹配池网段**自动关联）；中继跨网段（GIAddr 选池，**服务器回程路由是最常见死因**）。
2. VRRP 场景 DHCP 必须全局池+虚拟网关；中继接口配 `dhcp select relay` 而非 global。
3. 七步交互：Discover→Offer→（客户端选先到者）→广播 Request（为通知落选者销账）→ACK 加租期→免费 ARP 探冲突→T1 50% 单播续租、T2 87.5% 广播续租。
4. 八报文全表 + 字段六件套（OP/Flag/YiAddr/SiAddr/CHAddr/GIAddr）；静态绑定时 Offer 单播；服务器被动响应=协议安全性差。
5. Option：43（华为私有，AP 发现 AC，子选项 2 IP 直写最简；IPv6 用 52）、82（中继信息，运营商计费）、53/54/55/58/59 必背。
6. 五概念映射：BSSID=MAC、SSID=域名、VAP=VLAN、ESS=VLAN 跨交换机——映射思维在手，无线配置无障碍。
7. 上线五步：取 IP（VLAN 归接入交换机定）→发现 AC（二层广播/三层必须 Option 43）→CAPWAP、升级、配置下发；认证用 MAC 别裸奔。
8. CAPWAP 双通道：**控制 5246**（Echo 保活）/**数据 5247**（Keepalive）；防火墙放行 Trust→Untrust 目的端口两兄弟。
9. 隧道 vs 直接转发：交换机连 AP 口的放行差异（管理 only vs 管理+业务）、MAC 学习位置（AC 侧 vs AP 侧）、AC 负担三连对比；万能口配法（全放行+PVID 管理）一切模式通吃。
10. 高可靠选型：VRRP+HSB 园区首选（VIP+状态同步+track 定主备）；双链路热备 75s 支持负载分担（Profile 交叉）；N+1 低成本分支备胎；VLAN 池顺序/哈希两算法；漫游优先二层。

**落地注意事项**：Option 43 一定要在 AP 取地址**之前**配好，否则要重启 AP 端口重新 DHCP；三层组网 AC 回 AP 网段的静态路由，下一跳写互联 VLAN 对端地址（不是 AP 网关——"3.254？三你的鬼呀！"）；连 AP 口的 PVID 永远=管理 VLAN；HSB 双 AC 手动同步配置（模拟器）时删改业务两边都要动；漫游规划时同 SSID 的安全模板逐字段核对；高可靠方案没有最好只有最合适——"你不可能用大炮打蚊子"。

**学习建议**：DHCP 部分把"三形态实验+中继排错+抓包七步"做过一遍就立住了，重点抓 T1/T2 单播变广播和 Request 广播销账两个逻辑点。无线部分强烈建议完整复刻三 SSID 综合实验（IT 隧道转发+HR 直接转发+Guest 双 AP 同发）：做完你会自然掌握三模板四命令与两种转发的全部差异；再进阶做 VRRP 双机热备（重点体验 MSTP 误阻塞排错与 display hsb 三板斧）。四方案对比表和隧道/直接对比表建议抄写两遍——考试与面试都是送分题。学完本章，园区网接入侧闭环，接下来第 16 章《NAT 与 PPP》解决出口地址转换与广域网封装。

## 附：本章术语速查表

| 术语缩写 | 英文全称 | 中文含义 | 关键作用 |
|---|---|---|---|
| **DHCP** | Dynamic Host Configuration Protocol | 动态主机配置协议 | 基于 UDP 67/68 的自动化网络参数（IP/掩码/网关/DNS）分发协议 |
| **Option 43** | DHCP Option 43 (Vendor Specific) | 厂商自定义选项 | 华为私有子选项，用于在跨三层组网中向 Fit AP 通告 AC 的管理 IP 地址 |
| **Option 82** | DHCP Relay Agent Information Option | 中继代理信息选项 | 记录客户端接入的交换机 MAC、VLAN 与物理端口，用于运营商精确定位与计费 |
| **SSID** | Service Set Identifier | 服务集标识符 | 无线网络的名称标识，面向用户广播显示以区分不同的无线服务 |
| **BSSID** | Basic Service Set Identifier | 基本服务集标识符 | 无线网络的基本服务集硬件标识，通常取 AP 射频芯片的物理 MAC 地址 |
| **CAPWAP** | Control and Provisioning of Wireless APs | 无线接入点控制和配置协议 | AC 与 AP 间通信隧道协议，分为 UDP 5246（控制）与 UDP 5247（数据）通道 |
| **VAP** | Virtual Access Point | 虚拟接入点 | 将单个物理 AP 虚拟化为多个逻辑 AP，各绑定独立 SSID、安全策略与业务 VLAN |
| **HSB** | Hot Standby | 热备份服务 | 华为无线 AC 双机高可用机制，实时在主备 AC 间同步 AP 状态与用户上线信息 |
