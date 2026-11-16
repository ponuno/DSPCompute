# DSPMark

An open hardware benchmark for the **Nintendo 3DS XpertTeak DSP**, built to answer one question:
*is the DSP worth using as a math / neural coprocessor for a PICA-SR-style upscaler?*

DSPMark loads a **custom Teak component** through the DSP service, drives it over a shared-memory
mailbox, verifies every result against an ARM11 reference, and exports CSV/JSON for analysis. It runs
on New Nintendo 3DS (and, on a best-effort basis, Old 3DS), builds with devkitARM + libctru, and is
delivered as `DSPMark.3dsx`.

The project deliberately separates **what was measured on hardware** from **what was validated in an
emulator** and **what is only theoretically possible**. No invented performance numbers are included.

---

## Status at a glance

| Phase | Scope | Status |
|------|-------|--------|
| 1 | Custom DSP component: loading, mailbox, timeouts, unload, audio restore | **Implemented and hardware-confirmed** (v15: load, HOME, clean exit, audio handed back) |
| 2 | ADD16/SUB16 (plain, saturating, unrolled), MUL16, MAC16, DOT32, Q15, ACC32/ACC40, SHIFT16/NORM, REG_MPY, MAC2, dual-multiply DOT32/MAC16 | **Implemented; the scalar set is hardware-verified (84/84). The dual-multiply instruction is fast (8-10 ns/MAC) but still wrong on hardware; v8 adds a probe that isolates why** |
| 3 | DSP-local memory read/write/copy/random, ARM11 FCRAM reference, block sizes | **Implemented and measured**; DMA per-channel path is *not* in this build (see below) |
| 4 | Conv2D 3×3/5×5, depthwise, pointwise, dense, ReLU, pixel shuffle, SR-LUT, tiny CNN | **ARM11 side implemented and verified. DSP side: ReLU + SR-LUT verified on hardware, other Teak kernels scheduled for a later revision** |
| 5 | Hybrid GPU pipeline (render, readback, DMA, inference, compose) | **Not implemented** (documented, deliberately no numbers) |
| 6 | Stability (10 s, 1, 5, 30 min), error/timeout counting, cancel, recovery | **Implemented** |

The honest short version: **the DSP transport and kernels work, are verified word-for-word on
hardware (84/84 records, 0 mismatches), and the answer to the throughput question is still negative.**
On a New3DS at 804 MHz + L2 the ARM11 executes the same int16 kernels 2.7–9× faster per operation,
even after unrolling the DSP loops to their instruction-issue ceiling (~40 MMAC/s), and a mailbox
round trip adds ~75 ns per transferred word on top. Details and the full table are in
`BENCHMARKS.md` (section 10) and `results/new3ds-804mhz-2026-11-16-v02.json`.

What remains open is the *offloading* question (does moving work off the ARM11 free frame time even
when the DSP is slower in isolation), which Phase 9 has to answer.

**Component lifecycle is now solved.** The pieces that made it work are recorded in
`HARDWARE_NOTES.md` item 10: the component must set up the MIU paging exactly like OpenDSP's `crt0`,
answer the `REPLY0/1/2` boot handshake, raise `PSEM = 0x8000` when its pipe table is ready, and be
loaded with `flags=1`. With that in place the DSP loads, survives HOME, and is handed back to the
system on exit — i.e. a DSPCompute library can behave like `ndsp` instead of requiring a reboot.

The app marks missing backends as `unavailable` instead of guessing.

---

## What is verified, and how

Two independent layers of verification exist:

1. **Host emulator (Teakra).** `tools/dspsim --verify` executes the exact shipped component, drives
   it through the same mailbox protocol the ARM11 side uses, and compares every result word-for-word
   with a C reference. Current status: **49 checks pass, 0 fail**; the MAC2 record still reports
   *experimental* until a hardware run verifies both accumulators. This proves the encodings and the
   kernel logic, **not** hardware speed.
2. **On-console verification.** Every benchmark re-computes the result with the ARM11 reference and
   stores `verified`/`MISMATCH` in the output. A wrong DSP result can never silently become a
   performance number; it becomes a failed record.

The v0.2 suite adds kernel variants (unrolled loops, register-fed multiply, 32/40-bit accumulation,
Q15 multiply, shifts and block normalisation), and measures the DSP's true dual-MAC path separately as
an **experimental** record. Its findings are written up in `docs/PRIOR_ART.md` (what was reused from
Teakra and OpenDSP) and `BENCHMARKS.md`.

The emulator is never used to produce performance claims. Its "cycles" are Teakra's timing model and
are not comparable to the DSP clock.

---

## Build

Requirements:

* devkitPro with devkitARM and libctru (`/opt/devkitpro` or `C:\devkitPro`)
* `make` from devkitPro's msys2
* Python 3 (only to rebuild the DSP component)

```sh
make            # builds DSPMark.3dsx (uses the committed data/dspmark_cdc.bin)
make dsp        # optional: rebuild the DSP component from dsp/*.asm
make clean
```

Host tooling (optional, needs Visual Studio + LLVM/clang on Windows):

```bat
tools\build_host_tools.bat
```

The research checkouts are not vendored in git. Fetch them first:

```sh
git clone --depth 1 https://github.com/wwylele/teakra third_party/teakra
git clone --depth 1 https://github.com/Cellenseres/OpenDSP third_party/OpenDSP
```

This produces `third_party/teakra/build_host/{teaasm,makedsp1,dsp1_reader,dspsim,teacan}.exe`.
`dspsim` runs the component in the emulator and performs the correctness suite:

```sh
third_party/teakra/build_host/dspsim.exe --image data/dspmark_cdc.bin --verify
```

Rebuilding the DSP image needs `teaasm` from that host build:

```sh
python tools/asm.py dsp/dspmark.asm -o data/dspmark_cdc.bin --listing build/dspmark.lst
```

---

## Running on hardware

1. Copy `DSPMark.3dsx` to `sdmc:/3ds/DSPMark.3dsx`.
2. Launch it through the Homebrew Launcher on a console with **Luma3DS** (or another CFW that patches
   the DSP component signature check and maps DSP RAM into homebrew — see `HARDWARE_NOTES.md`).
3. The top screen shows the platform and then the DSP load outcome. If the component is refused, the
   `Result` code and the reason are printed and stored in `sdmc:/3ds/DSPMark/DSPMark.log`.
4. If startup ever stops at "Starting DSP service", hold **L** while launching: safe mode skips the
   DSP entirely and the ARM11-only suites still run. The load runs on a worker thread with a 4 s
   watchdog, so a blocked IPC degrades to ARM11-only mode instead of freezing the UI.
5. Run the suites, then **Export CSV + JSON** → `sdmc:/3ds/DSPMark/<timestamp>.{csv,json}`.
6. Quit normally: DSPMark unloads its component and restores stock `ndsp` audio. If `ndspInit` fails,
   the app tells you to place `dspfirm.cdc` in `sdmc:/3ds/`.

There is no overclocking, no voltage change, no NAND modification. The optional New3DS toggle only
requests the standard 804 MHz + L2 performance profile via `osSetSpeedupEnable`.

---

## Reading the output

A record is a row in CSV/JSON with, among others:

| Field | Meaning |
|-------|---------|
| `backend` | `dsp`, `arm-c`, `arm-vfp`, `arm-mt` (threaded), `gpu` |
| `latency_ns` | one round trip (DSP: mailbox command incl. polling) |
| `compute_ns` | per-iteration cost from the linear fit `T(K) = overhead + K·compute`, K ∈ {1,4,16} |
| `overhead_ns` | fitted fixed cost (transport + dispatch) |
| `ns_per_op` | `compute_ns / ops` (ns per MAC or per element) |
| `verified` | result matched the ARM11 reference |
| `experimental` | implemented but not hardware-confirmed (e.g. MAC2) |
| `available` | false when the backend/kernel is not present in this build |

The linear fit is the key to separating **compute** from **command transport**: each DSP command is
executed `K` times inside the firmware, so the slope is pure kernel time and the intercept is the
mailbox round trip.

Analyse exports with:

```sh
python tools/analyze_results.py results/*.json
python tools/analyze_results.py results/*.csv --csv-out summary.csv --plot dsp-vs-arm.png
```

---

## Repository layout

```
dsp/                 Teak firmware (boot, mailbox, arithmetic, neural module)
data/dspmark_cdc.bin Embedded DSP component (built from dsp/*.asm)
source/              ARM11 application (transport, benchmarks, neural, export)
tools/               assembler, emulator harness, verifier, analysis, host build
third_party/         teakra (MIT), OpenDSP (BSD-2) — research references and host tools
docs/                generated ISA reference (`DSP_ISA_FORMS.txt`), prior-art survey (`PRIOR_ART.md`)
results/             hardware exports (JSON/CSV/samples) with provenance
tests/dsp/           instruction-semantics probes used to pin down the ISA
BENCHMARKS.md        methodology, schema, current measurement status
HARDWARE_NOTES.md    findings, constraints, CFW requirements, toolchain quirks
```

`third_party/` is used for research and host tooling. The shipped firmware and app are original code;
no Nintendo firmware is included.

---

## Why some phases are not finished

* **Neural Teak kernels.** Four independent details of the XpertTeak ISA (operand order in Teakra's
  parser, `BKREP` executing `LC+1` times, 16-bit-only immediate compares, and the fact that
  two-memory `mpy`/`mac` forms multiply X by itself on real hardware) had to be established
  empirically before any kernel could be trusted. The arithmetic set passes on hardware; the first
  neural implementations did not, so they were pulled rather than shipped. `HARDWARE_NOTES.md`
  records exactly where they failed and what the next step is.
* **Phase 5.** A credible hybrid GPU benchmark needs the DSP convolution path from Phase 4, a
  render-to-texture readback pipeline and an external PICA-SR reference. Presenting a partial number
  as "PICA-SR vs DSP" would be misleading, so the menu explains the gap instead.
* **DMA.** The register sequence (from OpenDSP's hardware-tested path) is documented, but the
  implementation is not in this build; Phase 3 therefore measures DSP-local memory and ARM11 FCRAM,
  not DSP DMA. The CSV makes this explicit.

---

## Deliverables checklist

| Requested item | Where |
|----------------|-------|
| `DSPMark.3dsx` | build output (`make`) |
| Full source code | `source/`, `dsp/` |
| DSP assembly | `dsp/dspmark.asm`, `dsp/dspmark_kernels.asm` |
| Automated build | `Makefile`, `make dsp`, `tools/build_host_tools.bat` |
| `README.md` | this file |
| `BENCHMARKS.md` | methodology, schema, status |
| `HARDWARE_NOTES.md` | platform constraints, CFW, ISA findings |
| `LICENSE` | MIT + third-party notices |
| CSV/JSON export | on-console export to `sdmc:/3ds/DSPMark/` |
| Python analysis tool | `tools/analyze_results.py` |
| GPU hybrid benchmark | deferred in v0.1, explained in the app and docs |

---

## License

DSPMark is MIT licensed (`LICENSE`). Third-party components keep their own licenses:
Teakra (MIT), OpenDSP (BSD-2-Clause), libctru (zlib), citro3d (zlib). No GPL code was copied;
Luma3DS was read only to confirm the CFW requirements.
