# RTX 5090 on Hyper-V with GPU-PV and Nosana

This document describes a working setup for running an **NVIDIA GeForce RTX 5090** as a GPU compute node for **Nosana** inside an **Ubuntu 26.04.1 LTS virtual machine on Windows 11 Hyper-V**.

The GPU is exposed to the Linux VM using **Hyper-V GPU-PV** and `easy-gpu-pv-linux`.

The setup was verified with:

* NVIDIA GeForce RTX 5090
* Ubuntu 26.04.1 LTS
* Linux kernel `7.0.0-34-generic`
* Hyper-V Generation 2 VM
* Hyper-V GPU-PV
* `easy-gpu-pv-linux`
* NVIDIA driver supplied through the GPU-PV setup
* Docker `29.1.3`
* NVIDIA Container Toolkit `1.20.1`
* NVIDIA CDI
* Nosana package `1.1.66`
* Nosana Podman image `nosana/podman:v1.1.2`

The final architecture is:

```text
Windows 11
    │
    └── Hyper-V
          │
          └── Ubuntu 26.04.1 LTS VM
                │
                ├── Hyper-V GPU-PV
                │
                ├── NVIDIA driver / CUDA / NVML
                │
                ├── Docker
                │     └── NVIDIA CDI
                │
                └── Nosana
                      │
                      └── nosana/podman:v1.1.2
                            │
                            └── Podman
                                  │
                                  └── RTX 5090
```

---

## 1. Hardware and software

### Host

* Windows 11
* Hyper-V enabled
* NVIDIA GeForce RTX 5090

### Virtual machine

* VM name: `Ubuntu Nosana 5090`
* Generation: 2
* Ubuntu: 26.04.1 LTS
* Kernel: `7.0.0-34-generic`
* Secure Boot: **disabled**
* Dynamic Memory: initially **disabled**

> **Important:** Use fixed/static RAM while installing and verifying GPU-PV, Docker GPU access and the first Nosana benchmark. Dynamic Memory can be enabled later after the complete setup has been verified.

---

# 2. Create the Hyper-V VM

Create a normal **Generation 2** Ubuntu VM.

For the initial installation and GPU-PV setup:

* use static memory
* disable Secure Boot
* install Ubuntu 26.04.1 LTS
* make sure SSH is available

Example VM name:

```text
Ubuntu Nosana 5090
```

The Linux username used below is:

```text
<LinuxUser>
```

The VM address is represented as:

```text
<VM_IP>
```

---

# 3. Add the Hyper-V GPU partition

Power off the VM before configuring GPU-PV.

In an elevated PowerShell:

```powershell
Add-VMGpuPartitionAdapter -VMName "Ubuntu Nosana 5090"
```

Do not blindly copy MMIO/cache settings from an RTX 4080 setup. The RTX 5090 setup described here does not depend on manually specified DDA/MMIO values.

---

# 4. Install easy-gpu-pv-linux

The GPU-PV helper used for this setup is:

```text
easy-gpu-pv-linux
```

Repository:

```text
https://github.com/Ripthulhu/easy-gpu-pv-linux
```

Clone or download the repository on the Windows host.

The working directory used in this setup was:

```text
easy-gpu-pv-linux-main
```

Before running PowerShell scripts downloaded from GitHub, unblock them:

```powershell
Get-ChildItem *.ps1 | Unblock-File
```

---

# 5. SSH access

The helper uses SSH to configure the Ubuntu guest.

Example:

```powershell
ssh <LinuxUser>@<VM_IP>
```

The SSH key used by the helper is:

```text
C:\Users\<WindowsUser>\.ssh\id_ed25519_nosana
```

---

# 6. Passwordless sudo

The GPU-PV helper needs passwordless sudo inside the guest.

Configure sudo for the Linux user:

```bash
sudo visudo
```

Add:

```text
<LinuxUser> ALL=(ALL) NOPASSWD:ALL
```

Verify:

```bash
sudo -n true
echo $?
```

Expected result:

```text
0
```

---

# 7. Run the GPU-PV helper

Make sure the VM is running and SSH is working.

From the `easy-gpu-pv-linux-main` directory on Windows:

```powershell
.\Enable-GpupVM.ps1 `
  -VMName "Ubuntu Nosana 5090" `
  -User "<LinuxUser>" `
  -Ssh "<LinuxUser>@<VM_IP>" `
  -KeyFile "$env:USERPROFILE\.ssh\id_ed25519_nosana"
```

The helper configures the required Hyper-V GPU-PV components inside Ubuntu.

The working configuration verified:

* Hyper-V GPU partition
* Ubuntu 26.04.1
* kernel `7.0.0-34-generic`
* `/dev/dxg`
* NVIDIA GPU visibility
* NVML
* CUDA
* OpenGL/D3D12
* required GPU-PV kernel components

---

# 8. Verify GPU-PV

After the helper has completed and the VM has rebooted:

```bash
nvidia-smi
```

The expected GPU is:

```text
NVIDIA GeForce RTX 5090
```

Also verify CUDA:

```bash
sudo gpup-verify --trace
```

The important requirement is that NVIDIA NVML and CUDA can access the RTX 5090.

The GPU-PV environment may report unrelated optional failures such as NVENC or Vulkan. These are not required for the Nosana compute workload.

---

# 9. Verify the GPU directly from Linux

Run:

```bash
nvidia-smi --query-gpu=name --format=csv,noheader
```

Expected:

```text
NVIDIA GeForce RTX 5090
```

The working system reported approximately 32 GB of VRAM.

---

# 10. Docker and NVIDIA Container Toolkit

Install Docker and the NVIDIA Container Toolkit.

The verified versions in this setup are:

```text
Docker: 29.1.3
NVIDIA Container Toolkit: 1.20.1
```

Check:

```bash
docker --version
nvidia-ctk --version
```

---

# 11. Configure NVIDIA CDI

This setup uses **NVIDIA CDI** for container GPU access.

This is important because the standard Nosana Quick-Start script uses the older Docker `--gpus all` mechanism, while this Hyper-V GPU-PV setup works with CDI.

Verify the NVIDIA CDI configuration:

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

Check that the CDI device exists:

```bash
nvidia-ctk cdi list
```

You should see an NVIDIA GPU device available through:

```text
nvidia.com/gpu
```

---

# 12. Critical Docker GPU test

Before installing or starting Nosana, verify that Docker itself can access the RTX 5090.

Run:

```bash
docker run --rm --device nvidia.com/gpu=all ubuntu nvidia-smi
```

This must successfully display the RTX 5090.

Expected GPU:

```text
NVIDIA GeForce RTX 5090
```

This test is important.

If this command cannot see the GPU, do not continue with the Nosana installation yet.

The working setup uses:

```text
Docker
  ↓
NVIDIA CDI
  ↓
nvidia.com/gpu=all
  ↓
RTX 5090
```

---

# 13. Nosana directory

Nosana stores its configuration and wallet under:

```bash
~/.nosana
```

The wallet file is:

```text
~/.nosana/nosana_key.json
```

Protect the wallet:

```bash
chmod 600 ~/.nosana/nosana_key.json
```

Do not publish `nosana_key.json`.

---

# 14. Nosana Quick-Start script

The Nosana Quick-Start script used here is based on the upstream script, but the Hyper-V GPU-PV/CDI environment requires a few specific modifications.

The script is **not a complete rewrite**.

The working version differs from the upstream script in these areas:

### 14.1 Load required kernel modules

At the beginning of `nosana-start.sh`:

```bash
# Required kernel modules for Docker/Nosana networking
sudo modprobe ip_tables
sudo modprobe iptable_nat
```

These modules are required before starting the Nosana environment.

This makes the startup script self-contained instead of requiring the modules to be loaded manually first.

---

### 14.2 Remove the `nvidia-persistenced` requirement

The upstream script checks for:

```bash
/run/nvidia-persistenced
```

and aborts if the NVIDIA persistence daemon is not running.

That check is removed in the working Hyper-V GPU-PV version.

The corresponding Podman bind mount is also removed.

The Hyper-V GPU-PV environment does not require this `nvidia-persistenced` socket for the Nosana container setup.

---

### 14.3 Replace the upstream Docker GPU test

The upstream script tests NVIDIA Docker using:

```bash
docker run --rm --gpus all nvidia/cuda:11.0.3-base-ubuntu18.04 nvidia-smi
```

The working Hyper-V GPU-PV/CDI version instead uses:

```bash
docker run --rm --device nvidia.com/gpu=all ubuntu nvidia-smi
```

This verifies the actual NVIDIA CDI path used by the working configuration.

---

### 14.4 Use NVIDIA CDI inside the Podman container

The upstream Podman command uses:

```text
--gpus=all
```

and mounts:

```text
/run/nvidia-persistenced
```

The working version replaces this with:

```text
--device nvidia.com/gpu=all
```

The NVIDIA GPU is therefore passed into the Podman environment through CDI.

Conceptually:

```text
Docker
  │
  └── NVIDIA CDI
        │
        └── nosana/podman:v1.1.2
              │
              └── --device nvidia.com/gpu=all
                    │
                    └── RTX 5090
```

All other relevant parts of the upstream Nosana Quick-Start script remain unchanged.

---

# 15. Do not connect the Nosana Podman socket to Docker

Do **not** create a symlink such as:

```text
~/.nosana/podman/podman.sock -> /var/run/docker.sock
```

That is not the architecture used by this setup.

Nosana starts its own Podman environment using:

```text
nosana/podman:v1.1.2
```

The Podman socket used by the Nosana node belongs to that environment.

An empty file at:

```text
~/.nosana/podman/podman.sock
```

on an inactive/offline filesystem is not proof that it should be replaced by the Docker socket.

---

# 16. Start Nosana

The modified script can be started normally:

```bash
bash ~/.nosana/nosana-start.sh
```

No additional GPU parameters are required.

The node reads its configuration and wallet from the normal `.nosana` directory.

The startup script itself loads:

```bash
ip_tables
iptable_nat
```

before starting the Nosana environment.

---

# 17. Verify the first Nosana benchmark

After the node starts, verify that Nosana detects the GPU correctly.

The working benchmark reported:

```text
GPU name: NVIDIA GeForce RTX 5090
GPU verified: true
System environment: 7.0.0-34-generic
Package version: 1.1.66
```

The benchmark completed:

```text
4 passed
0 failed
4 optional
```

The reported GPU memory was approximately:

```text
32579 MB
```

The important result is:

```text
NVIDIA GeForce RTX 5090
verified: true
```

---

# 18. Dynamic Memory

Once all of the following have been verified:

* GPU-PV works
* `/dev/dxg` exists
* `nvidia-smi` sees the RTX 5090
* CUDA works
* Docker sees the GPU through CDI
* Nosana starts
* the first Nosana benchmark succeeds

Dynamic Memory can be enabled for the VM if desired.

For initial GPU-PV installation and troubleshooting, keep Dynamic Memory disabled.

---

# 19. Optional GPU power tuning

The RTX 5090 has a higher minimum power-limit setting than the RTX 4080 SUPER.

For example, MSI Afterburner may allow a minimum power limit around:

```text
70%
```

Unlike the RTX 4080 SUPER, this should not be treated as equivalent to a 62% power limit.

An alternative is to use a voltage/frequency curve and flatten the frequency at the desired operating voltage.

For example, the curve can be configured so that the GPU frequency stops increasing substantially above the selected voltage point.

Actual power consumption should always be measured under the real Nosana workload.

Do not assume that a particular curve equals a particular percentage power limit.

---

# 20. What not to change

Once the configuration is working, avoid unnecessary changes to:

* GPU-PV configuration
* NVIDIA driver files installed by `easy-gpu-pv-linux`
* `/dev/dxg`
* NVIDIA CDI configuration
* the Podman GPU device definition
* the Nosana Podman image
* the Nosana wallet
* the Podman socket architecture

In particular, do not replace the Nosana Podman socket with:

```text
/var/run/docker.sock
```

and do not switch the working CDI configuration back to the upstream:

```text
--gpus all
```

without a specific reason.

---

# 21. Known limitation

The GPU-PV verification may report failures for functionality that is not required by the Nosana compute workload, for example:

* NVENC
* Vulkan/Mesa Dozen

These are not blockers for this setup.

The relevant requirements are:

```text
/dev/dxg
        ↓
NVIDIA NVML
        ↓
CUDA
        ↓
Docker + NVIDIA CDI
        ↓
Podman
        ↓
Nosana
        ↓
RTX 5090
```

---

# 22. Final verified configuration

```text
HOST
Windows 11
│
└── Hyper-V
    │
    └── VM: Ubuntu Nosana 5090
        │
        ├── Ubuntu 26.04.1 LTS
        ├── Kernel 7.0.0-34-generic
        ├── Secure Boot OFF
        │
        ├── Hyper-V GPU-PV
        │   └── NVIDIA GeForce RTX 5090
        │
        ├── NVIDIA NVML
        ├── CUDA
        │
        ├── Docker 29.1.3
        │   └── NVIDIA Container Toolkit 1.20.1
        │       └── NVIDIA CDI
        │
        └── Nosana
            └── nosana/podman:v1.1.2
                └── Podman
                    └── NVIDIA CDI
                        └── RTX 5090
```

## Verified result

```text
GPU:              NVIDIA GeForce RTX 5090
VRAM:             ~32 GB
Ubuntu:           26.04.1 LTS
Kernel:           7.0.0-34-generic
Docker:           29.1.3
NVIDIA Toolkit:   1.20.1
Nosana:           1.1.66
Podman image:     nosana/podman:v1.1.2

Nosana benchmark:
4 passed
0 failed
4 optional

GPU verified:     true
```

This configuration provides a working RTX 5090 Nosana node inside a Hyper-V Ubuntu VM using GPU-PV, without requiring a native Linux installation or PCI passthrough.
