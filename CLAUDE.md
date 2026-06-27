# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Version 3 of slu4's **Minimal UART Computer** — a homebrew 8-bit breadboard/PCB computer built from ~30 74HCxx logic ICs plus RAM and FLASH, with no microprocessor. It is a textbook Von-Neumann machine (8-bit data bus, 16-bit address bus, 32KB RAM, 32KB FLASH "SSD" with a file system) controlled entirely by microcode. An ATmega328p serves only as a variable clock generator. The console is a serial UART terminal.

This repo is mostly **hardware design artifacts and software for that machine**, not a conventional software project. The four moving parts are:

1. **Hardware** — `KiCAD9/` schematics and PCB (`8-Bit CPU 32k.kicad_*`), `Schematics.pdf`, `Layout.jpg`, Gerbers zip. The CPU is a hierarchical schematic split into ALU, PC, IR, RegA, RegB, RAM, UART_TX/RX, VGA, PS/2, clock sheets.
2. **Microcode** — `microcode_def.csv` (human-readable instruction → control-signal microcode, using `#define`d control-line mnemonics like `RO`, `AI`, `EO`, `MIL`) and `microcode_rom.csv` (the compiled ROM). The `FLASH Images/ctrl_{lsb,msb,hsb}.bin` are the three control-ROM image bytes; `flash.bin` is the program/OS FLASH.
3. **Toolchain** (`Support/Emulator/src/`) — a cross-assembler and a cycle-exact GUI emulator, both built from one C++ tree.
4. **Software** (`Programs/`) — the operating system (`os.asm` → MinOS2), assembler (`asm.asm`, self-hosting on the machine), editor (`edit.asm`), and games/demos (`blocks.asm` Tetris, `mandel.asm`, `chars.asm`).

## Building the toolchain

All builds happen in `Support/Emulator/src/` via its `makefile`:

```sh
cd Support/Emulator/src
make            # or: make linux   — builds `asm` and `MinimalUART` for Linux
make windows    # cross-compiles .exe (needs x86_64-w64-mingw32 toolchain) or builds under MSYS2
make clean
```

- `asm` is the cross-assembler (`asm.cpp` + `asm.h`; `asm.h` holds the actual `Assembler()` implementation — `asm.cpp` is just the CLI frontend). `Programs/asm.asm` is a *separate* re-implementation that runs natively on the machine itself.
- `MinimalUART` is the emulator (`main.cpp`, ~1200 lines, OpenGL/GLFW; `glad`, `glm`, and static GLFW libs are vendored under `include/` and `lib/`). It loads `microcode_rom.csv` and `charset.bin` at build/run time.
- Prebuilt binaries already exist: `Support/Assembler/{Linux,Windows}/asm*` and `Support/Emulator/{Linux,Windows}/MinimalUART*`.

There is no test suite, linter, or CI. "Testing" means assembling a program and running it in the emulator (or on hardware).

## Assembling and running programs

```sh
asm Programs/blocks.asm            # emits Intel HEX to stdout
asm Programs/blocks.asm -s         # also emit a symbol table
asm Programs/blocks.asm > out.hex  # capture machine code
```

`Programs/send.sh` streams a source/text file line-by-line to the machine over `/dev/ttyUSB0` with a per-line delay (the machine needs time to decode each line). Change the device path for your setup; the machine is also driven from a terminal emulator (e.g. Tera Term) over USB-to-serial.

## Assembly language conventions

The assembly dialect (see any file in `Programs/`) is whitespace-driven and dense — multiple instructions per line are normal and idiomatic. Key points when editing `.asm`:

- **Directives:** `#org <addr>` (set assembly address), `#include "file"`, `#emit` (emit raw bytes), `#mute` (suppress output). Comments start with `;`.
- **64-instruction ISA** with the A and B ALU registers, a hardware stack, three flags (zero / negative / carry), conditional branches (`BCC`, `BEQ`, `BNE`, …), subroutines (`JPS`/`JPA`), and 8-bit + 16-bit ("word", e.g. `INW`, `LDW`) operations.
- `<sym` / `>sym` take the low / high byte of a 16-bit symbol (used to load pointers a byte at a time). Stack is passed via `PHS`/`PLS`.
- The OS (`os.asm`) begins with a **jump table** of `_`-prefixed entry points (`_Start`, `_Print`, `_LoadFile`, `_FlashWrite`, …) at fixed low addresses — programs call OS routines through these stable labels, so do not reorder the jump table.

## Editing microcode

`microcode_def.csv` is the source of truth for the instruction set: each line `#define <MNEMONIC> <signal>|<signal>, …` lists the control signals asserted per micro-step (16 steps max). The control signal mnemonics (`RO` RAM-out, `RI` RAM-in, `AI`/`AO` reg-A in/out, `EO` ALU-out, `MIL`/`MIH` memory-address-in low/high, `IC` instruction-counter clear, etc.) are defined by the hardware. Changing instruction behavior means editing the microcode here, regenerating `microcode_rom.csv` and the `ctrl_*.bin` ROM images, and keeping the emulator's copy (`Support/Emulator/src/microcode_rom.csv`) in sync.

## Other notes

- `Clock_ATmega328p/Clock_ATmega328p.ino` is the Arduino sketch for the ATmega328p clock generator (adjustable single-step–8MHz).
- Licensing is **per-component** (hardware and software parts carry individual licenses; MinOS2 is GPLv3). See `DISCLAIMER.md` and `README.md`. This is non-commercial; selling it violates the license.
- Authoritative external docs: the manual (Google Doc linked in `README.md`) and the author's YouTube channel.
