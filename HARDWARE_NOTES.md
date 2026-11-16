# DSPMark hardware and toolchain notes

Everything here is either documented from a primary source, confirmed by the project's own
experiments, or explicitly marked as unverified. This file is where the assumptions live.

## 1. The DSP component loading path

**What works.** The 3DS `dsp::DSP` service accepts a DSP1 `*.cdc` image through
`DSP:LoadComponent` (`IPC 0x001100C2`). The format is documented on 3dbrew (`DSP_Binary`):

* `0x000–0x0FF` RSA signature area
* `0x100` magic `DSP1`, size, memory layout (`0xFFFF` = all 32 KiB blocks), segment count, flags
* `0x120` up to 10 segment records: data offset, target word address, size, memory type
  (0 = program RAM at `0x1FF00000`, 2 = data RAM at `0x1FF40000`), SHA-256 of the segment

Our image is assembled by `tools/asm.py` from `dsp/*.asm` through Teakra's own parser
(`tools/teaasm`), so the encodings are identical to the emulator's and disassembler's tables.

**Signature enforcement.** The component has no RSA signature (the signature block is zeroed).
Loading it therefore requires a CFW that patches the DSP signature check. Luma3DS does this in its
loader (`patcher.c`, program id `0004013000001A02`, comment "Patch DSP signature check"), and its
homebrew loader maps DSP RAM (`0x1FF00000–0x1FF80000`) read/write into the `.3dsx` process. Both
facts are visible in Luma3DS's source and match what the Teakra hardware tests and OpenDSP rely on.

DSPMark does **not** assume success: it prints the `Result` code, the `loaded` flag and the boot
handshake result, and falls back to ARM-only benchmarks. Stock firmware without such a patch is
expected to refuse unsigned components; that case is reported, not worked around.

**Component flags.** DSPMark builds the component with **flags = 0**, so the sysmodule does not
expect the three boot reply registers. This is the configuration Teakra's hardware tests use, and it
removes the only startup step that can block `DSP_LoadComponent` (the sysmodule waiting for
`REPLY0..2` that never arrive because of an MMIO-base mismatch). The firmware therefore does **not**
write a boot handshake and does not announce a pipe table; the benchmark uses the shared mailbox only.

Unloading uses the protocol OpenDSP documents: on `CMD2 = 0x8000` the firmware writes
`REPLY2 = 0x8000` and halts, which is what `DSP_UnloadComponent` waits for. The polling happens in
the idle loop through pointer-based MMIO accesses, so the mailbox page register stays 0.

Earlier builds used flags = 1 with the full boot handshake. On at least one console that hung at
`DSP_LoadComponent`; the app now runs the whole load on a worker thread with a 4 s watchdog, so a
blocked IPC degrades to ARM11-only mode instead of freezing the UI.

## 2. Memory map used by DSPMark

DSP data RAM (bank 0) as 16-bit words, seen by the ARM11 at `0x1FF40000`:

| Range | Use |
|-------|-----|
| `0x0000–0x001F` | mailbox (doorbell, cmd, status, error, args, results, banner, repeat) |
| `0x0020–0x0047` | pipe table (8 slots, channels 0–3) |
| `0x0050–0x00CF` | 4 shared pipe buffers |
| `0x00D0–0x00FF` | kernel scratch |
| `0x0100–0x07FF` | free / stack grows down from `0x07FF` |
| `0x0800-0x47FF` | buffer A, 32 KiB |
| `0x4800-0x77FF` | buffer B, 24 KiB |
| `0x7800-0x87FF` | **reserved**: MMIO window for base `0x7800` *or* `0x8000` |
| `0x8800-0xBFFF` | buffer C, 28 KiB |
| `0xC000-0xFFFF` | buffer D, 32 KiB |

The reserved hole is why the buffers are not four equal 32 KiB blocks: the APBP/DMA/timer registers
live there for both candidate bases, and a `MEMCPY` across them would be a hardware fault (it was
exactly the first hardware failure — see section 3).

**ARM11 cache.** DSP RAM is ordinary cached memory from the CPU's point of view. Every mailbox write
is followed by `svcFlushProcessDataCache` and every poll by `svcInvalidateProcessDataCache`
(`__dsb` after). This is the same approach as Teakra's hardware tests. Reading the DSP's results
without invalidating produces stale data and the app would flag it as a mismatch.

## 3. MMIO base (`0x7800` vs reset `0x8000`)

The DSP core resets with MIU `MMIOBASE = 0x8000` (GBATEK/Teakra default); Nintendo's
firmware and OpenDSP use `0x7800`. A component loaded into a *reset* core therefore cannot
assume either value, and DSPMark's first hardware run proved it: an ADD16 destination buffer at
word `0x8000` was written while the base was still `0x8000`, i.e. into the APBP window. The DSP
froze mid-kernel (`status=BUSY`, heartbeat stopped); the ARM saw a timeout, which the early build
reported as "ADD16 FAIL".

Two independent guards now exist:

1. **The firmware forces the base to `0x7800` at boot** by writing MIU `MMIOBASE` (offset `0x011E`)
   through both candidate addresses (`0x791E` and `0x811E`). Whichever address is MMIO performs the
   write; the other lands in the reserved data hole and is harmless.
2. **No buffer or stack lives in `0x7800–0x87FF`**, the union of both possible MMIO windows. Buffers
   are A `0x0800–0x47FF`, B `0x4800–0x77FF`, C `0x8800–0xBFFF`, D `0xC000–0xFFFF`. Even if the base
   write were ignored, a buffer access could never become an MMIO access.

The host harness also sets the base to `0x7800` before running, which is why the emulator never
reproduced the failure. This is a concrete example of why emulator passes are not hardware passes.

## 4. Teak ISA quirks established empirically

These cost real debugging time and are encoded in the assembler and firmware comments:

1. **Operand order.** Teakra's *parser* (used by `makedsp1` and by DSPMark's assembler) takes
   `<source> <destination>` for `mov`/`add`/`cmp`. A memory operand first is a **load**, second a
   **store**. Teakra's *disassembler* prints the opposite order for `mov`. Consequence:
   `mov a0l [page:x]` stores, `mov [page:x] a0l` loads. `tools/asm.py` and the firmware comments follow
   the parser; do not trust the disassembly listing for operand order.
2. **`BKREP` iteration count is calibrated, not assumed.** Teakra's emulator runs the block `LC+1`
   times while some Teak documentation describes `LC`; the first hardware run of DSPMark failed
   ADD16 with a one-element difference, which is exactly that symptom. The firmware now measures the
   behaviour on the running core at boot (a one-instruction block with `LC=1`, counting executions)
   and stores a bias (0 = LC+1, 1 = LC) that every kernel adds to its loop count. The value is
   reported in `DSP_INFO`, shown by the self-test, logged in `DSPMark.log` and exported in JSON as
   `loop_bias`. The emulator reports 0, as expected.
   The block-repeat **end label must hold an instruction** (the loop ends at `PC == end+1`), so the
   firmware puts a trailing `nop` at every loop label; a label on a branch leaks a loop frame.
3. **Multiply operand semantics differ by addressing form, and the difference is only visible on
   hardware.** Two-memory forms (`mpy [r4++] [r0++] aX`, `mac [r4++] [r0++] aX`) returned exactly
   `2²+3²+4²+5² = 54` for the probe dot `(2,3,4,5)·(3,5,7,9) = 94`, on every idiom that used them:
   the core multiplies the X operand by itself, i.e. the second memory operand is not read as Y.
   Teakra aliases X and Y memory in that encoding, so an emulator can never show this.
   The working form on the New3DS is to load the Y factor into the multiplier register and multiply
   by X memory:

   ```
   mov [r4++] y0
   mpy y0 [r0++] a1     ; product in P (a1 scratch)
   add p* a0            ; accumulate
   ```

   This is the form OpenDSP's hardware-tested loops use. `MPY` does not accumulate into the named
   accumulator; `add p*` does. `MAC y0,[r0],aX` accumulates directly but its accumulator update is
   only visible after the next instruction, hence the explicit `nop` drain.
   DSPMark does not hard-code any of this: at startup the firmware runs eight idioms over the known
   dot and the ARM side selects the first one that returns 94 (`SET_IDIOM`). `dot4` and `MUL16` then
   use that idiom for the session, and the choice is exported as `mul_idiom` in the JSON/CSV.
   Hardware selected idiom 7 (`y0` + `mpy`/`add p*`); Teakra selects idiom 0.
4. **Immediate compares are 16-bit.** `cmp 0x7fff, a0` does not see the upper bits of a 40-bit
   accumulator. Saturating a 32-bit result to int16 must inspect the high word (see `satw_store` in
   `dsp/dspmark_kernels.asm`).
5. **Immediate loads sign-extend.** `mov 0x8000, a0` produces `0xFFFFFF8000`, while a 16-bit register
   load only writes the low word and leaves the rest untouched. Counter arithmetic must clear the
   accumulator first.
6. **`lim` clamps a 40-bit accumulator to signed 32-bit**, which is useful before packing a 32-bit
   result but not a 16-bit saturation.
7. **`0x78xx` is MMIO**; accessing APBP/DMA requires `page = 0x79` (base `0x7800`) plus the register
   offset. The DMA code documents this.
8. **A multiply and the use of `p*` must not be separated by an interrupt.** OpenDSP masks interrupts
   around every multiply sequence (`dint`/`eint`) because an interrupt between `mpy` and `add p*,a0`
   corrupts the product on hardware. DSPMark's component runs with interrupts disabled for its whole
   lifetime (`dint` at boot, interrupt vectors are inert), so no kernel here can be interrupted
   mid-product. Any future change that enables interrupts must re-check this.
9. **The core has extended parallel instructions; the dual-multiply one is now used.** The compound form
   `xy<- [arprni0+arpsi0] [arprnj0+arpsj0] acc+p0+p1 a0 mpysxsy mpysxsy dmodi dmodj` (0x9863) performs
   two signed multiplies and adds both products to an accumulator in one instruction.
   `DOT32_DUAL` / `MAC16_DUAL` use it with `arp0 = 0x018C` (address registers advance by 2, second
   element at +1), primed once to discard stale products and drained after the loop. Teakra verifies
   both the dual path and the odd/short fallback against the scalar dot.
   Still unused: `mac1` (the second multiplier pipeline with its own P1 accumulation) and the
   4-operand `mov a0h b0h [arrn0+ars0] x0<->x1 acc+p0+p1 b0 mpysxsy mpysxsy` form.
   `docs/PRIOR_ART.md` records where these forms were found.
10. **The boot prologue must match OpenDSP's, or the sysmodule never finishes booting.** A `flags=1`
    component must answer `REPLY0/1/2 = 1` **and** raise semaphore bit 15 (`PSEM` at `0x78CC = 0x8000`,
    "pipe table ready") after building its pipe table. It also has to configure the MIU the way
    OpenDSP's hardware-proven `crt0` does: `MIU_PAGE0CFG` (`0x114`) written once through the reset
    MMIO base, `MIU_MMIOBASE = 0x7800`, then `PAGE0CFG = 0x201E`, `PAGEMODE = 1`, `XPAGE = 0`,
    `ar0 = 0x110C`, `ar1 = 0x56A3`, with the 16-nop APBP settling delay after each reply.
    Earlier builds skipped the paging setup and the semaphore and still loaded (with `flags=0`), but
    every unload then hung the console: the sysmodule's state machine never left "booting", so the
    Home Menu could not get its audio firmware back.
    **Confirmed on hardware (New3DS, 2026-11-17, firmware v15):** with the full prologue and
    `flags=1`, loading works, **HOME returns to the menu, and the app exits normally** — the DSP is
    handed back to the system like any ndsp application. This is the prerequisite for a DSPCompute
    library: load at app start, unload at exit, no reboot.
11. **`DSP_UnloadComponent` can block forever, so nothing may rely on it.** It is an IPC round trip to
    the dsp sysmodule, which waits for the component's shutdown acknowledgement; the call cannot be
    cancelled. The app therefore watchdog-times the unload (2 s worker) and, if the DSP cannot be
    returned, offers a power-off instead of hanging on a black screen.
    The firmware's shutdown acknowledgement mirrors any APBP command channel (`CMD0/1/2` at
    `0x78C2/0x78C6/0x78CA`) to its reply register (`REPLY0/1/2` at `0x78C0/0x78C4/0x78C8`),
    acknowledges the ARM semaphore bits (`SEM 0x78D2` → `PSEM 0x78CC`, `PCLEAR 0x78D0`) and stops on
    `0x8000`. The register map is verified against Teakra's `MMIORegion` and libteak's `apbp.h`;
    libteak places the same registers at `0x80C0..0x78D8` with the default MMIO base `0x8000`, which
    is why the firmware accepts either base at boot.

`docs/DSP_ISA_FORMS.txt` is a generated, deduplicated instruction-form reference produced from
Teakra's disassembler/parser round trip; `tests/dsp/probe*.asm` are the probes that established the
behaviours above.

## 5. Toolchain

* `tools/asm.py` — two-pass Teak assembler producing DSP1 images. Instruction encoding is delegated to
  Teakra (`tools/teaasm.exe`), so a parser change cannot silently desynchronise the assembler.
* `tools/dspsim` — Teakra-based harness with two modes: low-level probe (set words, run N cycles,
  dump memory/registers) and `--verify`, which drives the shipped firmware exactly like the ARM11 and
  compares every result with a C reference.
* `tools/teacan` — generates `docs/DSP_ISA_REFERENCE.txt` by disassembling every 16-bit opcode and
  re-parsing the text; `tools/isa_forms.py` reduces it to unique forms.
* Host builds use Visual Studio's environment plus LLVM's `clang++`, because the project is developed
  on Windows and devkitPro ships no host compiler. `tools/build_host_tools.bat` documents the exact
  steps.

The shipped `data/dspmark_cdc.bin` is committed so that a plain `make` works without the host tools;
`make dsp` regenerates it.

## 6. What is verified where

| Item | Emulator (Teakra) | Hardware |
|------|-------------------|----------|
| Component format/load handshake | yes (format + boot replies) | **verified** (New3DS, layout 6) |
| Mailbox protocol + timeouts | yes | **verified** (0 timeouts in the reported run) |
| ADD16/SAT, MUL16, MAC16, DOT32, Q15, saturation | yes, word-exact | **verified**, all word-exact |
| Memory copy/read/write/random | yes, word-exact | **verified**, all word-exact |
| ReLU, SR-LUT | yes, word-exact | **verified**, all word-exact |
| Multiply idioms | `mpy`+`add p*` (idiom 0) | **idiom 7 only** (`y0` + `mpy`/`add p*`), selected by the on-console probe |
| MAC2 (dual accumulate) | **known mismatch** (one product of 15 extra) → experimental | still mismatched on hardware → stays experimental |
| Conv/depthwise/pointwise/dense Teak kernels | not shipped (first versions failed) | not shipped |
| DSP DMA | not shipped | not shipped |
| Neural ARM implementations | n/a (computed on the console) | verified on-device against scalar references |
| GPU hybrid | — | not implemented in v0.1 |

A hardware run is not a formality: it is the only way to confirm the signature patch situation, the
`0x7800` assumption, the mailbox cache behaviour and the multiplier semantics on real silicon — the
last of which turned out to differ from Teakra in a way the emulator cannot show
(`HARDWARE_NOTES.md`, item 3).

## 7. Missing pieces and their next steps

* **Neural Teak kernels.** The first Conv3×3/5×5 and matrix kernels produced wrong sums in the
  emulator's reference check even though the same `dot4` routine passed inside MAC16. The next step is
  a per-output-element trace of one row (compare the exact 9 product words), then extend to a 3×3
  unit test in the host verifier before any performance claim. They are intentionally absent rather
  than shipped with a "probably right" label.
* **DSP DMA.** The OpenDSP register sequence is the reference (AHBM channels 1/2, DMA channels 1/2,
  `DMA_SPACE 0x0607/0x0670`, `CTRL 0x40C0`). It needs an emulator-side AHBM/FCRAM model for a
  correctness pass, then a hardware run. Physical addresses must come from `osConvertVirtToPhys` and
  the buffers must be 32-byte aligned; only app-owned memory is touched.
* **Audio coexistence.** Replacing the DSP firmware excludes `ndsp` while DSPMark runs. Stock audio is
  restored on exit. A CSND-based concurrent test is future work; it must not be described as "DSP
  compute + DSP audio" because CSND is a separate hardware block.
* **GPU hybrid (Phase 5).** Needs the neural DSP path, a citro3d render-to-texture + display-transfer
  readback pipeline, and an external PICA-SR reference. Deliberately not faked.

## 8. Safety

No overclocking, no voltage changes, no NAND writes, no unknown MMIO pokes. The only clocks touched are
the standard New3DS performance profile request (`osSetSpeedupEnable`). Memory operations stay inside
app-owned buffers and the documented DSP RAM window. DMA, when implemented, will only accept physical
addresses derived from `osConvertVirtToPhys` for buffers the app allocated.

## 9. References

* Teakra — emulator, assembler, disassembler and hardware test tools (MIT):
  `src/*.md` (APBP, DMA, AHBM, MIU, timers, ICU) and `hwtest/` (`dsptester`, `dspmemorytester`).
* OpenDSP — an open DSP firmware replacement with hardware-tested APBP/DMA/MMIO constants (BSD-2).
* 3dbrew — `DSP_Binary`, `DSP:LoadComponent`, `DSP:UnloadComponent`, `DSP Services`,
  `CONFIG11 Registers`.
* GBATEK — DSi XpertTeak sections.
* libctru — `dsp.h`/`ndsp` behaviour, `svcFlushProcessDataCache`, `osConvertVirtToPhys`,
  `osSetSpeedupEnable`.
* Luma3DS — confirmation that unsigned DSP components load and that homebrew gets DSP RAM mapped.
