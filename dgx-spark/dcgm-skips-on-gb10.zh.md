# DGX Spark（GB10）上 dcgmi diag 几乎所有插件都跳过——以及怎么让它们跑起来

**结论先说** —— `dcgmi diag` 在 GB10 上跳过 10 个插件里的 8 个。原因不是插件检测到硬件不支持，插件根本不看硬件。它拿 PCI Device ID 去查一张编译进二进制的表，GB10 不在表里，于是退出。加 `-p "generic_mode=True"` 能把 8 个跳过里的 6 个变成 Pass。

硬件：NVIDIA DGX Spark（GB10，128 GB 统一内存）。DCGM 4.6.1，CUDA 13。

这篇是调查原文。英文版是从中提取的精简笔记，只保留 Symptom / Cause / Fix：[DCGM diag skips almost every plugin on GB10](dcgm-skips-on-gb10.md)

---

最近Nvidia DGX Spark到货了，按规矩做了一下压力测试，毕竟这东西的良品率也不是100%。

测试用Nvidia官方诊断工具：DCGM。
先用 dcgmi diag -r 1 探了一下，过了。然后运行 dcgmi diag -r 3，看memory/PCIe和targeted stress/power。

结果 -r 2 / -r 3 的硬件压力子测试全部 Skip：

```
| memory            | Skip |
| pcie              | Skip |
| diagnostic        | Skip |
| nvbandwidth       | Skip |
| memory_bandwidth  | Skip |
| targeted_stress   | Skip |
| targeted_power    | Skip |
```

问了一下AI，AI说这些插件是给数据中心独立显卡写的，依赖独立显存，PCIe 带宽，独立功耗墙。GB10 是集成统一内存 SoC，这些都没有，插件判定不适用就直接跳过了。

但是随后发现 dcgmi diag 还有个 -r 4，官方描述是 Extended (Longer-running System HW Diagnostics)。
跑了 dcgmi diag -r 4，这回发现没有全部跳过，而且竟然出现了 memtest Pass：

```
software            Pass
--- Hardware ---    memory Skip / diagnostic Skip / nvbandwidth Skip / pulse_test Skip
--- Integration --- pcie Skip
--- Stress ---      memtest Pass / memory_bandwidth Skip / targeted_stress Skip / targeted_power Skip
```

memtest就是Stress插件，说明并不存在什么独立显卡的问题，AI刚才当着我的面胡说八道。
memtest跑了20多分钟，96% util、2411 MHz、19.6W、58°C，Pass。

19.6W 看着很低，不过之前以为DCGM测不了压力测试的时候自己写了一个 torch matmul，测试结果是 94.8W。功率差5倍，区别在频率：两次都是 2200～2400 MHz，芯片没被限制，只是 memtest 这种负载吃不满功耗。

重新看skip的那些测试。

翻出skip reason，写的是：

```
Test targeted_stress: The targeted_stress test is skipped. Check DCGM and system configuration.
This error may be eliminated with an updated configuration.
```

似乎是配置问题导致的跳过。

这就引出了一个问题，DCGM到底在跑什么？

DCGM 开源，所以我去翻了一下源码。以下源码都来自 tag `v4.6.1`（SHA `64df9f89`），就是机器上跑的那个版本：

每个插件跑之前都经过一个确认，像targeted_stress（TargetedStress_wrapper.cpp:401）：

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

七个有源码的插件（对，不是所有插件都有源码），is_allowed是硬编码的：

```
memtest          "True"
memory           "False"
diagnostic       "False"
nvbandwidth      "False"
pcie             "False"
targeted_stress  "False"
targeted_power   "False"
```

只有 memtest 是 True，就是之前dcgmi diag -r 4跑过的那个。

那么问题就变成了，怎么将这些个False变成True。

nvvs/src/Allowlist.cpp 里有一个FillMap()，一张按 PCI Device ID 索引、编译进二进制的表。grep 整个 nvvs 和 dcgmlib，GB10 一次都没出现。

所以插件不看硬件，也当然就不知道当前被测试的机器是否有独立显存什么的。所以，这里就可以判断当时AI的解释是幻觉了，虽然听起来很合理。

细查发现，那句 skip reason 也是假线索。

dcgm_errors.h：

```c
DCGM_FR_TEST_DISABLED = 48, //!< 48 This test is disabled for this GPU
#define DCGM_FR_TEST_DISABLED_NEXT CONFIG_MSG
#define CONFIG_MSG "Check DCGM and system configuration. This error may be eliminated with an updated configuration."
```

CONFIG_MSG 是十几个错误码共用的通用后缀，对谁都这么说。真信息在错误码本身：This test is disabled for this GPU。

回到之前的 else 分支：

```cpp
else { log_debug("Proceeding in generic mode."); }
```

有个 generic_mode，is_allowed 是 False 的时候，只要 generic_mode 打开，插件就不退出。传参格式写在解析器自己的报错里（NvidiaValidationSuite.cpp:1435）：

```
Format should be <testname>[.<subtest>].<parameter name>=<parameter value> or generic_mode=<True|False>
```

跑之前确认了一下安不安全，毕竟不是官方文档明确写出来的用法。

先看 generic_mode 打开之后改了什么：

```
TargetedStress_wrapper.cpp:417   TARGET_PERF_MIN_RATIO  = 0.0
TargetedPower_wrapper.cpp:651    TARGET_POWER_MIN_RATIO = 0.0
```

这两个是合格线，被设成了 0。也就是说 generic_mode 关掉的是判定，不是放开什么限制。

然后 grep 了几个插件目录，看有没有写硬件状态的调用：

```
SetPowerManagementLimit / SetApplicationsClocks / SetGpuLockedClocks / nvmlDeviceSet
SetPersistenceMode / SetEccMode / dcgmConfigSet / ResetGpu
```

0 命中。这些插件只跑 CUDA 负载和读遥测，不改功耗墙、不锁频、不动固件。

开始跑：

```
nohup dcgmi diag -r 4 -p "generic_mode=True" -v > ~/dcgm_generic.log 2>&1 &
```

nohup 是必须的。没加之前有一次跑到 25 分钟 ssh 断了，我以为机器挂了，回连一看 uptime 连续，没有重启记录，dmesg 里是 Wi-Fi 在两个 AP 之间漫游，家里 eero mesh。监控断了而已，进程没问题。

整个过程53 分钟。结果如下：

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

六个检测从 Skip 变成 Pass。

targeted_stress 和 targeted_power 都有下面的内容：

```
Running in generic mode: stress threshold disabled; result reflects stability, not calibrated performance.
```

字面意思，Pass 说明的是稳定不是达标，53分钟没出错，不是性能合格，因为合格线已经被设成 0 了。

这里有几个点要注意：

memory 插件分配了 120593280555 bytes，DCGM 报 92.3%，约 120GB，Pass。

pcie 报 GPU→Host 58.37 GB/s，Host→GPU 58.23 GB/s，延迟 1.62 到 1.71 微秒。这是 Grace 和 Blackwell 之间的 C2C 互连，不是官方标称那个 273 GB/s 的显存带宽，别混。显存带宽没有拿到，memory_bandwidth Skip，nvbandwidth 虽然 Pass 但没打印数字，我用自己 torch 测的是238.3 GB/s，不能当官方数据看。

diagnostic 报了约 17597 gigaflops，内部混合精度负载，FP64 权重是 0，不能拿去跟 bf16 那 100 TFLOPS 比。

targeted_power 自己窗口里 max 31.0W、avg 25.7W；我在旁边采样，stress 阶段峰值 95.24W、69°C。这俩不是一个东西。

跑完重新看了一遍：HW Thermal Slowdown 0 us，HW Power Braking 0 us，SW Thermal Slowdown 0 us，XID 0，dmesg 无 GPU 错误，回落到 37°C / 3.54W / 208MHz。没触发任何硬件保护。

剩下 pulse_test 和 memory_bandwidth，generic_mode 也没用。回去看 nvvs/plugin_src/ 底下的实现目录：

```
contextcreate  diagnostic  memory  memtest  nccl_tests
nvbandwidth  pcie  targetedpower  targetedstress
```

没有 pulse，也没有 memorybandwidth。这两个在开源仓库里只剩名字和接口，实现是闭源的，只在 deb 包里发。

调查到此为止。
