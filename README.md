# Evaluation

A scenario-based evaluation of the model-driven access-control system: a multi-level PIP model (metamodel O2 →
domain model O1 → configuration O0) is edited in the studio, projected by a **fleet controller**, and served to
**device twins** on two controllers that decide NFC taps locally and drive real actuators through state machines.

**Results spreadsheet:** [CAISE2027 EVAL](https://docs.google.com/spreadsheets/d/1znLYLLexdbQq_OJ7q0Nj0TeGUoLVpa_ZyKtKS9JBBqA/edit)
— sheet `rq1` (one row per configuration) and sheet `rq2` (one row per step). It holds the same data as
[`results/rq1.csv`](results/rq1.csv) and [`results/rq2.csv`](results/rq2.csv).

## Research questions

**RQ1 — A PIP configuration is reflected at runtime, in three domains, without code changes.**
30 configurations, 10 per domain (office, smart home, fitness centre): row 01 restates the domain's demo model,
rows 02–10 are generated deterministically. Code, images and settings are the same for every row; only the model
changes. A configuration passes only if all four checks pass:

| check | claim |
|---|---|
| RQ1.1 twin | every probe tap is decided as the configuration prescribes, on the twin and in the fleet audit |
| RQ1.2 fleet | each twin receives exactly the principals and roles allowed on the space it guards; the fleet knows the model's sensors, indicators and goals |
| RQ1.3 injection | a statechart defined at fleet level reaches every twin and is enacted by the device (beep at 1777 Hz) |
| RQ1.4 override | one twin runs its own statechart (1999 Hz) without affecting the other, then returns under fleet control |

**RQ2 — A hardware change is detected, proposed as a model change, and its inconsistencies are surfaced.**
One model per domain, eight steps: plugging and unplugging sensor bricklets, modelling indicators and a goal on
them, replacing a controller. Each step records the runtime effect seen by the fleet, the modelling assistant's
proposal and the outcome of the commit — including step 6, where the conformance gate **must block** the change
(the indicator would lose its only input and the goal would become unevaluable).

**Oracle.** Expected outcomes are derived from each configuration's intent (`intents/*.json`) by an independent
implementation of the documented semantics, never from the code under test.

## Campaigns

| run | RQ | mode | content |
|---|---|---|---|
| `sim-2026-09-23` | RQ1 | simulated | 30 configurations, mock devices, unattended |
| `rq2-2026-09-23` | RQ2 | simulated | 3 domains × 8 steps, unattended |
| `hw-2026-09-24` | RQ1 | hardware | office-01, home-01, fitness-01 on two Raspberry Pi controllers, 30 physical taps |
| `hw-2026-09-24` | RQ2 | hardware | 3 domains × 8 steps, bricklets plugged and unplugged by hand |

Simulation and hardware share code, images and settings; only the device layer differs (mock device API vs real
Tinkerforge bricklets). Filter every analysis by the `run` column.

## Headline results

| | simulation | hardware |
|---|---|---|
| RQ1 | 30/30 configurations pass; 476 probes, 0 mismatches | 3/3 pass; 30 physical taps, 0 mismatches |
| RQ2 | 3/3 domains, 24/24 steps | 3/3 domains, 24/24 steps |
| M1 model commit → last device bundle | median 6.4 s | 3.7–7.5 s |
| M2 statechart apply → last twin | median 20 ms | 23–39 ms |
| M3 NFC reader → actuation | median 7 ms | not valid on this run (reader → decision: median 0.65 s) |
| M4 hardware change → observation | median 1.7 s | median 4.4 s |

The step-6 commit was blocked with the expected diagnostics (`P5.NO_INPUTS`, `P6.GOAL_OUT_OF_SCOPE`) in all six
domain runs. The claims rest on the pass/fail columns; timings are context (everything ran on one laptop, the Pis
only host the bricklet daemon). Caveats — why M3 on hardware is blank, why M4 differs between modes, the outliers —
are in [`RESULTS-GUIDE.md`](RESULTS-GUIDE.md) §6.

## Further reading

| file | content |
|---|---|
| [`RESULTS-GUIDE.md`](RESULTS-GUIDE.md) | column-by-column meaning of the results, caveats, where to find the evidence |
| [`EVALUATION-PROTOCOL.md`](EVALUATION-PROTOCOL.md) | the full protocol: subjects, oracle, procedures, measurements, threats |
| [`results/`](results/) | `rq1.csv`, `rq2.csv` |
| [`runs/<run>/`](runs/) | per-row / per-step evidence (models, bundles, every tap, commits) |
| [`intents/`](intents/), [`models/`](models/) | the 30 subjects and their expected outcomes |
| [`RUNNING.md`](RUNNING.md) | how the harness is run (maintainers) |
