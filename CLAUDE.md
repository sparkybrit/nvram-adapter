# nvram-adapter

A small KiCad 10 board that puts a battery-backed 512K × 8 SRAM on a breadboard. It is a **drop-in replacement
for the ROM/RAM breadboard adapter** (`../rom-ram-breadboard-adapter`,
[GitHub](https://github.com/sparkybrit/rom-ram-breadboard-adapter)) as it is used in Sparky1: there, the adapter
carries a Dallas DS1250AB-70 NVSRAM, a 512K × 8 SRAM with its lithium cell and power-fail control bonded into
one DIP-32 package. The DS1250 is no longer made, so this board builds the same thing from parts that are:

- the SRAM used on the sister SRAM board,
- a supervisor chip that does the DS1250's power-fail switching and write protection,
- a CR2032 in a holder.

Built for Sparky1 (`../sparky1`, [GitHub](https://github.com/sparkybrit/sparky1)), the MC68030 breadboard
computer, where the NVRAM holds the boot image at `0x01000000` and is written in-circuit by `../nvram-programmer`.
Sparky1's `CLAUDE.md` describes how the NVRAM is used; nothing about that changes with this board.

Public repo: <https://github.com/sparkybrit/nvram-adapter>. Started 2026-11-24.

## Parts

| Ref | Part | Notes |
|---|---|---|
| U1 | Lyontek **LY62W10248ML-55LLI**, 1M × 8 SRAM, TSOP-II-44 | As on `../sram-breadboard-adapter` (8 of them there, proven over 127 h of testing). 55 ns, 2.7–5.5 V, data retention down to 1.5 V. Symbol and footprint copied from that repo. |
| U2 | Maxim **MAX6365PKA46+T**, SOT23-8 | Supervisor with battery switchover and chip-enable gating. `46` = 4.50/4.63/4.75 V threshold; `P` = open-drain active-low `/RESET`; MAX6365 = the manual-reset variant. Bought from DigiKey, November 2026. |
| BT1 | MPD **BU2032SM-BT-GTR** SMD CR2032 holder | Bought from DigiKey, November 2026. KiCad has no footprint for it, so `lib/nvram-adapter.pretty` has one drawn from MPD's drawing (rev G). |
| J1, J2 | 1×20 2.54 mm headers | The ROM/RAM adapter's interface. See below. |
| J3 | Phoenix MKDS 1,5/2-5.08 screw terminal | +5 V and GND, in parallel with J1.20 and J2's GND pins. Wire entry faces the left board edge. |
| JP1 | 3-pad solder jumper, 1–2 bridged | SRAM A19: 1–2 (default) to GND for the lower 512K, 2–3 to VRAM for the upper. Two images on one chip. |
| TP1–TP6 | Keystone 5000 test points | +5V, GND, VBAT, VRAM, `/RESET`, `/CE_RAM`, labelled on the silkscreen by net. |
| C1–C5, R1 | 0603/0805 on top, 0402 on the bottom | See [Power](#power). |

Datasheets are in the private `../datasheets` repo ([GitHub](https://github.com/sparkybrit/datasheets)), each as the
PDF plus a Markdown transcription — read the `.md`:

- `Lyontek LY62W10248/`
- `Maxim MAX6365/` (19-1658 Rev 5, 10/11)
- `MPD BU2032SM-BT/` (drawing BU2032SM-BT-GTR rev G, 08/18/23)
- `Dallas DS1250AB/`, the part this board replaces

## The interface: same pins as the ROM/RAM adapter

J1 and J2 are at the ROM/RAM adapter's exact pad positions (J1 pin 1 at (170.175, 49.53) mm, J2 pin 1 at
(170.175, 67.31), both running to pin 20 at x = 121.915), the board outline is the same 108.56 × 20.65 mm, and the
nets match its interface file `module/interfaces/ROM_RAM_ADAPTER.json` pin for pin (checked 2026-11-24, 0
differences):

| | Pins |
|---|---|
| J1 | 1–19 `A0`–`A18`, 20 `+5V` |
| J2 | 1, 2 `GND`; 3–10 `D0`–`D7`; 11 `GND`; 12 `/RAMCE`; 13 `/ROMCE`; 14 `GND`; 15 `/RAMOE`; 16 `/ROMOE`; 17 `GND`; 18 `/RAMWE`; 19 `/ROMWE`; 20 `GND` |

**The pins carry what their names say.** The ROM/RAM adapter's U1 socket is drawn for an SST39SF040, which
differs from the DS1250 on pins 3, 29 and 31, so Sparky1's fitted v1 adapter is bodged so that J1.15 carries A14,
J1.16 A15 and J2.19 `ROMWE` the DS1250's `/WE` (Sparky1's `CLAUDE.md`, Hardware table). This board needs no bodge:
J1.15 is A14, J1.16 is A15 and J2.19 is `/WE`, exactly as the breadboard wiring already expects.

Only the ROM strobes are used. `/RAMCE`, `/RAMOE` and `/RAMWE` (the ROM/RAM adapter's empty U2 socket) are
labelled so the interface names match, and go nowhere; ERC's three "label connected to only one pin" warnings
are these.

## How it works

```
+5V ──► U2 VCC        OUT ──► VRAM ──► U1 VCC, CE2, C3–C5
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

**CE2 is tied to VRAM**, so `/CE_RAM` alone selects the chip. **A19** comes from JP1. **`/MR`** is left open (internal
20 kΩ pull-up). **`/RESET`** is open-drain, so R1 (100 kΩ) pulls it up to **+5 V, not VRAM**, which keeps it from
drawing on the cell; it is for TP5 only, not wired off the board.

Timing is better than what it replaces: the SRAM is 55 ns plus at most 7 ns in the gate, against the
DS1250AB-70's 70 ns.

### Battery life

Backup current is the SRAM's data-retention current plus U2's: the LLI grade's `IDR` is 4 µA typical, 50 µA
max over temperature (at 25 °C much less than the max), and U2's `IBACK` under 1 µA typical, 3 µA max. A CR2032
holds about 220 mAh, so roughly **five years at typical currents and six months at the worst-case figures**. The
DS1250 promises ten years. If that matters, the SL grade of the LY62W10248 (3 µA typ, 10 µA max at 25 °C) is the
part to buy; it is a drop-in.

The cell can be changed with +5 V applied without losing data (U2 does not switch while VCC is above the threshold).

## Power

- **+5V** comes in on J1.20 or J3, and reaches only U2's VCC, C1, C2, R1, TP1 and J3; it is routed as tracks.
- **VRAM** (U2's OUT) is the whole of **In2.Cu**. U1's two VCC pins, CE2, JP1 pad 3 and C3–C5 reach it by via.
- **GND** is the whole of **In1.Cu**, plus fills on F.Cu and B.Cu.

**Decoupling follows the SRAM board's lesson.** On `../sram-breadboard-adapter` the 100 nF capacitors sat off the
end of each package; that gave an occasional single-bit read error until 0402s were hand-soldered across each
chip's supply pins (Sparky1's "Improve decoupling capacitor placement"). So here C4 and C5 (100 nF 0402) are on
the **bottom side, directly under U1's two VCC/VSS pairs** (pins 11/12 and 33/34), C3 (10 µF) is at U2's OUT, C1
(100 nF) at U2's VCC and C2 (10 µF) is +5 V bulk.

**No copper on F.Cu under the cell.** A CR2032's can is its positive terminal; a rule area keeps tracks, vias and
fill out of a 20.6 mm circle under BT1.

## The board

- **Outline:** 108.56 × 20.65 mm, x 63.525–172.085, y 48.082–68.732, copied from the ROM/RAM adapter so it drops
  into the same place on the breadboard.
- **Four layers:** F.Cu signals + GND fill, In1.Cu GND plane, In2.Cu VRAM plane, B.Cu signals + GND fill.
- **Rules:** 0.2 mm tracks and 0.15 mm clearance by default (0.5 mm for the Power netclass: `+5V`, `GND`, `/VRAM`,
  `/VBAT`), 0.6/0.3 mm vias, 0.3 mm copper to edge.
- **Placement**, left to right: J3 at the left edge; BT1 (+ on the left); the six test points; then, between the
  header rows, C2, R1, U2 with C1 and C3, U1 (pin 1 bottom left, pins 1–22 facing J2), JP1. U1 fits between the
  rows with 1.7 mm to spare on each side.

### Status

**The schematic is complete; the board is placed, with its planes and fills, but not routed** (51 unconnected
items at the last DRC). With no autorouter installed, routing is to be done interactively in KiCad's PCB editor.
U1's address and data pins are mixed on both of its sides, so expect to cross between F.Cu and B.Cu; the space
under U1 on B.Cu is free apart from C4 and C5.

## Checks

```sh
kicad-cli sch erc nvram-adapter.kicad_sch                                  # expect 3 warnings: the RAM strobes
kicad-cli pcb drc --schematic-parity --severity-all nvram-adapter.kicad_pcb # expect 0 parity issues
```

Expected DRC warnings: silkscreen clipped by the board edge at the headers (their pads sit 0.6 mm from the edge,
as on the ROM/RAM adapter), and U1's own pin-1 marker text overlapping itself. Anything else is new.

Before ordering:

- Route, then DRC with **no** unconnected items.
- Re-check the BU2032SM footprint against MPD's drawing and a real holder (pads 3.20 × 4.20 mm at 29.30 mm
  centres, + on the left). It was drawn from the drawing only.
- Check the headers' position against a fitted ROM/RAM adapter or its `.kicad_pcb`.
- Recompare J1/J2 with `ROM_RAM_ADAPTER.json`: load the board with KiCad's `pcbnew` Python module (system
  `python3`), and for each connector in the JSON check each pad's net name (less the leading `/`) against it.

## Files

| Path | What |
|---|---|
| `nvram-adapter.kicad_pro`, `.kicad_sch`, `.kicad_pcb` | The project. One sheet. |
| `lib/nvram-adapter.kicad_sym` | `LY62W10248ML-55LLI` (copied from the SRAM board, footprint and datasheet fields updated) and `MAX6365PKA46` (new). |
| `lib/nvram-adapter.pretty/` | `TSOP-II-44_400MIL_LTK` (copied from the SRAM board) and `BatteryHolder_MPD_BU2032SM-BT_1x2032` (new). |
| `lib/kicad-tools/` | Submodule, [kicad-tools](https://github.com/sparkybrit/kicad-tools): the sister boards use it for their `module/` interface symbols. Its `kisch.py` wrote the first version of this schematic. |

The schematic and board were first generated by scripts (2026-11-24) and are now edited in KiCad like any other
project; the KiCad files are the source.

## To Do

- **Route the board** (see [Status](#status)).
- **Add a `module/` interface**, as the sister boards have (`module/modules.toml`, `make -C module`), so
  Sparky1's schematic can use this board's own symbol. Its pins would be identical to `ROM_RAM_ADAPTER`'s.
- **Decide the SRAM grade** (LLI as stocked, or SL for longer battery life), see [Battery life](#battery-life).
- **Bring-up on the bench**: with +5 V applied, check VRAM ≈ +5 V and `/RESET` high (TP4, TP5); remove +5 V and
  check VRAM follows VBAT and `/CE_RAM` follows VRAM; program an image with `nvram_write`, power-cycle, and
  `nvram_read` it back.
- **Update Sparky1's notes** once fitted: its Hardware table and Programming the NVRAM section name the DS1250AB.
