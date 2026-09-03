# Nosana GPU node on Windows 11 with Hyper-V GPU-PV and NVIDIA GPU

> **Experimental guide / real-world report**
>
> This is a record of a working setup we built and tested on a Windows 11 Pro machine with an NVIDIA GeForce RTX 4080 SUPER. The goal was to run a Nosana GPU node without replacing Windows with Linux and without changing the home router or network.
>
> The final working stack was:
>
> **Windows 11 → Hyper-V → Ubuntu 26.04.1 LTS → Hyper-V GPU-PV → easy-gpu-pv-linux → NVIDIA CUDA/NVML → Docker/Podman → Nosana node**
>
> This is deliberately written as a practical experiment report, not as an official Nosana installation guide.

## What we were trying to achieve

The original machine was a normal Windows 11 gaming/work PC with a GeForce RTX 4080 SUPER.

The idea was to use the otherwise idle GPU as a Nosana provider while keeping Windows as the main operating system.

There were two important constraints:

- the physical network configuration must not be changed;
- the GPU should remain usable from Windows whenever the Nosana VM is not running.

Native Linux was not the target. A normal native-Linux Nosana node is not especially interesting from a “can this be done from a Windows PC?” perspective.

## Attempt 1: WSL2

The first attempt was the obvious one: Windows 11 + WSL2 + Ubuntu.

A clean Ubuntu 24.04 WSL2 installation was able to see the RTX 4080 SUPER correctly:

```text
NVIDIA GeForce RTX 4080 SUPER
16376 MiB VRAM
```

So the Windows → WSL2 → NVIDIA/CUDA path itself worked.

The problem was Nosana.

The older Nosana quick-start script contains an explicit check which aborts when `/proc/version` contains `WSL2`. More importantly, the Nosana node's market health check itself rejected the environment because its `system_environment` contained:

```text
6.18.33.2-microsoft-standard-WSL2
```

and the market rule was:

```text
system_environment does not contain "WSL"
```

So this was not simply a missing CUDA package. The machine could have perfectly working GPU access and still fail the market eligibility check.

During testing we could see both sides clearly: the GPU and benchmark passed, while the WSL environment check failed.

**Conclusion:** ordinary WSL2 was not sufficient for the Nosana 4080 Community market.

## Attempt 2: native Linux

We also considered / tested native Linux.

That is the straightforward configuration and is outside the goal of this experiment.

**Conclusion:** technically straightforward, but not the solution we wanted.

## Attempt 3: Hyper-V with direct PCIe assignment (DDA)

The next idea was to create a normal Ubuntu Gen 2 VM in Hyper-V and give the RTX 4080 SUPER to it using Discrete Device Assignment (DDA).

This is the classic approach where the physical GPU is detached from Windows and assigned directly to the VM.

We prepared the VM for DDA with:

- Generation 2
- fixed (non-dynamic) RAM
- `GuestControlledCacheTypes = True`
- 3 GB low MMIO
- 32 GB high MMIO

The RTX 4080 SUPER was identified on Windows as:

```text
InstanceId:
PCI\VEN_10DE&DEV_2702&SUBSYS_511A1462&REV_A1\4&153C50D1&0&0008

LocationPath:
PCIROOT(0)#PCI(0100)#PCI(0000)
```

DDA was a plausible route, but it was not the method we ultimately kept.

The final solution used **Hyper-V GPU-PV (GPU partitioning)** instead. This had a major practical advantage for us: we did not have to permanently detach the GPU from Windows just to make the Linux VM work.

## The solution that actually worked: Hyper-V GPU-PV

The final working configuration was:

```text
Windows 11
    |
    +-- Hyper-V
          |
          +-- Ubuntu 26.04.1 LTS VM
                |
                +-- Hyper-V GPU-PV partition
                |
                +-- easy-gpu-pv-linux
                |
                +-- NVIDIA CUDA / NVML
                |
                +-- Docker + NVIDIA Container Toolkit
                |
                +-- Nosana node
```

The key component was:

**easy-gpu-pv-linux**

GitHub project:

`https://github.com/Ripthulhu/easy-gpu-pv-linux`

The project provides a way to expose the Windows NVIDIA driver to a Linux Hyper-V guest using GPU-PV. It installs the required payload into the Linux guest and provides the `/dev/dxg` / CUDA / NVML side needed by the Linux applications.

### 1. Create the Ubuntu VM

We used:

- Hyper-V Generation 2
- Ubuntu 26.04.1 LTS
- 8 vCPU
- 24 GB fixed RAM
- Dynamic Memory disabled
- Secure Boot disabled

The installed guest reported:

```text
Ubuntu 26.04.1 LTS
kernel 7.0.0-30-generic
8 CPU
~14 GiB visible RAM
4 GiB swap
systemd: running
```

### 2. Configure the Hyper-V GPU partition

The VM received a GPU partition with:

```powershell
Add-VMGpuPartitionAdapter -VMName "Ubuntu Nosana"
```

We also configured the VM for GPU-PV using the standard Hyper-V settings required by the experiment.

The exact values used for the VM were:

```powershell
Set-VM -Name "Ubuntu Nosana" -GuestControlledCacheTypes $true
Set-VM -Name "Ubuntu Nosana" -LowMemoryMappedIoSpace 3Gb
Set-VM -Name "Ubuntu Nosana" -HighMemoryMappedIoSpace 33280Mb
```

These settings were verified before continuing.

### 3. Prepare SSH access

The easy-gpu-pv helper uses SSH to configure the Linux guest.

We created an Ed25519 key on Windows:

```powershell
ssh-keygen -t ed25519
```

The key was saved as:

```text
C:\Users\dewil\.ssh\id_ed25519
```

The guest user was `vasek`.

One setup issue was that the helper needed passwordless sudo in the guest because it runs non-interactively.

The working configuration used:

```bash
echo 'vasek ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/gpup
```

### 4. Run the GPU-PV helper

From the `easy-gpu-pv-linux` directory on Windows:

```powershell
Get-ChildItem *.ps1 | Unblock-File
```

First perform a dry run:

```powershell
.\Enable-GpupVM.ps1 `
  -VMName "Ubuntu Nosana" `
  -User "vasek" `
  -KeyFile "$env:USERPROFILE\.ssh\id_ed25519" `
  -Unsupported `
  -WhatIf
```

`-Unsupported` was needed because the project had been tested against Ubuntu 24.04 / Debian 13 while we were using Ubuntu 26.04.1.

After checking the planned changes, the actual command used was:

```powershell
.\Enable-GpupVM.ps1 `
  -VMName "Ubuntu Nosana" `
  -User "vasek" `
  -KeyFile "$env:USERPROFILE\.ssh\id_ed25519" `
  -Unsupported `
  -Ssh "vasek@172.25.182.69"
```

The helper then:

- verified the Hyper-V guest;
- verified the GPU partition;
- verified Ubuntu and the kernel;
- installed the GPU-PV driver payload;
- staged the NVIDIA driver files from the Windows DriverStore;
- installed the guest-side files under `/usr/lib/gpup`;
- linked the required `/usr/lib/wsl` paths;
- rebooted the guest.

The payload in this run came from the Windows NVIDIA driver and was roughly 329 MB when transferred to the guest.

### 5. Verify NVIDIA access in Ubuntu

The first important verification command was:

```bash
sudo gpup-verify --trace
```

The successful GPU-PV state showed:

```text
PASS  /dev/dxg               opened
PASS  vmbus channels         2 claimed
PASS  NVML / nvidia-smi      NVIDIA GeForce RTX 4080 SUPER
PASS  CUDA                   1 device(s): NVIDIA GeForce RTX 4080 SUPER
```

One diagnostic check also reported:

```text
FAIL  nvcubins.bin MISSING
```

That message turned out not to mean that CUDA was unusable. The later CDI and Docker tests proved that the actual CUDA path worked.

The important point is that inside the normal Hyper-V Ubuntu VM we had working NVIDIA access.

### 6. Configure NVIDIA Container Device Interface (CDI)

The normal Nosana container workflow needed GPU access from inside containers.

We generated CDI with:

```bash
nvidia-ctk cdi generate --mode=wsl
```

Despite the `--mode=wsl` name, in this setup the NVIDIA toolkit used the GPU-PV / `/dev/dxg` driver store and generated a usable CDI definition.

The resulting CDI device was:

```text
nvidia.com/gpu=all
```

We verified it with:

```bash
nvidia-ctk cdi list
```

### 7. Verify that Docker can actually use the GPU

This was the critical test:

```bash
sudo docker run --rm \
  --device nvidia.com/gpu=all \
  ubuntu nvidia-smi
```

The container successfully reported the RTX 4080 SUPER.

That proved the entire chain:

```text
Hyper-V GPU-PV
    ↓
Linux guest
    ↓
NVIDIA CUDA / NVML
    ↓
NVIDIA CDI
    ↓
Docker container
    ↓
RTX 4080 SUPER
```

At that point the graphics stack itself was no longer the problem.

### 8. Install and run the Nosana node

We had already installed the Nosana CLI during the earlier WSL testing:

```bash
npm install -g @nosana/cli
```

At the time of the experiment it reported:

```text
nosana 1.0.133
```

The CLI exposed:

```bash
nosana node start --help
```

with support for Docker / Podman and a GPU selection.

The actual working node package used during the final test was:

```text
@nosana/node 1.1.59
```

The node's own health check showed that the 4080 Community requirements were satisfied:

```text
system_environment              7.0.0-30-generic
package_version                 1.1.59
ornith-9b__tokens_per_second    70.5
gpu.name                        NVIDIA GeForce RTX 4080 SUPER
gpu.verified                    true
gpu.vram_total_mb               16375.5
```

The node therefore passed all five market requirements.

There was one important exception: the node had to maintain a minimum SOL balance.

At one point the health check stopped the node because:

```text
SOL balance 0.00264922 should be 0.005 or higher.
```

This is a runtime/economic issue, not a GPU/Hyper-V issue.

## What happened with the WSL attempt

The WSL2 approach was abandoned before the final Hyper-V setup. Although the RTX 4080 SUPER and CUDA worked inside WSL2, Nosana rejected the environment because the reported `system_environment` contained `WSL`.

We did experiment with modifying the Nosana Node.js code to spoof that value, including changes around `NodeRepository.js`, but this did **not** provide a reliable solution. During metric collection the node obtained the WSL environment value again.

This workaround is therefore **not part of the final Hyper-V GPU-PV configuration** and is not required for the working setup documented below.

## What happened with the first real job

Once the node was accepted into the `NVIDIA 4080 Community`, it fetched the market's required resources, including large Docker images such as vLLM and TensorFlow.

The first actual workload we observed was labeled:

```text
foldingAtHome
```

The job completed and the wallet received:

```text
+0.3051 NOS
```

At the time that was about $0.09.

The Solana transaction history also showed several account-creation transactions consuming SOL.

The important lesson is that the initial account/setup costs and normal job settlement costs should not be confused with one another.

Later, when the node's SOL balance dropped below the required minimum of 0.005 SOL, the health check refused to start another job and shut the node down.

We therefore did **not** consider the one-job run a profitability test.

## What this setup proved

The final experiment proved that a Windows 11 desktop can host a normal Linux VM in Hyper-V and expose a consumer NVIDIA GPU through Hyper-V GPU-PV well enough for:

- Linux `nvidia-smi`
- CUDA
- NVIDIA NVML
- NVIDIA CDI
- Docker GPU containers
- Nosana's GPU benchmark
- Nosana market eligibility

The important trick was not to fight Nosana's old WSL-specific installation script. Instead, the final configuration presented a Linux guest that looked like a normal Linux environment to the node while GPU-PV supplied the NVIDIA device.

## What this guide does not claim

This is not a claim that:

- Nosana officially supports every Hyper-V GPU-PV configuration;
- every RTX 40/50-series card will behave identically;
- Ubuntu 26.04 is officially supported by the easy-gpu-pv project;
- running one home GPU on Nosana is economically profitable.

The guide documents one **real, working setup** and the problems encountered along the way.

## Troubleshooting notes from our experiment

### `This account is currently not available`

This happened in the Salad-provided WSL distribution. The `salad-enterprise-linux` image was not a normal interactive Linux environment.

Using a clean Ubuntu WSL distribution avoided that problem.

### WSL2 GPU works, but Nosana still rejects the host

This was the environment check:

```text
system_environment does not contain "WSL"
```

The WSL kernel string itself was the reason the market check failed.

### `nvidia-persistenced` is missing

The old Nosana quick-start script expected the traditional Linux NVIDIA driver stack and checked for:

```text
/run/nvidia-persistenced
```

The GPU-PV environment we used did not provide that daemon in the normal way.

This was another reason not to use the old quick-start wrapper unchanged.

The actual Docker/CDI GPU path worked without that daemon.

### Hyper-V VM has Linux but no NVIDIA GPU

Installing the GPU partition adapter alone is not enough.

The Linux guest also needs the GPU-PV userspace/driver payload. In our case this was supplied by `easy-gpu-pv-linux`.

### CDI works but the usual `--gpus all` syntax does not

The working container test used CDI explicitly:

```bash
--device nvidia.com/gpu=all
```

rather than relying only on the classic NVIDIA runtime syntax.

That distinction mattered in this GPU-PV setup.

## Final result

The final working chain was:

```text
Windows 11 Pro
    ↓
Hyper-V Gen 2 VM
    ↓
Ubuntu 26.04.1 LTS
    ↓
Hyper-V GPU-PV
    ↓
easy-gpu-pv-linux
    ↓
/dev/dxg + NVIDIA userspace
    ↓
CUDA / NVML
    ↓
NVIDIA CDI
    ↓
Docker GPU container
    ↓
Nosana node
    ↓
NVIDIA 4080 Community
```

The node really did accept a job and complete it.

The experiment also exposed an important practical issue: **technical success and economic viability are two separate questions**. Our one completed job paid 0.3051 NOS, while the node simultaneously consumed SOL for Solana-side operations and later refused new work after the balance dropped below the required 0.005 SOL.

For that reason, the economic section should be treated as a separate investigation rather than inferred from the first successful technical run.
