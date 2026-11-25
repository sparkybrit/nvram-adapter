# nvram-adapter

A small KiCad 10 board that puts a battery-backed 1M × 8 SRAM on a breadboard, **in the place and with nearly the
pinout of the ROM/RAM breadboard adapter** (`../rom-ram-breadboard-adapter`,
[GitHub](https://github.com/sparkybrit/rom-ram-breadboard-adapter)) as it is used in Sparky1: there, the adapter
carries a Dallas DS1250AB-70 NVSRAM, a 512K × 8 SRAM with its lithium cell and power-fail control bonded into
one DIP-32 package. The DS1250 is no longer made, so this board builds the same thing from parts that are:

- the SRAM used on the sister SRAM board,
- a supervisor chip that does the DS1250's power-fail switching and write protection,
- a CR2032 in a holder.

Built for Sparky1 (`../sparky1`, [GitHub](https://github.com/sparkybrit/sparky1)), the MC68030 breadboard
computer, where the NVRAM holds the boot image at `0x01000000` and is written in-circuit by `../nvram-programmer`.
Sparky1's `CLAUDE.md` describes how the NVRAM is used.

It is also **the prototype for the NVRAM and reset circuit of `../sparky1-backplane`'s NVRAM card**: U2's `/RESET`
goes off the board on J2.12, SW1 is a manual reset, and the whole 1 MB is addressed from the bus (A19 on J1.20),
so all of it is proven here first (2026-11-25).

**It is no longer a drop-in for Sparky1 as wired** (see [The interface](#the-interface-the-rom-ram-adapters-pins-with-four-changes)):
J1.20 is A19, not +5 V, so +5 V must come in on J2.18 or J3, and the breadboard row at J1.20 must be rewired first.

Public repo: <https://github.com/sparkybrit/nvram-adapter>. Started 2026-11-24.

## Parts

| Ref | Part | Notes |
|---|---|---|
| U1 | Lyontek **LY62W10248ML-55LLI**, 1M × 8 SRAM, TSOP-II-44 | As on `../sram-breadboard-adapter` (8 of them there, proven over 127 h of testing). 55 ns, 2.7–5.5 V, data retention down to 1.5 V. Symbol and footprint copied from that repo. |
| U2 | Maxim **MAX6365PKA46+T**, SOT23-8 | Supervisor with battery switchover and chip-enable gating. `46` = 4.50/4.63/4.75 V threshold; `P` = open-drain active-low `/RESET`; MAX6365 = the manual-reset variant. Bought from DigiKey, November 2026. |
| BT1 | MPD **BU2032SM-BT-GTR** SMD CR2032 holder | Bought from DigiKey, November 2026. KiCad has no footprint for it, so `lib/nvram-adapter.pretty` has one drawn from MPD's drawing (rev G). |
| J1, J2 | 1×20 2.54 mm headers | The ROM/RAM adapter's interface, with four pins changed. See below. |
| J3 | Phoenix MKDS 1,5/2-5.08 screw terminal | A +5 V input, alongside J2.18 (either will do). Pin 1 (upper) +5 V, pin 2 (lower) GND, labelled `+5V` and `GND` on the silkscreen above and below it. Wire entry faces the left board edge. |
| SW1 | C&K **PTS810** SMD tactile switch (KiCad `Button_Switch_SMD:SW_SPST_PTS810`), 4.2 × 3.2 mm | Manual reset: pulls U2's `/MR` to GND. Labelled `RST` on the silkscreen. Not yet bought; any switch on that footprint will do. |
| D1, R2 | Green LED, 0603; 1 kΩ, 0603 | Power LED on +5 V (about 3 mA), labelled `PWR`. On +5 V, not VRAM, so it never draws on the cell. |
| C1–C5, R1 | 0603/0805 on top, 0402 on the bottom | See [Power](#power). |

Datasheets are in the private `../datasheets` repo ([GitHub](https://github.com/sparkybrit/datasheets)), each as the
PDF plus a Markdown transcription — read the `.md`:

- `Lyontek LY62W10248/`
- `Maxim MAX6365/` (19-1658 Rev 5, 10/11)
- `MPD BU2032SM-BT/` (drawing BU2032SM-BT-GTR rev G, 08/18/23)
- `Dallas DS1250AB/`, the part this board replaces

## The interface: the ROM/RAM adapter's pins, with four changes

J1 and J2 are at the ROM/RAM adapter's exact pad positions (J1 pin 1 at (170.175, 49.53) mm, J2 pin 1 at
(170.175, 67.31), both running to pin 20 at x = 121.915). The board is the adapter's 20.65 mm high, with the same right-hand edge,
but shortened at the left to 99.735 mm (2026-11-25). The
nets match its interface file `module/interfaces/ROM_RAM_ADAPTER.json` except on four pins, changed on 2026-11-25
(bold):

| | Pins |
|---|---|
| J1 | 1–19 `A0`–`A18`; **20 `A19`** (the adapter's `+5V`) |
| J2 | 1, 2 `GND`; 3–10 `D0`–`D7`; 11 `GND`; **12 `/RESET`** (the adapter's `/RAMCE`); 13 `/ROMCE`; 14 `GND`; **15 NC** (`/RAMOE`); 16 `/ROMOE`; 17 `GND`; **18 `+5V`** (`/RAMWE`); 19 `/ROMWE`; 20 `GND` |

**The pins carry what their names say.** The ROM/RAM adapter's U1 socket is drawn for an SST39SF040, which
differs from the DS1250 on pins 3, 29 and 31, so Sparky1's fitted v1 adapter is bodged so that J1.15 carries A14,
J1.16 A15 and J2.19 `ROMWE` the DS1250's `/WE` (Sparky1's `CLAUDE.md`, Hardware table). This board needs no bodge:
J1.15 is A14, J1.16 is A15 and J2.19 is `/WE`, exactly as the breadboard wiring already expects.

Only the ROM strobes are used, because in Sparky1 the DS1250 sits in the adapter's U1, the ROM socket, where
`/ROMCE` carries the boot ROM's chip select (`/ROMCS`, `0x01000000`) and `/ROMWE` carries CPU R/W. The RAM
strobes (the ROM/RAM adapter's empty U2 socket) are unconnected on Sparky1 (its `schematic/WIRING.md` lists J2.12,
J2.15 and J2.18 as "U2 is empty"), so this board reuses them: **J2.12 carries `/RESET`**, **J2.18 is a +5 V input**
and **J2.15 is not connected**. Nothing on Sparky1's breadboard reaches J2.12 until a wire is added from it to the
68030's `/RESET`, nor J2.18 until a wire brings it +5 V.

**+5 V can come from the breadboard on J2.18 or from the screw terminal J3**; they are the same net, so the board
can be powered either way without wiring J3 (2026-11-25). J2.18 sits between `/WE` (J2.19) and `GND` (J2.17).
The board draws at most about 65 mA (SRAM 60 mA, LED 3 mA, U2), well within one header pin and breadboard contact.

**J1.20 is SRAM A19**, so the whole 1 MB is addressed from the bus (the solder jumper that used to pick a 512K half
is gone). **Before fitting this board to Sparky1, rewire the breadboard row at J1.20**: today it is +5 V. Left as
it is, it selects the upper half; worse, with neither J2.18 nor J3 wired the board is unpowered while +5 V sits on an SRAM input,
which back-powers VRAM through the SRAM's input clamp and can push current towards the cell. Wire J1.20's row to
the CPU's A19 (or to GND for the lower half) and +5 V to J2.18's row (or J3). Whether Sparky1's `/ROMCS` decode covers a full 1 MB
is Sparky1's business: with a 512 KB window A19 just picks the half.

## How it works

```
J3 +5V ──► U2 VCC        OUT ──► VRAM ──► U1 VCC, CE2, C3–C5
CR2032 ──► U2 BATT
/ROMCE ──► U2 /CE_IN  /CE_OUT ──► U1 /CE      /ROMOE ──► U1 /OE      /ROMWE ──► U1 /WE
```

- **Normal operation** (+5 V above the threshold): U2 connects VCC to OUT (3.1 Ω, up to 150 mA; the SRAM takes
  60 mA at most) and passes `/ROMCE` through a transmission gate to the SRAM's `/CE` in 1.5–7 ns.
- **Power failing** (+5 V below 4.50–4.75 V, typ 4.63): `/RESET` asserts and the chip-enable gate opens: `/CE_RAM`
  is pulled up to OUT, so the SRAM ignores the bus. If a cycle is in progress, `/CE_OUT` is held low for 12 µs
  first so it can finish. This is the DS1250AB's write protection (its V<sub>TP</sub> is 4.50/4.62/4.75 V, the
  same window).
- **Backup**: once VCC has also fallen below VBAT, OUT switches to the cell. `/CE_RAM` sits at OUT, which meets
  the SRAM's data-retention condition (`CE# ≥ VCC − 0.2 V`).
- **Power-up**: the reverse, with `/RESET` (and so the gate) held for 150–280 ms after VCC passes the threshold.
  **U2 never starts from the battery alone**: a new board must see +5 V once before it will back anything up.

**CE2 is tied to VRAM**, so `/CE_RAM` alone selects the chip. **A19** comes from J1.20.

**Reset.** SW1 pulls `/MR` low (internal 20 kΩ pull-up, debounced inside U2, so no capacitor). `/RESET` then
asserts while it is held and for 150–280 ms after release, and goes off the board on **J2.12**.
`/RESET` is open-drain, so it wire-ORs with the 68030's own `/RESET` (which the CPU drives during a `RESET`
instruction) and with the backplane's 74LS07. R1 (100 kΩ) pulls it up to **+5 V, not VRAM**, which keeps it from
drawing on the cell; the system's own, stronger pull-up does the real work. Two consequences:

- **A manual reset also write-protects the SRAM**: `/MR` asserts reset inside U2, which opens the chip-enable gate,
  so the SRAM ignores the bus for as long as `/RESET` is low. The CPU is held in reset over the same time.
- **Sink current.** The MAX6365 specifies `/RESET`'s V<sub>OL</sub> (0.3 V max) only up to 1.6 mA, that is a pull-up
  of 3.3 kΩ or more to 5 V. The backplane's 4k7 (1.1 mA) is within it; the 1 kΩ the breadboard needed (5 mA) is
  not, and V<sub>OL</sub> there is unspecified. If the bus keeps 1 kΩ, buffer `/RESET` through an open-drain gate
  (a 74LVC1G07, or the backplane's 74LS07), or check V<sub>OL</sub> on the scope against the 68030's V<sub>IL</sub>.

Timing is better than what it replaces: the SRAM is 55 ns plus at most 7 ns in the gate, against the
DS1250AB-70's 70 ns.

### Battery life

Backup current is the SRAM's data-retention current plus U2's: the LLI grade's `IDR` is 4 µA typical, 50 µA
max over temperature (at 25 °C much less than the max), and U2's `IBACK` under 1 µA typical, 3 µA max. A CR2032
holds about 220 mAh, so roughly **five years at typical currents and six months at the worst-case figures**. The
DS1250 promises ten years. If that matters, the SL grade of the LY62W10248 (3 µA typ, 10 µA max at 25 °C) is the
part to buy; it is a drop-in. The power LED is on +5 V and costs the cell nothing.

The cell can be changed with +5 V applied without losing data (U2 does not switch while VCC is above the threshold).

## Power

- **+5V** comes in on J2.18 or J3, and reaches U2's VCC, C1, C2, R1 and R2/D1; it is routed as tracks.
- **VRAM** (U2's OUT) is the whole of **In2.Cu**. U1's two VCC pins, CE2 and C3–C5 reach it by via. Until the
  board is routed nothing reaches it at all, so DRC reports the plane as isolated copper.
- **GND** is the whole of **In1.Cu**, plus fills on F.Cu and B.Cu.

**Decoupling follows the SRAM board's lesson.** On `../sram-breadboard-adapter` the 100 nF capacitors sat off the
end of each package; that gave an occasional single-bit read error until 0402s were hand-soldered across each
chip's supply pins (Sparky1's "Improve decoupling capacitor placement"). So here C4 and C5 (100 nF 0402) are on
the **bottom side, directly under U1's two VCC/VSS pairs** (pins 11/12 and 33/34), C3 (10 µF) is at U2's OUT, C1
(100 nF) at U2's VCC and C2 (10 µF) is +5 V bulk.

**Under the cell holder.** The holder has a plastic floor between the cell and the board, and MPD's drawing asks for
no keep-out (it marks the centre as an optional area for adhesive), so copper may run under BT1. The 20.6 mm
no-copper circle that used to be there was removed on 2026-11-25. One rule area remains, `BT1 spring finger`
(about 8.5 × 7 mm, from 4.5 mm right of BT1's centre to pad 2): it keeps tracks and vias, but not the GND fill,
out from under the holder's negative spring, which sits over a window in the floor and can be pushed down onto
the board by the cell.

## The board

- **Outline:** 99.735 × 20.65 mm, x 72.35–172.085, y 48.082–68.732. The ROM/RAM adapter's outline (which starts at
  x 63.525) with its left end cut back as far as the parts allow, so it still drops into the same place on the
  breadboard and takes 8.8 mm less of it.
- **Four layers:** F.Cu signals + GND fill, In1.Cu GND plane, In2.Cu VRAM plane, B.Cu signals + GND fill.
- **Rules:** 0.2 mm tracks and 0.15 mm clearance by default (0.5 mm for the Power netclass: `+5V`, `GND`, `/VRAM`,
  `/VBAT`), 0.6/0.3 mm vias, 0.3 mm copper to edge.
- **Placement**, left to right, packed courtyard to courtyard (about 0.1 mm gaps): J3 at the left edge; SW1;
  BT1 (+ on the left), its courtyard ending just short of the headers' pin 20; then, between the header rows, C2,
  R1, U2 with C1 and C3, U1 (pin 1 bottom left, pins 1–22 facing J2), R2 and D1. The holder's body would fit
  between the header rows, so BT1 could go about 4 mm further right, but only by overlapping the headers'
  courtyards and hiding the pin 19–20 names. U1 fits between the rows with 1.7 mm to spare on each side.

### Status

**The schematic is complete; the board is placed, with its planes and fills, but not routed** (50 unconnected
items at the last DRC). With no autorouter installed, routing is to be done interactively in KiCad's PCB editor.
U1's address and data pins are mixed on both of its sides, so expect to cross between F.Cu and B.Cu; the space
under U1 on B.Cu is free apart from C4 and C5.

## Checks

```sh
kicad-cli sch erc nvram-adapter.kicad_sch                                  # expect 0 errors, 0 warnings
kicad-cli pcb drc --schematic-parity --severity-all nvram-adapter.kicad_pcb # expect 0 parity issues
```

Expected DRC warnings: silkscreen clipped by the board edge at the headers (their pads sit 0.6 mm from the edge,
as on the ROM/RAM adapter), U1's own pin-1 marker text overlapping itself, two `lib_footprint_mismatch` for J1 and
J2 (their silkscreen outlines, and J2's pin-1 corner, were deleted on this board to make room for the pin names),
and, until routed, `isolated_copper` for the VRAM plane. Anything else is new.

**Pin names on the silkscreen**: every header pin is named in 0.8 mm text in the 1.7 mm between the header pads and
U1's pads, as the ROM/RAM adapter does. The strobes are shortened to `~{CE}`, `~{OE}`, `~{WE}` and `~{RST}` (J2.12)
to fit 2.54 mm. Mock-ups for checking by eye: `kicad-cli pcb export pdf --mode-single --layers
F.Cu,F.SilkS,Edge.Cuts --scale 2.5 -o prints/nvram-adapter-mockup.pdf nvram-adapter.kicad_pcb` (`prints/` is
gitignored).

**Editing the board by script**: `pcbnew`'s `BOARD.Remove()` and `FOOTPRINT.Remove()` segfault in KiCad 10.0.8's
Python module (2026-11-25), so remove items by editing the file through `lib/kicad-tools/tools/sexp.py`, and use
`pcbnew` only to add, move and refill.

Before ordering:

- Route, then DRC with **no** unconnected items.
- Re-check the BU2032SM footprint against MPD's drawing and a real holder (pads 3.20 × 4.20 mm at 29.30 mm
  centres, + on the left). It was drawn from the drawing only.
- Check the headers' position against a fitted ROM/RAM adapter or its `.kicad_pcb`.
- Recompare J1/J2 with `ROM_RAM_ADAPTER.json`: load the board with KiCad's `pcbnew` Python module (system
  `python3`), and for each connector in the JSON check each pad's net name (less the leading `/`) against it.
  Expect exactly four differences, the deliberate ones: J1.20 `A19` for `+5V`, J2.12 `~{RESET}` for `~{RAMCE}`,
  J2.15 unconnected and J2.18 `+5V` for `~{RAMWE}`.
- Settle the bus `/RESET` pull-up against U2's 1.6 mA sink (see [Reset](#how-it-works)).

## Files

| Path | What |
|---|---|
| `nvram-adapter.kicad_pro`, `.kicad_sch`, `.kicad_pcb` | The project. One sheet. |
| `lib/nvram-adapter.kicad_sym` | `LY62W10248ML-55LLI` (copied from the SRAM board, footprint and datasheet fields updated) and `MAX6365PKA46` (new). |
| `lib/nvram-adapter.pretty/` | `TSOP-II-44_400MIL_LTK` (copied from the SRAM board) and `BatteryHolder_MPD_BU2032SM-BT_1x2032` (new). |
| `lib/kicad-tools/` | Submodule, [kicad-tools](https://github.com/sparkybrit/kicad-tools): the sister boards use it for their `module/` interface symbols. Its `kisch.py` wrote the first version of this schematic; its `sexp.py` reads and writes KiCad files. |
| `prints/` | Gitignored: 1:1 prints and mock-ups for checking parts and layout by eye. |

The schematic and board were first generated by scripts (2026-11-24) and are now edited in KiCad like any other
project; the KiCad files are the source.

## To Do

- **Route the board** (see [Status](#status)).
- **Add a `module/` interface**, as the sister boards have (`module/modules.toml`, `make -C module`), so
  Sparky1's schematic can use this board's own symbol. Its pins would be `ROM_RAM_ADAPTER`'s, with J1.20 `A19`,
  J2.12 `/RESET`, J2.15 NC and J2.18 `+5V`.
- **Decide the SRAM grade** (LLI as stocked, or SL for longer battery life), see [Battery life](#battery-life).
- **Bring-up on the bench**: with +5 V on J3, D1 lights, VRAM ≈ +5 V (at C3) and `/RESET` high (J2.12); remove
  +5 V and check VRAM follows VBAT and `/CE_RAM` follows VRAM (at U2 pins 6 and 8); program an image with
  `nvram_write`, power-cycle, and `nvram_read` it back. Press SW1: `/RESET` low for 150–280 ms after release,
  `/CE_RAM` high meanwhile.
- **Rewire Sparky1's breadboard at J1.20** (+5 V today) before fitting, see [The interface](#the-interface-the-rom-ram-adapters-pins-with-four-changes).
- **Update Sparky1's notes** once fitted: its Hardware table and Programming the NVRAM section name the DS1250AB.
