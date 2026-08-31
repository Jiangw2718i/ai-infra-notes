# AI infra notes

Notes from running AI workloads on my own hardware. What broke, what the actual cause turned out to be, and what fixed it.

Error text is quoted verbatim so you can find these by searching for the error you are looking at. Everything here was run on my own machines. Where I could not settle something, it says so.

## DGX Spark (GB10)

- **[DCGM diag skips almost every plugin on GB10](dgx-spark/dcgm-skips-on-gb10.md)** — `The targeted_stress test is skipped. Check DCGM and system configuration.` The plugins never look at the hardware. They look up a PCI device ID in a table compiled into the binary, GB10 is not in it, and they exit. `-p "generic_mode=True"` turns 6 of the 8 skips into passes.

- **[DCGM install: "Detected unsupported Cuda version"](dgx-spark/dcgm-install-cuda13.md)** — `apt-get install datacenter-gpu-manager` gives you 3.3.9, which does not support CUDA 13. `datacenter-gpu-manager-4` does not exist. The package is `datacenter-gpu-manager-4-cuda13`.

## Hardware

```
NVIDIA DGX Spark (GB10)
20-core Arm: 10x Cortex-X925 @ 3900 MHz + 10x Cortex-A725 @ 2808 MHz
128 GB unified LPDDR5X (121 Gi visible to the OS, 130.7 GB GPU-addressable)
CUDA 13, DCGM 4.6.1
```
