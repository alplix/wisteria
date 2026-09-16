# Wisteria - FGRP5 at warp speed

A from-scratch modern reimplementation of the Einstein@Home FGRP5
gamma-ray pulsar search, packaged as ready-to-run BOINC anonymous-platform
apps for **Linux x86-64, Linux x86-64-v3 (AVX2/FMA), Linux aarch64 and
Windows x64**.

**Coded by Alperen Yavuz.** Binary releases only - source availability is
restricted per the upstream authors' request. GPLv2+ like upstream.

## Downloads

- **[Latest release](https://github.com/alplix/wisteria-fgrp5/releases/latest)** - v0.5.1
  - Linux x86-64 tarball: static build, baseline ISA (runs on any 64-bit x86 CPU)
  - Linux x86-64 v3 tarball: AVX2/FMA optimized static build (2013+ CPUs, ~15-30% faster)
  - Linux aarch64 tarball: Jetson, Raspberry Pi 5, Ampere
  - Windows x64 baseline zip: self-contained executable, runs on any x64 CPU
  - Windows x64 v3 zip: AVX2/FMA build (2013+ CPUs, ~15-30% faster)

### v0.5.1 (current) - bundled FFT wisdom, no wait for most users

The single biggest remaining per-step cost was the FFT itself, and the
"MEASURE" plan that makes it ~4x faster took a one-off 8-9 minute
measurement to find - so it stayed opt-in (`--fftwMeasure 1`). Wisteria
now tries a matching plan from a wisdom file **first, regardless of
`--fftwMeasure`**, since looking one up costs nothing when it fails: the
default path benefits for free whenever a plan is available.

**The Linux x86-64/x86-64-v3 and Windows x86-64/x86-64-v3 packages now
ship that plan, pre-measured on real hardware at the real work-unit FFT
size.** Verified on all four: default mode with zero extra flags finds
the bundled plan and produces a toplist that is bit-identical (Linux) or
hash-identical (Windows) to an explicit `--fftwMeasure 1` run - including
an end-to-end check that extracts the packaged archive fresh and runs it
exactly as BOINC would, no flags at all. Per-step FFT time on those four
platforms drops from an ~8-9 minute one-time measurement to about 0.5 s,
with **no wait at all**.

The aarch64 package does not include a bundled plan yet - no real ARM
hardware was available to measure one on, and a plan measured under QEMU
emulation would not reflect real timings (it could even pick a worse plan
than plain estimate). ARM users get the same win by running
`--fftwMeasure 1` once; the measured plan is then cached as wisdom and
reused by every later task on that host.

`app_info.xml` bumped to **151**.

### v0.5.0 - ~3x faster, and a critical fix for --skyRadius work units

**The most important fix first: `--skyRadius` was searching the wrong sky
positions.** Real FGRP5 work units send `--skyRadius` together with
`--firstSkyPoint`/`--numSkyPoints` (for example: a 535,504-point grid,
search points 120,771-120,791). Every version through v0.4.0 approximated
that grid with four rough rings and **ignored**
`--firstSkyPoint`/`--numSkyPoints` entirely - so every such task searched
sky positions that had nothing to do with the work unit. The sky grid is
now a line-by-line port of the upstream C `generate_skygrid()` - same
points, same order, same 0-based slice - checked against the unmodified
upstream C function on a real work unit: byte-identical on the
non-FMA Linux build, within 1 ULP of a single value on the AVX2/FMA,
Windows and ARM builds (ordinary floating-point rounding noise, not a
difference in which points get searched). `--writeskygrid 1` now writes
the searched points to `<output>.scsg`, as the stock app does.

**~3x faster on one core.** Photon pairs are bucket-sorted by FFT bin and
accumulated straight into the FFT input with vectorized phase and
lag-domain-interpolation kernels, replacing a 256 MB per-thread scratch
buffer that used to be rebuilt on every f1dot step. Real work unit, one
thread, per f1dot step: ~4.0 s -> ~1.5 s with the default FFT plan (a full
task ~2.5 h -> ~55 min). Results agree with v0.4.0 to 1.4e-7 in power,
same candidates.

**Optional cached FFT plan.** FFTW's better ("MEASURE") plan makes the FFT
itself another ~2x faster but takes several minutes to find. It stays off
by default; `--fftwMeasure 1` measures it once per host and caches it as
FFTW wisdom next to the project files, so only the very first task on a
host pays that cost (`-y/--wisdomfile`, `-L/--wisdTimeLimit`).

**Windows and ARM FFT was running fully scalar.** The FFTW library those
builds linked through v0.4.0 had no SIMD code at all - not SSE2, not AVX,
not NEON. Rebuilt with SSE2/AVX/AVX2 for Windows and NEON for ARM (FFTW
picks the right one at run time, so the baseline Windows build still runs
on SSE2-only CPUs). Windows, real work unit: FFT 3.18 s -> 1.73 s per step.

**Fix:** the heterodyne phase computation overflowed a 32-bit integer
above roughly 1 kHz.

All five packages rebuilt and verified against each other (same top
candidate on the demo case, `S=32.41130`) and, for the sky grid, against
the unmodified upstream C function on a real work unit. `app_info.xml`
bumped to **150**.

Known limitations: the Windows builds still link a stub BOINC API, so
Windows tasks show no progress in the client; `--bskyPointFile` (binary
sky-point files) is not supported.

### v0.4.0 - five production bugs fixed, CPU/RAM behaviour corrected

A full line-by-line review of the port against the upstream C source found
five bugs that broke real tasks, plus a set of latent memory-safety issues.

**1. Output files never reached the project directory.** `<result_name>_0`
and `_1` in a BOINC slot are not the output files - they are BOINC
`<soft_link>` indirection stubs pointing at the real files in the project
directory. Wisteria wrote over the stubs, so the upload file was never
created. That is why @baracutio's task 2018679709 logged
`Computation ... finished` and then `Output file ... absent` for both
files. Wisteria now writes `<result_name>_0.out` locally and moves the
finished files onto the resolved upload paths, exactly like the stock app
(`hs_boinc_extras.c`: *"don't overwrite original XML softlink"*).

**2. Wrong result file format.** Under BOINC the stock app does not write
text: each value is a little-endian float64 XORed with a fixed key
(`writeobs()`). That is what the server-side validator parses, so a
plain-text result could not have validated even when the task succeeded.
`--textOutput 1` keeps readable output for local runs.

**3. Corrupted spectrum with `--ldiBins > 0`.** In the lag-domain
interpolation path a cosine term overwrote the phase factor that was still
in use, so the real part of every FFT input was wrong. Real FGRP5 work
units use `--ldiBins 15`, so this affected every production task. On the
regression case the reported peak moved from 700.4909 Hz to 700.4650 Hz -
the LDI path now agrees with the non-LDI path to all digits.

**4. `--useWeights` was inverted.** The stock app *ignores* the
probability column unless `--useWeights` is non-zero, in which case it also
drops photons below `--scutoff`. Wisteria always used the column.
`--scutoff` is now supported.

**5. 32-bit overflow in the coherent follow-up.** Phase reduction used a
default `INTEGER(4)` rounding intrinsic on arguments of order 4e10, which
is undefined; the `.cohfu` output could be meaningless.

Also fixed: three out-of-bounds array reads (Fortran's `.AND.` does not
short-circuit), an out-of-bounds string read in the number parser, a
silently dropped Shapiro delay when the ephemeris lacked `GMS`, several
ephemeris-reader failure paths that produced garbage instead of an error,
diagnostics that never printed, and memory leaks in the ephemeris loader.

**CPU budget.** Wisteria now reads `<ncpus>` from the slot's
`init_data.xml` and sizes its thread pool accordingly. By default BOINC
assigns 1 CPU, so the app runs single-threaded and reported CPU time
matches run time - no more `Run time 1,646 / CPU time 6,123`. To use more
cores, set `<avg_ncpus>` in `app_info.xml` and/or pass `--nthreads N`
(thanks @Ian&Steve C. for the suggestion, and @baracutio for the report).

**Memory.** The per-thread interpolation buffer is allocated once instead
of on every f1dot step, and with a 1-CPU budget only one such buffer
exists. stderr now prints a one-line memory summary so peaks can be
attributed.

**Reproducibility.** FFT plans use `FFTW_ESTIMATE`. `FFTW_MEASURE`
benchmarks codelets at plan time and its choice varies between runs, so the
same binary produced different last digits for the same work unit; two runs
are now bit-identical. `--fftwMeasure 1` restores the old behaviour.

**Live progress.** gfortran block-buffers stderr once BOINC redirects it to
a file, which is why @toggleton saw a completely empty `stderr.txt` while a
task was running. Progress lines are now flushed as they are produced.

All five packages were rebuilt and cross-checked: same parameters give
f0 = 700.460174561 on every platform, with S agreeing to 1e-5 (the expected
FMA difference between the baseline, AVX2 and ARM builds). `app_info.xml`
bumped to **140**.

Known limitations: the Windows builds still link a stub BOINC API, so
Windows tasks show no progress in the client; `--skyRadius` uses a simpler
sky grid than the stock app and `--bskyPointFile` is not supported.

### v0.3.7 - hotfix: diagnostics now go to stderr

v0.3.6's diagnostics were written to **stdout**; the BOINC client only
captures **stderr**, so in BOINC the log showed just the banner, the
correct output names and a bare `STOP 1` - the new "bad LAL ephemeris
header" message and the file dump never appeared (confirmed by @baracutio's
v0.3.6 stderr on task `LATeah2223F_872.0_258069_0.0_0`).

v0.3.7 sends **every** progress and error line to stderr, so the ephemeris
diagnostics are now visible in the task log. With a non-FITS ephemeris the
next failing stderr should show:

```
% loading ephemeris: JPLEPH.405
% not JPL FITS; loading as LAL text: JPLEPH.405
ERROR: bad LAL ephemeris header, first token of:
> <the actual first non-# line>
file size: ...
first bytes (hex): ...
first bytes (text): ...
```

If you saw the bare `STOP 1` with v0.3.6, install v0.3.7 and paste the full
stderr of one failing task - that message identifies the real ephemeris
file format. `app_info.xml` bumped to **133**.

### v0.3.6 - ephemeris diagnostics ("Bad real number in item 1")

v0.3.5 fixed the output naming (confirmed by @toggleton and @baracutio: the
`_0` and `_1` outputs now appear under exactly the names BOINC expects) but
a second deterministic crash surfaced in the **ephemeris** loader:

```
At line 252 of file fgrp5_support.f90
Fortran runtime error: Bad real number in item 1 of list input
```

That is the generic **LAL-text** ephemeris parser being handed a file that
is not a JPL FITS file (a FITS file starts with `SIMPLE  =`). In the BOINC
slot the file referenced as `--ephemdir JPLEPH.405` turns out not to be the
standard FITS ephemeris, so detection falls through and the text parser
trips over the first non-numeric line, exiting with code 2.

**v0.3.6 turns that obscure crash into a real diagnostic:** the app now
prints the offending header line, the file size and the first 128 bytes of
the file (hex + as text) and stops with a clear
`ERROR: bad LAL ephemeris header` message - so the next stderr shows
exactly what that ephemeris file really is. Genuine LAL text tables and
JPL FITS files keep working unchanged. `app_info.xml` version bumped to
**132**.

### v0.3.5 - the real BOINC fix

Modern FGRP5 workunits from Einstein@Home no longer send `-o/--outputfile`
on the command line (newer workunit generator; the client_state.xml command
lines now end in `--debug 0 --debugCommandLineMangling` with no `-o`).
Every earlier build required `-o`, so at startup they printed an
`argc/argv` dump and exited before producing anything - and BOINC reported
**"Output file ... absent"** for every single task. That argv dump is
exactly what showed up in @baracutio's and @toggleton's stderr.

**Root cause found and fixed.** Wisteria now reads the result name from
`init_data.xml` (written by BOINC into the slot directory) and writes
exactly the files BOINC expects:

- `<result_name>_0` - the toplist,
- `<result_name>_1` - the coherent follow-up.

Verified with a BOINC-style command line (no `-o`, bare `--inputfile` and
`--ephemdir JPLEPH.405`, `--debugCommandLineMangling` flag): both output
files are created under the exact names the validator looks for, RC=0.
Standalone runs with `-o` keep working unchanged. `app_info.xml` version
bumped to **131** so BOINC picks up the new binaries.

### v0.3.4 - the semicoherent crash fix

The big one. Every real multi-sky-point BOINC task (a typical FGRP5 search
uses **12 sky points**) was dying with **"Output file ... absent"** right after
the first sky point. Root cause: the module-level pair arrays inside
`setup_pairs()` were allocated again on the 2nd sky point without being freed
first, raising a Fortran runtime **"already allocated"** error that terminated
the app (exit code 2). The arrays are now released before re-allocation.

Verified with the official BOINC command line and explicit multi-sky-point
runs: **12/12 sky points complete, RC=0, output file produced**. Single-sky
and `--ephemdir` tests still pass unchanged.

### v0.3.3 - BOINC app_info schema fix + --ephemdir directory support

Two follow-up fixes from the Einstein@Home forum (reported by @baracutio
and @toggleton):

- **app_info.xml `<file_name>` → `<name>`**: the `<file_info>` block must
  use `<name>wisteria</name>` (or `wisteria.exe`); the previous
  `<file_name>` tag is only valid inside `<file_ref>` and made BOINC report
  "missing application file". This was identified by @baracutio's own
  modified app_info.xml which worked correctly.
- **`--ephemdir` now accepts a directory**: BOINC passes
  `--ephemdir .../einstein.phys.uwm.edu/JPLEPH` which is a directory, not a
  file. Wisteria now probes the plain path first, then searches well-known
  ephemeris file names (`JPLEPH.405`, `lnxp1600p1658.405`, `DE430.dat`, ...)
  inside the directory before giving up.

All five packages rebuilt; same top candidate `f0=12.3457260` verified.

### v0.3.2 - BOINC CLI parser + app_config fixes

Reported on the Einstein@Home forum right after v0.3.1: the app received a
`STOP 1`/`process exited with code 1` from BOINC before doing any work, and
BOINC sometimes reported "Not requesting tasks: don't need (no
applications)". Three root causes found and fixed:

- **`app_config.xml` was invalid**: `<fraction_done_xml>1</fraction_done_xml>`
  is not a valid tag and an unknown `<options>` block made BOINC reject the
  file. It now uses the correct `<fraction_done_exact/>` and nothing else.
- **`app_info.xml` was missing the `<platform>` tag**. Without it BOINC did
  not properly match the app to workunits, so tasks either never arrived or
  the app was launched with no usable arguments (immediate `STOP 1`). All
  packages now ship the correct platform tag for their architecture.
- **The CLI parser was rebuilt from scratch**: it now handles the full
  official FGRP5 command line exactly as BOINC delivers it in
  anonymous-platform mode - split args, long flags, negative-number values
  (e.g. `--f1dot -1e-13`), and even the entire command line arriving as a
  single argv element. If required arguments are still missing at startup,
  the app prints the exact `argc/argv` it received so any remaining issue
  can be reported precisely.

Direct command line still works unchanged on every platform.

### Why are there two versions (Linux and Windows)?

Both the Linux and Windows releases come in a baseline and an x86-64-v3
(AVX2/FMA) variant - the same split the stock Einstein@Home app uses
(SSE2 vs AVX builds):

- **x86-64 (baseline)** - the safest choice. Runs on every 64-bit x86 CPU since 2003,
  including very old or low-end machines. Pick this if you don't know your CPU.
- **x86-64-v3 (AVX2/FMA)** - compiled for the x86-64-v3 instruction set (Intel Haswell+,
  AMD Excavator+/2013+). Clearly faster on supported CPUs: roughly 15-30% shorter
  runtimes thanks to the 256-bit SIMD FFT path and fused multiply-add.

The v3 build will **not** run on CPUs without AVX2/FMA - it crashes with SIGILL
(illegal instruction), which is why the baseline is kept. BOINC picks the correct
plan class per machine automatically; for manual installs, match your CPU.

The remaining package is single-variant: **aarch64** for ARM64 (Jetson,
Raspberry Pi 5, Ampere).

Every archive includes `app_info.xml` (with the confirmed
`<plan_class>FGRPSSE</plan_class>`) and `app_config.xml` - unpack into your
BOINC project folder, restart the client, done.

The executable is named `wisteria` (Linux) / `wisteria.exe` (Windows). On
start it prints a stderr banner with the GPL v2 notice, app name/version,
author (Alperen Yavuz) and project URL (github.com/alplix/wisteria-fgrp5).

## What changed in v0.3.1

Code-review pass over the full Fortran pipeline, 6 fixes applied and all
packages rebuilt/verified:

- Coherent follow-up: added the missing `- mjd_ref_frac` subtraction
  (consistent with the semicoherent stage; prevented a shift with
  fractional-day `--reftime`)
- Switched the monotonic clock to `system_clock` (no negative durations
  across midnight)
- Fixed `bfraction_done` normalization so progress reaches 1.0 on the last
  sky point
- Checkpoint `read`: clamped to the toplist size (no overflow on corrupt
  files); `write`: fixed Windows `rename()` failure by deleting the old
  checkpoint first (no stale `.tmp` residue)
- Removed a dead-statistic expression

## Verified

- Top candidate `f0=12.3457260` **identical on all four platforms**
- DE430 ephemeris, Windows 11 & Linux: `S=29.73155 P=47.64821`, top candidate
  bit-exact; lower toplist ranks match to 1e-5 (FMA rounding noise)
- `SHA256SUMS` checked with `sha256sum -c`

## This build is for you if...

- you want faster FGRP5 CPU tasks on any machine, old or new
- you run ARM boards (Jetson / RPi 5) that the stock app barely supports
- you like watching a task finish before your coffee does :)

## Skip it if...

- you only chase credit - validate your expectations against stock first.

Feedback very welcome - especially from ARM board owners and anyone running
long uninterrupted sessions.
