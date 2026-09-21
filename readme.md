# Closed-loop evaluation — scenario catalogue

Companion to the paper (RQ on closed-loop behaviour under variability). Suggested location in the repo:
`evaluation/SCENARIOS.md`, with one folder per scenario under `evaluation/scenarios/<id>/` holding the model
versions (Studio JSON export), the rule/KPI files if any, the oracle and the collected evidence.

## Set-up

- **Installation**: two physical controllers (ACS-1 guards `room_p1_28`, ACS-2 guards `meeting_a`), both in zone
  `floor_1`; fleet-controller; PIP Model Studio on Supabase; canonical configuration (Mode B, local PDP).
- **Baseline model**: `GSSI-office-2026` at a tagged version (`v-base`), exported to `evaluation/models/`.
- **Execution modes**
  - **P — physical**: the stimulus is applied to the real hardware (tap an NFC tag, unplug a bricklet, cut the
    network, vary light) and the effect is observed on the hardware (relay, LED, buzzer) and in the logs.
  - **S — simulated**: same installation and models, stimulus injected in software: simulated NFC events on the
    state-engine, the device-api mock hooks (detach/attach a bricklet, synthetic readings), or a device stack
    running in mock mode next to the two physical ones. Used to repeat a scenario N times for timing.
  - **P+S**: run once physically to show the loop closes on hardware, then N times simulated for timing.
- **Clocks**: NTP on both Raspberry Pis, fleet host and Studio host; timings are also cross-checked on the fleet
  clock alone (it observes both stimuli and effects).
- **Evidence sources** (existing APIs, no instrumentation added): fleet audit (`GET /api/pip/decisions`),
  restrictions (`/api/restrictions/stream`, `restrictions.jsonl`), rule state (`GET /api/rules/state`), twin
  snapshots (`/twin/stream`), device PDP status (`GET /api/pdp/status`), topology (`GET /api/topology`), runtime
  tables `sensor_readings` / `device_observations`, Studio `changes` log and model versions.

## Variability dimensions

| Dimension | Values |
|---|---|
| **V1 origin of the change** | physical (tag, bricklet, sensed environment, power/network) · digital (behaviour model) · federation (rule, KPI) · model (Studio edit) |
| **V2 scope of the effect** | single twin · zone / shared space (both twins) · fleet · model |
| **V3 connectivity** | nominal · fleet down · Studio down · device isolated · hardware (brickd) down |
| **V4 expected effect** | access decision · actuation · restriction emitted / cleared · model updated · state shown / audited |

Every value of every dimension is covered by at least one scenario; the critical combinations (model change under
degraded connectivity; physical change with a zone-level effect; physical change with a model-level effect) are
covered explicitly.

## Timing points (per hop)

`t_model` Studio commit · `t_fleet` projection applied on the fleet · `t_bundle` bundle fetched by the device ·
`t_stim` physical/simulated stimulus · `t_kpi` KPI value crossing the threshold · `t_rule` rule phase change ·
`t_restr` restriction emitted/cleared · `t_dec` decision taken on the device · `t_act` actuation · `t_audit`
decision in the fleet audit · `t_obs` runtime table updated · `t_disc` sensor committed to the model.

## Scenarios

Legend: **Mode** P / S / P+S · **V1/V2/V3/V4** as above · **Oracle** = expected, checked automatically where possible.

### Family A — model → twin (identity, policy, staleness)

| Id | Mode | V1 / V2 / V3 / V4 | Pre-model | Stimulus | Oracle | Timing |
|---|---|---|---|---|---|---|
| A1 | P+S | model / twin / nominal / decision | v-base | Studio: give `anna` role `staff`; then tap `anna` on ACS-1 | before: deny `no_role`; after: allow `ok`, relay opens | `t_model→t_bundle→t_dec` |
| A2 | P+S | model / zone / nominal / decision | v-base | Studio: revoke `ludovico`; tap on ACS-1 and ACS-2 | deny `revoked` on both | `t_model→t_dec` per device |
| A3 | S | model / twin / nominal / decision | v-base | Studio: `officeHours` 07–22 → 09–18; tap at 19:00 (clock injected) | deny `outside_hours` | `t_model→t_dec` |
| A4 | S | model / twin / nominal / decision | v-base | Studio: `room_p1_28.whenExceeded` `allow-cached` → `deny` (then run E2) | E2 outcome flips from allow-stale to deny | — |
| A5 | P | model / twin / nominal / decision | v-base | Studio: move `acs2.guards` from `meeting_a` to `server_room` | ACS-2 bundle now scoped to `server_room`; `staff` denied, `facility` allowed | `t_model→t_bundle` |

### Family B — digital → physical (behaviour distribution)

| Id | Mode | V1 / V2 / V3 / V4 | Pre-model | Stimulus | Oracle | Timing |
|---|---|---|---|---|---|---|
| B1 | P | digital / fleet / nominal / actuation | v-base | fleet: publish and apply `fleet-general` | both twins switch machine; tap → same LED/buzzer sequence on both | apply → active on each twin |
| B2 | P | digital / twin / nominal / actuation | v-base | ACS-2 activates a local machine (override) | cockpit shows ACS-2 overridden (derived from machine name); ACS-1 unchanged | — |
| B3 | S | digital / twin / nominal / actuation | v-base | ACS-2 returns to `fleet-general` | inheritance restored without fleet-side configuration | — |

### Family C — physical → model (auto-discovery and liveness)

| Id | Mode | V1 / V2 / V3 / V4 | Pre-model | Stimulus | Oracle | Timing |
|---|---|---|---|---|---|---|
| C1 | P+S | physical / zone / nominal / state + restriction | v-base, rule `floor1-air` enabled | unplug the CO₂ bricklet of ACS-1 (S: mock detach) | topology lists it as configured-but-missing; `sensor_readings.acs1_co2.online=false`; Studio node shows offline; `co2.avg` withheld on `floor_1`; rule stays in its phase (no restriction emitted or cleared) | `t_stim→t_obs`, `t_stim→` rule undecided |
| C2 | P+S | physical / zone / nominal / state | after C1 | re-plug the bricklet | reading back, sensor online, KPI and rule evaluated again | `t_stim→t_obs` |
| C3 | P | physical / model / nominal / model updated | v-base without `acs2_humidity` | plug a humidity bricklet on ACS-2 | device-api auto-configures it; `device_observations` lists key `humidity`; assistant proposes `acs2_humidity`; sync (manual or automatic) commits it as instance of `HumiditySensor` with `senses`; new model version; P5/P6 still pass | `t_stim→t_obs→t_disc` |
| C4 | P | physical / model / nominal / model updated | v-base | plug a bricklet whose reading key has no O1 sensor type | discovery reports and skips it (no guess); assistant proposes the missing sensor type in O1 | `t_stim→t_obs` |
| C5 | P | physical / fleet / hardware down / state | v-base | stop brickd on ACS-1 (or pull the HAT power) | all ACS-1 sensors offline; KPIs with ACS-1 inputs withheld; ACS-1 twin stays online (engine alive) | `t_stim→t_obs` |

### Family D — physical → federation → twins (collective decisions)

| Id | Mode | V1 / V2 / V3 / V4 | Pre-model | Stimulus | Oracle | Timing |
|---|---|---|---|---|---|---|
| D1 | P+S | physical / zone / nominal / restriction + decision | v-base, rule on `illuminance.avg` over `floor_1` (P) / `co2.avg` (S, injected) | raise light above threshold on both devices for `for` | rule `arming→active`; restriction emitted; guest denied `restricted:<rule>` on both ACS-1 and ACS-2 | `t_stim→t_kpi→t_rule→t_restr→t_bundle→t_dec` |
| D2 | P+S | physical / zone / nominal / restriction cleared | after D1 | lower light below clear threshold for `for` | `clearing→inactive`; restriction removed; guest allowed again | same chain |
| D3 | S | model / zone / nominal / restriction | v-base, rule follows goal `floor1Air` | Studio: goal value 900 → 800 while readings at 850 | rule fires after next sync without any fleet edit | `t_model→t_fleet→t_rule` |
| D4 | P | physical / twin / nominal / restriction + decision | v-base, rule `presence.staff ≥ 1` permits `guest` | staff taps on ACS-2, then guest taps | guest allowed only while staff presence holds; `ruleId` in decision | `t_dec(staff)→t_restr→t_dec(guest)` |
| D5 | S | federation / zone / nominal / restriction | v-base | enable a composite rule (`all` of CO₂ and light) | fires only when both leaves hold; `terms` in rule state | — |

### Family E — degraded connectivity

| Id | Mode | V1 / V2 / V3 / V4 | Pre-model | Stimulus | Oracle | Timing |
|---|---|---|---|---|---|---|
| E1 | P+S | physical / twin / fleet down / decision | v-base | stop fleet; tap within `maxAgeMs` | allow `ok` from cache | — |
| E2 | S | physical / twin / fleet down / decision | A4 applied (`deny`) | stop fleet; tap after `maxAgeMs` | deny `stale_cache` | — |
| E3 | S | physical / twin / fleet down / decision | v-base (`allow-cached`) | stop fleet; tap after `maxAgeMs` | allow `ok (cached, stale)`, `stale=true` | — |
| E4 | S | model / twin / Studio down / decision | v-base | stop Studio, fleet up; wait `maxAgeMs` | device reaches the stale outcome although the fleet answers (two-hop staleness), `fleetStale=true` in the audit | `t_stop→stale outcome` |
| E5 | P+S | physical / twin / device isolated / audit | v-base | isolate ACS-1, tap 3 times, reconnect | 3 decisions replayed, marked `offline`, no duplicates | reconnect → `t_audit` |
| E6 | S | federation / zone / fleet down / decision | D1 active | stop fleet while restriction is active | devices keep denying until `until`; after ttl the restriction lapses locally | — |
| E7 | S | model / fleet / fleet down / decision | v-base | revoke a user in the Studio while the fleet is down, then restart fleet | no effect while down; revocation applied within one poll + one refresh after restart | restart → `t_dec` |

## Per-scenario folder

```
evaluation/scenarios/C3/
  README.md          # description, mode, dimensions, steps (who does what, physically or via script)
  pre.model.json     # Studio export of the starting model (tag)
  post.model.json    # expected model after the scenario, when the scenario changes the model
  oracle.yaml        # expected decisions / restrictions / states, with tolerances
  runs/<timestamp>/  # collected evidence: audit.json, restrictions.jsonl, rule-state.json, timings.csv
```

`oracle.yaml` shape (example, D1):

```yaml
scenario: D1
expect:
  - rule: floor1-light
    phases: [arming, active]
  - restriction: { ruleId: floor1-light, effect: deny, roles: [guest], spacesInclude: [room_p1_28, meeting_a] }
  - decision: { device: acs1, user: chen, allowed: false, reason: "restricted:floor1-light" }
  - decision: { device: acs2, user: chen, allowed: false, reason: "restricted:floor1-light" }
timing:
  chain: [t_stim, t_kpi, t_rule, t_restr, t_bundle, t_dec]
  budget_ms: 20000   # for + KPI cycle (3 s) + refresh (10 s) + margin
```

## Reporting

For each scenario: pass/fail against the oracle, the model versions involved, and for P+S/S the distribution of the
per-hop times (median, p95, max over N runs; N = 30 unless stated). The paper reports one aggregated row per family;
this catalogue and the evidence folders are the replication package.
