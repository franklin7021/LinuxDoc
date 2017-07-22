# OLSRd 协议拓展

该说明基于 0.6.0 版本

> - 1.Credits
> - 2.Link Quality algorithms
> - 3.Fisheye
> - 4.NIIT(ipv4 over ipv6 traffic)
> - 5.Smart gateways
> - 6.NatThreshold

注：NIIT 和 Smart gateways 只支持 Linux 系统

## 1 Credits

ETX(Expected Transmission Count)由 Berlin Freifunk Network 设计，Douglas S. J. De Couto 开发，具体的代码和消息格式由 Thomas Lopatic 完成。详见 [wiki ETX](http://en.wikipedia.org/wiki/Expected_Transmission_Count)

Fisheye 由 Thomas Lopatic 于 2005 年实现

LQ-Plugin 由 Henning Rogge 于 2008 年重写

NIIT 内核模块由 lynxis 于 2009 年完成

非对称的 gateway tunnel 的概念来源于 B.A.T.M.A.N ，olsrd 中的功能由 Markus Kittenberger 和 Henning Rogge 完成

## 2 链路质量算法

### 2.1 概念

OLSRd 自 0.5.6 版本以来使用整数表示链路的开销。这个整数也可以被叫做链路质量(以 LQ 为简称)。OLSRd 有多个 LQ 插件全部使用 ETX-metric 的计算方式，其他的路由度量的方式也是可行的。

       A                  B
     +---+              +---+
     |   |  <--- LQ --- |   |
     |   |  ---- NLQ -->|   |
     +---+              +---+

每个链路由一对值 LQ/NLQ 表示，LQ 代指朝向节点的质量，NLQ 代指朝向邻居的质量。LQ 和 NLQ 的取值在 0 到 1 之间。

ETX = 1.0/(LQ * NLQ)

ETX 的值可以看做链路发送一个数据包需要重传的次数。ETX=1 表示没有丢包。

LQ 和 NLQ 是节点的属性：LQ 为 B 向 A 的包到达率。因此，在 ETX 的实现中 B 需要告知 A 它的 LQ。

OLSRd 在选择路径时会选择代价总和最小的路径。

best_path(A,B) = minimum_sum({set of all paths between A and B})

### 2.2 配置

激活链路质量系统需要在配置文件中将变量 LinkQualityLevel 设置为 2。可以更改 LinkQualityAlgorithm 参数选择使用哪一个链路质量算法。某些嵌入式的 OLSRd 只编译了一种算法，所以这些 OLSRd 不能使用该变量。

在 0.6.0 版本中有四种链路质量算法，其中两种为 Funkfeuer/Freifunk ETX 实现，另外两种为传统(Legacy)实现。

### 2.3 LinkQuality-Algorithm "etx_ff"

"Etx_ff" (ETX Funkfeuer/Freifunk) 是 OLSRd 中默认的 LQ 算法，利用接收到的 OLSR 包中的序列号计算链路的 loss rate。Etx_ff 包含了滞后机制，抑制了 LQ 的小的抖动。如果收不到邻居节点的数据包，计时器会逐渐减小 LQ 的数值，知道再次接收到邻居的数据包或者链路失效。

etx_ff 的消息格式兼容 etx_fpm 和 etx_float

### 2.4 LinkQuality-Algorithm "etx_ffeth"

etx_ffeth 与 etx_ff 不兼容，不同的节点分别使用两种不同的算法将不能一起工作。etx_ff，etx_float
和 etx_fpm 的问题在于他们在计算以太网链路时开销最小就为 1。这意味着如有两条以太网链路(假设每条 ETX=1.0)以及一条无线链路(假设 ETX=1.5)时，OLSRd 会优先选择无线链路，即便以太链路要好于无线链路。

Etx_ffeth 解决了该问题，为没有丢包的以太链路引入了特殊值 ETX=0.1。由于特殊的取值导致 etx_ffeth
与 etx_ff，etx_fpm，etx_float 不兼容。后三者在探测到 etx_ffeth 节点后，会将其 LQ 取 0。

etx_ffeth 算法只用到了整数计算，所以可以在嵌入式系统。

为了充分利用 etxff_eth 的优势，所有的以太网接口必须在接口配置中标记为 "mode ether" (详见 olsrd.conf.default.full)。

当前 etc_ffeth 是建立新型 mesh 网络(涉及到有线)较优的选择。

### 2.5 Legacy LinkQuality-Algorithm "etx_float"

etx_float 根据 HELLO 计算 ETX 使用了指数衰退(指数因子可配置)。etx_float 不能使用 TC 消息计算链路质量，相较于 etx_ff 实现简单但是效果不佳。etx_float 使用浮点型数学运算，在嵌入式系统中会占用更多的 cpu 资源。etx_float 的消息格式与 etx_fpm 和 etx_ff 兼容。

### 2.6 Legacy LinkQuality-Algorithm "etx_fpm"

etx_fpm 是 etx_float 的定点数的实现。etx_fpm 与 etx_float 的计算过程一致，但是速度更快。etx_fpm 与 etx_fpm 的消息格式与 etx_float 和 etx_ff 兼容。

### 2.7 建立自己的链路质量算法

OLSRd 可以轻松拓展不同的链路质量度量。参照 src/lq_plugin*.[ch] 文件创建自己的 metric。

## 3 Fisheye

通常情况下 OLSR 会将所有的 TC 消息转发给全部节点，在节点较多时这会带来很大的网络开销。减少 TC 消息的发送频率会拖延路由表的变化。

Fisheye(又叫做朦胧视觉链路状态路由 Hazy Sighted Link State Routing)的机制在上述两个问题之间做了折中，每隔 7 个 TC，下一个 TC 消息会被广播到所有节点。大部分的 TC 的 TTL 值被赋予特殊值。当前 Fisheye 机制中 TTL 序列为 2, 8, 2, 16, 2, 8, 2, 255(最大的 TTL 值)。

FIsheye 的问题在于人工引入了 TC 的洪泛边界，在理论上会引起路由不一致性以及洪泛边界的路由回环。在实践中，Fisheye 是大型网络必要的属性，效果也足够好。

## 4 NIIT(ipv4 over ipv6 traffic)

towrite

## 5 Smart gateways

towrite

## 6 NatThreshold

towrite