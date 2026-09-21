# 🌸 Wisteria — FGRP5 at warp speed

A from-scratch modern reimplementation of the Einstein@Home **FGRP5**
(Gamma-ray pulsar search #5) application, packaged as a ready-to-run
BOINC anonymous-platform app. Faster than the stock app, and every fix
is checked line-by-line against the original C source — no hand-waving,
just pulsars found quicker. ⚡

Runs on **CPUs of almost every kind** (x86-64, ARM, PowerPC, RISC-V; Linux,
Windows, macOS, FreeBSD, Android) and — new in v0.6.0, still experimental —
**on graphics cards** (NVIDIA, AMD, Intel, Apple and more).

**Coded by Alperen Yavuz.** Binary releases only — source availability
is restricted per the upstream authors' request. GPLv2+ like upstream.

[![Latest release](https://img.shields.io/github/v/release/alplix/wisteria?label=latest&color=9b59b6)](https://github.com/alplix/wisteria/releases/latest)
[![License: GPL v2+](https://img.shields.io/badge/license-GPLv2%2B-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html)
[![Platforms](https://img.shields.io/badge/platforms-Linux%20%7C%20Windows%20%7C%20macOS%20%7C%20Android%20%7C%20FreeBSD-informational)](#-supported-platforms)

---

## 📦 Downloads

**➡️ [Grab the latest release here](https://github.com/alplix/wisteria/releases/latest) ⬅️**

Every version's changelog — and every past release — lives on that
page, so this README stays put while the releases do the talking.

**Which file do I take?** The file names tell you:

| starts with | what it is | who it is for |
|---|---|---|
| **`CPU_`** | the normal app, runs on the processor | **everybody** — if you are unsure, take one of these |
| **`GPU_`** | the same app that also uses the graphics card | anyone who wants to try GPU support (experimental) |

---

## 🖥️ Supported platforms

### `CPU_` packages

| Package | CPU / device | Notes |
|---|---|---|
| 🐧 Linux x86-64 | any 64-bit x86 CPU | baseline, static build |
| 🐧 Linux x86-64-v3 | Intel Haswell+ / AMD Excavator+ (2013+) | AVX2/FMA, ~15-30% faster |
| 🐧 Linux aarch64 | 64-bit ARM | Raspberry Pi 4/5, Orange Pi, Jetson, Ampere, etc.; NEON |
| 🐧 Linux armhf | 32-bit ARM with NEON | Raspberry Pi 2+ on a 32-bit OS; tested in an emulator only |
| 🐧 Linux ppc64le | PowerPC64 little-endian, POWER8+ | tested in an emulator only |
| 🐧 Linux riscv64 | 64-bit RISC-V | tested in an emulator only |
| 🍎 macOS arm64 | Apple silicon (M1/M2/M3/M4) | needs nothing installed |
| 🪟 Windows x86-64 | any 64-bit x64 CPU | baseline |
| 🪟 Windows x86-64-v3 | Intel Haswell+ / AMD Excavator+ (2013+) | AVX2/FMA |
| 😈 FreeBSD amd64 | FreeBSD 14 | native, static |
| 🤖 Android aarch64 | phones / tablets, Android 7.0+ | confirmed working on a real device |

> ⚠️ The **x86-64-v3** builds will *not* run on CPUs without AVX2/FMA —
> they crash with SIGILL (illegal instruction). Not sure about your
> CPU? Pick the baseline build, it always works. BOINC's own plan-class
> matching picks the right one automatically in normal distribution;
> for a manual anonymous-platform install, match it yourself.

### `GPU_` packages — new in v0.6.0, experimental

The task is still an ordinary one-core CPU task as far as BOINC is
concerned (no GPU app to configure; credit and validation are unchanged);
the graphics card does the heavy parts while it runs. A production-size
task took 3955 s on the CPU package and 79–108 s with the CUDA package
(GeForce RTX 5070 Ti, one CPU thread). If no usable GPU is found the log
says so and the task simply runs on the CPU.

| Package | Graphics card | Tried on real hardware |
|---|---|---|
| `GPU_…_windows_x86-64_gpu` / `GPU_…_linux_x86-64_gpu` | **NVIDIA**, CUDA, GTX 750 (2014) up to RTX 50 | RTX 5070 Ti, RTX 3050 |
| `GPU_…_windows_x86-64_opencl` / `GPU_…_linux_x86-64_opencl` | **AMD**, **Intel**, NVIDIA and other OpenCL GPUs | RTX 5070 Ti, RTX 3050 (NVIDIA's OpenCL only) |
| `GPU_…_macos_arm64_opencl` | **Apple silicon** GPU | M1 |
| `GPU_…_android_aarch64_opencl` | phone / tablet GPUs (Mali, Adreno, …) | none yet |
| `GPU_…_linux_aarch64_opencl` | ARM boards and servers with an OpenCL GPU | CPU-based OpenCL only |
| `GPU_…_linux_aarch64_gpu` | **NVIDIA Jetson** (JetPack 6), ARM servers with NVIDIA cards | none yet |

Results checked against the stock app on a real work-unit slice: the
follow-up file is byte-identical and the powers agree to better than
4·10⁻⁶. **No GPU-made result has been through the project's validator
yet**, and AMD, Intel, Adreno, Mali and Jetson have not been tried on
real hardware — reports are very welcome (please include the `% GPU …`
lines of the task's stderr). Each GPU package carries its own README with
the details (GPU memory per task: about 1.1–1.4 GB).

---

## 🛠️ Installation

1. 🛑 Stop the BOINC client.
2. 📂 Unpack the release archive for your platform.
3. 📋 Copy **all** the files of the package — `wisteria` (or
   `wisteria.exe` on Windows), `app_info.xml`, `app_config.xml`, and for
   a `GPU_` package also the GPU library that sits next to the program
   (`wisteria_gpu.dll`, `wisteria_ocl.dll`, `libwisteria_ocl.so`,
   `libwisteria_ocl.dylib`, `cufft64_11.dll`, whichever the package has) —
   into your Einstein@Home project directory:
   - Linux: `/var/lib/boinc-client/projects/einstein.phys.uwm.edu/`
   - Windows: `C:\ProgramData\BOINC\projects\einstein.phys.uwm.edu\`
   - macOS: `/Library/Application Support/BOINC Data/projects/einstein.phys.uwm.edu/`
4. 🔐 On Linux, make it executable and owned by the user your BOINC
   client runs as (usually `boinc`):
   ```bash
   chmod +x wisteria
   chown boinc:boinc wisteria app_info.xml app_config.xml
   ```
   *(not sure who that is? `ps aux | grep boinc` will tell you)*
5. ▶️ Start the BOINC client. That's it — happy crunching! 🎉

The package's own `INSTALL.txt` / `README-WIN.txt` (CPU) or `README-…`
file (GPU) has the details for your system, including Android.

**Changed your mind?** Remove `app_info.xml` (and `app_config.xml`)
from the project directory and restart the client to go back to the
stock app.

---

## ⚙️ Configuration

Ships with sensible defaults out of the box — most people never need
to touch anything beyond the install steps above. For the curious:

- **🧵 CPU threads** — BOINC assigns 1 CPU by default, so the app runs
  single-threaded. Want more cores on a task? Set `<avg_ncpus>` in
  `app_info.xml` (see the packaged `app_config.xml` for an example), or
  pass `--nthreads N` directly. One task per core gives the best total
  throughput on most machines.
- **⚡ FFT plan / wisdom** — the app auto-detects a pre-measured FFTW
  plan (`wisteria_fftwf_wisdom.dat`, shipped for Linux x86-64/x86-64-v3,
  Windows x86-64/x86-64-v3, and macOS arm64) — zero setup, zero wait.
  Want a plan tuned to your *exact* host instead (or you're on another
  architecture, which ships without one)? Run once with
  `--fftwMeasure 1` — it gets cached next to the project files and
  reused by every later task.
- **🎮 GPU options** (`GPU_` packages, in `app_config.xml`'s `<cmdline>`):
  `--gpu 3` everything the GPU can do (default there), `--gpu 0` no GPU,
  `--gpuDevice N` pick card N, `--gpuApi 0/1/2` CUDA-then-OpenCL / CUDA
  only / OpenCL only. Limit `<max_concurrent>` so that
  (tasks) × 1.5 GB fits into your card's memory.
- **📊 Progress reporting** — Windows and Linux x86-64/x86-64-v3/aarch64
  link BOINC's real library and report progress, CPU time and checkpoints
  normally (and pause or stop cleanly when the client asks). The other
  builds (macOS, Android, FreeBSD, armhf, ppc64le, riscv64) still link a
  small stand-in instead, so those tasks show no progress percentage in
  the client — don't worry, they are still crunching away in the background.

The executable prints a short banner to stderr on startup (GPL notice,
version, and this project's URL) — handy for confirming which build a
task actually ran with. 🔍

---

## 💬 Questions / issues

Feedback is always welcome — especially from ARM board owners, GPU
testers (AMD, Intel, Jetson, Android phones) and anyone running long,
uninterrupted sessions. When reporting an issue, please include the
**full stderr** of the task in question — it has everything needed to
diagnose the problem. 🕵️

---

<p align="center">✧ May your candidates be strong and your CPUs stay cool ✧</p>
