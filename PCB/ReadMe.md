# LibreServo v4.0.0 — PCB

**This fork is at version 4.0.0.** Upstream LibreServo shipped v2.3.1 and had a v3 in
mind; the changes made here — new MCU family, CAN-FD alongside RS-485, an isolated
transceiver pair, and a hardware security stack — go past what that v3 was scoped to be,
so the fork takes the next major number rather than a v3.x that would collide with it.

**KiCad is the primary EDA tool for this fork.** Autodesk EAGLE is end-of-life and no
longer supported, so all new schematic and layout work happens in KiCad. The EAGLE
artefacts are retained for backward compatibility with upstream LibreServo and as
design history; they are frozen at the v2.3.1 they were forked from and are not
renumbered or updated.

## Layout of this directory

| Path | Version | Tool | Status |
| --- | --- | --- | --- |
| [`kicad/`](kicad/) | 4.0.0 | KiCad 9.0.2 | **Primary — edit here** |
| `LibreServo-v2.3.1.sch` / `.brd` | 2.3.1 | EAGLE 7.6.0 | Frozen, backward compatibility |
| [`old/`](old/) | 2, 2.1, 2.2 | EAGLE 7.6.0 | Frozen, superseded upstream revisions |
| [`Gerbers/`](Gerbers/) | 2.3.1 | — | Production output, EAGLE-era |
| [`datasheets/`](datasheets/) | — | — | Local copies of vendor documents |

`kicad/LibreServo-v4.0.0.kicad_sch` and `.kicad_pcb` were produced by KiCad 9.0.2's
EAGLE importer from the frozen `.sch`/`.brd`, then carried forward independently. Both
carry `rev 4.0.0` in their title block. The Gerbers have **not** been regenerated at
4.0.0 and must not be fabricated as if they had been.

### Reading the imported symbol library

The importer names symbols after the **EAGLE symbol**, not the EAGLE deviceset, so
`kicad/LibreServo-v4.0.0-eagle-import.kicad_sym` does not always agree with the part
number. Most notably `U1` uses the symbol `MSPM0G3519` while the fitted part is a
**MSPM0G3518-Q1** (`M0G3518QRHBRQ1`). Look parts up by reference designator, not by
value.

### Known import defect: `#U1`, `#U5`, `#U6`

The three parts that had no package in the EAGLE library — `U1` (MSPM0G3518-Q1), `U5`
(ADM2587E) and `U6` (ADM3055E) — imported as `#`-prefixed pseudo-components, the class
KiCad reserves for power flags. They are consequently **absent from the exported
netlist** and are not yet real board parts. Fixing this is the first layout task; see
[`MSPM0G3518-MCU-swap.md`](MSPM0G3518-MCU-swap.md) §6.2.

## Design-change records

Each MCU / interface upgrade pass has its own decision record. Superseded records are
kept with a banner rather than deleted.

- [`MSPM0G3518-MCU-swap.md`](MSPM0G3518-MCU-swap.md) — **current**: `U1` is a TI
  MSPM0G3518-Q1 in VQFN-32 (RHB).
- [`MSPM0G3507-MCU-swap.md`](MSPM0G3507-MCU-swap.md) — superseded.
- [`S32K144-MCU-swap.md`](S32K144-MCU-swap.md) — superseded.
- [`RS485-CANFD-TPM-upgrade.md`](RS485-CANFD-TPM-upgrade.md) — superseded.

Records written on or before 2026-08-03 state that EAGLE is the project standard and
that the final product must be delivered in EAGLE, and refer to the design as v2.3.1.
**Neither is true** as of 2026-08-04; where such a record and this file disagree, this
file governs.

## Bill of materials

`LibreServo-v2.3_BOM.txt` is the EAGLE-era BOM and is several MCU generations stale.
It still lists `U1` as an `STM32F301K8U6` and predates `ADM2587E` / `ADM3055E`. It
needs regeneration from the KiCad schematic, which cannot happen until the `#U1` /
`#U5` / `#U6` defect above is fixed.

![LibreServo v2.3.1 schematic, upstream](https://www.libreservo.com/sites/libreservo.com/files/imagenes/LibreServo-v2.3.1-esquema_wsmall.png)

## Attribution

Upstream board design © LibreServo, CC BY-SA 4.0 — see
[LibreServo](https://www.libreservo.com/en). This fork's modifications follow the same
licence. This file was rewritten by **Claude Opus 5** (Anthropic) on 2026-08-04 at the
direction of the human maintainer, who set both the EAGLE-to-KiCad policy and the
4.0.0 version number; the original upstream text described the `.sch`/`.brd` pair and
mislabelled it as KiCad.
