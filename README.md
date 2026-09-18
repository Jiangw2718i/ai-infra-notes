# AI infra notes

Notes from running AI workloads on my own hardware. What broke, what the actual cause turned out to be, and what fixed it.

Error text is quoted verbatim so you can find these by searching for the error you are looking at. Everything here was run on my own machines. Where I could not settle something, it says so.

## DGX Spark (GB10)

- **[The 200GbE QSFP link between two Sparks runs at 13 Gb/s until you reboot with the cable attached](dgx-spark/qsfp-link-13gbps-until-reboot.md)** — `ib_write_bw` reports 13.2 Gb/s per rail instead of ~98, and nothing at runtime moves it: not MTU 9000, not queue pairs, not forcing the link down to 40G, not removing the NIC from the PCI bus and rescanning. The cable has to be attached while the machine boots. Same cable after a reboot: 98.03 + 98.03 Gb/s. · 中文原文：[两台 DGX Spark 的 200GbE 直连只有 13 Gb/s——直到带着线重启](dgx-spark/qsfp-link-13gbps-until-reboot.zh.md)

- **[DCGM diag skips almost every plugin on GB10](dgx-spark/dcgm-skips-on-gb10.md)** — `The targeted_stress test is skipped. Check DCGM and system configuration.` The plugins never look at the hardware. They look up a PCI device ID in a table compiled into the binary, GB10 is not in it, and they exit. `-p "generic_mode=True"` turns 6 of the 8 skips into passes. · 中文原文：[DGX Spark（GB10）上 dcgmi diag 几乎所有插件都跳过](dgx-spark/dcgm-skips-on-gb10.zh.md)

- **[DCGM install: "Detected unsupported Cuda version"](dgx-spark/dcgm-install-cuda13.md)** — `apt-get install datacenter-gpu-manager` gives you 3.3.9, which does not support CUDA 13. `datacenter-gpu-manager-4` does not exist. The package is `datacenter-gpu-manager-4-cuda13`.

## Local LLM

- **[Claude Code with tool search gets a 400 on the first request of every session against vLLM](local-llm/tool-search-first-request-400.md)** — `'input': 'tool_addition'` … `Input should be 'text', 'image', 'tool_use', 'tool_result', 'tool_reference', 'thinking' or 'redacted_thinking'`. The first request carries `tool_addition` content blocks (beta `mid-conversation-tool-changes-2026-07-01`) that vLLM's `/v1/messages` does not accept; the client drops them and the beta header, resends, and nothing appears on the command line. I first blamed `input_schema`, filed two issues on that, installed a fix for it, and only then captured the real request. · 中文原文：[Claude Code 开着 Tool Search 接 vLLM，每个会话的第一个请求都被 400 拒掉](local-llm/tool-search-first-request-400.zh.md)

## Hardware

```
NVIDIA DGX Spark (GB10) x2, BIOS 5.36_0ACUM027
20-core Arm: 10x Cortex-X925 @ 3900 MHz + 10x Cortex-A725 @ 2808 MHz
128 GB unified LPDDR5X (121 Gi visible to the OS, 130.7 GB GPU-addressable)
CUDA 13, DCGM 4.6.1, driver 580.173.02, kernel 6.17.0-1032-nvidia
ConnectX-7 (MT2910) firmware 28.45.4028, one 200GbE QSFP DAC between the two units
```

---

How these notes are structured: [CONVENTIONS.md](CONVENTIONS.md)
