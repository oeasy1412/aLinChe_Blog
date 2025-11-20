---
title: WireShark
description: WireShark & iptables
published: 2025-09-01
tags: [net]
category: net
draft: false
---

## WireShark
```sh
sudo apt install wireshark tshark
# sudo dpkg-reconfigure wireshark-common # Yes
sudo usermod -aG wireshark $USER

	
sudo chgrp wireshark /usr/bin/dumpcap
sudo chmod 750 /usr/bin/dumpcap
sudo setcap cap_net_raw,cap_net_admin=eip /usr/bin/dumpcap

tshark -D # 列出可用网络接口
	
tshark -i eth0 -w output.pcap
tshark -r output.pcap         # 读取和分析已保存的抓包文件
tshark -r output.pcap -Y "ip.src == 192.168.1.100 and ip.dst == 192.168.1.102 and tcp.port == 80"

tshark -i eth0 -f "tcp port 80 and (src host 192.168.1.100 or src host 192.168.1.102)"
tshark -i eth0 -f "tcp port 80 and src net 192.168.1.0/24"
tshark -i eth0 -Y "http"
```

## iptables
在 Linux 运维和网络安全，虽然现在有了 Firewalld 甚至 eBPF 等新技术，但 iptables 依然是底层逻辑的基石
### 四表五链核心概念
首先需要明确一点：**iptables只是用户空间的一个管理工具**，**真正的防火墙功能是由内核中的 `netfilter` 实现**的。iptables 的作用就是帮我们在 Netfilter 上“挂”规则。
#### 五链（Chains）：数据包的必经关卡
五个链对应着数据包在Linux网络栈中传输路径上的五个关键"钩子点"（`hook points`），相当于数据包必须经过的五道关卡：
![chains](images/5chains.png)
1. **PREROUTING链**：数据包刚进入网络接口后，但在路由判断之前的阶段。这里是进行DNAT（目标地址转换）的理想位置。
2. **INPUT链**：当数据包经过路由判断，发现目标是本机系统时，就会进入INPUT链。这是**最常用的过滤点**，用于控制进入本机应用程序的数据包。
3. **FORWARD链**：当数据包需要经过本机转发到其他机器时（本机作为路由器或网关），会经过FORWARD链。
4. **OUTPUT链**：处理本机进程产生的、准备发送到外部的数据包。
5. **POSTROUTING链**：在数据包离开本机之前的最后阶段，这里是进行SNAT（源地址转换）的理想位置。

#### 四表（Tables）：规则的功能分类
四个表是按照**功能优先级**组织的规则集合，每个表包含不同的链：

| 表名 | 优先级 | 主要功能 | 包含的链 |
|------|--------|----------|----------|
| raw表 | 最高（1） | 决定是否跳过连接跟踪机制 | PREROUTING, OUTPUT |
| mangle表 | 2 | 修改数据包头部信息（TTL、TOS等） | 所有五个链 |
| nat表 | 3 | 网络地址转换（SNAT、DNAT） | PREROUTING, INPUT, OUTPUT, POSTROUTING |
| filter表 | 最低（4） | 过滤数据包（允许/拒绝） | INPUT, FORWARD, OUTPUT |

**iptables四表的优先级和功能对比**
- **filter表**是最常用的表，负责基本的包过滤功能，是iptables的默认表（当不指定`-t`参数时）。
- **nat表**专用于网络地址转换，包括端口转发、地址伪装等功能。
- **mangle表**用于高级包处理，如修改包头部信息，通常在日常管理中使用较少。
- **raw表**用于处理不需要连接跟踪的数据包，可提升高性能场景下的处理效率。

### 数据包在 iptables 中的完整处理流程
#### 1 入站数据包（目标为本机）
1. **物理网卡接收** → **PREROUTING链**（raw → mangle → nat）
2. → **路由判断**（识别数据包目标是本机）
3. → **INPUT链**（mangle → nat → filter）
4. → **本地进程处理**
*这是一个外部用户访问本机Web服务的典型路径。* 在PREROUTING链中，主要进行的是连接跟踪和可能的DNAT处理；而在INPUT链中，主要进行的是过滤决策。

#### 2 转发数据包（经本机路由）
1. **物理网卡接收** → **PREROUTING链**（raw → mangle → nat）
2. → **路由判断**（识别数据包需要转发到其他主机）
3. → **FORWARD链**（mangle → filter）
4. → **POSTROUTING链**（mangle → nat）
5. → **发送到目标主机**
*这是路由器或网关设备的典型数据流。* 需要注意的是，在FORWARD链中，只有mangle表和filter表参与处理，因为转发的数据包通常不需要在转发过程中进行NAT处理（NAT一般在PREROUTING或POSTROUTING阶段完成）。

#### 3 出站数据包（本机进程产生）
1. **本地进程产生数据** → **路由判断**
2. → **OUTPUT链**（raw → mangle → nat → filter）
3. → **POSTROUTING链**（mangle → nat）
4. → **物理网卡发送**
*这是本机应用程序访问外部服务的典型路径。* 在OUTPUT链中，数据包会经过所有四个表的处理，这是最复杂的路径。

#### 4 优先级决定处理顺序
**表的处理顺序是固定的**：raw → mangle → nat → filter。这一顺序是由数据包处理流程的内在逻辑决定的。

以PREROUTING链为例，当一个数据包进入时：
1. 首先由raw表处理（决定是否跳过连接跟踪）
2. 然后由mangle表处理（修改包头部信息）
3. 最后由nat表处理（目标地址转换）

**这个顺序很重要**！我曾经遇到一个案例：在filter表中写了规则拒绝某个IP，但在nat表中又写了DNAT规则转发同一IP的请求。结果发现DNAT规则生效了，因为nat表的优先级比filter表高，数据包在到达filter表之前就被转发了。

### 3 iptables命令语法与使用详解
```bash
# 基本命令格式：
iptables [-t 表名] 命令选项 [链名] [匹配条件] [-j 目标动作]
```

#### 1 命令管理选项
管理选项决定了操作类型，常用的有：

- **规则管理选项**：
  - `-A --append` 在链末尾追加规则
  - `-I --insert` 在指定位置插入规则（默认为链首）
  - `-D --delete` 删除指定规则
  - `-R --replace` 替换指定规则

- **链管理选项**：
  - `-N --new-chain` 创建用户自定义链
  - `-X --delete-chain` 删除用户自定义链
  - `-F --flush` 清空链中所有规则
  - `-L --list` 列出链中规则
  - `-P --policy` 设置链的默认策略

- **其他实用选项**：
  - `-n --numeric` 以数字形式显示IP和端口
  - `-v --verbose` 显示详细信息
  - `--line-numbers` 显示规则编号

#### 2 匹配条件参数
匹配条件用于筛选要处理的数据包，可分为基本匹配和扩展匹配。

**基本匹配条件**（常用且无需加载模块）：
- `-s --source`：匹配源IP地址或网段（如 `-s 192.168.1.100` 或 `-s 192.168.1.0/24`）
- `-d --destination`：匹配目标IP地址或网段
- `-p --protocol`：匹配协议类型（tcp、udp、icmp等）
- `-i --in-interface`：匹配数据包进入的网络接口
- `-o --out-interface`：匹配数据包发出的网络接口

**扩展匹配条件**（需用`-m`指定模块）：
- `-m tcp --dport`：匹配TCP目标端口（如 `-p tcp -m tcp --dport 80`）
- `-m tcp --sport`：匹配TCP源端口
- `-m state --state`：匹配连接状态（NEW、ESTABLISHED、RELATED、INVALID）
- `-m multiport --dports`：匹配多个不连续端口（如 `--dports 80,443,8080`）
- `-m limit --limit`：限制匹配速率（如 `--limit 5/minute`）

#### 3 目标动作（-j参数）
当数据包匹配规则时，需要指定要执行的动作：

- **ACCEPT**：允许数据包通过，继续后续处理
- **DROP**：直接丢弃数据包，不发送任何响应
- **REJECT**：拒绝数据包，并向发送端返回错误响应
- **LOG**：将数据包信息记录到系统日志，然后继续匹配后续规则
- **DNAT**：目标地址转换，用于端口转发或负载均衡
- **SNAT**：源地址转换，用于共享上网
- **MASQUERADE**：动态SNAT，适用于动态获取IP的场景
- **RETURN**：从当前链返回调用链继续处理

### iptables实战配置场景
#### 场景〇：保存
```sh
# iptables规则是在内存中的，重启后会丢失。持久化方法因发行版而异
sudo apt install iptables-persistent
netfilter-persistent save
```

#### 场景一：基础Web服务器防火墙配置
对于一台暴露在公网的Web服务器，安全配置是首要任务。以下是推荐配置步骤：
```bash
# 0. 清空所有现有规则，从零开始
# iptables -F
# iptables -t nat -F
# iptables -t mangle -F

# 1. 设置默认策略（谨慎操作！）
iptables -P INPUT DROP      # 默认拒绝所有入站连接
iptables -P FORWARD DROP    # 默认拒绝所有转发
iptables -P OUTPUT ACCEPT   # 允许所有出站连接

# 2. 允许本地回环接口，许多本地服务依赖它
iptables -A INPUT -i lo -j ACCEPT

# 3. 允许已建立连接和关联连接的通话（避免断开现有会话）
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 4. 开放SSH远程管理端口（建议限制源IP以提高安全性）
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 7. 开放Web服务端口
iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # HTTP
iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # HTTPS

# 8. 允许ICMP（ping请求）便于网络诊断
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
```
*此配置创建了一个**白名单模型**的防火墙，只有明确允许的服务才能访问。*

#### 场景二：NAT网关共享上网
将Linux服务器配置为NAT网关，让内网设备通过它共享上网：
```bash
# 1. 启用IP转发功能
sudo vim /etc/sysctl.conf # 取消注释 #net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf

# 2. 在nat表的POSTROUTING链设置地址伪装（SNAT）
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE

# 3. 在filter表允许内网到外网的转发
iptables -A FORWARD -s 192.168.1.0/24 -o eth0 -j ACCEPT

# 4. 允许外网返回数据的转发
iptables -A FORWARD -d 192.168.1.0/24 -i eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```
*此处`MASQUERADE`是`SNAT`的特殊形式，适用于动态获取公网IP的场景（如PPPoE拨号）。*

#### 场景三：端口转发（DNAT）
将公网IP的端口转发到内网服务器：
```bash
# 将本机8080端口的访问转发到内网服务器192.168.1.100的80端口
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.100:80

# 同时需要允许转发的数据包通过
iptables -A FORWARD -d 192.168.1.100 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -s 192.168.1.100 -p tcp --sport 80 -j ACCEPT
```
*这种配置常用于将一台公网服务器作为多个内网服务的入口网关。*

#### 场景四：高级流量控制
结合mangle表和limit模块进行高级流量控制：
```bash
# 1. 限制ICMP(ping)请求频率，防止洪水攻击
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/second -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

# 2. 为特定端口的流量设置DSCP标记（QoS）
iptables -t mangle -A OUTPUT -p tcp --dport 80 -j DSCP --set-dscp 46

# 3. 使用connlimit模块限制每IP的SSH连接数
iptables -A INPUT -p tcp --dport 22 -m connlimit --connlimit-above 3 -j DROP

# 4. 使用recent模块防御SSH暴力破解
iptables -A INPUT -p tcp --dport 22 -m recent --name ssh --set
iptables -A INPUT -p tcp --dport 22 -m recent --name ssh --update --seconds 60 --hitcount 4 -j DROP
```
*这些高级技巧可以帮助你构建更加健壮和安全的防火墙环境。*

### iptables注意事项与实用技巧
#### 1 避免常见陷阱
1. **规则顺序至关重要**：iptables规则**从上到下依次匹配**，一旦匹配成功即停止。常见的错误是将宽泛的拒绝规则放在前面，导致后面的允许规则永不生效：
```bash
# 错误示例（第二条规则永远无法匹配）：
iptables -A INPUT -j DROP
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 正确做法（先允许后拒绝）：
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -j DROP
```
2. **谨慎设置默认策略**：在远程服务器上设置`iptables -P INPUT DROP`前，**务必先允许SSH连接**，否则会立即断开连接并被锁在服务器外。建议的做法是：先添加所有必要规则，最后再修改默认策略。
3. **不要完全阻塞ICMP**：虽然出于安全考虑可能想禁止ping，但完全阻塞ICMP会导致路径MTU发现等机制失效，影响网络性能。至少应允许ICMP错误消息。
#### 2 性能优化技巧
1. **使用连接跟踪优化**：对高流量服务（如Web服务器），可考虑跳过连接跟踪以提升性能：
```bash
iptables -t raw -A PREROUTING -p tcp --dport 80 -j NOTRACK
iptables -t raw -A OUTPUT -p tcp --sport 80 -j NOTRACK
iptables -A INPUT -p tcp --dport 80 -m state --state UNTRACKED -j ACCEPT
```
2. **合理规划规则顺序**：将最常匹配的规则放在前面，减少匹配时间。使用`iptables -L -v`查看规则匹配计数器来优化顺序。
3. **避免过于复杂的规则集**：每个额外规则都会增加处理开销，定期清理不再需要的规则。

#### 4 故障排查命令
当遇到网络连接问题时，这些命令有助于快速定位问题：
```bash
# 1. 查看所有规则（带行号和计数器）
iptables -L -n -v --line-numbers
# 2. 检查NAT表规则
iptables -t nat -L -n -v
# 3. 追踪数据包处理路径（动态查看匹配过程）
iptables -t raw -A PREROUTING -p tcp --dport 80 -j TRACE
# 4. 检查连接跟踪表
cat /proc/net/nf_conntrack
# 5. 清空计数器重新开始统计
iptables -Z
```

### ipset
在掌握了iptables的基础后，管理大量IP地址或端口规则时，你可能会发现规则集变得臃肿且难以维护。此时，**ipset** 就成了你的得力助手。它允许你将大量的IP地址、端口号等元素放入一个**命名的集合**中，然后让一条iptables规则来引用整个集合，从而极大提升管理效率和规则匹配性能。

下面这个表格帮你快速了解ipset常见的集合类型。
| 集合类型 | 描述 | 典型应用场景 |
| :--- | :--- | :--- |
| `hash:ip` | 存储单个IP地址 | 管理黑名单或白名单中的独立IP |
| `hash:net` | 存储IP网段（CIDR格式） | 封禁或允许整个子网，如 `192.168.1.0/24` |
| `hash:ip,port` | 存储IP地址和端口号的组合 | 封禁特定IP对特定端口的访问，如 `192.168.1.100,80` |
| `hash:net,port` | 存储IP网段和端口号的组合 | 允许一个网段访问特定服务端口 |
| `list:set` | 存储其他ipset集合的名称，实现集合嵌套 | 创建更复杂的规则组合 |

*   **性能考量**：对于非常大的IP列表（例如成千上万个IP），使用 `hash:ip` 可能会比在iptables中使用数万条独立规则性能好得多，因为ipset基于哈希表，查找效率接近O(1)。

#### ipset基本操作
1.  **创建集合**
    使用 `ipset create` 命令。建议使用描述性的集合名，并指定合适的类型。还可以通过 `maxelem` 指定集合容量，`timeout` 设置条目的默认过期时间。
    ```bash
    # 创建一个名为"blacklist"的IP黑名单，最多支持10万个IP，条目默认永久有效
    ipset create blacklist hash:ip maxelem 100000
    # 创建一个名为"temp_ban"的集合，新添加的IP默认在600秒后自动移除
    ipset create temp_ban hash:ip timeout 600
    ```
2.  **管理集合条目**
    *   **添加条目**：`ipset add SETNAME ENTRY`
        ```bash
        ipset add blacklist 192.168.1.100
        ```
    *   **删除条目**：`ipset del SETNAME ENTRY`
        ```bash
        ipset del blacklist 192.168.1.100
        ```
    *   **查看集合内容**：`ipset list SETNAME` 或 `ipset list`（查看所有集合）
    *   **清空集合**：`ipset flush SETNAME` （删除集合内所有条目，但保留集合本身）
    *   **销毁集合**：`ipset destroy SETNAME` （彻底删除整个集合）
3.  **持久化配置**
    ipset的规则也默认保存在内存中，重启服务器后会丢失。必须手动保存和恢复。
    ```bash
    # 保存所有集合的规则到文件
    ipset save > /etc/ipset.rules
    # 从文件恢复规则
    ipset restore < /etc/ipset.rules
    ```
    你可以将保存命令放入关机脚本，或将恢复命令放入系统启动脚本（如 `/etc/rc.local`）以实现自动持久化。

#### ipset与iptables联动
创建好ipset后，需要在iptables中使用 `-m set --match-set` 选项来匹配它。

*   **匹配源IP**：使用 `src` 标志。
    ```bash
    # 丢弃来自黑名单中IP的所有数据包
    iptables -I INPUT -m set --match-set blacklist src -j DROP
    ```
*   **匹配目的IP**：使用 `dst` 标志。
    ```bash
    # 拒绝发往某个IP集合（如内部服务器列表）的流量
    iptables -I OUTPUT -m set --match-set internal_servers dst -j REJECT
    ```
*   **同时匹配源和目的**：可以组合多个标志。
    ```bash
    # 匹配特定源IP集合到特定目的端口集合的流量（如从办公网到Web服务）
    iptables -A FORWARD -m set --match-set office_net src -m set --match-set web_ports dst -j ACCEPT
    ```
*   **白名单（取反匹配）**：在 `--match-set` 前加 `!`。
    ```bash
    # 只允许白名单中的IP访问SSH，拒绝其他所有IP
    iptables -A INPUT -p tcp --dport 22 -m set ! --match-set whitelist src -j DROP
    ```
