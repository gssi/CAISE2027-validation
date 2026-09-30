# Evaluation results — a guide for interpreting them

You are given the results of a scenario-based evaluation of a model-driven access-control system (PIP model studio
→ fleet controller → device twins on two controllers). Use this guide to interpret `results/rq1.csv`,
`results/rq2.csv` and the evidence under `runs/`. The protocol is `EVALUATION-PROTOCOL.md` (§1 research questions,
§6 RQ2 steps, §7 measurements, §10 threats, §11 results); this file is the short version.

## 1. What the system is and what was evaluated

- A **multi-level model** (metamodel O2 → domain model O1 → level-0 configuration O0) describes principals, roles,
  credentials (NFC tags), spaces with containment, permit/deny policies with time windows, devices, sensors, KPIs
  and goals. A **fleet controller** projects the model and serves each **device twin** only the knowledge of the space
  it guards; each twin decides taps locally (Mode B) and drives real actuators through a state machine.
- **RQ1** — a PIP configuration is reflected at runtime, in three domains (office, home, fitness), without code
  changes. 30 level-0 models (10 per domain; row 01 derived from the domain's seed, rows 02–10 generated
  deterministically). Four checks per model:
  - **RQ1.1 twin**: every probe tap is decided as the oracle predicts (reason code), on the twin and in the fleet audit;
  - **RQ1.2 fleet**: each device's bundle holds exactly the principals/roles allowed on its space; sensors, KPIs and
    goals known to the fleet are those of the model;
  - **RQ1.3 injection**: a fleet-level statechart reaches every twin and is enacted (beep at 1777 Hz);
  - **RQ1.4 override**: one twin runs its own statechart (1999 Hz) without affecting the other, then returns under
    fleet control.
- **RQ2** — a hardware change is detected, proposed as a model change by the modelling assistant, and its
  inconsistencies surfaced by the conformance gate. One model per domain, 8 steps (table in §4).
- **Oracle**: expected outcomes are derived from the intent (`intents/*.json`) by an independent implementation of
  the documented semantics, never from the code under test.

## 2. Campaigns (filter every analysis by the `run` column)

| `run` | file | mode | content |
|---|---|---|---|
| `sim-2026-09-23` | rq1.csv | sim | 30 configurations, mock devices, unattended |
| `rq2-2026-09-23` | rq2.csv | sim | 3 domains × 8 steps, unattended |
| `hw-2026-09-24` | rq1.csv | hw | rows office-01, home-01, fitness-01 on two Raspberry Pi controllers, 30 physical taps |
| `hw-2026-09-24` | rq2.csv | hw | 3 domains × 8 steps, bricklets plugged/unplugged by hand |

Simulation and hardware use the same code, images and settings; only the device layer differs (mock device-api vs
real Tinkerforge bricklets). Everything ran on one laptop (fleet + engines); the Pis only run the bricklet daemon.

## 3. `rq1.csv` — one row per configuration

| column | meaning |
|---|---|
| `row`, `domain`, `origin`, `mode` | configuration id (`office-03`), domain, `seed` or `generated`, `sim`/`hw` |
| `pass` | true iff the four checks below are all true |
| `rq1_1_twin`, `rq1_2_fleet`, `rq1_3_injection`, `rq1_4_override` | the four checks (§1) |
| `probes`, `probes_failed` | taps performed / taps whose decision differed from the oracle |
| `m1_commit_to_last_bundle_ms` | M1: model commit in the cloud → updated bundle on the last device |
| `m1a_commit_to_fleet_ms` | M1a: model commit → fleet serving the new version |
| `m2_apply_to_last_twin_ms` | M2: fleet apply of a statechart → active on the last twin |
| `m3_reader_to_actuation_ms_median`, `m3_samples` | M3: NFC reader event → first actuation, median over allowed taps |
| `loaded_at`, `notes` | load instant; free-text notes (read them for hardware rows) |

Decision reason codes a probe can expect: `ok`, `unknown_credential` (the principal holds no role allowed on that
device's space, so it is not even in the device's bundle), `no_role`, `revoked`, `expired`, `not_yet_valid`,
`outside_hours`, `stale_cache`, `restricted:<rule>`. The same card can legitimately be `ok` on one reader and
`unknown_credential` on the other: that is the scoping RQ1.2 checks.

## 4. `rq2.csv` — one row per step

| step | change | expected |
|---|---|---|
| 1 | plug the ambient-light bricklet (`illuminance`) on dev1 | office: A2 add sensor; home/fitness (no sensor types): A4 + B2 per untyped key (+ B3), then A2 |
| 2 | plug the sound bricklet (`decibel`, untyped everywhere) on dev1, then both on dev2 | A4 + B2[decibel] at the domain model, A2 for dev1, one A2 for dev2 creating 2 sensors |
| 3 | operator models `decibel.max`, `illuminance.avg` and a goal measuring it | no proposal; commit accepted |
| 4 | unplug a sensor nobody uses | sensor offline; no proposal (the assistant never proposes removals); deletion accepted |
| 5 | unplug one of two inputs of `decibel.max` | KPI disabled, then decided again on one input after the deletion |
| 6 | unplug the only input of `illuminance.avg`, measured by the goal | deletion **blocked**: 422 `not_conforming` with `P5.NO_INPUTS`, `P6.GOAL_OUT_OF_SCOPE` (and `C8.TOO_FEW`), version unchanged |
| 7 | replace dev2's controller (new device id) | dev2 unmapped; A1 "use dev2" re-binds it |
| 8 | re-plug a bricklet, unplug it before accepting | A2 appears, then disappears (`→` separates the two moments) |

Columns: `change`; `runtime_effect` / `runtime_pass` (what the fleet observed and whether as expected);
`proposal` / `proposal_pass` (assistant proposals as `RULE:DETAIL`); `commit` / `commit_pass` (at step 6 "pass"
means *blocked as required*); `m4_change_to_observation_ms` (M4: hardware change → the observation carrying it is
visible to the studio; empty at step 3); `pass`; `notes`. `runtime_effect` and `commit` are JSON **truncated to 200
characters** — read `runs/<run>/<domain>-rq2/stepN.json` for the full content.

## 5. Headline results

| | simulation | hardware |
|---|---|---|
| RQ1 | 30/30 configurations pass; 476 probes, 0 mismatches, all confirmed in the audit | 3/3 pass; 30 physical taps, 0 mismatches |
| RQ2 | 3/3 domains, 24/24 steps | 3/3 domains, 24/24 steps |
| M1 commit → last bundle | median 6.4 s (min 3.0 s; one 203 s outlier, a retried cloud pull in fitness-08) | 3.7, 7.0, 7.5 s |
| M1a commit → fleet | median 0.8 s (same fitness-08 outlier: 198 s) | 0.6–1.3 s |
| M2 apply → last twin | median 20 ms (11–35 ms) | 23–39 ms |
| M3 reader → actuation | median 7 ms (4–12 ms, 188 samples) | not valid (see §6); reader → decision median 0.65 s (0.63–1.70 s) |
| M4 change → observation | plug/unplug median 1.7 s (0.2–2.7 s) | plug/unplug median 4.4 s (18 changes, 14 within 0.1–6.5 s) |
| M4 controller replacement | 6.4–9.1 s | 13.1–20.1 s |

The step-6 commit was blocked with the expected diagnostics in all six domain runs (3 sim + 3 hw). The demo models'
checksum was unchanged after every campaign.

## 6. Caveats — read before drawing conclusions

1. **M3 on hardware is blank on purpose.** The runner took an actuation of the state machine's occupancy region
   (driven by live motion/sound/CO₂/light readings) that preceded the tap. Fixed after the run; the simulated M3 is
   unaffected (static mock readings). Use reader → decision (`decidedAt − readerAt` in
   `runs/hw-2026-09-24/*/rq1_1.json`) as the hardware latency figure.
2. **M4 is measured differently per mode.** Simulation: DB clock corrected by an offset estimated once (±~1 s), so
   values of a few hundred ms, even slightly negative, mean "within the same observation cycle". Hardware: laptop
   clock at both ends (device-api detection → runner reads the row, 250 ms polling). Do not compare the two
   distributions as if they were the same instrument; the fleet samples devices every 3 s, which dominates both.
3. **Four M4 hardware outliers** (10.7, 25.1, 57.7 s, home/fitness steps 5 and 8) have no established cause; 57.7 s
   matches the fleet's 60 s write back-off. Report medians.
4. **Timing figures are single-laptop.** Fleet and engines ran on one machine: M1–M3 exclude controller↔fleet LAN
   latency. The claims of RQ1/RQ2 rest on the pass/fail columns; the timings are context.
5. **Harness defects, not system defects.** Several earlier attempts failed on runner bugs (audit lookup, NFC re-reads,
   stale mounts, cleanup); they were fixed and are excluded from the CSVs. §11 of the protocol lists them. The
   recorded rows contain no failure of the system under test.
6. **Hardware constraint:** the bricklet daemon detects hot-plugged bricklets only on ports occupied at its start;
   the attended RQ2 run was prepared accordingly.
7. **Redaction:** physical tag UIDs are replaced by `TAG_1…TAG_4` / `TAG_OTHER`; names in the seed office row and
   the demo model names are neutralised (`davide`, `office-demo`). Element ids in the evidence are those of the
   models as run.

## 7. Drilling down

- A configuration's verdict: `runs/<run>/<row>/checks.json` (each check with expected / observed).
- Every tap: `runs/<run>/<row>/rq1_1.json` → `probes[]` with `reason` (expected), `result.reason` (twin),
  `audit.reason` (fleet), `result.readerAt / decidedAt / firstActuationAt`, `attempts`.
- Knowledge scoping: `rq1_2.json` (`bundles[]` users/roles per device, `sensors`, `kpis`, `projection`).
- Statecharts: `rq1_3.json` (`sig` frequencies per device, `m2`), `rq1_4.json` (`forked`, `back`).
- Timing sources: `load.json` (commit instant, clock offset), `switch.json` (M1, M1a).
- RQ2: `runs/<run>/<domain>-rq2/stepN.json` (runtime, proposal, commit, m4, notes), `setup.json`.
- Expected outcomes: `models/<row>.expected.json`; subjects: `intents/<row>.json`, `models/<row>.model.json`.

Importing: the CSVs are comma-separated UTF-8 with quoted JSON fields — use a real CSV parser
(`pandas.read_csv`), not a naive split.
