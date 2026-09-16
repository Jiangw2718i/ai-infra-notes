# 两台 DGX Spark 的 200GbE 直连只有 13 Gb/s——直到带着线重启

**结论先说** —— 两台 Spark 开着机的时候把 QSFP 线插上，链路会正常建立，能 ping 通，也能跑流量，速度是 13 Gb/s,而不是应有的 98。运行期间你能改的东西没有一样管用：改 MTU 不行，多队列不行，强制降速不行，把网卡从 PCI 总线上摘掉再重新枚举也不行。**带着线把两台都重启一次**,同一根线、同样的驱动和固件，每条通道 98.03 Gb/s。

硬件：两台 NVIDIA DGX Spark(GB10),BIOS 5.36_0ACUM027。内核 6.17.0-1032-nvidia,驱动 580.173.02,ConnectX-7(MT2910)固件 28.45.4028。一根无源 QSFP DAC,两台都插 0 号口。2026 年 9 月。

这篇是调查原文，包含走过的弯路。英文版是从中提取的精简笔记：[The 200GbE QSFP link between two DGX Sparks runs at 13 Gb/s until you reboot with the cable attached](qsfp-link-13gbps-until-reboot.md)

---

## 起因

两台 Spark 放在一起很久了，一直没有直连。线到货那天晚上 21:53,直接在开机状态下插了上去。

插上就有反应，热插拔机制是设计好的：没插线时 DGX Spark 会把 ConnectX-7 从 PCI 总线上摘掉省电，插上线由 udev 规则调用 `/opt/nvidia/dgx-spark-mlnx-hotplug/mtk-hotplug-handler.sh` 把它加回来。日志很干净：

```
cx7-pcie-hotplug MTKP0001:00: Cable plugin
mlx5_core 0000:01:00.0: Port module event: module 0, Cable plugged
mlx5_core 0000:01:00.0 enp1s0f0np0: Link up
```

两台互相 ping 通，延迟 1 毫秒出头。看起来一切正常。

## 第一次测速，13 Gb/s

配好地址(两条通道分属 192.168.100.0/24 和 192.168.101.0/24)、把 MTU 调到 9000、巨型帧用 `ping -M do -s 8972` 验证过之后，开始测。

RDMA 写带宽，单队列：

```
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 1048576    5000             13.22              13.22  		   0.001576
```

然后是一串排列组合，全都一样:

| 变化 | 结果 |
|---|---|
| 4 个队列对 | 12.78 |
| 另一条通道 | 12.72 |
| TCP(`iperf -P 8 -w 4M`) | 13.1 |
| MTU 1500 | 13.2 |
| 两条通道同时加压 | 13.33 + 13.23 |

NVIDIA 自己的基准指南里，同样的拓扑是 92.57 + 97.28 = 189.85 Gb/s。我们是它的七分之一。

## 我的第一个判断：线不对（错了）

用 `ethtool -m` 把线缆芯片里的信息读出来:

```
Identifier                                : 0x11 (QSFP28)
Transceiver type                          : 40G Ethernet: 40G Base-CR4
Encoding                                  : 0x08 (PAM4)
BR, Nominal                               : 25500Mbps
Length (Copper or Active cable)           : 1m
Transmitter technology                    : 0xa0 (Copper cable unequalized)
Vendor name                               : Amphenol
Vendor PN                                 : NJAAKK-N911
```

QSFP28、40G-CR4、每通道 25.5 Gbps。200G 需要每通道 50 Gbps(甚至 100G)。看起来实锤了。

再看链路状态，我更确信:

```
active_width:		2X (16)
active_speed:		100.0 Gbps (128)
```

我当时读成「四条通道只跑起来两条，而且只有 100G」,加上物理层错误计数不小(`rx_err_lane_1_phy` 两百多万),结论就写成了「线的等级不够，信号质量差，要换线」。

**这个判断是错的，错在两个地方。**

## 两个反证

**第一个反证：强制降速，数字一动不动。**

如果瓶颈在线缆的信号质量，那么降低速率应该让链路更稳、吞吐更高——至少应该**变化**。两端都执行:

```bash
sudo ethtool -s enp1s0f0np0 autoneg off speed 40000 duplex full
```

结果 13.23 Gb/s。和「名义 200G」时一模一样，小数点后都一样。

**一个随速率变化的物理层问题，不会给出一个与速率无关的固定上限。** 瓶颈不在 PHY。

**第二个反证：给网卡单独断电重来，也没用。**

按平台自己的热插拔脚本的做法，把四个 ConnectX-7 的 PCI 功能全部摘掉，再让总线重新扫描:

```bash
for d in 0000:01:00.1 0000:01:00.0 0002:01:00.1 0002:01:00.0; do
  echo 1 > /sys/bus/pci/devices/$d/remove
done
echo 1 > /sys/bus/pci/rescan
```

网卡干净地回来了，IP 和 MTU 由 NetworkManager 自动恢复，推理服务全程没受影响。驱动重新 probe、链路重新训练、配置重新下发——整条路径都走了一遍。

13.24 Gb/s。

## 更正：我读错了两个正常现象

查了资料之后，前面那两条「证据」都塌了。

**`active_width 2X / 100 Gbps` 是正常的。** Spark 的一个物理 QSFP 口对应两个逻辑网口，分别挂在两个控制器上，每个控制器只有 PCIe Gen5 x4,上限约 100 Gb/s。所谓 200G 是两条加起来。所以 `ethtool` 里「支持的速率最高 100000baseCR4」也不是缺陷，是这台机器本来的样子。

**`Detected insufficient power on the PCIe slot (27W)` 是良性提示。** 这条 `mlx5_pcie_event` 每次开机都会刷四条。NVIDIA 官方人员在论坛里说过它 "is considered benign",并且在该提示持续出现的情况下实测到 111 Gb/s。我们后来修好之后，这条提示照样每次开机出现四次，带宽照样满速。

顺便，PCIe 本身也没问题:`LnkSta: Speed 32GT/s, Width x4`,和 `LnkCap` 一致。

## 真正的原因：网卡必须在「线已经插着」的状态下开机

社区里有人遇到过一模一样的数字。NVIDIA 开发者论坛上至少三个帖子(370035、363461、373538)都是 13–16 Gb/s,其中 363461 修好后是每个接口 98.01 Gb/s——和我们最后测到的几乎一样。修复办法一致:**带着线冷启动**。

回头查日志，时间线解释了一切。cc07 从 9 月 5 日到 9 月 16 日一直没重启，它的日志是完整的:

```
2026-09-15T21:53  两台热插线，链路 up            → 13 Gb/s
2026-09-15T22:02  9f38 内核 Oops(见最后一节)
2026-09-15T22:15  cx7-pcie-hotplug: Cable removal   ← 线被拔掉
2026-09-15T22:17  9f38 重启,四个口全报 Cable unplugged ← 这次重启时线是拔着的
2026-09-16T06:21  cx7-pcie-hotplug: Cable plugin     ← 再次热插
                  ...此后我测到的全部是 13 Gb/s
2026-09-16T09:54  9f38 重启(线插着)
2026-09-16T09:55  cc07 重启(线插着)
```

中间那次 22:17 的重启特别有迷惑性：机器确实重启过，线也确实插过，但**两件事没有重叠**——重启的那一刻线是拔着的。如果不查日志，很容易以为「重启过了，没用」,从而排除掉正确答案。

重启之后:

```
rail 0: 98.03 Gb/s
rail 1: 98.03 Gb/s
```

合计 196 Gb/s,对比 NVIDIA 文档里的 189.85。同一根线，同样的驱动和固件，什么都没换。

**机制我说不清楚。** 我只能确认：热插拔这条路径给出的是一个能用但降级的链路，而带线开机给出的是满速链路。网卡固件在这两种情况下到底做了什么不同的事，从操作系统这一侧看不见。

## 三个坑

**一、`sudo systemctl reboot` 可能什么都不做。**

我先后两次对两台执行它，两台都没重启——`uptime -s` 没变，日志里也没有关机记录。换成:

```bash
sudo systemctl reboot --no-block
```

十秒内就下线了。为什么阻塞形式不生效，我没查出来。当时两台都有 8 个排队中的 systemd 任务(`plymouth-quit-wait.service` 在 running、`graphical.target` 在 waiting),可能有关，也可能无关。

**二、RoCE 的 GID 索引取决于网口有没有 IPv4 地址。**

配 IP 之前，`show_gids` 每个口只有两条记录，RoCE v2 是索引 1;配上 IPv4 之后多出索引 2 和 3,其中 IPv4 的 RoCE v2 是索引 3:

```
rocep1s0f0	1	1	fe80:0000:0000:0000:4ebb:47ff:fee8:9f39			v2	enp1s0f0np0
rocep1s0f0	1	3	0000:0000:0000:0000:0000:ffff:c0a8:640a	192.168.100.10  	v2	enp1s0f0np0
```

`-x` 给错会用最不友好的方式失败：把整个配置表头打印出来，**不打印结果行，退出码 0**。最早还报过一句 `Failed to modify QP 306 to RTR / Unable to Connect the HCA's through the link`,后来干脆什么都不说。我在这上面浪费了好几轮。

**三、只测一条通道说明不了问题。** 每条通道设计上就只有约 100 Gb/s,官方基准是两条同时压、把数字相加。只测一条，你既看不出坏(13 对 98 还是能看出的),也证明不了好。

## 顺带发现的崩溃：mstflint 把内核搞崩了

查日志时发现，第一次插线之后九分钟,9f38 内核 Oops:

```
Internal error: Oops: 0000000096000004 [#1]  SMP
CPU: 7 UID: 0 PID: 3648517 Comm: mstflint Tainted: G           O        6.17.0-1032-nvidia #32-Ubuntu
Hardware name: NVIDIA NVIDIA_DGX_Spark/P4242, BIOS 5.36_0ACUM027 06/12/2026
pc : pci_bus_read_config_dword+0x6c/0x118
Call trace:
 _vendor_specific_sem+0xc0/0x178 [mstflint_access]
 get_space_support_status+0x90/0x2d0 [mstflint_access]
 mst_ioctl+0x97c/0x1cd8 [mstflint_access]
```

70 秒后开始 `rcu: INFO: rcu_preempt detected stalls on CPUs/tasks`,机器 14 分钟后被重启。空指针地址是 `0x00000000000000c0`,发生在读 PCI 配置空间的时候。

`mstflint` 是 Ubuntu 的 `4.26.0+1-2ubuntu3`,配套的树外模块 `mstflint_access` 由 `nvidia-mstflint-loader` 在开机时加载，两台上都还装着。

**在一条会被守护进程动态添加和移除设备的 PCI 总线上，用工具去读那个设备的配置空间**——这个组合看起来就是它丢了设备。但我没有刻意复现过，所以这只是推测，不能当成结论。要在刚热插过的 ConnectX-7 上跑 mstflint 的话，知道有这么回事。

## 没有解决的

- 热插拔为什么给出降级链路，带线开机为什么不会。从 OS 侧看不见。
- 这根线的芯片信息(QSFP28 / 40G-CR4 / 1m)和它的标称型号(NJAAKK-N911,按公开资料是 40 cm 的 QSFP112 400G DAC)对不上。是芯片信息写错了，还是贴牌，无法确认。**但它冷启动后能跑满每通道 98 Gb/s,所以这件事最终不影响结果。**
- `systemctl reboot` 阻塞形式为什么不生效。
