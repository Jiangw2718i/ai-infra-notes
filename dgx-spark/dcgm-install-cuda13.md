# DCGM: "Detected unsupported Cuda version" on DGX Spark (GB10, CUDA 13)

**TL;DR** — `apt-get install datacenter-gpu-manager` gives you 3.3.9, which does not support CUDA 13 or GB10. The package you want is `datacenter-gpu-manager-4-cuda13`. `datacenter-gpu-manager-4`, the obvious guess, does not exist.

Hardware: NVIDIA DGX Spark (GB10). CUDA 13.

## Symptom

Install the package with the name you would expect:

```
sudo apt-get install datacenter-gpu-manager
```

Then run a diagnostic and it fails immediately:

```
Detected unsupported Cuda version
```

## Cause

The CUDA repo serves 3.3.9 under that name:

```
apt-cache madison datacenter-gpu-manager
```

Only 3.3.6 through 3.3.9 are listed. DCGM 3.x does not know CUDA 13 or GB10.

Appending the major version does not work either:

```
sudo apt-get install datacenter-gpu-manager-4
E: Package 'datacenter-gpu-manager-4' has no installation candidate
```

## Fix

The version 4 packages carry the CUDA version in the name. Search first:

```
apt-cache search datacenter-gpu-manager
```

That lists `datacenter-gpu-manager-4-cuda13` and `datacenter-gpu-manager-4-core`. You only need to ask for the first one:

```
sudo apt-get install -y datacenter-gpu-manager-4-cuda13
```

`-4-core` is a hard dependency and the proprietary packages are pulled in as well:

```
apt-cache depends datacenter-gpu-manager-4-cuda13
  Depends: datacenter-gpu-manager-4-core
  Recommends: datacenter-gpu-manager-4-proprietary-cuda13
```

After the install, four packages are present:

```
ii  datacenter-gpu-manager-4-core                1:4.6.1-1   arm64
ii  datacenter-gpu-manager-4-cuda13              1:4.6.1-1   arm64
ii  datacenter-gpu-manager-4-proprietary         1:4.6.1-1   arm64
ii  datacenter-gpu-manager-4-proprietary-cuda13  1:4.6.1-1   arm64
```

Check:

```
dcgmi version          → 4.6.1
dcgmi discovery -l     → NVIDIA GB10
```

`nv-hostengine` starts and the diagnostics run.

## If you already installed the 3.x package

Remove it before installing version 4:

```
sudo apt-get remove -y datacenter-gpu-manager
```

The package names do not collide, so in principle both can be installed at once. But both ship `/usr/bin/dcgmi`, and I did not test what happens when they coexist. Removing first is the cautious path, not a measured result.

## Next

`dcgmi diag` will now start, but on GB10 it skips 8 of its 10 plugins. That has a different cause and a workaround — see [DCGM diag skips almost every plugin on GB10](dcgm-skips-on-gb10.md).
