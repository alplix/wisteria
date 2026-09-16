# Wisteria - FGRP5 at warp speed

A from-scratch modern reimplementation of the Einstein@Home FGRP5
(Gamma-ray pulsar search #5) application, packaged as a ready-to-run
BOINC anonymous-platform app. Faster than the stock app, and every fix
is verified against the original C source.

**Coded by Alperen Yavuz.** Binary releases only - source availability
is restricted per the upstream authors' request. GPLv2+ like upstream.

## Downloads

**[Latest release](https://github.com/alplix/wisteria/releases/latest)**
- changelog for every version, and every past release, lives there.

## Supported platforms

| Package | CPU / device | Notes |
|---|---|---|
| Linux x86-64 | any 64-bit x86 CPU | baseline, static build |
| Linux x86-64-v3 | Intel Haswell+ / AMD Excavator+ (2013+) | AVX2/FMA, ~15-30% faster |
| Linux aarch64 | 64-bit ARM | Jetson, Raspberry Pi 5, Ampere, etc. |
| Windows x86-64 | any 64-bit x64 CPU | baseline |
| Windows x86-64-v3 | Intel Haswell+ / AMD Excavator+ (2013+) | AVX2/FMA |

The x86-64-v3 builds will **not** run on CPUs without AVX2/FMA - they
crash with SIGILL (illegal instruction) - so pick the baseline build if
you are unsure. BOINC's own plan-class matching picks the right one
automatically when the app is distributed normally; for a manual
anonymous-platform install, match it to your own CPU.

## Installation

1. Stop the BOINC client.
2. Unpack the release archive for your platform.
3. Copy `wisteria` (or `wisteria.exe` on Windows), `app_info.xml`, and
   `app_config.xml` into your Einstein@Home project directory:
   - Linux: `/var/lib/boinc-client/projects/einstein.phys.uwm.edu/`
   - Windows: `C:\ProgramData\BOINC\projects\einstein.phys.uwm.edu\`
4. On Linux, make sure the file is executable and owned by the user
   your BOINC client runs as (usually `boinc`):
   ```
   chmod +x wisteria
   chown boinc:boinc wisteria app_info.xml app_config.xml
   ```
   (check the actual user/group with `ps aux | grep boinc` if your
   setup differs)
5. Start the BOINC client.

To go back to the stock app, remove `app_info.xml` (and
`app_config.xml`) from the project directory and restart the client.

## Configuration

Everything ships with sensible defaults - most users don't need to
touch anything beyond the install steps above. A few things worth
knowing:

- **CPU threads**: BOINC assigns 1 CPU by default, so the app runs
  single-threaded. To use more cores on a task, set `<avg_ncpus>` in
  `app_info.xml` (see the packaged `app_config.xml` for an example) or
  pass `--nthreads N` directly.
- **FFT plan / wisdom**: the app looks for a pre-measured FFTW plan
  (`wisteria_fftwf_wisdom.dat`, shipped in the package on Linux
  x86-64/x86-64-v3 and Windows x86-64/x86-64-v3) and uses it
  automatically - no setup needed. If you want a plan measured on your
  own exact host instead (or you're on aarch64, which ships without
  one), run once with `--fftwMeasure 1`; the result is cached next to
  the project files and reused by every later task.
- **Progress reporting**: Linux and aarch64 link the real BOINC API and
  report progress normally. The Windows build links a stub BOINC API
  (the real one can't be cross-compiled here), so Windows tasks show no
  progress percentage in the client - the task is still running
  normally.

The executable prints a short banner to stderr on startup (GPL notice,
version, and this project's URL) - useful for confirming which build a
task actually ran with.

## Questions / issues

Feedback is welcome, especially from ARM board owners and anyone
running long, uninterrupted sessions. Please include the full stderr
of the task in question - it has everything needed to diagnose a
problem.
