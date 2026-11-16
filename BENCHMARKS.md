# DSPMark benchmark methodology

This document defines what each measurement means, how it is taken, and what is deliberately not
claimed. It is the contract between the numbers in a result file and the hardware that produced them.

## 1. Provenance rules

Every result file states its provenance:

* `hardware` — produced by `DSPMark.3dsx` on a console. Only these are performance data.
* `emulator` — produced by `tools/dspsim` (Teakra). Correctness only; its cycle model is not the DSP
  clock and must never be compared with hardware numbers.
* `theoretical` — calculations such as MACs per instruction. Always labelled as such.

No figure in this repository is a hardware measurement until a real export exists. The app marks
unimplemented backends `available=false`; the analyser prints `unavailable` instead of zero.

## 2. Timing

* Clock source: `svcGetSystemTick` (`SYSCLOCK_ARM11`, 268 MHz). It does not change with the CPU clock
  mode, so tick-to-ns conversion stays valid at 268 and 804 MHz.
* Timer overhead is measured (median of empty start/stop pairs) and exported as
  `timer_overhead_ticks`; sub-microsecond records must be read against it.
* Warm-up: each case runs once before samples are taken.
* Repetition: **21 samples per repeat point** (K = 1, 4, 16). Statistics are computed **per repeat
  point**, never over a mixture of batch sizes: min, median, P95 and P99. The headline fields
  (`latency_ns`, `ns_min`, `ns_p95`, `ns_p99`) always describe one round trip, i.e. the K=1 samples;
  `median_k4_ns` / `median_k16_ns` expose the batch totals and `compute_ns`/`overhead_ns` come from the
  fit below. Every raw sample is exported (JSON `samples_ns`, plus a separate `-samples.csv`), so the
  statistics can be recomputed offline.
* Repeats: every DSP command carries a repeat count. The firmware executes the kernel `K` times per
  command, with K ∈ {1, 4, 16}. A least-squares fit `T(K) = overhead + K·compute` separates kernel
  time from mailbox transport time. `compute_ns` is the slope; `overhead_ns` is the intercept.
  This is how "pure compute" is isolated without trusting a DSP-side timer.
* Separated measurements, each with its own record:
  * **pure compute** — `compute_ns` from the fit (DSP) or the kernel call (ARM);
  * **command submission** — `SUBMIT`, a command that does no work (`DSPM_CMD_NOP`);
  * **input transfer** — `TRANSFER_WRITE`, uncached ARM11 stores into DSP RAM;
  * **output transfer** — `TRANSFER_READ`, uncached ARM11 loads from DSP RAM;
  * **end-to-end** — `MAC16_E2E`, which writes the inputs, submits one command and reads the results
    back, i.e. the latency a real pipeline pays rather than the fitted slope.
* ARM baselines are the same kernels compiled by devkitARM `-O2` from the same int16 data, which is
  already resident in FCRAM; results are stored through a `volatile` sink so nothing is optimised
  away. Compiler and flags are recorded in the JSON `method` block.

## 3. Phase 1 — DSP foundation

Measured and reported:

* `DSP_LoadComponent` result and the `loaded` flag (the console's answer, not an assumption).
* Boot handshake latency: time from load to the firmware banner (`magic 0xD5A1`, version).
* Mailbox ping round trip: 64 echoes, median/min/p95, explicitly described as *including ARM polling*.
* ADD16 correctness on 256 words against the ARM reference.
* Unload/restore: the app unloads the component and starts stock `ndsp`; failure is reported with the
  `ndspInit` result code.
* Stability runs exercise recovery: a timeout triggers one component reload and the counters record
  it.

Not measured here: APBP register transfers via `DSP_SendData`, and DSP pipes (`DSP_WriteProcessPipe`).
The firmware implements the pipe-table handshake required for a successful load but the benchmark
uses the shared mailbox, which is the lower-overhead path for bulk work.

## 4. Phase 2 — arithmetic

All vectors are int16, results are checked word-for-word.

| Test | Kernel | Notes |
|------|--------|-------|
| `ADD16` | wrapping 16-bit add | low 16 bits only |
| `ADD16_U4` / `ADD16_U8` | same, 4 / 8 elements per loop iteration | tail falls back to `ADD16` |
| `ADD16_SAT` | saturating add | explicit clamp, branchy |
| `SUB16`, `SUB16_SAT`, `SUB16_U4` | wrapping / saturating subtract, plus unrolled | |
| `MUL16_LO` / `MUL16_HI` | 16×16→32 multiply | low / high half (Q15) |
| `MUL16_U2_LO` / `_HI` | same, two elements per iteration | idiom-7 loop only |
| `MAC16` / `MAC16_U4` / `MAC16_DUAL` | rows × vector dot products | int32 accumulate; `_U4` uses the unrolled dot, `_DUAL` the dual-multiply dot |
| `DOT32` / `DOT32_U4` | single dot product | 1 vs 4 MACs per loop iteration |
| `DOT32_DUAL` | dot with the compound dual-multiply instruction | **2 signed products + 2 accumulations per instruction** (`xy<- ... acc+p0+p1 a0 mpysxsy mpysxsy`) |
| `Q15_DOT` | dot then `>>15` + 32-bit clamp | fixed-point path |
| `Q15_MUL` | `sat16((a*b) >> 15)` elementwise | fixed-point multiply |
| `ACC32` / `ACC40` | dot with explicit 32-bit wrap vs the native 40-bit accumulator | same inputs, different accumulator width |
| `SHIFT16` | `dst[i] = src[i] << SV` | SV from the parameter block |
| `NORM` | block normalisation to int16 full scale | returns the shift, data shifted by `shfc` |
| `REG_MPY` | register-fed multiply (`mpy y0,r6` + `add p*`) | no memory operand: arithmetic ceiling |
| `SAT_TEST` | saturation probe | validates clamp behaviour |
| `MAC2` | two interleaved MAC chains | **experimental** until a hardware run returns correct values for both accumulators |

Every record carries the variant it measured (`ADD16_U4`, `REG_MPY`, ...), and the ARM11 C baseline is
recorded under the same test name, so the analyser pairs them automatically.

The DSP's multiply semantics are **calibrated on the console at startup, not assumed**. Eight probe
idioms run over the known dot `(2,3,4,5)·(3,5,7,9) = 94`; the first idiom that returns 94 is selected
(`SET_IDIOM`) and used by the shared dot routine and `MUL16` for the rest of the session. The choice is
logged and exported as `mul_idiom`. On the New3DS hardware the winner is idiom 7: load Y into the
multiplier register, `mpy y0,[r0],a1`, then `add p*,a0`. Every two-memory form (`mpy/mac [r4] [r0]`)
returns 54 on hardware because the core multiplies X by itself, so the Y operand is not read. Teakra
aliases X and Y memory and therefore cannot reproduce that; see `HARDWARE_NOTES.md`.

Because of this, `MUL16`, `DOT32`, `MAC16` and `Q15_DOT` use the same verified multiply form rather
than a second, unverified code path (`tools/dspsim --verify` checks all of them). `MUL16` dispatches on
the idiom once per call and inlines the loop body per idiom, so its timing measures the loop, not the
dispatch.

`BKREP` loop counts are **calibrated on the console at boot**: the firmware measures whether the core
executes `LC` or `LC+1` iterations and applies the difference to every kernel. The self-test prints the
result (`BKREP loop: LC+1 iterations (bias 0)` or `LC iterations (bias 1)`), it is logged and exported
as `loop_bias`. This exists because the host emulator and the first hardware run disagreed by one
element in ADD16.

The self-test's ADD16 line is diagnostic by design. It reports either `no reply` (with the mailbox
status/error) or the exact mismatch count, first index and the surrounding DSP/reference words. A
timeout disables the DSP link for the rest of the session (fail fast) instead of retrying a blocking
unload.

`MAC2` is classified as experimental by design. The task's rule — do not treat dual MAC as
hardware-confirmed before correct results — is enforced in code: it is never marked `verified` and the
analyser flags it.

## 5. Phase 3 — memory

DSP-local (data RAM, bank 0):

* `MEMCPY` word copy, sizes 256 / 1024 / 4096 / 8192 words.
* `MEM_READ` sequential sum (read bandwidth proxy).
* `MEM_WRITE` sequential fill (write bandwidth proxy).
* `MEM_RANDOM` indexed gather through a 512-entry index list.

ARM11 reference: FCRAM `memcpy` of the same sizes (cached path, which is what a real pipeline uses).

Cache handling: DSP RAM is written through the ARM11 data cache, so the transport flushes before
raising the doorbell and invalidates before reading results (`svcFlushProcessDataCache` /
`svcInvalidateProcessDataCache`, `__dsb`). Results read without a flush would be meaningless, and the
self-test would catch it as a mismatch.

Not in this build: DSP DMA transfers (FCRAM↔DSP). The register sequence is documented in
`HARDWARE_NOTES.md`; it is listed as future work, so Phase 3 does not claim DMA numbers.

### DSP DMA (firmware v16)

Two commands implement the AHBM path with the register sequence taken from libteak's `dma.c`:

* `DMA_READ` — ARM physical memory → DSP data RAM (`ARG0` DSP word address, `ARG1/ARG2` physical
  address low/high, `ARG3` words).
* `DMA_WRITE` — DSP data RAM → ARM physical memory.

Limits enforced by the bench: at most **512 words (1 KB) per transfer**, because the AHBM must not
cross a 1 KB boundary on the ARM side, and the buffers are `memalign(1024, ...)`. Every transfer is
verified word for word against the ARM copy, and the firmware bounds its completion poll so a stuck
DMA fails the command instead of hanging the DSP.

The benchmark issues 16 blocks of 512 words (8192 words total) and reports ns/word including the
per-block setup, next to the CPU `TRANSFER_WRITE`/`TRANSFER_READ` numbers in the same suite.## 6. Phase 4 — neural

Operators: Conv2D 3×3, Conv2D 5×5, depthwise 3×3 (8 channels), pointwise 1×1 (8→8), dense (256→64),
ReLU, pixel shuffle 2×, SR-LUT-style 4-tap LUT, and a fused tiny CNN (conv + ReLU + pointwise + ReLU
+ dense).

* Data: deterministic int16 (xorshift32), weights deterministic, no training.
* Resolutions: 100×60, 200×120, 280×168, 320×192, 400×240.
* Validation: each op is compared with a scalar reference before timing.
* Backends: ARM11 C int16 (verified), ARM11 VFP float (conv, not bit-identical), ARM11 threaded
  (conv rows split over up to 4 threads), DSP (ReLU and SR-LUT only in this build, chunked to the
  32 KiB kernel buffers and verified; other Teak kernels are `unavailable`).
* `ns_per_op` is ns per MAC for convolution/matrix ops and ns per element for elementwise ops.

DSP neural chunking: buffers are 32/24/32/32 KiB; larger tensors are split into ≤ 8192-word chunks and
the transfer overhead is included in the DSP record. The chunk count is exported in the note, so a
chunked result is never mistaken for a single-shot kernel.

## 7. Phase 5 — hybrid GPU

Not implemented in v0.1. No GPU or hybrid numbers are produced. The intended measurement plan is in
`README.md` and `HARDWARE_NOTES.md`; the reason for the gap is the missing DSP convolution path and
the external PICA-SR reference, not a missing timer.

## 8. Phase 6 — stability

Runs 10 s / 1 min / 5 min / 30 min. Each iteration: deterministic ADD16 over 512 words, DSP result
compared with ARM, timeouts counted, one reload attempted after a timeout. START aborts. The record
carries iteration/error/timeout counts.

Audio coexistence is investigated as follows: DSP compute requires replacing the audio firmware, so
`ndsp` cannot run simultaneously. DSPMark therefore restores stock audio on exit and reports whether
`ndspInit` succeeded. CSND-based concurrent output is *not* implemented; it is listed as future work
rather than claimed.

## 9. CSV / JSON schema

CSV columns (one row per record):

```
suite,test,backend,variant,resolution,problem_size,available,verified,experimental,timed_out,
latency_ns,compute_ns,overhead_ns,ns_per_op,ns_min,p95_ns,p99_ns,median_k4_ns,median_k16_ns,ops,note
```

A second file, `<stamp>-samples.csv`, contains the raw measurements:
`suite,test,backend,variant,repeat,sample_ns,ops` — one row per sample per repeat point.

JSON layout:

```json
{
  "tool": "DSPMark", "schema": 1, "generated": "...",
  "provenance": { "kind": "hardware" },
  "platform": { ... },
  "method": { "timer": "...", "repeat_points": [1,4,16], ... },
  "dsp": { "state": "...", "component_loaded": true, ... },
  "results": [ { "suite": "...", "test": "...", "backend": "dsp", ... } ]
}
```

`note` carries the verification verdict (`verified` / `MISMATCH` / `unavailable`) and context such as
chunk counts.

## 10. Hardware results — New3DS, 804 MHz + L2

### v0.2 run (current)

Results: `results/new3ds-804mhz-2026-11-17-v15.{json,csv}` plus `-v15-samples.csv` (every raw sample).
Firmware protocol **v15** (OpenDSP boot prologue, `flags=1`), `loop_bias=0`, `mul_idiom=7`,
handshake 19 µs. Component lifecycle **confirmed on hardware in this run**: the log shows
`exit: restore ok (component unloaded; stock ndsp firmware accepted the DSP)` followed by a normal
`aptExit` — HOME and a clean return to the menu work with the DSP loaded.

Records: 166 — **131 verified**, 0 unexpected mismatches, 27 unavailable (the Teak convolution
kernels), and the two `*_DUAL` rows that are expected to mismatch (see below). All scalar numbers are
identical to the v0.2 `flags=0` run, i.e. the MIU paging mode does not change kernel timing.

Arithmetic, ns per operation (same int16 workload, 1024 elements; MAC counts as one op):

| Test | DSP | ARM11 C | DSP vs ARM |
|------|-----|---------|-----------|
| `ADD16` | 30.25 | 7.81 | 0.26× |
| `ADD16_U4` | 25.85 | 7.82 | 0.30× |
| `ADD16_U8` | 25.09 | 7.83 | 0.31× |
| `ADD16_SAT` | 137.99 | 17.53 | 0.13× |
| `SUB16` | 31.41 | 7.82 | 0.25× |
| `SUB16_SAT` | 139.04 | 17.35 | 0.12× |
| `SUB16_U4` | 25.98 | 7.82 | 0.30× |
| `MUL16_LO` | 60.39 | 11.55 | 0.19× |
| `MUL16_U2_LO` | 50.53 | 11.58 | 0.23× |
| `DOT32` | 30.79 | 7.82 | 0.25× |
| `DOT32_U4` | 26.16 | 7.86 | 0.30× |
| `MAC16` (16 rows) | 30.50 | 9.21 | 0.30× |
| `MAC16_U4` | 24.90 | 9.21 | 0.37× |
| `MAC2` (2 chains) | 26.86 | — | — |
| `Q15_DOT` | 30.93 | — | — |
| `Q15_MUL` | 165.88 | 17.83 | 0.11× |
| `ACC32` | 46.73 | 7.86 | 0.17× |
| `ACC40` | 31.79 | 9.11 | 0.29× |
| `SHIFT16` | 46.65 | 9.11 | 0.20× |
| `NORM` | 195.70 | 24.02 | 0.12× |
| `REG_MPY` | 24.50 | 7.82 | 0.32× |
| `DOT32_DUAL` | 9.79 | 7.82 | 0.80× — **MISMATCH** |
| `MAC16_DUAL` | 8.16 | 9.21 | 1.13× — **MISMATCH** |

Interpretation:

* **The unrolled kernels work and reach the arithmetic ceiling.** `MAC16_U4` is 24.90 ns/MAC and
  `REG_MPY` — a register-fed multiply with no memory operand — is 24.50 ns/MAC. The kernel is
  therefore within 2% of the core's multiplier throughput: the remaining cost is instruction issue
  (`mpy` + `add p*` per product), not memory.
* **That ceiling is ~40 MMAC/s, not 134 MMAC/s.** The 134 MMAC/s theoretical target would need one
  MAC per cycle, i.e. a single instruction that multiplies and accumulates — this core needs at least
  two (`mpy` → `P`, then `add p*`). At 24.5 ns/MAC, `MAC16_U4` sustains 40.2 MMAC/s and `MAC2`
  37.2 MMAC/s.
* **Two chains are not a parallel dual-MAC.** `MAC2` interleaves two independent accumulators with
  *sequential* multiplies: +14% over `DOT32`, all of it from amortising the loop, not from a second
  multiplier. The parallel forms (`mac1`, `acc+p0+p1`) remain unimplemented and unclaimed.
* **The ARM11 baseline is still 2.7-9× faster per operation.** Best DSP case: `MAC16_U4` at 0.37× of
  ARM. Saturation, Q15 multiply and block normalisation are the worst (7-9× slower) because they add
  branches or a second pass.
* **End-to-end** (`MAC16_E2E`, write inputs + run + read results): 915 µs for 16384 MACs, 55.9 ns/MAC,
  versus 24.9 ns/MAC for the kernel alone. Transport is more than half of the real cost.
* **Dual-multiply status.** `DOT32_DUAL` and `MAC16_DUAL` are fast but wrong (9.9 and 8.2 ns/MAC) and
  stay `MISMATCH` on every vector. The probe pinned the semantics: with `(2,3)*(3,5)` it returns
  `p0 = 9` and `p1 = 15`, i.e. `p0 = X[1]*Y[0]`, `p1 = X[1]*Y[1]` — both products share one X operand
  while the Y pair advances. The instruction is a **broadcast 1×2 multiply-accumulate** (one
  coefficient × two samples), which is the FIR/window pattern, not a two-tap dot of two independent
  vectors. Those numbers are therefore not a valid dot throughput and are not used anywhere.

### v0.1 baseline (kept for regression)

Results file: `results/new3ds-804mhz-2026-11-16.{json,csv}`.
Build: `Nov 16 2026 22:11:29`, protocol v5, DSP image 3814 bytes, crc `C08789C0`.
Records: 126 — **92 verified, 0 mismatch**, 7 experimental (`MAC2`), 27 unavailable (the unshipped
Teak convolution kernels), 0 timeouts.

| Test | DSP | ARM11 C |
|------|-----|---------|
| `ADD16` | 30.29 | 7.94 |
| `MUL16_LO` | 60.35 | 11.53 |
| `DOT32` | 30.85 | 7.80 |
| `MAC16` (16 rows) | 30.49 | 9.12 |
| `Q15_DOT` | 30.88 | — |
| `ADD16_SAT` | 137.94 | — |

### Transport — ns per word, DSP mailbox (both runs agree)

| Operation | 1024 words | 8192 words |
|-----------|-----------:|-----------:|
| ARM→DSP write (uncached) | 23.07 | 23.01 |
| DSP→ARM read (uncached) | 52.51 | 52.25 |
| FCRAM cache flush | 20.57 | 4.38 |
| FCRAM cache invalidate | 11.70 | 3.06 |

A round trip therefore costs **~75 ns per word** before any DSP arithmetic happens.

### Neural operators (v0.1 run)

| Operator | DSP | ARM11 C |
|----------|-----|---------|
| `RELU` 100×60 (6000 elements) | 885.9 µs total, 147.7 ns/element | 60.0 µs, 10.0 ns/element |
| `SR_LUT` 4-tap 100×60 | 1.91 ms total, 318.2 ns/element | not timed (the scalar reference only verifies the DSP result) |
| Conv2D / dense / pointwise / depthwise / tiny CNN | unavailable in this build | 9.4–21.3 ns/MAC |

### Verdict

For a throughput coprocessor the answer is **no**: on a New3DS at 804 MHz + L2 the Teak DSP is
2.7–9× slower per operation than the ARM11 C code, and transport adds ~75 ns per word on top.

* Best DSP kernel (`MAC16_U4`) runs at 24.9 ns/MAC against 9.2 ns/MAC on the ARM11; even the
  register-fed ceiling (24.5 ns/MAC) is 2.7× behind.
* The break-even condition `75·W + 24.9·M < 9.2·M` has no positive solution: DSP compute alone is
  already slower, so no transfer size or kernel shape reverses the sign.
* Elementwise work is the worst case (`RELU` 147.7 vs 10.0 ns/element, `NORM` 195.7 vs 24.0,
  `Q15_MUL` 165.9 vs 17.8).
* End-to-end MAC16 costs 55.9 ns/MAC (kernel 24.9 + transport), so a real pipeline sees less than
  half of the already-slow kernel rate.

This is a throughput statement, not a correctness one: 84/84 records verified on hardware with zero
mismatches, including the calibrated multiply idiom and both MAC accumulators. The 134 MMAC/s
theoretical figure is not reachable on this core because a MAC needs at least two instructions
(`mpy`, then `add p*`); the measured instruction-issue ceiling is ~40 MMAC/s.

What the DSP could still be worth is *offloading*, not speed: a kernel does not have to be faster in
isolation to free ARM11 time, which is what Phase 9 (GPU + ARM + DSP concurrency) has to measure.
Nothing here should be read as an architectural limit of the XpertTeak core; it is a limit of this
DSP clock, this transport and these kernel shapes.

## 11. Measurement status

| Item | Status |
|------|--------|
| Host self-test (Teakra) | 49 checks pass, 0 fail |
| Hardware performance | **measured** — `results/new3ds-804mhz-2026-11-16-v02.json` (New3DS, 804 MHz + L2) |
| Old3DS | not measured in this repository |
| DSP neural kernels | ReLU + SR-LUT only; the rest are marked unavailable |
| DSP DMA | not in this build |
| GPU hybrid | not in this build |

Never copy emulator numbers into a hardware results file; the JSON `provenance.kind` field records
which is which.
