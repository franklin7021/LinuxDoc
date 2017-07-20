# olsr-nl80211

## 概要

本文档描述了利用 linux nl80211 收集无线链路信息实现 OLSRd 的链路质量拓展。

## 设计

OLSRd 每一秒从 nl80211 中获取最新的数据，数据中包含了邻居节点的MAC地址。OLSRd 调用 arp 缓存将 IP 之地与 MAC 地址关联。

## 实现

本拓展需要附加外部依赖 libnl 进行编译。libnl可以简化进程与内核之间通信。

src/linux/nl80211_link_info.* 这些文件负责无线链路状态信息的收集。修改后的 link ffeth quality 插件利用收集来的信息计算链路质量。链路质量插件使用 #ifdef LINUX_NL80211 合并原有的 ffeth 插件。合并原有插件的好处是减少了重复代码。

## 开销计算

在旧的开销计算的基础上根据信号强度和链路带宽加入了 penalty。两个 penalty 的最大值为 1.0。

Costs = EXT + BandwidthPenalty + SignalPenalty

BandwidthPenalty = 1 - ( ActualBandwidth / ReferenceBandwidth)

SignalPenalty = LookupSignalPenaltyTable(SignalStrenghtOfNeighbor)

在发送 LQ_HELLO 消息时，本地节点会在消息中加入 2 bytes 的 penalty 信息。但是，当前的开销计算中只用到了从 nl80211 获取的信息。

## 说明

主要面向 IPv4 设计，也可用于 IPv6。未来的工作将集中于在 IPv6 测试该拓展。

netlink 的代码是阻塞的，这不会引发大的问题，但是理应是非阻塞的。

当前版本不使用从邻居节点收到的 NL80211 的信息。使用与否还在讨论中。

目前 penalty 的最大值为 1.0，可能会不充分。可能存在数乘效应。

ReferenceBandwidth 目前为常量 54 MBit。信号强度 penalty 的列表也是固定的。理想的情况下是通过配置文件进行配置。

可以添加配置选项禁用 NL80211，那么就会使用 ff_eth 版本的链路质量。