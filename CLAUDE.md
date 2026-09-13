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

## Bill of materials and front-panel LEDs

`BOM.csv` (repo root) is generated from the **PCB** (`KiCAD9/8-Bit CPU 32k.kicad_pcb`), the same source the Gerbers were exported from — **not** from `8-Bit CPU 32k.net`, which is a stale older revision and must not be used for ordering. The board is a 253 × 236 mm "blinkenlights" machine: ~207 components, of which **~98 are 5 mm THT LEDs**. After converting to KiCad 10, regenerate the BOM from the schematic (**Tools → Generate BOM**) and diff it against `BOM.csv` — trust the schematic for values, the PCB for what is physically placed.

The LEDs are all the same footprint (`LED_THT:LED_D5.0mm`), so colour is a **free build choice** used to make the panel readable. The reference designators / silkscreen values group them functionally — assigning one colour per group is recommended:

- **Control / microcode signals** (named LEDs: `AI AO BI BO EO RI RO MIL MIH CIL CIH COL COH TI TO ME FI II IC CE EC ES`) — these mirror the `microcode_def.csv` control lines; give them one dominant colour (e.g. **red**).
- **Data bus** bits — a second colour (e.g. **green**).
- **Address bus / program counter** bits — a third colour (e.g. **blue**).
- **Registers A/B contents** — a fourth colour (e.g. **amber/yellow**).
- **Flags and clock** (`C` carry, `N` negative, `CLK`) — a distinct accent colour (e.g. **white**) since they are watched constantly.

**As-built colours are captured in the schematic.** The bullets above are the design rationale; the actual per-LED colour was assigned on the PCB (stored in each LED footprint's `Description`, e.g. `LED GREEN, 2000mcd, 60°`). That colour has now been copied onto every LED **symbol** as a hidden **`Color`** property (all 98 LEDs), so it travels with the schematic-generated BOM the same way `MPN` does — **add `Color` to the BOM export field list**. The source of truth for colour is now the schematic `Color` field; the PCB footprint `Description` is the independent original it was derived from, and KiCad does *not* auto-sync the two, so if you recolour a LED, update the schematic `Color` field. The as-built distribution is **Green 75, Yellow 11, Red 7, White 4, Blue 1** — note the physical build uses green as the dominant bus/signal colour rather than the red-dominant scheme suggested above.

**LED sourcing (Element14 only).** LEDs are ordered exclusively from Element14, so every LED is identified by a **`Supplier Ref`** symbol property (the Element14 reference) — LEDs deliberately carry **no `MPN`**. (`MPN` is reserved for the socketed chips, which are bought as manufacturer parts, e.g. `SST39SF010A-70-4C-PHE`.) Add `Supplier Ref` to the BOM export field list alongside `Color` and `MPN`. Codes assigned so far:

| Colour | Qty | `Supplier Ref` (Element14) |
|---|---|---|
| Green | 75 | `L-53SGC` |
| Red | 7 | `2112111` |
| Yellow | 11 | `MCL053YD` |
| White | 4 | `C512A-WNN-CZ0B0151` |
| Blue | 1 | `C503B-BCS-CV0Z0461` |

All 98 LEDs are coded.

**LED drive currents (per-colour forward-voltage handling).** Traced from a fresh netlist (`kicad-cli sch export netlist`). The design **does** account for the different forward voltages of the LED colours — but in two different ways depending on the LED's role:

- **Bus / data / address / register LEDs (70 of them)** are all **green** and hang off the **`RN1`–`RN11` bussed 3.3 kΩ arrays** → ~0.85 mA each. All one colour, so a single array value is correct. These cannot be retuned per-LED (they share 8-way packs), but they never need to be — they're monochrome.
- **Named control / flag / clock LEDs (28 of them)** mix colours and each sits on its **own discrete resistor** (`R4`–`R33`), valued *by colour* so the high-Vf parts aren't starved:

| Colour | Signals | R | I (≈) | Vf |
|---|---|---|---|---|
| Yellow | AI BI RI CIL CIH MIL MIH TI FI II IC | 470 Ω | 6.2 mA | ~2.1 V |
| Blue | R (flag, D97) | 1 kΩ | 1.9 mA | ~3.1 V |
| Red | AO BO EO TO COH COL RO | 2 kΩ | 1.55 mA | ~1.9 V |
| White | ME CE EC ES | 2 kΩ | 0.95 mA | ~3.1 V |
| Green | C (flag), CLK, D1–D3 status | 3.3 kΩ | 0.85 mA | ~2.2 V |

So the high-Vf blue (1 kΩ) and white (2 kΩ) are deliberately given lower series resistors; blue actually ends up driven harder than the green banks. The tuning targets *desired brightness by function* more than strict equal-luminance — yellow at 6 mA visibly dominates, whites at ~1 mA are the most modest — but nothing is Vf-starved. **Any named LED can be rebalanced** by changing its discrete resistor (e.g. whites 2 kΩ → ~1.2–1.5 kΩ, or calm the yellows 470 Ω → ~1 kΩ); the green bus banks cannot (bussed arrays) but are monochrome so it's moot. Evaluate lit-up on the prototype before changing anything.

**`element14_order.csv`** (repo root) is the submit-ready Element14 parts list covering the whole board — one line per orderable part with the numeric Element14 **order code**, quantity (spares included), MPN, and a line note; upload it via Element14's parts-list import. It also includes DIP sockets, which are needed for the build but absent from the schematic BOM. The red LED's `Supplier Ref` is the numeric Element14 order code `2112111` (its Multicomp MPN is `703-0100`); the other colours' refs are manufacturer part numbers that Element14 also matches.

Order ~10% spare LEDs (≈110 total). Other order-sensitive items: **11× SIP-9 bussed resistor arrays** (must be the 9-pin common-bus type, not isolated) drive the LED banks; the four `SST39SF010` flash chips are identical parts (HSB/LSB/MSB/SSD are roles); memory is **AS6C1008** (128K SRAM, DIP-32 0.6″) and the **ATmega328P is on-board** (DIP-28 narrow 0.3″) with a 16 MHz crystal. Buy flash, RAM, MCU, and crystal genuine from a reputable distributor; 74HCxx logic, sockets, and passives can be sourced cheaply in bulk.

### Identifying the socketed chips (and their orderable parts)

All four flash chips are the **same physical part** — identity comes from the **silkscreen U-number** (the socket), not the chip marking. The "Value" field in `BOM.csv` names the family + role; the **`Footprint`** field (`DIP-32_W15.24mm` = DIP-32, 0.6″) is what binds the BOM to the board; you complete the match at order time by picking the **PDIP** package variant. `BOM.csv` now carries an explicit **`MPN`** column, and the same MPNs are stored as `MPN` symbol properties on these instances in the KiCad 10 schematic (so they survive BOM regeneration — add `MPN` to the BOM export field list).

| Ref | Sheet | Role | Orderable MPN (PDIP) |
|---|---|---|---|
| U7 | IR | control ROM **HSB** (`ctrl_hsb.bin`) | `SST39SF010A-70-4C-PHE` |
| U9 | IR | control ROM **MSB** (`ctrl_msb.bin`) | `SST39SF010A-70-4C-PHE` |
| U11 | IR | control ROM **LSB** (`ctrl_lsb.bin`) | `SST39SF010A-70-4C-PHE` |
| U15 | RAM | program/OS FLASH **SSD** (`flash.bin`) | `SST39SF010A-70-4C-PHE` |
| U17 | RAM | **AS6C1008** SRAM (128K×8) | `AS6C1008-55PCN` |
| U36 | IR | **ATmega328P** clock generator | `ATMEGA328P-PU` |

Package suffix matters: SST39SF010 ships in PDIP-32 (`-PHE`), PLCC (`NHE`), and TSOP (`THE`) — only **`-PHE`** fits the DIP socket. The plain non-A `SST39SF010` is obsolete; use the drop-in `SST39SF010A`.

## Other notes

- `Clock_ATmega328p/Clock_ATmega328p.ino` is the Arduino sketch for the ATmega328p clock generator (adjustable single-step–8MHz). The ATmega is **not** a CPU — it is purely the variable clock/baud source (Timer1 → CLK, Timer2 → UART bit-clock), driven by the SELECT/SINGLE buttons.
- **Programming U36 (ATmega):** the board has **no on-board ISP path** — verified from the PCB copper: MISO/SCK (PB4/PB5) are unrouted, MOSI/PB3 is repurposed as `UART_OSC` (drives U3), the USART pins (PD0/PD1) are unconnected, and RESET only carries its 10k pull-up (R3). So U36 must be **programmed off-board and socketed**: flash the sketch on a **DIP-socketed Arduino Uno R3** (Board = "Arduino Uno"), then swap the chip into U36 — the Uno already runs at 16 MHz so the external-crystal fuses are correct. (Uno R4 = Renesas, won't work; SMD-Uno chips aren't swappable.) A ZIF universal programmer (e.g. XGecu T48) is the alternative and also burns the `SST39SF010` ROMs — for that route export the sketch to `.hex` and set the 16 MHz crystal fuses.
- Licensing is **per-component** (hardware and software parts carry individual licenses; MinOS2 is GPLv3). See `DISCLAIMER.md` and `README.md`. This is non-commercial; selling it violates the license.
- Authoritative external docs: the manual (Google Doc linked in `README.md`) and the author's YouTube channel.
