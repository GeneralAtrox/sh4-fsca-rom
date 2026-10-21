# sh4-fsca-rom: where the Dreamcast's FSCA sine table comes from

The SH-4 CPU in the Sega Dreamcast has an instruction, `FSCA`, that returns the sine and cosine of a 16-bit angle. It
doesn't compute them exactly: a small fixed-point circuit evaluates a cubic around the nearest of 64 nodes, using
coefficients from a 17-word ROM. To match the console bit for bit, an emulator has to reproduce that circuit and that
ROM.

The circuit has been modelled already (see [Credits](#credits)), and three of each ROM word's four fields have closed
formulas. The fourth field, C0 (the sine at the node, in 26-bit fixed point), had no known rule. Models carry the 15
interior C0 words as constants measured from hardware. They are close to `round(sin(mπ/32) · 2^26)` but not equal: the
offsets run from −4 to +1 with no visible pattern.

**This repository gives the rule.** It rebuilds all 17 C0 words exactly from first principles. With the rule in place,
the circuit model reproduces the full 65,536-entry table bit for bit, in both of the CPU's rounding modes.

`fsca_rom.py` is the generator: one Python file, standard library only, about 220 lines.

## The rule

For node m = 1..16, let F be `sin(mπ/32)` rounded to float32 (24 significant bits, round to nearest even). The ROM's
exponent field E (0..3) splits the nodes in two halves, and each half derives C0 from F differently:

- **Nodes 1..7 (E ≥ 1): C0 is the smallest 26-bit word that rounds back to F.** That is, the smallest integer c for
  which float32(c · 2^-26) == F.
- **Nodes 8..16 (E = 0): C0 goes through the slope's fixed-point domain and back.** First F · π/2 is truncated to 23
  fraction bits and a 1 is jammed in below them: `M = 2 · floor(F · π/2 · 2^23) + 1`. Then
  `C0 = round(M · 8/π)`, to nearest even. That is M, read in units of 2^-24, scaled back by 2/π into units of 2^-26.

Node 0 is 0.

Things the rule explains with no special case:
- **The last word, C0[16] = 2^26 − 1.** For m = 16, F is exactly 1, M = 26,353,589, and M · 8/π = 67,108,863.32, so C0
  lands one below 2^26. Models had treated this as a clamp.
- **The large offsets at nodes 10, 11, 13 and 14** (+1, +1, +1, −4 from plain rounding). The jammed bit puts M on a
  coarse lattice. Scaled back by 8/π, that lattice lands C0 on a few specific values, and plain rounding of the sine
  would not.

## How strong is the evidence?

We guarded the search against finding a rule by luck. Every family of hypotheses was registered with its size before it
was tested. Each hypothesis was also tried on decoy targets (nearby made-up word sets), and a rule only counted if the
chance of a hit was under 1%.

- **Nodes 8..16:** strong. Nine words match exactly, several of them far from plain rounding. The chance of that
  happening by luck is about one in a million.
- **Nodes 1..7:** the rule is exact, but it says less about how the words were made. Several simpler rules also give
  nodes 1..4. Only nodes 5..7 tell the candidates apart.

Before finding the rule, we ran an exhaustive search. It scored every C0 within ±4,096 of plain rounding against 22
error criteria (max error, max ulp error, squared error, bias, correctly rounded count and others, on the raw and
rounded outputs). No criterion reproduced the hardware words. So the words weren't tuned node by node for the
circuit's error. The rule suggests that the designers took the float32 sine and converted it through the circuit's own
fixed-point formats.

## The other ROM fields and KC

These come from the existing model; the generator computes them from formulas:

- `L = round(sin · π/2 · 2^(22+E))`, where E (at most 3) is the largest shift that keeps L below 2^23.
- `Q = round(sin · 4π³/3 · 2^6)`, which is about sin · 2646.

The circuit also uses one global constant, the quadratic coefficient `KC = 0x648340` (102,925 · 2^6). **KC is still
credited.** We found that it is exactly the value minimising the circuit's maximum absolute error over all 65,536
angles, given the C0 rule above. It is a unique optimum. But other reasonable criteria land 2 to 4 steps away, so the
match didn't pass our 1% luck guard, and we don't claim it as the rule.

## Use

```bash
python fsca_rom.py                    # print the ROM (C0, L, E, Q per node) and the table's SHA-256
python fsca_rom.py --table sin.bin    # write the table
python fsca_rom.py --check dump.bin   # compare with a table of your own
python fsca_rom.py --rz               # the round-toward-zero table (FPSCR.RM = 1)
```

A table file is `sin(2π·a/65536)` for a = 0..65535 as float32 bit patterns, little-endian (256 KiB). FSCA's cosine is
`sin[(a + 0x4000) & 0xffff]`, so it isn't stored. A run takes about a second.

Expected SHA-256 of the table files:

| Mode | SHA-256 |
|---|---|
| Round to nearest (the default, FPSCR.RM = 0) | `500664c1e60659a3210c020db24e9c534beb6f8c6bea354e052424c2b91669bd` |
| Round toward zero (`--rz`) | `1eb396fa419325cafb4130a9d6d47e5deeb13ee40655ba386374f771393d5ef8` |

Both zero crossings (sin at 0 and at 0x8000) are +0, as on hardware.

## How it was checked

- **The ROM:** all 17 C0 words, and every L, E and Q word, equal the hardware-derived words in skmp's model.
- **The table:** with the generated ROM, both of the model's tables match ours on all 65,536 entries, comparing all 32
  bits (round to nearest and round toward zero). That model reports 0 mismatches against a real Dreamcast for every
  angle in both modes.
- **The math:** π and the node sines are computed to 60 digits, and every rounding uses exact fractions. No step
  depends on the host's floating point.

We don't have a Dreamcast to test on directly. If you do, `--check` against a dump of FSCA over all 65,536 angles is the
direct test.

## Credits

- **The circuit:** Hitachi's patents US 6,327,605 (the FSCA datapath, figs. 28–29) and US 6,038,582 (the shared
  multiplier block).
- **The bit-exact model:** Stefanos Kornilios Mitsis Poiitidis (skmp),
  [`sh4_fpu_approx.h`](https://gist.github.com/skmp/4bd7b38149fc455978b5104201573236), MIT license. It is verified
  against real hardware. It provides the multiplier and datapath model that `fsca_rom.py` re-implements in Python, the
  L/E/Q formulas, KC and the hardware C0 words we checked against. Its notice is in [LICENSE](LICENSE).
- **The C0 rule, and the KC minimax observation:** this repository.

## License

MIT. See [LICENSE](LICENSE).
