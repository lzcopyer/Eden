> Netfilter 是 Linux 操作系统核心层内部的一个**数据包处理**模块集合的统称, 是一种**流量控制系统**。一种网络筛选系统，对数据包进入以及出去本机进行的一些控制与管理。

# Netfilter 的设计与实现

netfilter 的定义是一个工作在 Linux 内核的网络数据包处理框架，为了彻底理解 netfilter 的工作方式，我们首先需要对数据包在 Linux 内核中的处理路径建立基本认识。

Netfilter 功能的所有模块可以通过下图所示的目录进行查找，其中还包括 ipvs 等。

``` Bash
~]# find /usr/lib/modules/$(uname -r)/kernel -name netfilter -exec realpath {} \;
/usr/lib/modules/5.15.0-102-generic/kernel/net/bridge/netfilter
/usr/lib/modules/5.15.0-102-generic/kernel/net/netfilter
/usr/lib/modules/5.15.0-102-generic/kernel/net/ipv6/netfilter
/usr/lib/modules/5.15.0-102-generic/kernel/net/ipv4/netfilter
```

## 数据包的内核之旅

数据包在内核中的处理路径，也就是处理网络数据包的内核代码调用链，大体上也可按 TCP/IP 模型分为多个层级，以接收一个 IPv4 的 tcp 数据包为例：

1. 在物理-网络设备层，网卡通过 DMA 将接收到的数据包写入内存中的 `ring buffer`，经过一系列中断和调度后，操作系统内核调用 `__skb_dequeue` 将数据包加入对应设备的处理队列中，并转换成 `sk_buffer` 类型（即 socket buffer - 将在整个内核调用栈中持续作为参数传递的基础数据结构，下文指称的数据包都可以认为是 `sk_buffer`），最后调用 `netif_receive_skb` 函数按协议类型对数据包进行分类，并跳转到对应的处理函数。如下图所示：
    
    ![network-path](network-path.png)
    
2. 假设该数据包为 IP 协议包，对应的接收包处理函数 `ip_rcv` 将被调用，数据包处理进入网络（IP）层。`ip_rcv` 检查数据包的 IP 首部并丢弃出错的包，必要时还会聚合被分片的 IP 包。然后执行 `ip_rcv_finish` 函数，对数据包进行路由查询并决定是将数据包交付本机还是转发其他主机。假设数据包的目的地址是本主机，接着执行的 `dst_input` 函数将调用 `ip_local_deliver` 函数。`ip_local_deliver` 函数中将根据 IP 首部中的协议号判断载荷数据的协议类型，最后调用对应类型的包处理函数。本例中将调用 TCP 协议对应的 `tcp_v4_rcv` 函数，之后数据包处理进入传输层。
    
3. `tcp_v4_rcv` 函数同样读取数据包的 TCP 首部并计算校验和，然后在数据包对应的 TCP control buffer 中维护一些必要状态包括 TCP 序列号以及 SACK 号等。该函数下一步将调用 `__tcp_v4_lookup` 查询数据包对应的 socket，如果没找到或 socket 的连接状态处于 **TCP_TIME_WAIT**，数据包将被丢弃。如果 socket 处于未加锁状态，数据包将通过调用 `tcp_prequeue` 函数进入 `prequeue` 队列，之后数据包将可被用户态的用户程序所处理。传输层的处理流程超出本文讨论范围，实际上还要复杂很多。

## netfilter hooks

> hooks function(钩子函数) 是 Linux 网络栈中的流量检查点。所有流量通过网卡进入内核或从内核出去都会调用 Hook 函数来进行检查，并根据其规则进行过滤。Netfilter 框架中一共有 5 个 Hook，这些 Hooks 组成了下文定义的“五链”。

### hook 触发点

对于不同的协议（IPv4、IPv6 或 ARP 等），Linux 内核网络栈会在该协议栈数据包处理路径上的预设位置触发对应的 hook。在不同协议处理流程中的触发点位置以及对应的 hook 名称（蓝色矩形外部的黑体字）如下，本文仅重点关注 IPv4 协议：

![netfilter-flow](2022-03-07-netfilter-flow.png)

所谓的 hook 实质上是代码中的枚举对象（值为从0开始递增的整型）：

``` C
enum nf_inet_hooks {
	NF_INET_PRE_ROUTING,
	NF_INET_LOCAL_IN,
	NF_INET_FORWARD,
	NF_INET_LOCAL_OUT,
	NF_INET_POST_ROUTING,
	NF_INET_NUMHOOKS
};
```

每个 hook 在内核网络栈中对应特定的触发点位置，以 IPv4 协议栈为例，有以下 netfilter hooks 定义：

![netfilter-hooks-stack](2022-03-07-netfilter-hooks-stack.png)

- NF_INET_PRE_ROUTING: 这个 hook 在 IPv4 协议栈的 `ip_rcv` 函数或 IPv6 协议栈的 `ipv6_rcv` 函数中执行。所有接收数据包到达的第一个 hook 触发点（实际上新版本 Linux 增加了 INGRESS hook 作为最早触发点），在进行路由判断之前执行。
- NF_INET_LOCAL_IN: 这个 hook 在 IPv4 协议栈的 `ip_local_deliver()` 函数或 IPv6 协议栈的 `ip6_input()` 函数中执行。经过路由判断后，所有目标地址是本机的接收数据包到达此 hook 触发点。
- NF_INET_FORWARD: 这个 hook 在 IPv4 协议栈的 `ip_forward()` 函数或 IPv6 协议栈的 `ip6_forward()` 函数中执行。经过路由判断后，所有目标地址不是本机的接收数据包到达此 hook 触发点。
- NF_INET_LOCAL_OUT: 这个 hook 在 IPv4 协议栈的 `__ip_local_out()` 函数或 IPv6 协议栈的 `__ip6_local_out()` 函数中执行。所有本机产生的准备发出的数据包，在进入网络栈后首先到达此 hook 触发点。
- NF_INET_POST_ROUTING: 这个 hook 在 IPv4 协议栈的 `ip_output()` 函数或 IPv6 协议栈的 `ip6_finish_output2()` 函数中执行。本机产生的准备发出的数据包或者转发的数据包，在经过路由判断之后， 将到达此 hook 触发点。

### NF_HOOK 宏和 netfilter 向量

所有的触发点位置统一调用 `NF_HOOK` 这个宏来触发 hook：

``` C
static inline int NF_HOOK(uint8_t pf, unsigned int hook, struct sk_buff *skb,
	struct net_device *in, struct net_device *out,
	int (*okfn)(struct sk_buff *))
{
	return NF_HOOK_THRESH(pf, hook, skb, in, out, okfn, INT_MIN);
}
```

`NF-HOOK` 接收的参数如下：

- pf: 数据包的协议族，对 IPv4 来说是 `NFPROTO_IPV4`。
- hook: 上图中所示的 netfilter hook 枚举对象，如 NF_INET_PRE_ROUTING 或NF_INET_LOCAL_OUT。
- skb: SKB 对象，表示正在被处理的数据包。
- in: 数据包的输入网络设备。
- out: 数据包的输出网络设备。
- okfn: 一个指向函数的指针，该函数将在该 hook 即将终止时调用，通常传入数据包处理路径上的下一个处理函数。

`NF-HOOK` 的返回值是以下具有特定含义的 netfilter 向量之一：

1. NF_ACCEPT: 在处理路径上正常继续（实际上是在 `NF-HOOK` 中最后执行传入的 `okfn`）。
2. NF_DROP: 丢弃数据包，终止处理。
3. NF_STOLEN: 数据包已转交，终止处理。
4. NF_QUEUE: 将数据包入队后供其他处理。
5. NF_REPEAT: 重新调用当前 hook。

回归到源码，IPv4 内核网络栈会在以下代码模块中调用 `NF_HOOK()`：

![NF_HOOK](2022-03-07-NF_HOOK.png)

实际调用方式以 **[`net/ipv4/ip_forward.c`](https://elixir.bootlin.com/linux/v2.6.39.4/source/net/ipv4/ip_forward.c#L115)** 对数据包进行转发的源码为例，在 `ip_forward` 函数结尾部分的第 115 行以 **NF_INET_FORWARD** hook 作为入参调用了 **`NF_HOOK`** 宏，并将网络栈接下来的处理函数 `ip_forward_finish` 作为 `okfn` 参数传入**：**

``` Bash
int ip_forward(struct sk_buff *skb)
{
.....(省略部分代码)
if (rt->rt_flags&RTCF_DOREDIRECT && !opt->srr && !skb_sec_path(skb))
		ip_rt_send_redirect(skb);

	skb->priority = rt_tos2priority(iph->tos);

	return NF_HOOK(NFPROTO_IPV4, NF_INET_FORWARD, skb, skb->dev,
		       rt->dst.dev, ip_forward_finish);
.....(省略部分代码)
}
```

## 回调函数与优先级

netfilter 的另一组成部分是 hook 的回调函数。内核网络栈既使用 hook 来代表特定触发位置，也使用 hook （的整数值）作为数据索引来访问触发点对应的回调函数。

内核的其他模块可以通过 netfilter 提供的 api 向指定的 hook 注册回调函数，同一 hook 可以注册多个回调函数，通过注册时指定的 **priority** 参数可指定回调函数在执行时的优先级。

注册 hook 的回调函数时，首先需要定义一个 `nf_hook_ops` 结构（或由多个该结构组成的数组），其定义如下：

``` C
struct nf_hook_ops {
	struct list_head list;

	/* User fills in from here down. */
	nf_hookfn *hook;
	struct module *owner;
	u_int8_t pf;
	unsigned int hooknum;
	/* Hooks are ordered in ascending priority. */
  int priority;
};
```

在定义中有 3 个重要成员：

- hook: 将要注册的回调函数，函数参数定义与 `NF_HOOK` 类似，可通过 `okfn` 参数嵌套其他函数。
- hooknum: 注册的目标 hook 枚举值。
- priority: 回调函数的优先级，**较小的值优先执行**。

定义结构体后可通过 `int nf_register_hook(struct nf_hook_ops *reg)` 或 `int nf_register_hooks(struct nf_hook_ops *reg, unsigned int n);` 分别注册一个或多个回调函数。同一 netfilter hook 下所有的 `nf_hook_ops` 注册后以 priority 为顺序组成一个链表结构，注册过程会根据 priority 从链表中找到合适的位置，然后执行链表插入操作。

在执行 `NF-HOOK` 宏触发指定的 hook 时，将调用 `nf_iterate` 函数迭代这个 hook 对应的 `nf_hook_ops` 链表，并依次调用每一个 `nf_hook_ops` 的注册函数成员 `hookfn`。示意图如下：

![netfilter-hookfn1](2022-03-07-netfilter-hookfn1.png)

这种链式调用回调函数的工作方式，也让 netfilter hook 被称为 Chain，下文的 iptables 介绍中尤其体现了这一关联。

每个回调函数也必须返回一个 netfilter 向量；如果该向量为 **NF_ACCEPT，**`nf_iterate` 将会继续调用下一个 `nf_hook_ops` 的回调函数，直到所有回调函数调用完毕后返回 **NF_ACCEPT**；如果该向量为 **NF_DROP**，将中断遍历并直接返回 **NF_DROP；**如果该向量为 **NF_REPEAT**，将重新执行该回调函数**。** `nf_iterate` 的返回值也将作为 `NF-HOOK` 的返回值，网络栈将根据该向量值判断是否继续执行处理函数。示意图如下：

![netfilter-hookfn2](2022-03-07-netfilter-hookfn2.png)

netfilter hook 的回调函数机制具有以下特性：

- 回调函数按优先级依次执行，只有上一回调函数返回 **NF_ACCEPT** 才会继续执行下一回调函数。
- 任一回调函数都可以中断该 hook 的回调函数执行链，同时要求整个网络栈中止对数据包的处理。

# 用户空间管理工具

> **Netfilter**是 Linux 内核的一部分，它真正负责在网络堆栈中检查、修改、转发或丢弃数据包。而用户空间管理工具则是告诉 `Netfilter` “该怎么做。常用的用户空间管理工具有`iptables`,  `nftables`, `firewalld`

## iptables

iptables则是netfilter的操作接口，iptables 在用户空间管理应用于数据包的自定义规则，netfilter执行规则所对应的策略对数据包进行处理。

图：iptables与netfilter的关系

![](iptables-6d948ee3.png)

iptables 分为两部分：

- 用户空间的 iptables 命令向用户提供访问内核 iptables 模块的管理界面。
- 内核空间的 iptables 模块在内存中维护规则表，实现表的创建及注册。

### iptables 的核心概念

它的三个核心组成部分：**表 (Tables)**、**链 (Chains)** 和 **规则 (Rules)**。

数据包进入系统后，会按照一个预定义的流程（“流程图”）穿过一系列的“关卡”。

- **表 (Tables)**：代表不同类型的功能。一个表包含多个“链”。
    
- **链 (Chains)**：代表数据包在特定“表”中所处的不同**处理阶段**。一个链包含多条“规则”。
    
- **规则 (Rules)**：是具体的“如果...那么...” (If...Then...) 条件。
    
    - **匹配 (Match)**：即“如果”部分，定义了数据包的特征（如源 IP、目的端口、协议等）。
        
    - **目标 (Target)**：即“那么”部分，定义了如何处理匹配到的数据包（如 `ACCEPT`, `DROP` 等）。
        

#### 表 (Tables)

`iptables` 默认有 4 个主要的表（按功能划分）：

1. **`filter` 表 (过滤表)**
    
    - **用途**：这是**最常用**的表，用于真正意义上的“防火墙”功能，即**过滤数据包（允许/拒绝）**。
        
    - **包含的链**：`INPUT`, `OUTPUT`, `FORWARD`。
        
2. **`nat` 表 (网络地址转换表)**
    
    - **用途**：用于修改数据包的源或目的 IP/端口，实现 NAT。
        
    - **包含的链**：`PREROUTING`, `POSTROUTING`, `OUTPUT`。
        
3. **`mangle` 表 (修改表)**
    
    - **用途**：用于修改数据包的特定字段，如 TOS（服务质量）、TTL（生存时间）或设置内核内部的“标记”(mark)。
        
    - **包含的链**：全部 5 个内建链 (`PREROUTING`, `INPUT`, `FORWARD`, `OUTPUT`, `POSTROUTING`)。
        
4. **`raw` 表**
    
    - **用途**：优先级最高。用于在系统进行连接跟踪 (Connection Tracking) 之前处理数据包。一个常见的用途是给某些数据包设置 `NOTRACK` 目标，让内核“跳过”对它们的跟踪，以提高性能。
        
    - **包含的链**：`PREROUTING`, `OUTPUT`。
        

#### 链 (Chains)

“链”是数据包流经的路径。有 5 个内建的“钩子”(hook) 链，它们在 `Netfilter` 框架中的触发时机不同：

1. **`PREROUTING` (路由前)**
    
    - 数据包刚进入网络接口，**尚未**进行路由决策（即还不知道这个包是给本机的，还是需要转发的）。
        
    - _主要用于 `nat` 表（做 DNAT/端口转发）和 `mangle` 表。_
        
2. **`INPUT` (输入)**
    
    - 数据包经过路由决策后，确定是**发往本机**的（目标 IP 是本机）。
        
    - _主要用于 `filter` 表（控制谁能访问本机服务）。_
        
3. **`FORWARD` (转发)**
    
    - 数据包经过路由决策后，确定是**需要转发**到另一台机器的（源 IP 和目标 IP 都不是本机）。
        
    - _主要用于 `filter` 表（控制本机作为路由器时，允许哪些流量通过）。_
        
4. **`OUTPUT` (输出)**
    
    - 由**本机进程产生**，准备发出去的数据包。
        
    - _主要用于 `filter` 表（控制本机主动发起的连接）和 `nat` 表（修改出站包的目的地址）。_
        
5. **`POSTROUTING` (路由后)**
    
    - 数据包即将离开网络接口时（已经过了路由决策）。
        
    - _主要用于 `nat` 表（做 SNAT/MASQUERADE，即共享上网）。_

我们扩充一下图1. 一个IP包经过 iptables 的处理流程如下

![](iptables-chain-4de09730.png)

实际上 iptables的规则就是挂在netfilter钩子上的函数，用来修改IP数据包的内容或者过滤数据包，iptables的表就是所有规则的逻辑集合。

一般情况下一条iptables的规则包含两个部分：`匹配条件`和`动作`。匹配条件比如协议类型、源ip、目的ip、源端口号等，匹配条件可以组合，匹配之后动作有如下几种：

- `DROP`：直接将数据包丢弃
- `REJECT` 给客户端返回 `connection refused` 或 `destination unreachable`报文。
- `QUEUE` 将数据包放入用户空间队列，供用户空间程序使用
- `RETURN` 跳出当前链，后续规则不再处理
- `ACCEPT` 允许数据包通过
- `JUMP` 跳转到用户自定义的其他链继续执行

理解iptables的链、表、规则的概念之后，我们来介绍一下iptables的命令用法。

### iptables 规则用法

**iptables 更新延迟的问题**

规则是链中的具体条目。`iptables` 会**按顺序**检查链中的每一条规则。

- 如果数据包**匹配**某条规则，就执行该规则的**目标 (Target)**。
    
- 一旦被某个目标（如 `ACCEPT`, `DROP`）“最终处理”了，就不再检查该链中的后续规则。
    

**常见的匹配 (Matches)：**

- `-p`：协议 (如 `tcp`, `udp`, `icmp`)
    
- `-s`：源地址 (Source IP 或网段)
    
- `-d`：目的地址 (Destination IP 或网段)
    
- `-i`：入站接口 (Input interface, 如 `eth0`)
    
- `-o`：出站接口 (Output interface, 如 `eth1`)
    
- `--dport`：目的端口 (如 `--dport 80`) (需配合 `-p tcp` 或 `-p udp`)
    
- `--sport`：源端口
    
- `-m state --state`：连接状态 (非常重要！)
    
    - `NEW`：新发起的连接。
        
    - `ESTABLISHED`：已经建立的连接（即“已握手”的后续包）。
        
    - `RELATED`：与已有连接相关的包（如 FTP 的数据连接）。
        

**常见的目标 (Targets)：**

- **`ACCEPT`**：**接受**。允许数据包通过。
    
- **`DROP`**：**丢弃**。悄悄地丢掉数据包，不给任何回应（推荐，安全性高，对方会超时）。
    
- **`REJECT`**：**拒绝**。丢掉数据包，但会回复一个错误信息（如 "port unreachable"）。
    
- **`LOG`**：**记录**。将数据包信息记录到系统日志 (dmesg 或 /var/log/messages)，然后**继续匹配下一条规则**。
    
- **`MASQUERADE`**：**地址伪装**。用于 `nat` 表的 `POSTROUTING` 链，自动将源 IP 改为出站网卡的 IP。主要用于动态 IP（如 ADSL）的共享上网。
    
- **`DNAT`**：**目的地址转换**。用于 `nat` 表的 `PREROUTING` 链，实现端口转发（如将外部 80 端口转到内部 8080）。

### 如何判断 iptables 进行了哪些流量过滤

判断 `iptables` 正在做什么，主要依赖于**查看当前的规则集**，并**理解这些规则的含义**。

核心命令是：`iptables -L`

但 `iptables -L` 本身信息有限，您需要配合以下参数才能获得有用的信息：

**最佳查看命令：`iptables -L -v -n --line-numbers`**

让我们分解这个命令：

- `-L`：(List) 列出规则。
    
- `-v`：(Verbose) 显示详细信息。**这是最关键的参数**，它会显示：
    
    - `pkts`：(Packets) 匹配到此规则的数据包**数量**。
        
    - `bytes`：匹配到此规则的数据包**总字节数**。
        
    - `in` 和 `out`：对应的输入/输出接口。
        
- `-n`：(Numeric) **以数字形式**显示 IP 地址和端口号。这可以避免 `iptables` 尝试反向 DNS 解析，速度更快，结果也更清晰。
    
- `--line-numbers`：显示规则的**行号**，这在您需要删除或插入特定规则时非常有用。

**分析步骤：**

#### 第 1 步：查看 `filter` 表（默认表）

直接运行：

``` Bash
sudo iptables -L -v -n --line-numbers
```

您会看到类似这样的输出：

```
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1       50  3420 ACCEPT     all  --  lo     * 0.0.0.0/0            0.0.0.0/0
2      850 69200 ACCEPT     all  --  * * 0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
3        1    60 ACCEPT     tcp  --  eth0   * 192.168.1.100        0.0.0.0/0            tcp dpt:22
4        0     0 DROP       all  --  eth0   * 1.2.3.4              0.0.0.0/0
5        5   300 REJECT     all  --  * * 0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 1050 110K bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

**如何解读：**

1. **看默认策略 (Policy)**：
    
    - `Chain INPUT (policy ACCEPT ...)`：`INPUT` 链的默认策略是 `ACCEPT`。这意味着，如果一个数据包没有匹配到 `INPUT` 链中的任何一条规则，它最终会被接受。这是一种“黑名单”策略（默认全通，只禁特定的）。
        
    - `Chain FORWARD (policy DROP ...)`：`FORWARD` 链的默认策略是 `DROP`。这意味着服务器默认**不转发**任何流量。
        
2. **看规则顺序 (num)**：
    
    - `iptables` 从上到下（`num` 从 1 开始）匹配。
        
3. **看 `pkts` (包计数器)**：
    
    - **这是判断过滤行为的核心！**
        
    - 规则 2 (`state RELATED,ESTABLISHED`) 的 `pkts` 是 850，说明大量已建立的连接流量被这条规则放行了（这是防火墙能正常工作的基础）。
        
    - 规则 3 (SSH, `dpt:22`) 的 `pkts` 是 1，说明有 1 个新的 SSH 连接请求被接受了。
        
    - 规则 4 (DROP `1.2.3.4`) 的 `pkts` 是 **0**。这说明**到目前为止，这条规则还没有阻止过任何来自 `1.2.3.4` 的流量**。
        
    - 如果规则 4 的 `pkts` 是一个很大的数字，那就说明 `iptables` 正在**积极地**阻止来自 `1.2.3.4` 的流量。
        
4. **总结 `INPUT` 链的行为**：
    
    - 允许本地回环（`lo`）流量 (规则 1)。
        
    - 允许所有已建立的连接 (规则 2)。
        
    - 允许来自 `192.168.1.100` 的新 SSH (22端口) 连接 (规则 3)。
        
    - 丢弃来自 `1.2.3.4` 的所有流量 (规则 4)。
        
    - 拒绝所有其他不匹配的流量 (规则 5)。
        
    - (如果规则 5 不存在，那么流量会走到默认策略 `ACCEPT`)。
        

#### 第 2 步：查看 `nat` 表

如果您的服务器在做路由或端口转发，`filter` 表不足以说明全部情况。您必须检查 `nat` 表：

Bash

```
sudo iptables -t nat -L -v -n --line-numbers
```

- `-t nat`：指定查看 `nat` 表。
    

您可能会看到：

```
Chain PREROUTING (policy ACCEPT 150 packets, 10K bytes)
num   pkts bytes target     prot opt in     out     source               destination
1       10   600 DNAT       tcp  --  eth0   * 0.0.0.0/0            1.2.3.4              tcp dpt:80 to:192.168.1.10:8080

Chain POSTROUTING (policy ACCEPT 30 packets, 2K bytes)
num   pkts bytes target     prot opt in     out     source               destination
1       80  5120 MASQUERADE all  --  * eth0    192.168.1.0/24       0.0.0.0/0
```

**解读 `nat` 表：**

- `PREROUTING` 链的规则 1：
    
    - `pkts` 是 10，说明有 10 个数据包被处理了。
        
    - **行为**：当 `eth0` 接口收到访问 `1.2.3.4`（公网IP）的 80 端口的 TCP 流量时...
        
    - **目标 (Target)**：`DNAT`（目的地址转换）...
        
    - **结果**：`to:192.168.1.10:8080`。流量被**转发**到了内网 `192.168.1.10` 的 8080 端口。
        
    - **结论**：这台服务器配置了**端口转发**。
        
- `POSTROUTING` 链的规则 1：
    
    - `pkts` 是 80，说明有 80 个数据包被处理了。
        
    - **行为**：当来自 `192.168.1.0/24` (内网) 的流量准备从 `eth0` (外网) 接口出去时...
        
    - **目标**：`MASQUERADE`（地址伪装）。
        
    - **结论**：这台服务器配置了**共享上网**，让内网 `192.168.1.0/24` 网段的机器可以通过它访问互联网。
        

#### 总结：如何判断过滤

1. **首选命令**：`sudo iptables -L -v -n`。
    
2. **检查 `filter` 表**：查看 `INPUT`, `OUTPUT`, `FORWARD` 链的默认 `policy` 和规则。
    
3. **检查 `nat` 表**：`sudo iptables -t nat -L -v -n`，查看 `PREROUTING` (端口转发) 和 `POSTROUTING` (共享上网)。
    
4. **核心看点**：
    
    - **默认策略 (Policy)**：是 `ACCEPT` (黑名单模式) 还是 `DROP` (白名单模式)？
        
    - **`pkts` 计数器**：非零的 `DROP` 或 `REJECT` 规则意味着**正在发生过滤**。非零的 `ACCEPT` 规则意味着**正在允许流量**。非零的 `DNAT` 或 `MASQUERADE` 规则意味着**正在发生 NAT**。
        

---

### 4. iptables 常用法（示例）

**注意：操作 `iptables` 具有风险，尤其是在远程 SSH 连接时。错误的 `INPUT` 规则（如 `iptables -P INPUT DROP`）可能会立即切断您的连接。**

**1. 查看规则 (已讲)**

```
# 查看 filter 表 (默认)
sudo iptables -L -v -n
# 查看 nat 表
sudo iptables -t nat -L -v -n
```

**2. 清空规则 (危险！)**

``` Bash
# 清空 filter 表的所有规则
sudo iptables -F
# 清空 nat 表的所有规则
sudo iptables -t nat -F
# 删除所有用户自定义链
sudo iptables -X
```

**3. 设置默认策略 (白名单模式，推荐)** _在执行此操作前，必须先允许 SSH 和已建立的连接！_

``` Bash
# 1. 允许已建立的连接 (必须！)
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# 2. 允许 SSH (必须！)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
# 3. 允许本地回环 (推荐)
sudo iptables -A INPUT -i lo -j ACCEPT
# 4. 最后，设置默认拒绝
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT # (通常允许出站)
```

**4. 添加 (Append) 规则**

``` Bash
# 允许来自 1.2.3.4 的 HTTP (80) 访问
sudo iptables -A INPUT -p tcp -s 1.2.3.4 --dport 80 -j ACCEPT
# 阻止 (DROP) 来自 5.6.7.8 的所有流量
sudo iptables -A INPUT -s 5.6.7.8 -j DROP
```

**5. 插入 (Insert) 规则** _`-A` (Append) 是追加到末尾，`-I` (Insert) 是插入到指定位置（默认第 1 行）。_

``` Bash
# 紧急：在 INPUT 链的第 1 条规则插入，阻止 5.6.7.8 (优先处理)
sudo iptables -I INPUT 1 -s 5.6.7.8 -j DROP
```

**6. 删除 (Delete) 规则**

``` Bash
# 方式一：按行号删除 (先用 --line-numbers 查看)
# 假设 "DROP 5.6.7.8" 是 INPUT 链的第 1 条
sudo iptables -D INPUT 1

# 方式二：按规则内容删除 (必须完全匹配)
sudo iptables -D INPUT -s 5.6.7.8 -j DROP
```

### 规则持久化

> 如果仅仅是通过`iptables`增加`netfilter`，那么当服务器重启时规则将会被重置。

#### 使用持久化工具

以`iptables-services`为例

Red Hat 系（包括 RHEL, CentOS 7, Rocky Linux, AlmaLinux）使用一个不同的包 `iptables-services`。

**注意**：在 RHEL/CentOS 7+ 系统上，默认的防火墙是 `firewalld`。如果您想使用 `iptables`，您**必须**先禁用 `firewalld`，否则两者会冲突。

**1. 禁用 `firewalld` (如果它在运行)：**

``` Bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
sudo systemctl mask firewalld
```

**2. 安装 `iptables-services`：**

``` Bash
# RHEL / CentOS
sudo yum install iptables-services

# Fedora
sudo dnf install iptables-services
```

**3. 启动并启用 `iptables` 服务：**

``` Bash
# 启动 iptables 服务
sudo systemctl start iptables
# 设置开机自启
sudo systemctl enable iptables

# (如果需要) IPv6
sudo systemctl start ip6tables
sudo systemctl enable ip6tables
```

**4. 如何保存规则？ (关键！)** 与 Debian 不同，您不应该直接编辑文件。而是使用 `service` 命令或 `iptables-save` 工具来保存**当前内存中**的规则：

``` Bash
# 推荐的方法
sudo service iptables save
# 或者 (效果相同)
sudo iptables-save > /etc/sysconfig/iptables

# (如果需要) IPv6
sudo service ip6tables save
# 或者
sudo ip6tables-save > /etc/sysconfig/ip6tables
```

**5. 文件保存位置：**

- IPv4 规则保存在: `/etc/sysconfig/iptables`
    
- IPv6 规则保存在: `/etc/sysconfig/ip6tables`
    

**工作原理：** `iptables.service` 在启动时会读取 `/etc/sysconfig/iptables` 文件来加载规则。

#### 手动 "DIY" 方法 (适用于所有系统)

**1. 手动保存规则：** 首先，将您当前满意的规则集保存到一个安全的文件中。

``` Bash
sudo iptables-save > /etc/iptables.rules
```

**2. 创建一个 `systemd` 服务来恢复规则：** 这是现代 Linux 系统中最干净的手动方式。

- 创建一个新的服务文件：
    
    ``` Bash
    sudo nano /etc/systemd/system/iptables-restore.service
    ```
    
- 将以下内容粘贴到文件中：
    
    ``` TOML
    [Unit]
    Description=Restore iptables rules
    # 必须在网络启动之前加载规则
    Before=network-pre.target
    Wants=network-pre.target
    
    [Service]
    Type=oneshot
    # 确保在失败时不会继续
    RemainAfterExit=yes
    # 使用绝对路径
    ExecStart=/usr/sbin/iptables-restore /etc/iptables.rules
    
    [Install]
    WantedBy=multi-user.target
    ```
    
- **启用并启动服务：**
  
    ``` Bash
    # 重新加载 systemd 配置
    sudo systemctl daemon-reload
    # 设为开机启动
    sudo systemctl enable iptables-restore.service
    # 立即测试一次
    sudo systemctl start iptables-restore.service
    ```
---

##### ⚠️ **重要警告：防火墙操作的“安全网”**

在您修改防火墙规则并使其**持久化**之前，请务必小心。一个常见的错误是：

1. 您设置了一条规则 `iptables -P INPUT DROP`（默认拒绝所有输入）。
    
2. 您**忘记**添加 `iptables -A INPUT -p tcp --dport 22 -j ACCEPT` 来允许 SSH。
    
3. 您保存了规则并重启了服务器。
    
4. **结果：** 服务器启动后，`INPUT` 链默认为 `DROP`，SSH 规则不存在，您将**永远无法**通过 SSH 再次登录您的服务器（需要通过VNC或物理控制台救援）。
    

**最佳实践：**

1. **先允许 SSH：** 始终确保允许您访问的规则（如 SSH）在规则集的顶部。
    
2. **测试规则文件：** 在重启 _之前_，测试您的规则文件是否可以被正确加载。
    
    ``` Bash
    # 1. 保存规则
    sudo iptables-save > /tmp/test.rules
    # 2. 清空所有规则 (模拟重启)
    sudo iptables -F
    sudo iptables -P INPUT ACCEPT
    sudo iptables -P FORWARD ACCEPT
    sudo iptables -P OUTPUT ACCEPT
    # 3. 尝试恢复
    sudo iptables-restore < /tmp/test.rules
    # 4. 检查是否仍然可以访问
    ```
    
1. **使用 `at` 命令：** 在执行危险操作时，设置一个“自动反悔”的 `at` 命令。
    
    ``` Bash
    # 在5分钟后执行 "iptables -P INPUT ACCEPT"，以防您把自己锁在外面
    sudo at now + 5 minutes
    # (在提示符处输入)
    iptables -P INPUT ACCEPT
    # (按 Ctrl+D 结束)
    
    # ...现在开始执行您的危险操作...
    
    # 如果操作成功，您可以取消 at 任务
    sudo atq # 查看任务编号
    sudo atrm <任务编号>
    ```