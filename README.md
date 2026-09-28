# Nosana on Hyper-V with NVIDIA GPU-PV

This repository documents experiments and working configurations for running **Nosana GPU compute nodes inside Ubuntu virtual machines on Windows 11 Hyper-V**.

The main goal is to make use of an NVIDIA GPU from a Linux VM without requiring a separate physical Linux installation.

The project started with an **RTX 4080 SUPER** and was later extended to a newer **RTX 5090** setup.

---

## Current status

The **RTX 5090 configuration is the newer and currently preferred solution**.

It uses:

* Windows 11
* Hyper-V Generation 2
* Hyper-V GPU-PV
* Ubuntu 26.04.1 LTS
* `easy-gpu-pv-linux`
* NVIDIA CUDA / NVML
* Docker + NVIDIA Container Toolkit
* NVIDIA CDI
* Nosana + Podman

The RTX 5090 configuration has been verified with a working Nosana benchmark and GPU detection.

---

## Documentation

### 🟢 RTX 5090 — current setup

**Recommended starting point**

The latest working configuration for running an RTX 5090 as a Nosana node inside an Ubuntu Hyper-V VM.

➡️ **[RTX 5090 Hyper-V / GPU-PV README](./RTX5090-HyperV/README.md)**

This document contains the complete installation and configuration procedure, including:

* Hyper-V GPU-PV
* `easy-gpu-pv-linux`
* NVIDIA driver / CUDA / NVML
* Docker
* NVIDIA Container Toolkit
* NVIDIA CDI
* Nosana
* Podman
* the required modifications to the Nosana Quick-Start script
* GPU verification
* Nosana benchmark verification

---

### 🟡 RTX 4080 SUPER — previous setup

The original working experiment using an RTX 4080 SUPER.

➡️ **[RTX 4080 SUPER Hyper-V / GPU-PV README](./RTX4080S-HyperV/README.md)**

This documentation describes the earlier GPU-PV configuration and the experiments that led to a working Hyper-V GPU setup.

It is kept as a separate reference because it uses different hardware and was developed before the RTX 5090 configuration.

---

## Why this project exists

Running GPU workloads from Linux normally means using a physical Linux installation or a dedicated Linux machine.

The goal of this project was to investigate whether a Windows 11 machine could remain the primary desktop environment while an Ubuntu VM could use the physical NVIDIA GPU for GPU compute workloads.

The resulting architecture is:

```text
Windows 11
    │
    └── Hyper-V
          │
          └── Ubuntu Linux VM
                │
                └── Hyper-V GPU-PV
                      │
                      └── NVIDIA GPU
                            │
                            └── CUDA / NVML
                                  │
                                  └── Docker
                                        │
                                        └── NVIDIA CDI
                                              │
                                              └── Nosana
```

This makes it possible to keep Windows as the host operating system while using the GPU from Linux applications running inside the VM.

---

## Project history

The project evolved through several different approaches.

The important outcome is not the individual experiments, but the final working **Hyper-V GPU-PV** configuration.

The two documented hardware generations are intentionally kept separate:

```text
RTX 4080 SUPER
    │
    └── RTX4080S-HyperV/
          └── README.md

RTX 5090
    │
    └── RTX5090-HyperV/
          └── README.md
```

The **RTX 5090 documentation represents the newer configuration** and should be used as the starting point for a new installation.

---

## Repository structure

```text
nosana-hyperv-gpu-pv/
│
├── README.md
│
├── RTX4080S-HyperV/
│   └── README.md
│
└── RTX5090-HyperV/
    └── README.md
```

---

## Notes

These documents describe configurations that were actually tested on the corresponding hardware.

Hardware-specific settings should not automatically be copied from one GPU generation to another.

In particular, the RTX 4080 SUPER and RTX 5090 configurations should be treated as separate setups.

For the latest configuration, start here:

**[RTX 5090 Hyper-V / GPU-PV](./RTX5090-HyperV/README.md)**
