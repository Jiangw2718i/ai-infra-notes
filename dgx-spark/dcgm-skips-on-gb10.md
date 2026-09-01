# DCGM diag skips almost every plugin on DGX Spark (GB10) — and how to run them anyway

**TL;DR** — `dcgmi diag` skips 8 of 10 plugins on GB10. It is not because the plugins detect unsupported hardware. They never look at the hardware. They look up the PCI device ID in a table compiled into the binary, GB10 is not in it, and they exit. Adding `-p "generic_mode=True"` turns 6 of the 8 skips into passes.

Hardware: NVIDIA DGX Spark (GB10, 128 GB unified memory). DCGM 4.6.1, CUDA 13.

## Symptom

`dcgmi diag -r 1` passes. `-r 2` and `-r 3` skip every hardware and stress plugin:

```
| memory            | Skip |
| pcie              | Skip |
| diagnostic        | Skip |
| nvbandwidth       | Skip |
| memory_bandwidth  | Skip |
| targeted_stress   | Skip |
| targeted_power    | Skip |
```

`-r 4` (Extended) runs one more, `memtest`, which passes:

```
software            Pass
--- Hardware ---    memory Skip / diagnostic Skip / nvbandwidth Skip / pulse_test Skip
--- Integration --- pcie Skip
--- Stress ---      memtest Pass / memory_bandwidth Skip / targeted_stress Skip / targeted_power Skip
```

The skip reason looks like a configuration problem:

```
Test targeted_stress: The targeted_stress test is skipped. Check DCGM and system configuration.
This error may be eliminated with an updated configuration.
```

It is not a configuration problem. That sentence is boilerplate — see below.

## The explanation that is wrong

I asked an AI first. It said these plugins are written for datacenter discrete GPUs and depend on dedicated VRAM, PCIe bandwidth and a separate power rail; GB10 is an integrated unified-memory SoC, so the plugins decide they do not apply and skip.

Every word of that fits the facts I had. It is still wrong.

`memtest` is a Stress-section plugin and it runs on GB10. If the plugins were skipping because the SoC lacks dedicated VRAM, `memtest` would skip too.

## Root cause

DCGM is open source. Source below is from tag `v4.6.1` (SHA `64df9f89`), the version running on the machine.

Every plugin passes the same gate before it runs. From `nvvs/plugin_src/targetedstress/TargetedStress_wrapper.cpp:401`:

```cpp
if (!GetBoolFromString(TS_STR_IS_ALLOWED))
{
    if (GetBoolFromString(PS_USE_GENERIC_MODE) == false)
    {
        DCGM_ERROR_FORMAT_MESSAGE(DCGM_FR_TEST_DISABLED, d, TS_PLUGIN_NAME);
        SetResult(testName, NVVS_RESULT_SKIP);
        return;
    }
    else { log_debug("Proceeding in generic mode."); }
}
```

The hardcoded default of `is_allowed`, for the seven plugins that have source in the repo:

```
memtest          "True"
memory           "False"
diagnostic       "False"
nvbandwidth      "False"
pcie             "False"
targeted_stress  "False"
targeted_power   "False"
```

Only `memtest` defaults to `True`. That is exactly the observed result: `memtest` runs, everything else skips.

What flips `is_allowed` to `True` is `FillMap()` in `nvvs/src/Allowlist.cpp`, a table indexed by PCI device ID and compiled into the binary. (`nvvs/configfile_examples/allowlist.txt` is only an example file — editing it does nothing.) Lookup failure takes this path:

```cpp
DCGM_LOG_INFO << "DeviceId " << deviceId.c_str() << " is NOT allowlisted";
return false;
```

`GB10` does not appear anywhere in `nvvs/` or `dcgmlib/`.

So the plugin never inspects the hardware. It looks up an ID, does not find it, and exits.

## Why the skip message is misleading

From `dcgmlib/dcgm_errors.h`:

```c
DCGM_FR_TEST_DISABLED = 48, //!< 48 This test is disabled for this GPU
#define DCGM_FR_TEST_DISABLED_NEXT CONFIG_MSG
#define CONFIG_MSG "Check DCGM and system configuration. This error may be eliminated with an updated configuration."
```

`CONFIG_MSG` is a generic "next steps" suffix shared by more than a dozen error codes. It says the same thing to everyone and carries no diagnostic information.

The real information is in the error code: **This test is disabled for this GPU.**

## Fix: generic mode

The `else` branch above is the way out. When `is_allowed` is `False`, the plugin still runs if `generic_mode` is on. The parameter format is documented in the parser's own error text (`nvvs/src/NvidiaValidationSuite.cpp:1435`):

```
Format should be <testname>[.<subtest>].<parameter name>=<parameter value> or generic_mode=<True|False>
```

```
dcgmi diag -r 4 -p "generic_mode=True"
```

`dcgmi` forwards the parameter correctly; there is no need to call `nvvs` directly. It also works per plugin, e.g. `dcgmi diag -r pcie -p "generic_mode=True"`.

## Is it safe

I checked the source before running it, because this is an uncalibrated path on a new machine.

Generic mode sets the pass thresholds to zero:

```
TargetedStress_wrapper.cpp:417   TARGET_PERF_MIN_RATIO  = 0.0
TargetedPower_wrapper.cpp:651    TARGET_POWER_MIN_RATIO = 0.0
```

So it disables the verdict, not a limit.

Grepping the plugin directories for calls that write hardware state returns nothing:

```
SetPowerManagementLimit / SetApplicationsClocks / SetGpuLockedClocks / nvmlDeviceSet
SetPersistenceMode / SetEccMode / dcgmConfigSet / ResetGpu
```

These plugins only run CUDA workloads and read telemetry. They do not change the power cap, lock clocks, or touch firmware. `targeted_power` does not force a power level either — `TP_STR_TARGET_POWER = 100.0` is a target to observe, and its failure message is `Max power of X did not reach desired power minimum`.

After a 53-minute run: `HW Thermal Slowdown 0 us`, `HW Power Braking 0 us`, `SW Thermal Slowdown 0 us`, XID count 0, no GPU errors in dmesg, GPU back to 37 °C / 3.54 W / 208 MHz. No hardware protection was triggered.

## Result

```
software          Pass
--- Hardware ---
memory            Pass
diagnostic        Pass
nvbandwidth       Pass
pulse_test        Skip
--- Integration ---
pcie              Pass
--- Stress ---
memtest           Pass
memory_bandwidth  Skip
targeted_stress   Pass
targeted_power    Pass
```

Six plugins went from Skip to Pass. Run time 53 minutes.

`targeted_stress` and `targeted_power` both print:

```
Running in generic mode: stress threshold disabled; result reflects stability, not calibrated performance.
```

Read that literally. Pass here means the machine ran for 53 minutes without errors. It does not mean it met a performance bar, because the bar was set to zero.

## Numbers, with the traps

- `memory` allocated 120593280555 bytes, reported by DCGM as 92.3%, about 120 GB. Pass.
- `pcie` reported GPU→Host 58.37 GB/s, Host→GPU 58.23 GB/s, latency 1.62–1.71 µs. **This is the Grace↔Blackwell C2C interconnect, not memory bandwidth.** Do not compare it to the 273 GB/s figure in the spec.
- Memory bandwidth was not obtained. `memory_bandwidth` still skips and `nvbandwidth` passes without printing a number. My own torch measurement was 238.3 GB/s, which is a floor for torch operators, not a device peak.
- `diagnostic` reported roughly 17597 gigaflops from a mixed-precision internal workload with the FP64 weight at 0. Not comparable to a bf16 figure.
- `targeted_power` reported max 31.0 W / avg 25.7 W inside its own measurement window. Separate sampling during the stress phase peaked at 95.24 W / 69 °C. These are different measurements, not a contradiction.

## What still skips

`pulse_test` and `memory_bandwidth` skip even in generic mode.

The plugin implementation directories in `nvvs/plugin_src/` are:

```
contextcreate  diagnostic  memory  memtest  nccl_tests
nvbandwidth  pcie  targetedpower  targetedstress
```

There is no `pulse` and no `memorybandwidth`. Both exist in the open-source repo only as names and interfaces (`PULSE_TEST_PLUGIN_NAME`, `MEMBW_PLUGIN_NAME`, `DCGM_PULSE_TEST_INDEX`, `DCGM_MEMORY_BANDWIDTH_INDEX`). The implementations are closed source and ship only as binaries in the deb package.

So why they ignore `generic_mode` cannot be answered from source. The gate is not in the open tree.

---

中文原文（更长，含调查过程）：[dcgm-skips-on-gb10.zh.md](dcgm-skips-on-gb10.zh.md)
