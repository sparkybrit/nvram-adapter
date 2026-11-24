# nvram-adapter

A battery-backed 512K × 8 SRAM board for a breadboard: a drop-in replacement for the
[ROM/RAM breadboard adapter](https://github.com/sparkybrit/rom-ram-breadboard-adapter) when it carries a
Dallas DS1250AB NVRAM, which is no longer made.

- **U1**: Lyontek LY62W10248ML-55LLI, 1M × 8 SRAM in TSOP-II-44, the chip used on the
  [SRAM breadboard adapter](https://github.com/sparkybrit/sram-breadboard-adapter). A solder jumper (JP1) picks which
  512K half is used.
- **U2**: Maxim MAX6365PKA46+T: switches the SRAM to the coin cell when +5 V fails, and blocks chip enable below
  4.63 V, matching the DS1250AB's 4.62 V write protection.
- **BT1**: CR2032 in an MPD BU2032SM-BT-GTR holder.
- **J1, J2**: the ROM/RAM adapter's two 20-pin headers, in the same places and with the same pinout.
- **J3**: a screw terminal for +5 V, and six test points.

KiCad 10, four layers. Built for [Sparky1](https://github.com/sparkybrit/sparky1), a 68030 breadboard computer.

**Status:** schematic done, board placed but not yet routed.
