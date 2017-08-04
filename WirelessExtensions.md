# Wireless Extensions

译者 zhangmc 邮箱 zhangmc1993@qq.com

翻译自 [Wireless Extensions for Linux](https://hewlettpackard.github.io/wireless-tools/Linux.Wireless.Extensions.html)

该文章做于 __97__ 年 __1__ 月，是 [Linux Wireless LAN Howto](https://hewlettpackard.github.io/wireless-tools/Linux.Wireless.html) 的一部分

文章较为古老，使用的内核也较为陈旧，在现代 Linux 发行版中有些功能不再支持，请谨慎参考

## 1 简介

最初的开发需求是定义无线的API，允许用户通过标准且统一的方式操作无线设备。当然，所有设备是完全不同的，所以标准化工作只是针对设备的操作方式，不针对设备的具体数值。需要定义灵活、可拓展的无线接口。最为迫切的需求是对设备的配置(configuration)，其次是无线的数据统计(wireless statistics)和无线感知应用(wireless aware applications)的开发；接口还需要遵从 Linux 标准，简单易实现并且易于维护和分享；随着设备的变更，接口要易于升级。

Wireless Extensions 适用于 Wlan 设备；AX25(Amateur Radio)设备有其特定的网络工具；Wireless Extensions 不适用于 Cellular 以及长距离的无线通信，因为他们的运行方式与 wlan 相去甚远；红外(InfraRed)通信与 wlan 的工作方式相似，可以利用相关接口。

Wireless Extensions 是 Linux 网络接口(networking interface)的拓展。

## 2 可行性

Wireless Extensions 由三个部分实现。第一部分是用户接口，既操作拓展的工具；第二部分是拓展(Extensions)的定义和 Linux 内核的支持；第三部分是硬件接口，既网络设备驱动实现拓展(Extensions)到硬件操作的映射。Linux 内核已从 2.1 版本开始支持 Wireless Extensions。当前系统默认不开启 Wireless Extensions，需要激活内核中的 *CONFIG_NET_RADIO* 选项，该选项同时激活可用的无线设备列表。

> 译者注：当前内核中默认开启 Wireless Extensions

设备驱动的修改是最为重要的。驱动需要支持无线拓展，并通过相应的会话操作特定硬件。Wavelan ISA 和 Wavelan PCMCIA 的驱动支持无线拓展，在内核中可以获得 Wavelan ISA 的驱动。

## 3 用户接口和工具

用户接口由 3 个程序和 1 个 /proc 条目组成。

### 3.1 /proc/net/wireless

/proc 是伪文件，提供当前系统的信息以及数据统计，可以使用 cat 命令查看。/proc/net/wireless 给出了每个无线接口的数据统计。该文件实际上是标准驱动数据统计文件 /proc/net/dev 的翻版。

每个设备都会提供如下信息：

- Statrs：当前状态，设备的信息
- Quality-link：总体的接收质量
- Quality-level：接收到的信号强度
- Quality-noise：无接收时的信号强度
- Discarded-nwid：因为网络 ID 无效而丢弃的包的数量
- Discarded-crypt：不能译码的包的数量
- Discarded-misc：该信息暂未使用

这些信息可以反馈系统当前状态。Discarded-nwid 数值反映 nwid 配置问题或者临近网络；Quality-level 数值反映阴影区域(shadow areas)。

Quality-link 反映了接收情况，比如数据正确的接收比例。Quality-level 反映了接收信号的强度。

### 3.3 iwconfig

iwconfig 是配置驱动以及硬件的工具，参照了标准设备配置 ifconfig。可以设置如下参数：

- freq or channel：频率和信道
- nwid：网络 ID 或是网络域，用于区分不同的逻辑网络
- name of the protocol used on the air：空口使用的协议，比如802.11或 HiperLan
- sens：接收信号的门限值
- enc：加密或加扰

不加任何参数时，iwconfig 输出 /proc/net/wireless 中的数据。加入命令行选项可以修改对应参数。比如修改频点到2.462GHz，禁用 eth2 的 nwid 检查：

> iwconfig eth2 freq 2.462G nwid off

用户还可以查看设备 eth2 可用的频率：

> iwconfig eth2 list_freq

### 3.4 iwspy

> 译者注：iwspy 使用 ioctl() 函数以及特定操作码获取内核信息，在新的内核中不再支持该操作码，意味着 iwspy 在较新的系统中不可用。

iwspy 允许用户在驱动中添加需要监视的 IP，驱动在接收到对应 IP 发来的数据包后，会对相应的信道进行质量评估。iwpsy 可以将这些 IP 对应的信道信息展示出来。

iwpsy 接受的参数为 IP 地址或者 MAC 地址。IP 地址在传给驱动之前会转变为 MAC 地址。使用命令行添加硬件地址时，需要加入 *hw* 关键字，如：

> iwspy eth2 15.144.104.4 hw 08:00:0E:21:3A:1F

使用 “*+*” 添加新的监视地址：

> iwspy eth2 + hw 08:00:0E:2A:26:FA

显示监视信息：

> iwspy eth2

如果显示信息为全0，可能是因为设备之间没有数据传输，也就没有计算相应数值，可以使用 __ping__ 命令模拟数据传输。监视地址的数量上限为8。

### 3.4 iwpriv

iwpriv 用于配置额外的参数。

## 4 驱动接口与编程

Wireless Extensions 尽可能少的改动内核，获得简单易拓展的解决方案。所有的无线接口(类型与常数)定义在 /usr/include/linux/wireless.h 文件中。

### 4.1 /proc/net/wireless

该 /proc 条目是 /proc/net/dev 的翻版。Linux 网络栈使用结构体 device 跟踪系统中的每一个设备。device 的第一部分为标准接口，包括了一些参数，如 I/O 地址、网络地址，以及回调函数接口。现已在该结构体中加入了新的回调函数 *get_wireless_stats* 用于数据统计。当输出 /proc 的内容时，实际上是调用回调函数，返回并打印信息。当调用 *get_wireless_stats* 时，返回结构体 *iw_statistics* 。

### 4.2 ioctl

Unix 系统中设置及获取网络设备参数的通用方法是调用 ioctl 函数。ioctl 的操作对象是文件描述符，也可以是 socket。 ioctl 是非场重要的系统调用。

使用 ifconfig 命令更改 eth2 的 IP 地址，实际上是调用 ioctl (参数为 eth2, SIOCSIFADDR, new address)。结构体等定义参见 /usr/include/linux/if.h，操作码 SIOCSIFADDR 定义在 /usr/include/linux/sockios.h 中。Wireless Extensions 定义了一系列新的操作码(比如更改频点的 SIOCSIWFREQ)，也定义了这些操作的相关参数。

当然还可以使用初始化的方式设定参数，但是这种方式只能在模块初始化阶段完成，不够灵活。ioctl 的另一个优点是它不仅仅是用户接口更是编程接口，任何程序都可以调用。

## 5 当前的维护情况

翻译自 [wireless.wiki.kernel.org](https://wireless.wiki.kernel.org/en/developers/documentation/wireless-extensions)

Wireless Extensions 不再开发新的功能，仅接受 bug 修改。

### 5.1 为什么放弃使用 Wireless Extensions？

Wireless Extensions 基于 ioctl。ioctl 是用户空间到内核空间标准的传输工具。ioctl 的函数原型如下：

> int ioctl (int fd, unsigned long cmd, ...);

该原型支持多参数的系统调用。但是在真实的系统中，系统调用不支持可变参数，系统调用必须有明确规范的函数原型。用户程序通过硬件门(hardware gates)可以直接获得系统调用。"..." 代表的不是可变参数，而是 __一个__ 可选参数，一般情况下定义为 "char *argp"。

ioctl 较差的结构化特性使得它在内核开发者间不再流行。ioctl 的调用不会留下系统记录，也无法使用简单的方法检查这些调用。结构化较差的特点使得 ioctl 不适用于所有系统；比如一个64位系统，用户空间的进程使用32位模式。

### 5.2 Wireless Extensions 的替代者

[cfg80211](https://wireless.wiki.kernel.org/en/developers/documentation/cfg80211)

[nl80211](https://wireless.wiki.kernel.org/en/developers/documentation/nl80211)