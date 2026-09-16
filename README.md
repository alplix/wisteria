# 🌸 Wisteria — FGRP5 at warp speed

A from-scratch modern reimplementation of the Einstein@Home **FGRP5**
(Gamma-ray pulsar search #5) application, packaged as a ready-to-run
BOINC anonymous-platform app. Faster than the stock app, and every fix
is checked line-by-line against the original C source — no hand-waving,
just pulsars found quicker. ⚡

**Coded by Alperen Yavuz.** Binary releases only — source availability
is restricted per the upstream authors' request. GPLv2+ like upstream.

[![Latest release](https://img.shields.io/github/v/release/alplix/wisteria?label=latest&color=9b59b6)](https://github.com/alplix/wisteria/releases/latest)
[![License: GPL v2+](https://img.shields.io/badge/license-GPLv2%2B-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html)
[![Platforms](https://img.shields.io/badge/platforms-Linux%20%7C%20Windows%20%7C%20ARM64-informational)](#-supported-platforms)

---

## 📦 Downloads

**➡️ [Grab the latest release here](https://github.com/alplix/wisteria/releases/latest) ⬅️**

Every version's changelog — and every past release — lives on that
page, so this README stays put while the releases do the talking.

---

## 🖥️ Supported platforms

| Package | CPU / device | Notes |
|---|---|---|
| 🐧 Linux x86-64 | any 64-bit x86 CPU | baseline, static build |
| 🐧 Linux x86-64-v3 | Intel Haswell+ / AMD Excavator+ (2013+) | AVX2/FMA, ~15-30% faster |
| 🐧 Linux aarch64 | 64-bit ARM | Jetson, Raspberry Pi 5, Ampere, etc. |
| 🪟 Windows x86-64 | any 64-bit x64 CPU | baseline |
| 🪟 Windows x86-64-v3 | Intel Haswell+ / AMD Excavator+ (2013+) | AVX2/FMA |

> ⚠️ The **x86-64-v3** builds will *not* run on CPUs without AVX2/FMA —
> they crash with SIGILL (illegal instruction). Not sure about your
> CPU? Pick the baseline build, it always works. BOINC's own plan-class
> matching picks the right one automatically in normal distribution;
> for a manual anonymous-platform install, match it yourself.

---

## 🛠️ Installation

1. 🛑 Stop the BOINC client.
2. 📂 Unpack the release archive for your platform.
3. 📋 Copy `wisteria` (or `wisteria.exe` on Windows), `app_info.xml`,
   and `app_config.xml` into your Einstein@Home project directory:
   - Linux: `/var/lib/boinc-client/projects/einstein.phys.uwm.edu/`
   - Windows: `C:\ProgramData\BOINC\projects\einstein.phys.uwm.edu\`
4. 🔐 On Linux, make it executable and owned by the user your BOINC
   client runs as (usually `boinc`):
   ```bash
   chmod +x wisteria
   chown boinc:boinc wisteria app_info.xml app_config.xml
   ```
   *(not sure who that is? `ps aux | grep boinc` will tell you)*
5. ▶️ Start the BOINC client. That's it — happy crunching! 🎉

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
  pass `--nthreads N` directly.
- **⚡ FFT plan / wisdom** — the app auto-detects a pre-measured FFTW
  plan (`wisteria_fftwf_wisdom.dat`, shipped for Linux x86-64/x86-64-v3
  and Windows x86-64/x86-64-v3) — zero setup, zero wait. Want a plan
  tuned to your *exact* host instead (or you're on aarch64, which ships
  without one)? Run once with `--fftwMeasure 1` — it gets cached next
  to the project files and reused by every later task.
- **📊 Progress reporting** — Linux and aarch64 link the real BOINC API
  and report progress normally. The Windows build links a stub BOINC
  API (the real one can't be cross-compiled here), so Windows tasks
  show no progress percentage in the client — don't worry, it's still
  crunching away happily in the background.

The executable prints a short banner to stderr on startup (GPL notice,
version, and this project's URL) — handy for confirming which build a
task actually ran with. 🔍

---

## 💬 Questions / issues

Feedback is always welcome — especially from ARM board owners and
anyone running long, uninterrupted sessions. When reporting an issue,
please include the **full stderr** of the task in question — it has
everything needed to diagnose the problem. 🕵️

---

<p align="center">✧ May your candidates be strong and your CPUs stay cool ✧</p>
