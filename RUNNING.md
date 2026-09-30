# Evaluation — how to run (maintainers)

Protocol: `EVALUATION-PROTOCOL.md`. Results: `results/rq1.csv`, `results/rq2.csv`; evidence: `runs/<run-id>/`.

## Prerequisites

- Docker Desktop, Node ≥ 20, the repo built once (`mvn -DskipTests package` is not needed: the stacks build their
  images with `--build`).
- The fleet's Studio account in the repo's `.env` (`FLEET_PIP_URL`, `FLEET_PIP_ANONKEY`, `FLEET_PIP_EMAIL`,
  `FLEET_PIP_PASSWORD`; an **operator** of the org) — that is all RQ1 needs.
- `evaluation/prod-devices.env` (copy of `prod-devices.env.example`, git-ignored): for RQ2 and for cleanup a
  Supabase **personal access token** (`SUPABASE_ACCESS_TOKEN`, `SUPABASE_PROJECT_REF`); for the hardware rows the
  real X-Device-Ids, the brickd hosts and the four tag UIDs.

```bash
cd evaluation && npm install
npm test                       # the 30 intents build, conform, project; the oracle on the seed rows
npm run gen                    # regenerate intents/, models/ (only after changing the generator)
```

## RQ1, simulated (unattended, ≈ 50 min)

```bash
npm run eval -- stack up --mode sim --build          # fleet + ACS-1 + ACS-2 on evaluation/data, mock devices
npm run eval -- rq1 --mode sim --run sim-$(date +%F)  # all 30 rows; --rows office-01,home-02 for a subset
npm run eval -- stack down --mode sim                 # stop; re-create the production containers (stopped)
```

Detached, with sleep prevented (a laptop that sleeps mid-run breaks the timings and the fleet's cloud calls):
`nohup caffeinate -i npm run eval -- rq1 --mode sim > ../scripts/e2e/out/eval-full.log 2>&1 &` and watch the log for
`ROW PASS|FAIL` and `E2E_EVAL_DONE`. The run ends with a results table, deletes the evaluation models from the
Studio (needs the token; otherwise it lists them and `npm run eval -- clean` does it later) and compares the demo
models' checksum before/after.

## RQ1, hardware (attended)

Fill `prod-devices.env`, put the Pis on the LAN with brickd listening, then:

```bash
set -a; . prod-devices.env; set +a
npm run eval -- stack up --mode hw
npm run eval -- rq1 --mode hw --run hw-$(date +%F)   # rows office-01, home-01, fitness-01; the runner prompts for every tap
npm run eval -- stack down --mode hw
```

## RQ2 (needs the management token, ≈ 12 min)

```bash
npm run eval -- rq2 --mode sim --run rq2-$(date +%F) --up   # fresh stack with the RQ2 mock config (no ambient /
                                                            # soundPressure at boot); office, home, fitness
npm run eval -- stack down --mode sim
```

`--up` re-creates the containers on fresh `evaluation/data/*` dirs (a container kept across a data reset holds a
stale bind mount); `--domains office` runs one domain; `--keep` leaves the models in the Studio. The cloud edge
functions must be the ones of the sources under test: `cd pip-studio && npm run build:edge && supabase functions
deploy commit-model projection export-model --project-ref <ref>` (the commit gate is the deployed `commit-model`).
Detached: `nohup caffeinate -i -s npm run eval -- rq2 --mode sim --run rq2-$(date +%F) --up --no-down > ../scripts/e2e/out/eval-rq2.log 2>&1 &`.

Hardware: `--mode hw` with the Pis; the runner prompts to plug and unplug the bricklets and restarts dev2's engine
for the controller replacement. The HAT's brickd sees a hot-plugged bricklet only on a port that was occupied at its
start: plug `ambient` and `soundPressure` into both Pis, `sudo systemctl restart brickd` on each, then unplug them
and launch. Attended runs are interactive (Enter per tap / plug): run them in a terminal, e.g.
`npm run eval -- rq2 --mode hw --run hw-$(date +%F) --up 2>&1 | tee ../scripts/e2e/out/eval-hw-rq2.log`.
`npm run eval -- tags --mode hw` tells which physical card is TAG_1…TAG_4.

## Layout

```
EVALUATION-PROTOCOL.md   the protocol (what, why, how measured)
intents/                 30 intents (+ the RQ2 ones are built in code: src/gen/rq2.ts)
models/                  the models built at the reference instant + expected outcomes
statecharts/             eval-signature (1777 Hz), eval-override (1999 Hz)
stack/                   compose overrides, RQ2 mock config
src/gen/                 intent → model (build), oracle (expected), generator, anchoring (resolve), RQ2 subjects
src/runner/              env, http, studio, fleet, device, stack, rq1, rq2, report, cli
results/                 rq1.csv, rq2.csv (appended per run)
runs/<run-id>/<row>/     evidence: model, expected, load, switch, rq1_2, rq1_1, rq1_3, rq1_4, checks, log
```
