# Surface 1 (Pool / Spa) — Contract Reconciliation

**Author:** intellicenter expert · **Date:** 2026-06-08 · **Scope:** read-only analysis of `custom_components/intellicenter/` against `ENTITY_CONTRACT.md` Surface 1.
**Status of this doc:** PROPOSAL to coordinator. I do not edit `ENTITY_CONTRACT.md`. The protocol/model layer (`pyintellicenter`) is external and not in this repo; attribute *sources* are inferred from the platform modules and flagged where I cannot see them.

---

## 0. How entity_id and unique_id are actually constructed (read this first)

This is the single most important finding for b-panels binding.

### unique_id (STABLE — this is what to bind by)
`PoolEntity.unique_id` (`__init__.py:455-461`):
```
f"{entry_id}_{objnam}"            # base
+ attribute_key                   # appended only when attribute_key != STATUS_ATTR
```
Subclasses extend it:
- `PoolClimate.unique_id` → base + `"_climate"` (`climate.py:136-139`)
- `PoolWaterHeater.unique_id` → base + `LOTMP` (`water_heater.py:255-258`)
- `PumpModeSelect.unique_id` → `f"{entry_id}_{objnam}_{SELECT}"` (`select.py:125-127`)

`entry_id` is the random config-entry UUID (different on every install). `objnam` is the IntelliCenter object name (e.g. `B1101`, `C0006`, `PMP01`) — **stable per panel config but opaque and install-specific**, not a friendly string. So unique_id is stable across restarts/reloads but **not predictable from the contract** — b-panels cannot hardcode it either.

### entity_id (DERIVED — do NOT hardcode)
Every entity sets `_attr_has_entity_name = True` (`__init__.py:352`) and all entities share ONE device (`device_info`, `__init__.py:463-474`, `identifiers={(DOMAIN, entry_id)}`). HA therefore builds the initial `entity_id` as `slugify(device_name) + "_" + slugify(entity_name)`:
- **device_name** = `system_info.prop_name` (the panel's property name, user-set) or `"IntelliCenter"` (`__init__.py:472`).
- **entity_name** = `PoolObject.sname` (the user's circuit/body/sensor label, e.g. "Pool", "Spa", "Pool Light"), optionally with a `"+ suffix"` (`PoolEntity.name`, `__init__.py:409-453`), and with a `" 1"` stripped when the type is singleton (`_simplify_name`).

**Consequence:** every proposed `entity_id` in the contract (`climate.pool_heat`, `switch.pool_pump`, `sensor.pool_water_temp`, …) is a **guess at the slug** and will almost never match the real one. Real IDs depend on (a) the property name and (b) the user's equipment labels. They are also user-renameable in the UI. **None of the Surface-1 `entity_id`s are fixed.**

### Binding recommendation (applies to ALL Surface-1 rows)
b-panels must NOT bind by guessed `entity_id`. Bind by a stable selector resolved at runtime:
1. Filter to the intellicenter device (config entry / `identifiers={("intellicenter", entry_id)}`), then
2. Match on `(domain, device_class, unit_of_measurement)` plus the `OBJTYPE`/`OBJNAM` exposed in `extra_state_attributes` (every entity exposes `OBJNAM` + `OBJTYPE`, `__init__.py:485-488`), or
3. Resolve `unique_id` once via the entity registry and persist the mapping.

The contract's `entity_id` column should be relabeled "display target / nominal" and the contract should state binding is by registry lookup, not literal id.

---

## 1. Row-by-row reconciliation

| Contract proposed entity_id | Integration actually produces (source object → entity) | id derivation | device_class | state_class | unit | verdict | recommended final row |
|---|---|---|---|---|---|---|---|
| `climate.pool_heat` / `climate.spa_heat` | `PoolClimate` per BODY **only if `body_supports_cooling()` is true** (UltraTemp heat-pump bodies). `climate.py:69-78` | `slug(prop)_slug(body sname)`; uid base+`_climate` | none (climate) | n/a | temp unit from system (°F/°C) `climate.py:152-154` | **MISMATCH (conditional)** | Heat/cool only exists for cooling-capable bodies. Heating-only bodies have **no climate entity** — they are `water_heater.*`. See gap G1. |
| `switch.pool_pump` / spa pump | **NOT a switch on the pump.** Pump on/off surfaces as a `binary_sensor` (RUNNING) `binary_sensor.py:83-91`. The user-controllable on/off is the **BODY** circuit `PoolBody` switch `switch.py:51-52` | body: `switch.slug(body sname)` | switch: SWITCH; binary: RUNNING | n/a | n/a | **MISMATCH** | Rename: pool/spa run = `switch.<body>` (RW). Pump running = `binary_sensor.<pump>` (R). There is no direct pump on/off switch. |
| `number.pool_pump_speed` | `PumpSpeedNumber` (VSF, dynamic RPM/GPM) `number.py:322-336` **or** `PoolNumber` SPEED (RPM-only) `number.py:337-354` **or** GPM-only `number.py:355-372`. Built per **PMPCIRC** object, requires parent pump present | `number.slug(pump) Speed (circuit)` | none | n/a | `rpm` or `gpm` (CONST_RPM/CONST_GPM, dynamic on VSF) | **PARTIAL MATCH** | One number **per pump-circuit (PMPCIRC)**, not one per pump. Unit is `rpm`/`gpm` (custom strings), never `%`. EntityCategory=CONFIG. |
| `sensor.pool_pump_rpm` | `PoolSensor` RPM, only if `obj[RPM_ATTR]` present `sensor.py:98-108` | `sensor.slug(pump)_rpm` | **None** | MEASUREMENT (default) | `rpm` (CONST_RPM) | **MATCH (class caveat)** | device_class is `None` (no HA rpm class). state_class MEASUREMENT. Conditional on pump reporting RPM. |
| `sensor.pool_pump_power` | `PoolSensor` POWER, only if `obj[PWR_ATTR]` `sensor.py:86-97` | `sensor.slug(pump)_power` | POWER | MEASUREMENT | W (`UnitOfPower.WATT`) | **MATCH** | rounding_factor=25. Good to lock if pump reports PWR. |
| `sensor.pool_pump_flow` | `PoolSensor` GPM, only if `obj[GPM_ATTR]` `sensor.py:109-119` | `sensor.slug(pump)_gpm` | None | MEASUREMENT | `gpm` (CONST_GPM) | **MATCH (VSF only)** | Correct that this is VSF-only. device_class None. |
| `sensor.pool_pump_energy` (kWh, total_increasing) | **NOT PRODUCED.** No energy/kWh sensor anywhere in `sensor.py`. Only instantaneous POWER (W). | — | — | — | — | **NOT-PRODUCED** | Gap G2: must be an HA-side Riemann-sum `integration` sensor off the W power sensor → coordinator/template work, not integration. |
| `switch/number/sensor.spa_pump_*` | Same machinery as pool pump — depends on a distinct spa pump existing in config | as above | as above | as above | as above | **CONFIG-DEPENDENT** | Mirror only exists if there is a separate spa pump object. Single shared pump = no separate spa set. |
| `edge_pump_a/b_*` | Same machinery; GATED in contract | as above | as above | as above | as above | **GATED (correct)** | Stays GATED on pump selection. No change. |
| `sensor.pool_water_temp` / `sensor.spa_water_temp` | Water temp is `LSTTMP` on the BODY, exposed via `water_heater.current_temperature` / `climate.current_temperature` — **NOT a standalone `sensor.*`** `water_heater.py:287-289`, `climate.py:169-171` | — | (temperature, on the climate/water_heater entity) | n/a | temp unit | **MISMATCH / NOT a sensor** | Gap G3: there is no `sensor.pool_water_temp`. Body temp lives as `current_temperature` on the body's `water_heater`/`climate` entity. The standalone temp **sensors** are SENSE-type probes only (next row). |
| `sensor.air_temp` | `PoolSensor` for every `SENSE` object (air/water/solar probes), device_class TEMPERATURE, `attribute_key=SOURCE` `sensor.py:76-84` | `sensor.slug(sense sname)` | TEMPERATURE | MEASUREMENT | temp unit (auto °F/°C) `sensor.py:387-392` | **MATCH (id derived from sname)** | Which probe is "air" vs "water" vs "solar" is **entirely the user's SENSE sname**. Bind by OBJTYPE=SENSE + sname, not by `sensor.air_temp`. |
| `sensor.pool_salt` | `PoolSensor` SALT on CHEM/ICHLOR, only if `SALT` attr `sensor.py:273-285` | `sensor.slug(ichlor) (Salt)` | **None** | MEASUREMENT | ppm (`CONCENTRATION_PARTS_PER_MILLION`) | **MATCH (class caveat)** | No HA salt device_class; device_class None, unit ppm. Requires IntelliChlor. |
| `number.pool_swg_output` (%) | `PoolNumber` on ICHLOR `PRIM`/`SEC` per body `number.py:115-138` | `number.slug(ichlor) Output % (body)` | None | n/a | `%` (PERCENTAGE) | **MATCH** | One per body served by the chlorinator (PRIM for body0, SEC for body1). EntityCategory=CONFIG. RW via `set_chlorinator_output`. |
| `sensor.pool_swg_cell_life` (%) | **NOT PRODUCED.** No cell-life attribute/sensor in `sensor.py` (grep confirms none). | — | — | — | — | **NOT-PRODUCED** | Gap G4: integration does not expose SWG cell life. Drop from contract or mark GATED pending integration work (needs a pyintellicenter attr). |
| `sensor.pool_ph` | `PoolSensor` PHVAL on ICHEM `sensor.py:183-192` | `sensor.slug(ichem) (pH)` | **PH** | MEASUREMENT | none (HA PH class implies pH) | **MATCH** | device_class PH. Requires IntelliChem. |
| `sensor.pool_orp` (mV) | `PoolSensor` ORPVAL on ICHEM `sensor.py:193-204` | `sensor.slug(ichem) (ORP)` | None | MEASUREMENT | `mV` | **MATCH** | device_class None, unit mV. Requires IntelliChem. |
| `sensor.pool_ph_tank` / `_orp_tank` (%) | `PoolSensor` PHTNK / ORPTNK `sensor.py:216-241`, **DIAGNOSTIC**, `value_offset=-1`, **no unit set** | `sensor.slug(ichem) (pH Tank Level)` etc. | None | MEASUREMENT | **none** (no unit_of_measurement passed) | **MISMATCH (unit)** | Tank levels carry NO unit (raw 1-6 level minus 1), and are DIAGNOSTIC (hidden by default). Contract says `%` — incorrect. Fix unit to "level"/none. |
| `light.pool_light` / `light.spa_light` | `PoolLight` for circuits with light subtypes; `effect_list` only when `supports_color_effects` `light.py:40-64`, `110-123` | `light.slug(light sname)` | n/a | n/a | n/a | **PARTIAL MATCH** | **No `rgb_color`.** `_attr_color_mode = ONOFF` only `light.py:83-84`. Supports on/off + `effect`/`effect_list` (named MagicStream/IntelliBrite shows), NOT rgb. Contract's `rgb_color` attr is wrong. |
| `switch.<water_feature>` | `PoolCircuit` for **featured** circuits (`is_featured`) and CIRCGRP groups `switch.py:67-78` | `switch.slug(circuit sname)` | SWITCH | n/a | n/a | **MATCH (config-dependent)** | Only circuits the user marked "Featured" become switches. Per-feature ids derive from circuit sname. |
| `binary_sensor.pool_freeze_protection` (running) | `PoolBinarySensor` for CIRCUIT subtype `FRZ`, device_class **COLD**, **DIAGNOSTIC** `binary_sensor.py:59-68` | `binary_sensor.slug(frz sname)` | **COLD** (not RUNNING) | n/a | n/a | **MISMATCH (class + category)** | device_class is COLD, entity_category DIAGNOSTIC (hidden by default). Contract says `running`. Also only exists if a FRZ circuit is configured. |
| `sensor.pool_turnover` / `sensor.pool_total_flow` | **NOT PRODUCED** (correctly flagged "derived HA-side" in contract). | — | — | — | measurement | **NOT-PRODUCED (expected)** | Gap G5: HA-side template/derived sensors. Coordinator/template work, not integration. |

---

## 2. Extra entities the integration emits that the contract omits (FYI for coordinator)

These are real Surface-1 outputs not in the contract; coordinator may want rows:
- `switch.<...> Superchlorinate` — ICHLOR SUPER attr `switch.py:58-66` (RW).
- `switch.<...> Vacation mode` — SYSTEM, CONFIG `switch.py:79-81`, `138-162` (RW).
- `sensor.* Firmware Version`, `sensor.* System Mode` (ENUM auto/service/timeout) — SYSTEM `sensor.py:286-305`, `402-454`. System Mode is a primary (non-diagnostic) enum sensor.
- `number.* Max Temperature` (HITMP, °F, CONFIG) per body `number.py:235-252` — this is the actual high-temp setpoint number.
- `number.*` IntelliChem pH/ORP setpoints + Alkalinity/Calcium/Cyanuric (CONFIG) `number.py:140-232`.
- `binary_sensor.* Heater` (HEAT, diagnostic) `binary_sensor.py:69-75`, `199-263`; `binary_sensor.Schedule (...)` (RUNNING, diagnostic, disabled-ish) `binary_sensor.py:76-82`, `269-303`; IntelliChem pH/ORP Hi/Lo alarm binaries (PROBLEM, diagnostic) `binary_sensor.py:92-141`.
- `sensor.* (Water Quality)`, `sensor.* (pH/ORP Dosing Volume)` (mL, TOTAL_INCREASING, diagnostic) `sensor.py:205-270`.
- Pump diagnostic sensors: Max/Min RPM, Max/Min GPM `sensor.py:120-180`.
- `select.<pump> Mode (circuit)` RPM/GPM for VSF pumps `select.py` (CONFIG).
- `cover.<...>` for EXTINSTR subtype COVER `cover.py:45-47` (SHADE class, open/close). **Note for AKVO:** this is a Pentair-attached cover, unrelated to the AKVO movable floor (Surface 3, Modbus). Do not conflate.
- `water_heater.<body>` for **every** body with any heater `water_heater.py:96-99` — this is the primary heat control for heating-only bodies.

---

## 3. Gaps (contract rows the integration does NOT produce)

- **G1 — `climate.*` only for cooling-capable bodies.** Gas/solar/heat-only pool & spa surface as `water_heater.*`, not `climate.*` (`climate.py:71` gate). If b-panels wants a single "heat" control per body, it must handle BOTH `water_heater` (heating-only) and `climate` (UltraTemp heat/cool). Coordinator decision needed.
- **G2 — `sensor.pool_pump_energy` (kWh) not produced.** Only instantaneous W. Energy must be an HA-side Riemann `integration` sensor. Coordinator/template work.
- **G3 — `sensor.pool_water_temp` / `sensor.spa_water_temp` not produced as sensors.** Body water temp = `current_temperature` on the body's `water_heater`/`climate` entity. Standalone temp sensors exist only for SENSE probes (and which is "water" depends on the user's sname). If b-panels needs a plain water-temp sensor, either bind to the water_heater's `current_temperature` attribute, or add a template sensor (coordinator), or request an integration enhancement.
- **G4 — `sensor.pool_swg_cell_life` not produced.** No cell-life attribute exposed. Needs pyintellicenter + sensor.py work, or drop from contract.
- **G5 — `sensor.pool_turnover` / `sensor.pool_total_flow` not produced** (expected; HA template work).

---

## 4. Recommended LOCK LIST

**Caveat binding all locks:** lock the *semantic contract* (domain + device_class + state_class + unit + R/W), NOT a literal `entity_id`. b-panels binds by registry lookup over the intellicenter device using OBJTYPE/OBJNAM + class, never by the guessed id (see §0). Several rows are also config-dependent — they exist only if the equipment is present, but their *shape* is stable, so they are safe to lock as "appears iff equipment present."

### Safe to flip PROPOSED → LOCKED now (shape confirmed, stable id derivation)
| Nominal id | Lock as | Evidence |
|---|---|---|
| pump power | `sensor`, device_class=POWER, MEASUREMENT, unit=W | `sensor.py:86-97` |
| pump rpm | `sensor`, device_class=None, MEASUREMENT, unit=`rpm` | `sensor.py:98-108` |
| pump flow | `sensor`, device_class=None, MEASUREMENT, unit=`gpm` (VSF only) | `sensor.py:109-119` |
| pump speed setpoint | `number`, unit `rpm`/`gpm` (dynamic), CONFIG, RW, per PMPCIRC | `number.py:322-372` |
| salt | `sensor`, device_class=None, MEASUREMENT, unit=ppm | `sensor.py:273-285` |
| SWG output | `number`, unit=%, CONFIG, RW (per body) | `number.py:115-138` |
| pH | `sensor`, device_class=PH, MEASUREMENT | `sensor.py:183-192` |
| ORP | `sensor`, device_class=None, MEASUREMENT, unit=mV | `sensor.py:193-204` |
| temp probe(s) (air/water/solar) | `sensor`, device_class=TEMPERATURE, MEASUREMENT, unit auto | `sensor.py:76-84`, `387-392` |
| body run (pool/spa) | `switch`, SWITCH, RW (replaces "pool_pump" switch) | `switch.py:51-52`, `123-135` |
| featured water features | `switch`, SWITCH, RW (per featured circuit) | `switch.py:67-78` |
| pool/spa light | `light`, ONOFF + effect/effect_list (NO rgb_color) | `light.py:76-123` |
| water_heater (heating-only bodies) | `water_heater`, target/current temp + operation_mode + on/off, RW | `water_heater.py:116-` |

### Must STAY PROPOSED / change before lock
| Nominal id | Why | Action |
|---|---|---|
| `climate.pool_heat`/`spa_heat` | Only exists for UltraTemp cooling-capable bodies; heating-only bodies are water_heater | Coordinator: split into climate (cooling-capable) + water_heater (heating-only) rows. |
| `sensor.pool_water_temp`/`spa_water_temp` | Not produced as a sensor (G3) | Re-spec as water_heater `current_temperature` binding or template sensor. |
| `sensor.pool_ph_tank`/`_orp_tank` | Wrong unit (no unit, level-1, not %) + DIAGNOSTIC | Fix class/unit in contract before lock. |
| `binary_sensor.pool_freeze_protection` | device_class COLD (not RUNNING) + DIAGNOSTIC (hidden) | Fix class to COLD; note diagnostic; then lock. |
| `light.* rgb_color` attr | Not supported (ONOFF + effects only) | Remove rgb_color from contract attrs. |
| `sensor.pool_swg_cell_life` | NOT-PRODUCED (G4) | Drop or mark GATED pending integration work. |
| `sensor.pool_pump_energy` | NOT-PRODUCED (G2) | Mark as HA-side template/integration sensor (coordinator). |
| `sensor.pool_turnover`/`total_flow` | NOT-PRODUCED (G5, expected) | Keep as derived HA-side template (coordinator). |
| spa pump / edge pump sets | config/hardware-dependent | edge stays GATED; spa set locks only if a distinct spa pump object exists. |

---

## 5. Notes / uncertainties (pyintellicenter is external)
- Attribute *values* (`on_status`, `STATUS_ON/OFF`, `SOURCE`, `is_featured`, `is_a_light`, `supports_color_effects`, `body_supports_cooling`) live in the external `pyintellicenter` package, not in this repo. I inferred entity shape from how the platform modules consume them; the exact protocol strings (e.g. SERVICE enum values for System Mode) are documented in-code as inferred/unconfirmed (`sensor.py:395-399`, `412-417`).
- Temperature unit is dynamic: °F or °C from `system_info.uses_metric` (`__init__.py:558-571`). The contract's hardcoded "°F" is only correct if the panel is in ENGLISH mode. b-panels should read the unit off the entity, not assume °F.
- `entry_id` in every unique_id is install-specific, so unique_ids cannot be pre-written into the contract; they must be resolved per-install via the entity registry.

---

## 6. Currency verdict (2026-06-08)

**Library: CURRENT — no lag.** `manifest.json` pins `pyintellicenter>=0.1.19`; PyPI's latest is **0.1.19** (`pip index versions pyintellicenter` → newest = 0.1.19). `pyproject.toml` pin matches (`tests/test_versions.py::test_pyintellicenter_pin_matches` enforces this). The installed contract surface (`tests/test_library_contract.py`) passes against 0.1.19 — every controller method and symbol the integration imports still exists, and the SAMMOD light show fix is present.

**Unexposed firmware/protocol features (evidence-based, library is the limiter):** I enumerated the full `*_ATTR` surface exported by the installed library (`pyintellicenter` `attributes/equipment.py:60-119`). Findings:
- **IntelliChlor cell life: NOT EXPOSED by pyintellicenter.** The full ICHLOR attribute set is `PRIM, SEC, SALT, SUPER, TIMOUT` (`equipment.py:79-91`). There is no cell-life / cell-percent attribute anywhere in the library (grep for `cell|life` returns nothing). The IntelliChlor hardware does report cell life on the panel, but exposing it requires a **pyintellicenter** change first (add the attribute to the ICHLOR set + model), then a one-line sensor here. Per the thin-wrapper rule I did **not** synthesize it in the integration. → **Library enhancement request, not integration work.**
- **Energy (kWh): NOT EXPOSED.** The pump exposes only `PWR` (instantaneous watts, `equipment.py`), no cumulative energy register. A correct kWh figure is a time-integral of W and must be an HA-side Riemann-sum (`integration`) helper off the existing power sensor — not something the integration can emit truthfully. The integration's power sensor is already `device_class=POWER, state_class=MEASUREMENT, unit=W`, which is exactly what the Riemann helper needs. → **HA-side helper (coordinator/template), documented below.**
- Body `COOL`/`HEATING` and heater `BOOST` attributes exist in the library and are already consumed via the controller's `is_body_heating/is_body_cooling` helpers (climate `hvac_action`). No gap.

**device_class / state_class / unit audit (long-term statistics):** all already correct — POWER+W, PH, TEMPERATURE (dynamic °F/°C), ppm salt, mV ORP, `rpm`/`gpm` custom units with `device_class=None` (no HA class exists for these), MEASUREMENT state_class throughout, TOTAL_INCREASING on dosing volumes. No corrections required.

---

## 7. What I implemented this loop (branch `feat/contract-and-currency`)

**Additive, read-only, library-backed. No unique_id changes, no migration, no new actuation.**

- **NEW: body water-temperature sensor** (`sensor.py` `_build_entities`, BODY_TYPE branch). Closes gap **G3** (`sensor.pool_water_temp` / `sensor.spa_water_temp` were NOT-PRODUCED as sensors). Emits a `SensorDeviceClass.TEMPERATURE` sensor off the body `LSTTMP` attribute (already tracked in `coordinator.DEFAULT_ATTRIBUTES_MAP[BODY_TYPE]`), `state_class=MEASUREMENT`, dynamic °F/°C unit. One per body that reports `LSTTMP`. `unique_id = {entry_id}_{body_objnam}LSTTMP` — distinct from the body's water_heater (`…LOTMP`) and climate (`…_climate`) entities, so no collision and no migration. Gives pool/spa water temperature a standalone, statistics-bearing, stably-bindable entity (previously only available as the water_heater's `current_temperature` attribute).
- **NOT implemented (deferred, with reason):** cell-life sensor (library does not expose the data — deferred to a pyintellicenter enhancement); pump energy kWh sensor (must be an HA-side Riemann helper — coordinator/template work, cannot be truthfully emitted by the integration). No equipment-actuating behavior was added.

Tests: added 3 (`tests/test_sensor.py`): properties/unique_id/isUpdated of the water-temp sensor, setup creates exactly one per body with LSTTMP, and none for a body without LSTTMP. **Full suite: 299 passed** (296 baseline + 3). ruff + mypy clean on changed files.

---

## 8. PROPOSED FINAL ROWS — Surface 1 (for coordinator to transcribe)

**Binding strategy (applies to every row):** b-panels resolves entities at runtime by querying the entity registry for the intellicenter config entry / device, then matching on `(domain, device_class, unit)` plus the `OBJTYPE`/`OBJNAM` in `extra_state_attributes`. **Do NOT bind by literal `entity_id`** — ids derive from the user's property name + equipment labels and are renameable (see §0). The `selection` column below is that stable matcher. Temperature `unit` is dynamic (°F/°C from panel mode); read it off the entity.

Legend: **LOCK** = shape stable, flip PROPOSED→LOCKED now (exists iff the equipment is present, but the shape is fixed). **HOLD** = keep PROPOSED/GATED, needs a contract decision or external work.

| nominal id | domain | selection (bind by) | device_class | state_class | unit | R/W | rec |
|---|---|---|---|---|---|---|---|
| pool/spa water temp | sensor | OBJTYPE=BODY + attr LSTTMP | temperature | measurement | °F/°C dyn | R | **LOCK** (NEW this loop) |
| air/water/solar probe temp | sensor | OBJTYPE=SENSE (+ sname) | temperature | measurement | °F/°C dyn | R | **LOCK** |
| pump power | sensor | OBJTYPE=PUMP + attr PWR | power | measurement | W | R | **LOCK** |
| pump rpm | sensor | OBJTYPE=PUMP + attr RPM | none | measurement | rpm | R | **LOCK** |
| pump flow | sensor | OBJTYPE=PUMP + attr GPM | none | measurement | gpm | R | **LOCK** (VSF only) |
| salt | sensor | OBJTYPE=CHEM/ICHLOR + attr SALT | none | measurement | ppm | R | **LOCK** |
| pH | sensor | OBJTYPE=CHEM/ICHEM + attr PHVAL | ph | measurement | — | R | **LOCK** |
| ORP | sensor | OBJTYPE=CHEM/ICHEM + attr ORPVAL | none | measurement | mV | R | **LOCK** |
| body run (pool/spa) | switch | OBJTYPE=BODY | switch | — | — | RW | **LOCK** (replaces `switch.pool_pump`) |
| water feature | switch | OBJTYPE=CIRCUIT, featured/CIRCGRP | switch | — | — | RW | **LOCK** (per featured circuit) |
| pool/spa light | light | OBJTYPE=CIRCUIT light/lightshow | — | — | — | RW | **LOCK** (ONOFF + effect/effect_list; **no rgb_color**) |
| pump speed setpoint | number | OBJTYPE=PMPCIRC | none | — | rpm/gpm dyn | RW (CONFIG) | **LOCK** (per pump-circuit) |
| SWG output % | number | OBJTYPE=CHEM/ICHLOR + PRIM/SEC | none | — | % | RW (CONFIG) | **LOCK** (per body) |
| body max temp setpoint | number | OBJTYPE=BODY + attr HITMP | temperature | — | °F/°C dyn | RW (CONFIG) | **LOCK** |
| pump running | binary_sensor | OBJTYPE=PUMP | running | — | — | R | LOCK (note: this is the pump, distinct from body-run switch) |
| freeze protection | binary_sensor | OBJTYPE=CIRCUIT/FRZ | **cold** | — | — | R | **LOCK** (correct class=cold + diagnostic; NOT `running`) |
| water heater (heating-only bodies) | water_heater | OBJTYPE=BODY w/ heater, not cooling-capable | — | — | °F/°C dyn | RW | **LOCK** |
| climate (heat/cool body) | climate | OBJTYPE=BODY, body_supports_cooling | — | — | °F/°C dyn | RW | **HOLD** — only exists for UltraTemp cooling bodies; coordinator must split the single contract "heat" row into water_heater (heating-only) + climate (cooling-capable). |
| pH/ORP tank level | sensor | OBJTYPE=CHEM/ICHEM + PHTNK/ORPTNK | none | measurement | **none** (level, value−1) | R | HOLD — fix contract unit (not %); DIAGNOSTIC/hidden by default. |
| SWG cell life | — | — | — | — | — | — | **HOLD / drop** — NOT-PRODUCED; library doesn't expose it (§6). Needs pyintellicenter enhancement first. |
| pump energy (kWh) | sensor | (HA-side) Riemann helper off pump-power sensor | energy | total_increasing | kWh | R | **HOLD** — coordinator builds an `integration` helper; not integration output. |
| pool turnover / total flow | sensor | (HA-side) template | — | measurement | — | R | **HOLD** — derived template (coordinator). |
| spa pump set | switch/number/sensor | OBJTYPE=PUMP (distinct spa pump) | as pool pump | — | — | mixed | LOCK iff a separate spa pump object exists; else N/A (shared pump). |
| edge pump A/B | switch/number/sensor | OBJTYPE=PUMP | as pool pump | — | — | mixed | **GATED** (pump selection) — unchanged. |

**Lights attribute correction:** drop `rgb_color`; the real attrs are `effect` + `effect_list` (named MagicStream/IntelliBrite shows), color_mode ONOFF only.
**Climate/water_heater correction:** the single contract `climate.pool_heat`/`spa_heat` row does not match reality — heating-only bodies are `water_heater`, only UltraTemp cooling-capable bodies are `climate`. Coordinator should publish both rows.
