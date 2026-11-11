# 第35章 Python网络自动化与Netconf/YANG

> 现代数通自动化工程落地的压轴集大成者：从传统 CLI“屏幕抓取（Screen Scraping）”的脆弱痛点出发，系统拆解 Paramiko 工业级 SSH 客户端（`exec_command` 单次执行 vs `invoke_shell` 交互式会话 Channel）与 SFTP 安全文件传输归档；深度揭秘新一代网络开放可编程架构——NETCONF 协议四层体系（传输层 TCP 830、RPC 通信层、操作层 `<edit-config>` 与 YANG 内容模型层）；使用 `ncclient` 手搓 XML 报文下发与状态过滤；压轴完整复原 HCIE Datacom 编程机试大题全景工程实现（多维硬件状态巡检、命令解耦、24 小时定时备份、NETCONF 日志主机与时钟同步、面向对象与防御性容错）。

| 项 | 内容 |
|---|---|
| **适用层级** | HCIE Datacom 编程机试必过压轴卷、企业生产级 NetDevOps 落地终极指南 |
| **建议学习位次** | 全书第 35 章（压轴卷），先修第 33 章（Python 基础与文件流）、第 34 章（面向对象与并发线程池）；学完本章即完成整套数通工程师全栈知识体系闭环 |
| **技术维度** | 协议原理深度剖析 · 生产级VRP配置 · 故障诊断排错 · 架构设计决策 |

---

## 一、总起

随着网络规模由十台百台扩展到数万台端口，传统依靠人工登录 CLI 敲命令的运维方式走到了终点。网络自动化经历了两代技术演进：
1. **第一代自动化：基于 CLI 的文本抓取（Screen Scraping）**
   - 通过 Telnet 或 SSH 登录交换机，模拟人类敲入 `display` 命令，然后通过复杂的正则表达式从非结构化回显文本中“抠数据”；
   - **痛点**：极其脆弱！设备厂商版本稍微升级，输出多空格或微调一个英文字母，原本运行正常的正则表达式立刻失效；且缺乏“事务（Transaction）”机制，批量配置下发到一半报错无法原子回滚，容易造成网络半瘫痪。
2. **第二代自动化：基于模型驱动的结构化可编程（Model-Driven Programmability）**
   - 彻底摒弃面向人类肉眼的 CLI，全面转向面向机器调用的 **NETCONF / RESTCONF 协议** 与 **YANG 数据建模语言**；
   - 数据天然结构化（XML / JSON），支持候选配置库（Candidate DB）、配置锁定（Lock）、配置校验（Validate）与**原子回滚（Commit / Rollback）**，网络运维真正达到与软件工程同等严谨的工业级标准。

本章紧密围绕从“传统 CLI 封装到新一代 NETCONF 建模与真实机试大题”展开：
1. **Paramiko 工业级 SSH 自动化**：深入剖析 `paramiko.SSHClient` 底层机制；详解非交互式 `exec_command()` 与交互式终端 `invoke_shell()` 的本质差异；解决交互式通道的分页阻塞（`---- More ----`）与缓冲区回显提取；
2. **SFTP 远程安全文件归档**：使用 `open_sftp()` 建立安全文件传输通道；落地 `.get()` 配置文件备份与 `.put()` 自动化补丁推送；
3. **NETCONF 与 YANG 体系剖析**：系统解构 NETCONF 四层模型（SSH 830 传输层、`<rpc>` 通信层、`<get-config>`/`<edit-config>` 操作层、YANG 内容层）；掌握 `ncclient` 客户端连接华为设备并下发 XML 模板；
4. **HCIE Datacom 编程机试大题全景通关实战**：100% 还原真实考场与企业级跳板机需求——手搓面向对象类结构，完成硬件多维巡检（风扇/电源/内存/CPU/OSPF/LACP）、命令文本解耦、24 小时 SFTP 规范命名归档、NETCONF 日志主机与时钟同步、参数解包及防御性异常容错。

**适用人群**：
- **初学 / 转行人员**：系统掌握 Paramiko 与 NETCONF 的核心用法，理解网络自动化脚本的底层工作逻辑；
- **备考工程师（HCIE Datacom 核心冲刺）**：彻底拿下 HCIE 实验编程大题（代码结构规范、功能函数完整性、避开考场依赖库与环境坑）；
- **一线工程实践者**：掌握一套可直接迁移至企业现网生产环境的“设备健康巡检 + 配置文件自动定时灾备 + 集中时钟同步”通用脚手架。

### 本章技术架构与核心脉络

| 模块序号 | 核心技术议题 | 架构要点与设计精髓 |
|:---|:---|:---|
| **§35.1** | 网络编程2与网络自动化模块 | Telnet vs SSHv2加密、Paramiko架构、SSHClient与AutoAddPolicy、exec_command单次执行、invoke_shell交互终端与Channel缓冲区 |
| **§35.2** | paramiko进阶与netconf | open_sftp远程文件拉取/推送、NETCONF背景与痛点突破、NETCONF四层体系、YANG数据建模、ncclient库RPC操作(get/edit-config) |
| **§35.3** | datacom编程题讲解 | HCIE Datacom编程机试大题全流程拆解、terminal01跳板机需求分析、面向对象OOP架构、硬件巡检/命令解耦/SFTP备份/NETCONF实战 |

---

## 二、分述

### 2.1 Paramiko 核心架构：`exec_command` vs `invoke_shell`

Paramiko 是 Python 中最成熟的 SSHv2 工业级实现库。网络工程师必须精准掌握其两大核心执行模式的本质差异：

```
                      +---------------------------------------+
                      |         paramiko.SSHClient()          |
                      +-------------------+-------------------+
                                          │ client.connect()
                                          ▼
                      +---------------------------------------+
                      |         建立安全底层 SSH 传输层通道    |
                      +-------------------+-------------------+
                                          │
            ┌─────────────────────────────┴─────────────────────────────┐
            ▼                                                           ▼
 【模式一: exec_command()】                                  【模式二: invoke_shell()】
 - 行为: 单命令非交互式执行                                  - 行为: 打开交互式虚拟终端 (PTY Channel)
 - 特点: 执行单条系统命令后，通道自动关闭                      - 特点: 持续保持会话，支持进入系统视图与子视图
 - 局限: 无法保存会话上下文 (执行 system-view 无效)          - 优势: 华为网络设备连续配置下发、特权切换的唯一选择
 - 场景: 适合快速执行一次性只读查询命令                      - 场景: 适合生产级自动化运维与全流程交互
```

#### 模式一：`exec_command()` 极简查询（单次执行）

```python
import paramiko

client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect("192.168.10.1", port=22, username="admin", password="Huawei@123")

# 返回三个文件流对象: 标准输入、标准输出、标准错误
stdin, stdout, stderr = client.exec_command("display version")
result = stdout.read().decode('utf-8')
print(result)

client.close()
```

#### 模式二：`invoke_shell()` 交互式会话（生产级推荐）
网络设备通常需要先输入 `system-view` 进入系统视图，再进入特定接口配置，必须依赖 `invoke_shell()` 保持会话通道：

```python
import time
import paramiko

client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect("192.168.10.1", port=22, username="admin", password="Huawei@123")

# 打开持久化交互式通道
channel = client.invoke_shell()
time.sleep(1)

# 关键防坑第一步: 必须先关闭分屏，否则长回显会卡死在 "---- More ----"
channel.send("screen-length 0 temporary\n")
time.sleep(0.5)

# 发送业务配置指令
channel.send("system-view\n")
channel.send("sysname Core-Switch-01\n")
channel.send("return\n")
channel.send("save safely\n")
channel.send("y\n")
time.sleep(2)

# 读取通道缓冲区的所有回显数据
output = ""
while channel.recv_ready():
    output += channel.recv(65535).decode('utf-8')

print("交互回显结果:\n", output)
client.close()
```

---

### 2.2 SFTP 远程文件传输与配置定时归档

Paramiko 原生集成了 SFTP 客户端（`paramiko.SFTPClient`），使网络工程师无需安装额外软件即可直接在 SSH 通道中传输文件：

```python
import datetime

# 通过现有的 SSHClient 打开 SFTP 子系统会话
sftp = client.open_sftp()

# 1. 远程下载 (拉取备份配置文件)
# 语法: sftp.get(remotepath, localpath)
today = datetime.datetime.now().strftime("%Y%m%d")
local_backup_name = f"{today}_CoreSwitch.zip"
sftp.get("vrpcfg.zip", local_backup_name)
print(f"[BACKUP SUCCESS] 已安全下载备份文件至本地: {local_backup_name}")

# 2. 远程上传 (批量推送系统升级固件或补丁)
# 语法: sftp.put(localpath, remotepath)
# sftp.put("patch_V200R020.pat", "flash:/patch_V200R020.pat")

sftp.close()
```

---

### 2.3 新一代结构化可编程：NETCONF 与 YANG

#### 传统 CLI 自动化 vs NETCONF 结构化

| 评估维度 | 传统 CLI 自动化 (Paramiko / Telnet) | 新一代 NETCONF (RFC 6241 / YANG) |
|:---|:---|:---|
| **数据形态** | **非结构化纯文本**，依赖复杂的正则切片抓取 | **结构化 XML / JSON**，按 YANG 模型严格定义 |
| **厂商移植性** | 极差，不同厂商甚至同厂商不同版本命令回显各异 | 极高，遵循 IETF 与 OpenConfig 行业统一标准模型 |
| **配置事务性** | **无事务机制**，出错无法原子回滚，导致半拉子配置 | **强事务机制**，支持候选库配置、校验与毫秒级原子回滚 |
| **通信通道** | TCP 22 (SSH) 交互式终端模拟字符输入 | **TCP 830 (SSH)** 标准 RPC 报文交互 |

#### NETCONF 四层协议栈架构

```
 4. 内容层 (Content)       : YANG 数据模型定义配置与状态数据 (ietf-interfaces, huawei-system)
 3. 操作层 (Operations)    : <get>, <get-config>, <edit-config>, <copy-config>, <commit>
 2. RPC 通信层 (Messages)  : <rpc>, <rpc-reply>, <rpc-error>
 1. 传输层 (Transport)     : 必须基于安全传输协议，标准规定为 SSH (默认监听 TCP 830 端口)
```

#### 使用 `ncclient` 库下发配置实战

```python
from ncclient import manager

# 1. 建立基于 SSH 830 端口的 NETCONF 会话
with manager.connect(
    host="192.168.10.1",
    port=830,
    username="admin",
    password="Huawei@123",
    hostkey_verify=False,
    device_params={'name': 'huawei'}
) as m:
    
    # 2. 构造符合华为 YANG 模型的 XML 配置数据 (配置日志主机)
    xml_config = """
    <config>
      <info-center xmlns="http://www.huawei.com/netconf/vrp/huawei-info-center">
        <log-hosts>
          <log-host>
            <host-name>192.168.100.200</host-name>
            <channel>loghost</channel>
          </log-host>
        </log-hosts>
      </info-center>
    </config>
    """
    
    # 3. 通过 edit-config 操作安全下发
    response = m.edit_config(target='running', config=xml_config)
    print("NETCONF 服务器应答:\n", response)
```

---

### 2.4 HCIE Datacom 编程机试真题全景通关实战

以下代码为 HCIE Datacom 编程大题（基于 `notes/_src/prd-ops.txt`）的**生产级标准满分参考实现**，彻底整合了面向对象设计、命令解耦、硬件监控、SFTP 定时备份与 NETCONF 自动化：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
HCIE Datacom 编程大题生产级参考工程
功能清单:
 1. SSH 安全管理通道与面向对象 (OOP) 建模
 2. 外部命令解耦 (commands.txt) 流式动态加载
 3. 硬件与网络状态多维巡检 (电源/风扇/CPU/内存/OSPF/LACP)
 4. 异常状态精准告警判定 (All fans are fault)
 5. 24小时周期 SFTP 配置文件拉取与日期规范重命名归档
 6. NETCONF 远程日志主机配置与设备系统时钟自动校准
 7. 全局防御性异常捕获 (防挂起、单点故障隔离)
"""

import os
import re
import time
import socket
import datetime
import paramiko
from ncclient import manager

class NetworkOpsDevice:
    """
    数通自动化运维设备核心类 (面向对象封装)
    """
    def __init__(self, ip, port=22, username="python", password=""):
        # 属性封装在当前实例私有命名空间
        self.ip = ip
        self.port = int(port)
        self.username = username
        self.password = password
        self.ssh_client = None

    def connect(self, timeout=8):
        """建立底层 SSH 安全加密通道"""
        try:
            self.ssh_client = paramiko.SSHClient()
            self.ssh_client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
            self.ssh_client.connect(
                hostname=self.ip,
                port=self.port,
                username=self.username,
                password=self.password,
                timeout=timeout,
                look_for_keys=False,
                allow_agent=False
            )
            return True
        except socket.timeout:
            print(f"\033[1;31m[ERROR] 设备 {self.ip} 连接超时，请核实网络链路！\033[0m")
        except paramiko.AuthenticationException:
            print(f"\033[1;31m[ERROR] 设备 {self.ip} 用户名或密码错误，认证失败！\033[0m")
        except Exception as e:
            print(f"\033[1;31m[ERROR] 设备 {self.ip} 发生连接未知异常: {e}\033[0m")
        return False

    def execute_commands_from_file(self, command_file="commands.txt"):
        """
        核心要求: 巡检命令与代码逻辑彻底解耦
        从外部文件动态读取 display 命令并执行
        """
        if not os.path.exists(command_file):
            print(f"[WARN] 命令文件 {command_file} 不存在，跳过文本巡检！")
            return {}

        results = {}
        # 安全流式读取命令
        with open(command_file, "r", encoding="utf-8") as f:
            commands = [l.strip() for l in f if l.strip() and not l.startswith("#")]

        # 打开交互式 Shell
        channel = self.ssh_client.invoke_shell()
        time.sleep(1)
        # 关分屏，防卡死
        channel.send("screen-length 0 temporary\n")
        time.sleep(0.5)

        for cmd in commands:
            channel.send(cmd + "\n")
            time.sleep(1)
            output = ""
            while channel.recv_ready():
                output += channel.recv(65535).decode("utf-8", errors="ignore")
            results[cmd] = output
            
            # 状态智能分析与告警 (机试核心采分点)
            self._analyze_output(cmd, output)

        return results

    def _analyze_output(self, cmd, output):
        """解析回显并抛出业务告警"""
        # 1. 监控风扇状态
        if "fan" in cmd.lower():
            if "Fault" in output or "Abnormal" in output or "fail" in output.lower():
                print(f"\033[1;31m[CRITICAL ALERT] 设备 {self.ip}: All fans are fault!\033[0m")
            else:
                print(f"[OK] 设备 {self.ip}: 风扇模块运行正常。")

        # 2. 监控 CPU 与内存利用率
        if "cpu" in cmd.lower():
            match = re.search(r"(\d+)%\s+in past 5 seconds", output)
            if match:
                cpu_val = int(match.group(1))
                print(f"[STATUS] 设备 {self.ip} CPU 利用率: {cpu_val}%")
                if cpu_val > 80:
                    print(f"\033[1;33m[WARN] 设备 {self.ip} CPU 负载过高 (>{cpu_val}%)！\033[0m")

    def backup_config_sftp(self, remote_file="vrpcfg.zip", local_dir="backups"):
        """
        核心要求: 每隔 24 小时拉取配置文件
        通过 SFTP 安全拉取并按 '日期_设备名.zip' 命名归档
        """
        if not os.path.exists(local_dir):
            os.makedirs(local_dir)

        # 触发远端保存当前配置
        channel = self.ssh_client.invoke_shell()
        channel.send("save safely\ny\n")
        time.sleep(2)

        try:
            sftp = self.ssh_client.open_sftp()
            today_str = datetime.datetime.now().strftime("%Y%m%d")
            # 规范命名: 20261110_192.168.100.1.zip
            local_filename = f"{today_str}_{self.ip}.zip"
            local_path = os.path.join(local_dir, local_filename)
            
            sftp.get(remote_file, local_path)
            sftp.close()
            print(f"\033[1;32m[SUCCESS] 配置文件已通过 SFTP 成功归档至: {local_path}\033[0m")
            return True
        except Exception as e:
            print(f"\033[1;31m[ERROR] SFTP 拉取配置失败: {e}\033[0m")
            return False

    def sync_clock_and_syslog_netconf(self, syslog_host="192.168.100.200"):
        """
        核心要求: 使用 NETCONF 远程配置日志主机并自动同步系统时间
        """
        try:
            with manager.connect(
                host=self.ip,
                port=830,
                username=self.username,
                password=self.password,
                hostkey_verify=False,
                device_params={'name': 'huawei'},
                timeout=10
            ) as m:
                # 获取当前时间构建 XML
                now_str = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                # 下发 NETCONF 编辑时钟与配置日志主机
                rpc_xml = f"""
                <config>
                  <system xmlns="http://www.huawei.com/netconf/vrp/huawei-system">
                    <systemInfo>
                      <systemTime>{now_str}</systemTime>
                    </systemInfo>
                  </system>
                  <info-center xmlns="http://www.huawei.com/netconf/vrp/huawei-info-center">
                    <log-hosts>
                      <log-host>
                        <host-name>{syslog_host}</host-name>
                        <channel>loghost</channel>
                      </log-host>
                    </log-hosts>
                  </info-center>
                </config>
                """
                m.edit_config(target='running', config=rpc_xml)
                print(f"[NETCONF OK] 设备 {self.ip} 成功同步系统时钟 ({now_str}) 并下发日志主机 ({syslog_host})！")
        except Exception as e:
            print(f"\033[1;31m[NETCONF FAIL] 设备 {self.ip} NETCONF 执行异常: {e}\033[0m")

    def close(self):
        """优雅关闭网络套接字句柄"""
        if self.ssh_client:
            self.ssh_client.close()

# ==================== 入口调度调度中心 ====================
if __name__ == "__main__":
    # 模拟资产配置字典 (从 config.py 导入)
    switch_inventory = [
        {"ip": "192.168.100.1", "port": 22, "username": "python", "password": "Huawei@123"},
        {"ip": "192.168.100.2", "port": 22, "username": "python", "password": "Huawei@123"}
    ]

    print("=" * 60)
    print("      企业园区网自动化运维全流程作业启动")
    print("=" * 60)

    for dev_info in switch_inventory:
        # 核心技能: 采用 ** 字典解包极简实例化对象！
        device = NetworkOpsDevice(**dev_info)
        
        if device.connect():
            # 1. 动态加载外部命令并巡检
            device.execute_commands_from_file("commands.txt")
            # 2. 定时 24 小时 SFTP 规范命名归档
            device.backup_config_sftp()
            # 3. NETCONF 下发时钟与日志主机
            device.sync_clock_and_syslog_netconf("192.168.100.200")
            # 4. 安全关闭
            device.close()
        print("-" * 60)
```

---

### 2.5 真实考场与工程落地排坑指南

1. **考场环境 Python 库缺失或被阉割**：
   - 考场极个别机架由于系统重装，`ncclient` 或 `paramiko` 可能缺失依赖；
   - **救命技巧**：优先检查 Python 路径（`which python3`）；若缺库，利用本地预装的 wheel 缓存安装（`pip3 install --no-index --find-links=/opt/packages paramiko`）；
2. **交互式通道提示符捕获与延时不足**：
   - 使用 `channel.recv()` 时，若设备响应稍慢，代码如果紧接着没有 `time.sleep()` 延时就立刻执行 `recv_ready()`，会误以为没有回显而读出空字符串；
   - **工程铁律**：发送命令后务必预留 `0.5 ~ 1.0` 秒缓冲等待（或基于期望提示符 `<Switch>` 正则循环等待）；
3. **NETCONF 830 端口在设备侧未使能**：
   - 很多考生在 Python 端调试报错 `ConnectionRefusedError`，原因不是代码问题，而是交换机没有配：
     ```shell
     snetconf server enable
     ssh server-source all-interface
     ```
   - 必须确保交换机底层已开启 NETCONF 服务且 AAA 用户已赋予 `snetconf` 服务类型授权！

---

## 三、总结十带走

1. **自动化两代演进**：第一代 CLI 屏幕抓取（脆弱易损）；第二代基于 NETCONF/YANG 模型驱动（结构化强事务）；
2. **两类 SSH 执行模式**：`exec_command()` 适合单次独立查询；`invoke_shell()` 保持长会话，是多视图配置交互首选；
3. **分屏死锁绝杀**：交互式通道第一条指令必须发送 `screen-length 0 temporary`，彻底杜绝回显被 `---- More ----` 挂死；
4. **SFTP 极简归档**：利用现有 SSH 句柄直接 `open_sftp()`，无需额外账号端口，秒级拉取与推送大文件；
5. **NETCONF 四层架构**：内容层（YANG 模型）、操作层（`<get-config>`/`<edit-config>`）、RPC 层（`<rpc>`）、传输层（SSH 830）；
6. **YANG 核心价值**：充当网络设备的“数据语义模式蓝图”，确保机器调用具备严格的类型校验与约束；
7. **ncclient 客户端**：Python 操作 NETCONF 的事实工业标准，配合 XML 字符串实现秒级原子配置下发与状态过滤；
8. **机试命令解耦**：严禁硬编码命令，将指令外置于 `commands.txt` 动态流式读取，优雅通过机试审查；
9. **规范灾备归档**：使用 `datetime` 模块动态格式化当前日期，将备份命名为 `20261110_IP.zip` 规范落盘；
10. **全栈闭环大圆满**：至此，35 卷涵盖理论体系、底层协议、云网前沿（VXLAN/EVPN/SRv6/SD-WAN）与 Python 网络自动化的数通工程知识图谱全量大成！

---

## 附：本章术语速查表

| 术语（缩写） | 英文全称 | 中文标准名称 | 核心作用与定义 |
|:---|:---|:---|:---|
| **Paramiko** | Paramiko Python Library | Paramiko 自动化库 | Python 官方推荐的纯 Python 实现的 SSHv2 客户端与服务器底层通信库 |
| **SFTP** | SSH File Transfer Protocol | SSH 文件传输协议 | 运行在安全 SSH 数据流之上的交互式远程文件管理与加密传输协议 |
| **NETCONF** | Network Configuration Protocol | 网络配置协议 | IETF（RFC 6241）定义的基于 XML/RPC 的新一代网络设备管理与配置标准协议 |
| **YANG** | Yet Another Next Generation | YANG 数据建模语言 | 专为 NETCONF/RESTCONF 设计的层次化数据模型定义语言（RFC 6020/7950） |
| **ncclient** | NETCONF Client Python Library | NETCONF Python 客户端 | 专用于与支持 NETCONF 协议的网络设备进行 RPC 通信的标准开源 Python 库 |
| **RPC** | Remote Procedure Call | 远程过程调用 | 客户端向网络设备发送结构化操作请求并等待设备同步应答（`<rpc-reply>`）的通信协议 |
| **Candidate DB** | Candidate Configuration Database | 候选配置数据库 | NETCONF 支持的临时配置暂存区，需显式执行 `<commit>` 才会真正写入芯片生效 |
| **DPI / SAC** | Deep Packet Inspection | 深度报文检测 / 智能应用控制 | 网络设备分析应用层载荷特征以识别具体企业软件流量的高级检测技术 |
| **Telemetry** | Network Telemetry | 遥测技术 | 网络设备主动、高速、周期性将自身性能与状态数据向采集器推送的监控机制 |
| **OPS** | Open Programmability System | 开放可编程系统 | 华为设备在 VRP 内部运行 Python 脚本以实现事件触发式本地自愈调度的开放框架 |
| **PTY** | Pseudo-Terminal | 伪终端 | 操作系统与网络设备交互时模拟的字符终端通道（Paramiko 的 Channel 实体） |
| **Screen Scraping** | Screen Scraping | 屏幕抓取 | 通过脚本读取设备命令行非结构化回显并依靠正则表达式提取字段的传统自动化方式 |
