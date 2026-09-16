# The 200GbE QSFP link between two DGX Sparks runs at 13 Gb/s until you reboot with the cable attached

**TL;DR** — Plug the QSFP cable in while both Sparks are running and the link trains, pings, and carries traffic — at 13 Gb/s instead of ~98. Nothing you can change at runtime moves that number: not MTU, not queue pairs, not the negotiated speed, not even removing and re-adding the NIC on the PCI bus. Reboot both machines **with the cable already attached** and the same cable, the same everything, gives 98.03 Gb/s per rail.

Hardware: 2× NVIDIA DGX Spark (GB10), BIOS 5.36_0ACUM027. Kernel 6.17.0-1032-nvidia, NVIDIA driver 580.173.02, ConnectX-7 (MT2910) firmware 28.45.4028. One passive QSFP DAC, port 0 on both machines. perftest `ib_write_bw`, September 2026.

## Symptom

Both rails loaded at the same time, which is how NVIDIA's own benchmark guide measures it:

```
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 1048576    9534             0.00               13.33  		   0.001589
 1048576    9465             0.00               13.23  		   0.001578
```

26.6 Gb/s aggregate. NVIDIA's `connect-two-sparks` performance guide shows 92.57 + 97.28 = 189.85 Gb/s for this cable topology.

The link itself looks healthy. `ibv_devinfo -d rocep1s0f0 -v`:

```
state:			PORT_ACTIVE (4)
phys_state:		LINK_UP (5)
active_width:		2X (16)
active_speed:		100.0 Gbps (128)
```

None of this changes the 13 Gb/s:

| Change | Result |
|---|---|
| MTU 1500 → 9000 (jumbo frames verified with `ping -M do -s 8972`) | 13.2 |
| `ib_write_bw -q 4` instead of 1 queue pair | 12.8 |
| The other rail (`roceP2p1s0f0`) | 12.7 |
| TCP instead of RDMA (`iperf -P 8 -w 4M`) | 13.1 |
| `ethtool -s enp1s0f0np0 autoneg off speed 40000` on both ends | 13.23 |
| Remove all four ConnectX-7 functions from the PCI bus and `echo 1 > /sys/bus/pci/rescan` | 13.24 |

The last two are the useful ones. A cable or signal-integrity fault scales with the link rate — forcing the link down to 40G and getting the *same* 13.23 Gb/s says the ceiling is not in the PHY. And a PCI-level remove/rescan re-runs driver probe, link training and NetworkManager configuration end to end, yet changes nothing.

## Cause

The NIC has to be initialised at boot with the cable already present.

DGX Spark powers the ConnectX-7 down when no cable is attached — this is deliberate, a udev rule (`/usr/lib/udev/rules.d/90-mtk-hotplug.rules`) driving `/opt/nvidia/dgx-spark-mlnx-hotplug/mtk-hotplug-handler.sh`, which removes the devices from the PCI bus. Insert a cable and the handler brings them back:

```
cx7-pcie-hotplug MTKP0001:00: Cable plugin
mlx5_core 0000:01:00.0: Port module event: module 0, Cable plugged
mlx5_core 0000:01:00.0 enp1s0f0np0: Link up
```

That path produces a working link. It does not produce a full-speed one. The machine has to go through a boot with the cable in place.

**This is observed, not explained.** I can see that the hot-plug path yields 13 Gb/s and that a boot with the cable attached yields 98, on the same hardware, the same cable, and the same driver and firmware versions. Whatever the NIC or its firmware does differently in the two cases is not visible from the OS side. Other people report the same number and the same fix on the NVIDIA developer forum (threads 370035, 363461, 373538 — 13.39 Gb/s → 98.01 Gb/s per interface in 363461). I read those through a summary, not the original threads.

## Fix

Attach the cable, then reboot both machines. Not one — the degraded state is per machine.

```bash
sudo systemctl reboot --no-block
```

Then measure both rails at once, one `ib_write_bw` per rail:

```
rail 0: 98.03 Gb/s
rail 1: 98.03 Gb/s
```

196 Gb/s aggregate, against NVIDIA's documented 189.85. Everything else came back by itself: the cable was detected during boot, all four PCI functions enumerated, and the NetworkManager profiles restored both fabric addresses and MTU 9000 with no manual step.

## Three traps around this

**`sudo systemctl reboot` can return without rebooting.** Twice I issued it on both machines and both stayed up — `uptime -s` unchanged, no shutdown in the journal. `sudo systemctl reboot --no-block` rebooted them within ten seconds. I did not establish why the blocking form did nothing; there were 8 queued systemd jobs (`plymouth-quit-wait.service` running, `graphical.target` waiting) on both machines, which may or may not be related.

**The RoCE GID index depends on whether the interface has an IPv4 address.** `show_gids` before assigning IPs lists only two entries per port, and RoCE v2 is index 1. After assigning IPv4, indexes 2 and 3 appear and the IPv4 RoCE v2 GID is index 3:

```
rocep1s0f0	1	1	fe80:0000:0000:0000:4ebb:47ff:fee8:9f39			v2	enp1s0f0np0
rocep1s0f0	1	3	0000:0000:0000:0000:0000:ffff:c0a8:640a	192.168.100.10  	v2	enp1s0f0np0
```

Give `ib_write_bw` the wrong `-x` and it fails in the least helpful way available: it prints its full configuration header, prints no result row, and exits 0. Early on it printed `Failed to modify QP 306 to RTR / Unable to Connect the HCA's through the link`; after that it printed nothing at all.

**Measuring one rail tells you nothing about the pair.** Each physical QSFP port maps to two logical interfaces on two controllers, each on PCIe Gen5 x4, so a single rail tops out near 100 Gb/s by design. `active_width 2X` and an `ethtool` supported-modes list that stops at `100000baseCR4/Full` are normal here, not evidence of a fault. The 200G figure only exists as the sum.

## What was not the cause

- **The cable.** `ethtool -m` makes a correct 400G-class DAC look like a 40G one. Every odd line is a legacy-format artifact — see [What `ethtool -m` gets wrong about a DAC](#what-ethtool--m-gets-wrong-about-a-dac) below.

  *This note first said the EEPROM "does not match what it is sold as" and that I could not settle whether it was mislabelled or relabelled. That was wrong, and reading the raw bytes against SFF-8636 and SFF-8024 disproved it.*
- **`Detected insufficient power on the PCIe slot (27W)`.** This `mlx5_pcie_event` line appears four times per boot on both machines, including the boot that measured 98 Gb/s per rail. It is not a throttle indicator.
- **PCIe negotiation.** `LnkSta: Speed 32GT/s, Width x4` on both controllers, matching `LnkCap`.
- **FEC errors.** RS-FEC is active and does correct a small number of bits under load (`rx_corrected_bits_phy` +4671 over a 16 GiB transfer), with `rx_crc_errors_phy` at 0 throughout. Not enough to explain a 7× shortfall.

## What `ethtool -m` gets wrong about a DAC

While suspecting the cable I read its EEPROM, and every field that looked damning turned out to be a legacy-encoding artifact. The decoded output:

```
Identifier                                : 0x11 (QSFP28)
Transceiver type                          : 40G Ethernet: 40G Base-CR4
Encoding                                  : 0x08 (PAM4)
BR, Nominal                               : 25500Mbps
Length (OM1 62.5um)                       : 7m
Length (Copper or Active cable)           : 1m
Vendor PN                                 : NJAAKK-N911
Revision Compliance                       : Unallocated
```

Read against the specs, byte by byte (`sudo ethtool -m <dev> hex on`):

| Field | What it looks like | What the spec says |
|---|---|---|
| `Identifier: 0x11 (QSFP28)` | An old 100G-class module | SFF-8024 Rev 4.14 Table 4-1: `11h` = "QSFP28 **or later** with SFF-8636 management interface". The byte selects the management memory map, not the speed. SFF-8024 4.14 has no QSFP112 identifier at all; `1Eh` means the module implements CMIS instead. A passive DAC on SFF-8636 must report `11h` |
| `40G Base-CR4` | A 40G cable | Byte 131 = `0x88`. Bit 3 (`0x08`) is the 40GBASE-CR4 compatibility claim; bit 7 (`0x80`) means "real specification is in byte 192" (SFF-8636 §6.3.4). Byte 192 = `0x3F` = **100GBASE-CR1 / 200GBASE-CR2 / 400GBASE-CR4**, 802.3ck Clause 162 (SFF-8024 Rev 4.14 Table 4-4). The 200G-class code would be `40h` |
| `BR, Nominal: 25500Mbps` | 25 Gb/s | Byte 140 = `0xFF`, the documented saturation value meaning "see byte 222". Byte 222 = `0xD5` = 213 × 250 = **53.25 GBd**. SFF-8636 Rev 2.12 §6.3.6 renamed these fields from "BR, nominal" in Mb/s to "signaling rate, nominal" in **MBd** precisely to stop this confusion; with PAM4 (byte 139) that is ~106 Gb/s per lane, ~426 Gb/s over four |
| `Length (Copper): 1m` on a 0.5 m cable | A wrong length | SFF-8636 §6.3.12: units of 1 metre, and "link lengths less than 1 meter shall indicate 1 meter". 0.5 cannot be encoded. Sub-metre granularity exists only in CMIS (byte 202, 0.1 m multiplier) |
| `Length (OM1 62.5um): 7m` | A fibre length on a copper cable | Not a length. When byte 147 bits 7–4 are `1010b` (copper, unequalized — here `0xa0`), byte 145 is **attenuation at 25.78 GHz in dB** = 7 dB. `ethtool` prints the field as OM1 metres unconditionally |
| `Revision Compliance: Unallocated` | A malformed EEPROM | Byte 1 = `0x08` = SFF-8636 Rev 2.8. `ethtool`'s table stops at `07h` |

Two integrity checks pass: the base and extended checksums recomputed from bytes 128–190 and 192–222 match the stored `0x2C` and `0x12`. Editing a part number normally breaks those.

An NVIDIA support attachment from May 2026 shows a different DGX Spark DAC — a Mellanox `QSFP-200G-CU0.5M` — reporting the same `Identifier 0x11`, the same `BR 25500Mbps`, and the same `Length (Copper): 1m` for a 0.5 m cable, with byte 192 = `0x40` for its 200G class. Same artifacts, working cable.

So: `ethtool -m` alone cannot tell you a DAC's speed class. Read byte 131 bit 7, then byte 192, then byte 222 with byte 139.

**The spec references in this section were checked against SFF-8024 Rev 4.14 and SFF-8636 Rev 2.9/2.12 by a research pass, not by me reading the documents end to end.**

## Unrelated crash found in the same journal

Worth knowing if you are poking at a freshly hot-plugged ConnectX-7: nine minutes after the cable first went in, `mstflint` took the kernel down.

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

`rcu: INFO: rcu_preempt detected stalls on CPUs/tasks` followed 70 seconds later, and the machine was restarted 14 minutes after that. `mstflint` is `4.26.0+1-2ubuntu3` from Ubuntu, with the out-of-tree `mstflint_access` module loaded at boot by `nvidia-mstflint-loader`. Both are still installed here. A null pointer at `0x00000000000000c0` in a PCI config read suggests it lost the device underneath it, which is plausible on a bus where a daemon adds and removes that device — but I have not reproduced it deliberately and cannot say that is the trigger.

---

中文原文（更长，含调查过程和走过的弯路）：[qsfp-link-13gbps-until-reboot.zh.md](qsfp-link-13gbps-until-reboot.zh.md)
