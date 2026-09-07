# Silicon Dreams · Module 3 · Tape-out Starter

[![Lvs](../../workflows/lvs/badge.svg)](../../actions/workflows/lvs.yml)
[![Drc](../../workflows/drc/badge.svg)](../../actions/workflows/drc.yml)
[![Timing](../../workflows/timing/badge.svg)](../../actions/workflows/timing.yml)
[![Docs](../../workflows/docs/badge.svg)](../../actions/workflows/docs.yml)
[![GDS](../../workflows/gds/badge.svg)](../../actions/workflows/gds.yml)
[![FPGA](../../workflows/fpga/badge.svg)](../../actions/workflows/fpga.yml)
[![Test](../../workflows/test/badge.svg)](../../actions/workflows/test.yml)

Week 3 of the ChipMango × ChipFoundry Silicon Dreams course (`CM-HW-101`). This starter is where the work you've done in Modules 1 and 2 becomes a real chip. By the end of the week you will push a tag called `v1.0.0-final`, upload a tarball to the ChipFoundry shuttle portal, and wait approximately twelve weeks for real silicon to arrive on your desk.

Unlike M2, this repo is **not** a patch on your M1 fork. Module 3 restructures the project around multi-module integration, the LibreLane digital-design flow, and the ChipFoundry shuttle submission pipeline. You install your hardened M2 elevator into `src/elevator.v` and integrate it alongside three new sub-modules you will write this week.

You've cleared the parallel shaft: your elevator recovers from reset pulses, survives single-event upsets, ignores corrupted input, and no longer bloats the die with a counter that only ever needed to count to twelve. Module 1 built it, Module 2 hardened it — this module ships it. The last piece of the story is AXIOM: the thing your fault-injection harness knocked out of the simulator in Module 2 never really left. It's waiting inside the black-box IP you're about to instantiate, and this week you finally have to design around it instead of just detecting it.

## What this module covers

Module 3 introduces four things Modules 1 and 2 didn't, plus the back half of the physical-design flow:

| Concept | Why it shows up this week | Chapter |
|---|---|---|
| Multi-module integration | The elevator stops being a solo act — handshake signals, reset domains, and pinout budgets all get real. | TB-M3-02 |
| Arbitration | Multiple floor requests at once means priority *and* fairness both matter; the naive mux from Module 1 isn't enough. | TB-M3-03 |
| Clock gating + power | The die has a power budget. Gated clocks cut dynamic power on idle sub-modules, and the tool flow has to be taught to honour it. | TB-M3-04 |
| Defensive IP wrapping | AXIOM is a black box you can't fully audit. You build the top level assuming it can misbehave. | TB-M3-05 |
| LibreLane / DRC | The physical-design flow you've been treating as magic, now running end to end. | TB-M3-06 |
| LVS + timing closure | The part where your design either is a chip, or is a story about why it isn't. | TB-M3-07 |
| Shuttle + post-silicon | Submission day, the portal, the bring-up kit, and what happens once the chips come back. | TB-M3-08 |

## What you build this week

| Sub-module         | What it does                                                    | Study guide |
|--------------------|--------------------------------------------------------------------|-------------|
| `src/top.v`        | tt_um wrapper. Binds all four sub-modules to the shuttle pinout. | SG-M3-02    |
| `src/arbiter.v`    | Round-robin with priority override. Has one deliberate bug.      | SG-M3-03    |
| `src/clock_gate.v` | ICG wrapper around `sky130_fd_sc_hd__dlclkp_1`.                  | SG-M3-04    |
| `src/axiom_shim.v` | Defensive wrapper for the adversarial AXIOM black-box.           | SG-M3-05    |

## What is in this repository

| Path | What it is |
|---|---|
| `src/top.v` | Top wrapper. Wires the elevator, arbiter, clock-gate, and AXIOM shim together. |
| `src/elevator.v` | Slot for your hardened M2 RTL — paste your fault-tolerant Module 2 design here. |
| `src/arbiter.v` | Round-robin-with-priority arbiter. Ships with one deliberate bug. |
| `src/clock_gate.v` | Thin ICG wrapper around `sky130_fd_sc_hd__dlclkp`. |
| `src/axiom_blackbox.v` | The AXIOM black-box stub. Do **not** modify — it's instantiated by name and bound at implementation time. |
| `src/axiom_shim.v` | Defensive wrapper around the black box. |
| `librelane/config.json` | Flow configuration — `CLOCK_PERIOD`, `SYNTH_STRATEGY`, `PL_TARGET_DENSITY_PCT`, etc. |
| `librelane/pin_order.cfg` | Pad-ring order — maps top-level ports to the `tt_um` pins. |
| `test/` | cocotb tests for the arbiter, clock gate, and AXIOM shim; the integration smoke test lives in `test/top/`. |
| `.github/workflows/` | `drc.yml`, `lvs.yml`, `timing.yml` — every PR must pass all three. |

## Flow and CI

Module 3 runs the **LibreLane 2025.04** digital-design flow end-to-end on every push. Three CI workflows must stay green for you to tag a final release:

- `.github/workflows/drc.yml` — DRC (Magic + KLayout) on the full GDS.
- `.github/workflows/lvs.yml` — LVS (Netgen) match on layout vs schematic.
- `.github/workflows/timing.yml` — OpenSTA setup + hold on the worst corner.

Each workflow POSTs a JSON payload to the ChipFoundry grader service, which tracks your iteration count and final rubric score. The flow version is pinned — the grader rejects any run that isn't `efabless/librelane:2025.04`, so don't edit that pin in `librelane/config.json`.

## How the week is paced

Unlike Module 1 (bottom-up, one module at a time) or Module 2 (per-floor escape room), Module 3 is wide-and-shallow for the first three days and deep-and-narrow for the next four:

| Day | Focus | Typical hours |
|---|---|---|
| 1 | Integration top-level wrapper + pinout contract (TB/SG-02) | 1.5 h |
| 2 | Arbiter (TB/SG-03) | 2.0 h |
| 3 | Clock gating + SDC (TB/SG-04) | 1.5 h |
| 4 | AXIOM black-box wrapping (TB/SG-05) | 1.5 h |
| 5 | First LibreLane run + DRC fixing (TB/SG-06) | 3.0 h |
| 6 | LVS + timing closure (TB/SG-07) | 3.5 h |
| 7 | Launch-day submission + reflection (TB/SG-08) | 1.5 h |

Total expected: ~14.5 hours over seven calendar days. If Days 5–6 are running much longer than that, come to office hours — LibreLane debugging is usually a one-line fix that takes four hours to find and forty seconds to apply.

## Local quick-start

```bash
# 1. Fork and clone
gh repo fork chipmango-design/silicon-dreams-m3-starter --clone --remote
cd silicon-dreams-m3-starter
git checkout -b my-tapeout

# 2. Install your M2 elevator (confirm it passes M2's smoke test first —
#    a broken reset or 2-bit state encoding will fail the integration
#    smoke test before you even reach the LibreLane flow)
cp ../silicon-dreams-m2/src/elevator.v src/elevator.v

# 3. Pull the pinned LibreLane image and PDK
docker pull efabless/librelane:2025.04
./scripts/install-pdk.sh          # first run only, ~20 min / ~4 GB
./scripts/run-librelane.sh --version-check

# 4. First hardening pass (expect DRC/LVS fails — this is the calibration run)
./scripts/run-librelane.sh --tag first-run

# 5. Integration smoke (six tests, all must pass before you iterate the flow)
cd test/top && make smoke
```

Prefer to run LibreLane natively instead of in Docker? SG-M3-01 §2 covers the Nix-based install (Linux, WSL2, and macOS/Apple Silicon) if Docker isn't an option on your machine.

Expected flow stages, in order — each should tick green:

| Stage | What it does |
|---|---|
| Synthesis (Yosys) | Runs `synth -flatten`; emits gate count and longest-path estimate. |
| Floorplanning | Creates the die outline, pad ring, and power grid. |
| Placement (OpenROAD) | Global + detailed placement; target density ~0.55–0.60. |
| Clock-tree synthesis | Builds the clock tree, inserts buffers, balances skew. |
| Routing (OpenROAD) | Global + detailed routing — watch for congestion warnings. |
| DRC (Magic + KLayout) | Design-rule checks against SKY130A. Must be zero violations. |
| LVS (Netgen) | Layout-vs-schematic. Must match. |
| Timing (OpenSTA) | Setup + hold against corners. Must meet `CLOCK_PERIOD`. |
| Antenna (Magic) | Antenna ratio checks. Should be zero. |
| GDS write | Final GDSII + LEF written to `runs/<tag>/final/`. |

The first LibreLane run is not supposed to be clean — it's a calibration run to learn which knobs move which reports. Don't try to fix DRC on run 1; that's typically run 6.

### Where reports live

```
runs/<tag>/
├── reports/
│   ├── synthesis/1-synth.stat   # gate count, area, longest path
│   ├── placement/                # density, wirelength
│   ├── cts/                      # clock tree skew, latency
│   ├── routing/                  # congestion, shorts
│   ├── signoff/
│   │   ├── drc.rpt               # <-- your eyes live here
│   │   ├── lvs.rpt
│   │   ├── sta-best.rpt
│   │   └── sta-worst.rpt
│   └── metrics.csv               # one-line summary for CI parsing
└── final/
    ├── gds/top.gds
    ├── lef/top.lef
    └── def/top.def
```

If a stage fails, the run exits non-zero and the last report file written tells you where to look.

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `LibreLane version mismatch` | You pulled the wrong image. Explicitly pull `efabless/librelane:2025.04` and re-run — the grader rejects any other version. |
| `PDK sky130A not found` | First-time setup needs `./scripts/install-pdk.sh`. Takes ~20 min and ~4 GB disk. |
| Synthesis times out | Your elevator RTL likely has a combinational loop or an infinite-depth `always` block. Re-run your M2 smoke test to confirm the RTL is clean before blaming the flow. |
| Routing emits 100+ shorts | `PL_TARGET_DENSITY_PCT` is too high. Lower it from 0.60 to 0.55 in `config.json`. |
| Magic exits on segfault | Magic needs Xvfb in headless mode. The run script wraps this automatically; if running manually, prefix with `xvfb-run`. |

## Adding the repo

**Starter repo** (your Week 3 workspace — a fresh repo, not an extension of your M1/M2 fork):

```
https://github.com/chipmango-design/silicon-dreams-m3-starter.git
```

```bash
gh repo fork chipmango-design/silicon-dreams-m3-starter --clone --remote
cd silicon-dreams-m3-starter
git checkout -b my-tapeout
```

**Solution repo** (optional, to cross-check your work, understand the intended integration behaviour, and verify the complete tape-out flow — just like Modules 1 and 2):

```
https://github.com/chipfoundry/silicon_dearms.git
```

(mirrored at [github.com/chipmango-design/silicon_dreams](https://github.com/chipmango-design/silicon_dreams))

```bash
git clone https://github.com/<your-username>/silicon_dreams.git
cd silicon_dreams

# Switch to the Module 3 reference branch
git checkout mod3

# Run the full flow to see the expected PASS results
./scripts/run-librelane.sh   # expect: all stages pass with zero violations
```

Compare individual files against your own work, e.g.:

```bash
diff test/top/test_integration.py /path/to/silicon-dreams-m3-starter/test/top/test_integration.py
```

## Why this module matters

AXIOM is a plot device, but what it stands in for is real. Every chip you ship in your career will contain black-box IP someone else wrote, under assumptions you can't fully audit — a DSP core, a serialiser, a PLL. The vendor hands you an encrypted netlist and a datasheet; you integrate it and hope the datasheet is right.

That's not a story problem, it's the day job. The work is the same whether the black box is malicious or just buggy: wrap it defensively, isolate its reset and clock, clamp its outputs, and test the chip as if the IP could do anything. If you take one skill out of this module, make it this — when someone hands you a black box, you wrap it before you trust it.

## AXIOM

AXIOM is the adversarial IP block you wrap in the shim (`src/axiom_shim.v`). It's delivered as an encrypted binary bound by the grader at P&R time; the behavioural stub at `src/axiom_blackbox.v` is used for local simulation and must not be modified. Three misbehaviours are published in TB-M3-05; your shim's job is to contain all of them. There is also a documented easter egg on the command sequence `0x4, 0x7, 0x2` — treat it as a Chekhov's gun, rule #3 already handles it.

## Deliverables

Pushed as a release on tag `v1.0.0-final`:

1. `src/*.v` — your four sub-modules plus the integrated top.
2. `librelane/config.json` + `librelane/constraints.sdc` — the flow config.
3. `runs/final/final/gds/top.gds` — the GDSII that goes to fab.
4. `runs/final/final/lef/top.lef` — the LEF for the shuttle integrator.
5. `info.yaml` — shuttle metadata. Author and Discord fields filled in.
6. `notes/escape-log.md` — your per-module post-mortem.
7. `notes/m3-reflection.md` — six reflection prompts, 500 words.

## Course context

- Course code: **CM-HW-101**
- Module: **M3 (Week 3 of 3)**
- Cohort: **2026-spring**
- Partners: **ChipMango × ChipFoundry**
- Shuttle: **ChipFoundry chipIgnite 2026-Q2**
- PDK: **SKY130A**, standard cell library **sky130_fd_sc_hd**
- LibreLane version: **2025.04** (pinned; CI rejects any other)

## Rubric (max 1700 XP)

| Section                                      | XP  |
|----------------------------------------------|-----|
| Clean DRC/LVS/timing on tag v1.0.0-final     | 500 |
| Arbiter fairness + 50 XP bonus               | 250 |
| Clock gating with measurable power drop      | 150 |
| AXIOM shim (all three misbehaviour tests)    | 250 |
| Reflection (500+ words)                      | 300 |
| Bring-up success at week 12                  | 150 |
| First-try shuttle accept                     | 100 |

Minimum pass: 1000 XP. Distinction: 1400 XP.

## Resources

- ChipFoundry platform docs — [chipfoundry.io/docs](https://chipfoundry.io/docs)
- Silicon Dreams course home — [chipmango.com/silicon-dreams](https://chipmango.com/silicon-dreams)
- LibreLane documentation — [librelane.readthedocs.io](https://librelane.readthedocs.io)
- SkyWater SKY130 PDK — [skywater-pdk.readthedocs.io](https://skywater-pdk.readthedocs.io)
- Course Discord — link distributed with enrolment.

## Licence

RTL and harness code are released under **Apache-2.0**. Course materials (`docs/`, `notes/`, study guides, videos) are **CC BY-NC-SA 4.0** by ChipMango × ChipFoundry.

---
*ChipMango × ChipFoundry · MoU Partnership 2026 · CM-HW-101-M3*
