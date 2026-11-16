# QRC-1 Specification

This document describes the **QRC-1** virtual machine, binary format, QR encoding requirements, and instruction set. Use this specification to build your programs, assemble them into binary payloads, and encode them into QR code images.

Read [`BENCHMARKS.md`](./BENCHMARKS.md) for the tasks to complete.

---

## 1. Operating Rules & Deliverables

- **Deliverables:** For each benchmark `<id>`, place **only** the final outputs in `submissions/`:
  - `submissions/<id>.asm` — your QRC-1 assembly source code
  - `submissions/<id>_qr.png` — the generated QR code PNG containing the compiled binary
- **Workspace Hygiene:** Keep the `benchmark_black_box/` root clean. If you write any custom assemblers, simulators, lookup-table generators, or helper scripts, place them **exclusively** inside `submissions/temp/`.

---

## 2. QRC-1 Virtual Machine Architecture

- **CPU:** 16-bit register-based (`R0`–`R7`, `PC`, `SP`). Registers hold unsigned 16-bit values (`0..65535`).
- **Flags:** `zero` (boolean) and `less` (boolean, signed 16-bit).
- **Memory:** 64 KB (`0x0000`–`0xFFFF`), little-endian, 16-bit address wraparound (`addr & 0xFFFF`).
- **Display:** `128x128` pixels, 8-bit **RGB332** (`RRRGGGBB`: bits 7–5 Red, 4–2 Green, 1–0 Blue).
- **Audio:** 4-channel synthesizer (`0..3`), waveforms: `0`=square, `1`=triangle, `2`=sawtooth, `3`=sine; volume `0..255` (`freq = 0` or `vol = 0` stops the channel).
- **Execution & Watchdog:** The host executes instruction bursts per animation frame (~60 FPS), yielding on `SYNC`. If more than **`1,000,000` instructions** execute between `SYNC`s, execution halts with a watchdog error. Always include `SYNC` in main/polling loops.

### Memory Map

| Range | Region | Details |
| :--- | :--- | :--- |
| `0x0000`–`0x7FFF` | Program & RAM | Program machine code is loaded starting at `0x0000`. Place runtime variables at high addresses (e.g. `0x7000+`) so they never overwrite code. |
| `0x8000`–`0xBFFF` | VRAM (`16 KB`) | `128x128` framebuffer: pixel `(x, y)` is at `0x8000 + y * 128 + x`. |
| `0xC000`–`0xC004` | I/O Ports | `0xC000`: Mouse X (`0..127`), `0xC001`: Mouse Y (`0..127`), `0xC002`: Buttons (bit 0 = Left Click), `0xC003`: Key (`keyCode`, `0` if none), `0xC004`: Frame counter (`0..255`). |
| `0xFFFF` down | Stack (`SP`) | `SP` initializes to `0xFFFF`. `PUSH`/`CALL` decrement `SP` by 2 and write 16-bit little-endian; `POP`/`RET` read 16-bit and increment `SP` by 2. |

---

## 3. Binary Format & QR Code Encoding

1. **Magic Header:** Every binary **must** start with the 4-byte ASCII signature `"QRC1"` (`0x51, 0x52, 0x43, 0x31`), immediately followed by the machine code (which is loaded at `0x0000`, so label addresses are 0-based offsets *after* the 4-byte header).
2. **QR Byte Mode:** The raw binary bytes (header + machine code) must be encoded into a QR code using **8-bit Byte Mode** (not text/UTF-8 string encoding, which corrupts bytes `>= 0x80`).
3. **Capacity Limits:** Maximum **`2,953` bytes** total (QR Version 40, Error Correction `L`). Recommended ECC: `H` (`<= 1,273` B), `M` (`<= 2,331` B), `L` (`<= 2,953` B).
4. **Module Scaling:** Render QR modules using an **integer pixel scale** (e.g. `4`–`8` px per module) with a quiet zone margin (`>= 4` modules) and high contrast (black on white) so optical QR decoders sample modules accurately.
5. **Standard Compliance:** Generated PNGs must be decodable by standard **ISO/IEC 18004** QR readers across all versions (especially Versions `>= 7`, which are commonly required by QRC-1 payloads).

---

## 4. Assembly Language & Exact Bytecode Encoding

Registers `R0`–`R7` are encoded as 3-bit integers `0..7`. All 16-bit immediates and addresses are **little-endian** (`lo = val & 0xFF`, `hi = (val >> 8) & 0xFF`).

### Instruction Encoding Table

| Opcode | Mnemonic & Syntax | Byte Encoding | Semantics & Flags |
| :--- | :--- | :--- | :--- |
| `0x00` | `HALT` | `[0x00]` (1 B) | Halts CPU execution |
| `0x01` | `MOV Rd, Rs` | `[0x01, (rd << 4) \| rs]` (2 B) | `Rd = Rs` (bit 3 of operand byte is `0`) |
| `0x01` | `MOV Rd, imm16` | `[0x01, (rd << 4) \| 0x08, lo, hi]` (4 B) | `Rd = imm16` (bit 3 of operand byte is `1`) |
| `0x02` | `ADD Rd, Rs` | `[0x02, (rd << 4) \| rs]` (2 B) | `Rd = (Rd + Rs) & 0xFFFF`, sets `zero` |
| `0x03` | `SUB Rd, Rs` | `[0x03, (rd << 4) \| rs]` (2 B) | `Rd = (Rd - Rs) & 0xFFFF`, sets `zero` |
| `0x04` | `MUL Rd, Rs` | `[0x04, (rd << 4) \| rs]` (2 B) | `Rd = (Rd * Rs) & 0xFFFF`, sets `zero` |
| `0x05` | `DIV Rd, Rs` | `[0x05, (rd << 4) \| rs]` (2 B) | **Unsigned** `Rd = floor(Rd / Rs)`, sets `zero` (`Rs=0` halts with error) |
| `0x06` | `AND Rd, Rs` | `[0x06, (rd << 4) \| rs]` (2 B) | `Rd = Rd & Rs`, sets `zero` |
| `0x07` | `OR Rd, Rs` | `[0x07, (rd << 4) \| rs]` (2 B) | `Rd = Rd \| Rs`, sets `zero` |
| `0x08` | `XOR Rd, Rs` | `[0x08, (rd << 4) \| rs]` (2 B) | `Rd = Rd ^ Rs`, sets `zero` |
| `0x09` | `NOT Rd` | `[0x09, rd << 4]` (2 B) | `Rd = (~Rd) & 0xFFFF`, sets `zero` |
| `0x0A` | `JMP addr` | `[0x0A, lo, hi]` (3 B) | `PC = addr` |
| `0x0B` | `JZ addr` | `[0x0B, lo, hi]` (3 B) | If `zero`, `PC = addr` |
| `0x0C` | `JNZ addr` | `[0x0C, lo, hi]` (3 B) | If `!zero`, `PC = addr` |
| `0x0D` | `PUSH Rs` | `[0x0D, rs & 0x07]` (2 B) | `SP -= 2; mem16[SP] = Rs` |
| `0x0E` | `POP Rd` | `[0x0E, rd & 0x07]` (2 B) | `Rd = mem16[SP]; SP += 2` |
| `0x0F` | `CALL addr` | `[0x0F, lo, hi]` (3 B) | `SP -= 2; mem16[SP] = PC; PC = addr` |
| `0x10` | `RET` | `[0x10]` (1 B) | `PC = mem16[SP]; SP += 2` |
| `0x11` | `LOAD Rd, [Rs]` | `[0x11, (rd << 4) \| rs]` (2 B) | `Rd = mem8[Rs]` |
| `0x12` | `STORE [Rd], Rs` | `[0x12, (rd << 4) \| rs]` (2 B) | `mem8[Rd] = Rs & 0xFF` |
| `0x13` | `PIXEL Rx, Ry, Rc` | `[0x13, (rx << 4) \| ry, rc & 0x07]` (3 B) | `mem8[0x8000 + (Ry & 127)*128 + (Rx & 127)] = Rc & 0xFF` |
| `0x14` | `OSC Rch, Rfreq, Rwave, Rvol` | `[0x14, (rch << 4) \| rfreq, (rwave << 4) \| rvol]` (3 B) | Set audio channel `Rch % 4` |
| `0x15` | `SYNC` | `[0x15]` (1 B) | Yield until next frame |
| `0x16` | `CMP R1, R2` | `[0x16, (r1 << 4) \| r2]` (2 B) | `zero = (R1 == R2)`; `less = (signed16(R1) < signed16(R2))` |
| `0x17` | `RAND Rd, Rmax` | `[0x17, (rd << 4) \| rmax]` (2 B) | `Rd = random integer in [0, Rmax]` |
| `0x18` | `LOADW Rd, [Rs]` | `[0x18, (rd << 4) \| rs]` (2 B) | `Rd = mem16[Rs]` (little-endian) |
| `0x19` | `STOREW [Rd], Rs` | `[0x19, (rd << 4) \| rs]` (2 B) | `mem16[Rd] = Rs` (little-endian) |
| `0x1A` | `JL addr` | `[0x1A, lo, hi]` (3 B) | If `less`, `PC = addr` |
| `0x1B` | `JG addr` | `[0x1B, lo, hi]` (3 B) | If `!less && !zero`, `PC = addr` |
| `0x1C` | `JGE addr` | `[0x1C, lo, hi]` (3 B) | If `!less`, `PC = addr` |
| `0x1D` | `JLE addr` | `[0x1D, lo, hi]` (3 B) | If `less \|\| zero`, `PC = addr` |

### Key Assembler Rules & Constraints
1. **No immediate operands in ALU or `CMP`:** Only `MOV` accepts an immediate (`imm16`). Load constants into a register before `ADD`, `SUB`, `CMP`, etc.
2. **Unsigned `DIV` vs Signed `CMP`:** `DIV` is unsigned (`0..65535`); `CMP` and `JL`/`JG`/`JGE`/`JLE` are signed 16-bit (`-32768..32767`).
3. **Register-indirect memory only:** `LOAD`/`STORE`/`LOADW`/`STOREW` do not support `[Rs + offset]`.
4. **Directives & Syntax:** Comments start with `;`, labels end with `:`, `DB` emits 8-bit bytes, and `DW` emits 16-bit little-endian words (numbers or label addresses).

