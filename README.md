# nvram-adapter

A battery-backed 1M × 8 SRAM board for a breadboard, in the place of the
[ROM/RAM breadboard adapter](https://github.com/sparkybrit/rom-ram-breadboard-adapter) when it carries a
Dallas DS1250AB NVRAM, which is no longer made.

- **U1**: Lyontek LY62W10248ML-55LLI, 1M × 8 SRAM in TSOP-II-44, the chip used on the
  [SRAM breadboard adapter](https://github.com/sparkybrit/sram-breadboard-adapter).
- **U2**: Maxim MAX6365PKA46+T: switches the SRAM to the coin cell when +5 V fails, and blocks chip enable below
  4.63 V, matching the DS1250AB's 4.62 V write protection.
- **BT1**: CR2032 in an MPD BU2032SM-BT-GTR holder.
- **J1, J2**: the ROM/RAM adapter's two 20-pin headers, in the same places and with the same pinout except for
  four pins: J1.20 carries A19 (not +5 V), J2.12 carries U2's open-drain `/RESET`, J2.15 is not connected and J2.18
  is a +5 V input. Every pin is named on the silkscreen.
- **J3**: a screw terminal for +5 V, for when it doesn't come from the breadboard on J2.18.
- **SW1**: a manual reset button on U2's `/MR`.
- **D1**: a power LED.

It is also the prototype for the NVRAM and reset circuit of the Sparky1 backplane's NVRAM card.

KiCad 10, four layers. Built for [Sparky1](https://github.com/sparkybrit/sparky1), a 68030 breadboard computer.

**Status:** schematic done, board placed but not yet routed.
