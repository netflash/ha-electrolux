# AC Finish Open Work — 2026-05-27

Scope: AC-only. Wrap open PR, fix two open AC bugs, verify already-merged fixes on live hardware, sync HA install.

## Context

**HA install**: `/config/custom_components/electrolux/manifest.json` reports `version: 3.6.6`. Latest release `v3.6.7` (2026-05-18) does NOT contain PRs merged 2026-05-21..25.

**My PRs (last 3 weeks)**:

| PR | State | In release | Notes |
|----|-------|-----------|-------|
| #47 remove log1 | merged | v3.6.6 | — |
| #48 restore last user temp on power-on | merged | v3.6.6 | — |
| #49 Bogong catalog gaps | merged | v3.6.6 | — |
| #50 fix temp_float test | merged | v3.6.6 | — |
| #51 AGENTS.md | merged | v3.6.6 | — |
| #52 pre-commit | merged | v3.6.6 | — |
| #53 redact userId | merged | v3.6.6 | — |
| #54 HTTP 500/502/504 mapping | merged | v3.6.6 | — |
| #56 README Bogong verified | merged | v3.6.6 | — |
| #57 case-insensitive mode + Bogong docs | merged | **none** | merged 2026-05-25 |
| #61 switch post-cmd refresh | merged | v3.6.7 | — |
| #62 applianceState power-off detection | merged | **none** | merged 2026-05-25 |
| #63 guard set_temp when off | **OPEN** | — | test plan unchecked |
| #64 phantom-filter regression test | merged | **none** | merged 2026-05-21 |

**Open AC issues**:

- **#43** talondnb — phantom turn-on: HA entity flips to `cool`, physical unit stays off. Owner kept open after `/closes #57` auto-close. Likely fixed by #62 (applianceState read).
- **#59** mine — `number.<name>_target_temperature_f` stuck at 16.0°F regardless of `_c` setpoint.
- **#58** mine — `select.<name>_mode` shows `unknown` when device reports disabled value (`autoClean`, `OFF`).

**Out of scope**: #66 (WM "appliance is on"), #65 (oven guided programs), #67/#68/#69 (coserotondo appliances), #44 (DQ90X fridge), #42 (WM load weight). Re-evaluate after AC work done.

## Plan

### 1. Code changes

#### 1.1 PR #63 — guard set_temperature when off
- Branch: `fix/climate-set-temp-guard` (already pushed, code complete, tests written).
- Action: switch to branch, run tests locally, manual test on Bogong unit, tick test plan boxes, mark ready-for-review.
- No code edits expected.

#### 1.2 Issue #59 — sync `_f` and `_c` target temperature

Owner hint: ovens already do F↔C conversion. Read oven path before designing.

Files to inspect: `custom_components/electrolux/number.py`, `catalogs/catalog_ac.py`, plus oven catalog.

Proposed approach (revisit after reading oven code):
- `_f` reads as `round(_c * 9/5 + 32)` from the same source DPCode.
- `_f` write converts `(value - 32) * 5/9` → sends `targetTemperatureC` to API.
- Step = 1°F. Min/max derived from C bounds in capability.

Open design choice — confirm during implementation:
- Mirror C entity (option A) vs. hide F entity for AC if oven pattern is different (option B).

#### 1.3 Issue #58 — disabled-mode read-only display

Currently:
- `options` excludes disabled values (correct).
- `current_option` returns `""` when device reports disabled value → HA shows `unknown`.

Approach: when reported `mode` matches a disabled capability value, inject the translated label into `options` for the duration the device is in that state and return it as `current_option`. Pop it on next state change.

Risk: HA logs a warning when `current_option` not in `options`. Injection avoids the warning. Verify no regressions for non-disabled values.

Tests:
- `mode=autoClean` (disabled) → `current_option = "Auto Clean"`, `"Auto Clean" in options`.
- After `mode=cool` → `"Auto Clean"` removed from options.
- `mode=OFF` (disabled, transient) → handled the same way OR explicitly skipped if PR #57 already covers.

### 2. HA upgrade

Installed: v3.6.6. Need: upstream main (contains #57, #62, #64).

Steps:
1. `git fetch upstream && git checkout upstream/main -- custom_components/electrolux/` into a clean working tree.
2. Bump `version: "3.6.7+main.<short-sha>"` for traceability.
3. `scp -r custom_components/electrolux/ root@10.170.248.4:/config/custom_components/`.
4. Restart HA.
5. Confirm logs clean, no schema warnings.

When v3.6.8 ships, revert to HACS-tracked.

### 3. Issue/PR comments

- **#43**: after live-verifying #62 (applianceState path) fixes phantom turn-on, comment confirming and ask talondnb to re-test on main build.
- **#63**: tick manual test plan boxes once tests pass on live unit; mark ready-for-review (currently effectively open but unchecked).
- **#58 / #59**: link draft PRs when ready.

### 4. Manual testing (live, 3 Bogong WSD27HWAI units)

Test matrix:

| # | Test | Source PR | Expected |
|---|------|-----------|----------|
| T1 | Physical remote OFF → HA `hvac_mode` | #62 | `off` (not last active mode) |
| T2 | Physical remote ON `heat` → HA `select.mode` | #57 | `Heat` (lowercase `heat` matched, no duplicate option) |
| T3 | Physical remote OFF → mode options unchanged | #57 | no `OFF` injected into options |
| T4 | HA `set_temp` while off, with `hvac_mode=cool` | #63 | 3 cmds (ON, mode, temp); final state correct |
| T5 | HA `set_temp` while off, no `hvac_mode` | #63 | `HomeAssistantError`, zero API calls |
| T6 | Set 25°C → `_f` reads 77 | #59 fix | synced |
| T7 | Set 70°F → `_c` reads ~21 | #59 fix | synced |
| T8 | Trigger autoClean from remote → `select.mode` | #58 fix | `Auto Clean` shown, no `unknown` |
| T9 | Restart HA mid-cool → `_last_user_temperature` restored | #48 | last setpoint restored on next power-on |

Capture HA debug log per failed test, attach to corresponding issue.

### 5. Cleanup (housekeeping; do at end)

- Add `.envrc` to `.gitignore` (contains live API tokens).
- Move local diagnostic dumps (`956007*.txt`, `config_entry-electrolux-*.json`) to `~/nobackup/electrolux-diagnostics/` or delete if no longer needed.
- Delete squash-merged local branches:
  ```
  fix/climate-appliance-state-power
  fix/climate-restore-state
  fix/select-case-insensitive-mode-lookup
  fix/switch-platform-setup-test
  fix/switch-post-command-refresh
  fix/test-select-temp-float
  fix/userid-redaction
  chore/pre-commit
  docs/bogong-verified-readme
  feat/bogong-catalog
  fix/climate-temp-memory
  fix/remove-log1
  ```
- Sync local `main` to `upstream/main`, fast-forward `origin/main`.
- Decide fate of `investigate/tuya-kettle-temp` (unrelated, separate project).

## Order of execution

1. Cleanup (low-risk; clears working tree).
2. HA upgrade to upstream main.
3. Manual tests T1–T3, T9 (verify already-merged fixes #57, #62, #48).
4. Manual tests T4, T5 (verify #63 code, then mark PR ready).
5. Comment on #43 with #62 result.
6. Implement #59 fix → manual tests T6, T7 → PR.
7. Implement #58 fix → manual test T8 → PR.

## Risks / unknowns

- #58 injection approach may interact with HA SelectEntity validation in unexpected ways. Read HA core SelectEntity source before coding.
- #59 oven F/C path may differ from AC catalog shape; design choice (A vs B) deferred to implementation.
- HA install via `scp` from main bypasses HACS tracking — must remember to revert when v3.6.8 lands.
- Live testing on 3 units across multiple sessions — risk of state drift between tests; reset each unit to known state (off, 22°C, cool, fan auto) at start of each block.

## Live test session — 2026-05-28 (office Bogong WSD27HWAI VM211_A_04.43.06)

HA installed: `3.6.7+main.69804d7` (upstream main copied over HACS install).

### Verified (already-merged fixes)

| Test | Source | Result | Notes |
|------|--------|--------|-------|
| T1 physical OFF → `hvac_mode=off` | #62 | ✅ | SSE `applianceState=off` arrived; despite `mode=heat`, climate state correctly `off`. |
| T2 physical heat → `select.mode=Heat` | #57 | ✅ | Lowercase `heat` matched case-insensitively, no duplicate option. |
| T3 OFF not injected into mode options | #57 | ✅ | Options remain `[Auto, Cool, Heat, Dry, Fanonly]`. |
| T9 `_last_user_temperature` across restart + power cycle | #48 | ✅ | Restored from RestoreEntity attr; re-applied on next HA-driven power-on. |

### NOT verified

- **T4 / T5** (PR #63 — `set_temp` while off): not run. Branch `fix/climate-set-temp-guard` not deployed; main currently has no off-guard, confirmed on S6 (HTTP 500 from API on `targetTemperatureC` while `applianceState=off`).

### New bugs discovered

#### Bug 1 — Trigger logic phantom-lies (root cause of #43)

**Severity**: high. Source of talondnb's "device shows 16 after mode change" complaint and the 16-flicker on every HA mode change.

`entity.py:573` (`_apply_triggers`) writes catalog mode-default `targetTemperatureC` into the local reported state cache on every mode change, with log line `"Trigger applied: mode=X → targetTemperatureC set to Y (will be confirmed by SSE)"`.

For Bogong AC, the catalog mode-default table:

| Mode | targetTemperatureC default | targetTemperatureF default |
|------|----------------------------|----------------------------|
| COOL | 16 | 60 |
| HEAT | 16 | 60 |
| AUTO | 16 | 60 |
| DRY | 16 | 60 |
| FANONLY | 23 | 73 |

Live evidence: with device actually at `targetTemperatureC=28`, cycling `select.office_mode` heat → auto → dry → fan_only triggered `_apply_triggers` to write 16, 16, 16, 23 respectively into HA's reported cache. Force-poll confirmed device kept 28 throughout. SSE never confirms because device didn't change.

HA UI therefore lies until next coordinator poll (every 6 h) or until a temp-setting action arrives. Automations reading `temperature` see false setpoint changes.

**Fix candidates**:
1. Stop writing trigger defaults into `reported` state for `targetTemperatureC` / `targetTemperatureF`. Catalog defaults are init hints, not live state.
2. Skip trigger write when the property already has a value in `reported`.
3. For climate-card path, the existing `#48` re-apply masks the dip but causes a brief flicker; for select-dropdown path, no re-apply, so phantom value persists.

#### Bug 2 — `_last_user_temperature` cache pollution

`climate.async_set_temperature` writes `self._last_user_temperature = float(temperature)` BEFORE `await self._send_command(...)`. If the API rejects (e.g. HTTP 500 on off device), the cache keeps the bad value. Next HA-driven power-on / mode change re-applies the polluted value via #48's re-apply path.

Reproduced live (S6 → S7): drag temp to 19 while off → API 500 → HA cache = 19. Subsequent `set_hvac_mode=cool` re-applied 19 instead of the previous 22.

**Fix**: cache only after successful command, or roll back on exception. PR #63 happens to fix this for the off-with-hvac_mode branch but not for the fall-through path.

#### Bug 3 — `hvac_mode` kwarg ignored when device ON

`climate.set_temperature(temperature=X, hvac_mode=Y)` is a standard HA combined call. Main code drops `hvac_mode` when device is ON (only handled inside PR #63's off-state branch). Reproduced S8: `temperature=28, hvac_mode=heat` while device cool → only temp command sent, mode stayed cool.

**Fix**: always honour `hvac_mode` kwarg if provided; route through `async_set_hvac_mode` first when current and requested differ.

#### Bug 4 — `number.async_set_native_value` missing off-guard

PR #63 only patches `climate.py`. Direct drag of `number.<name>_target_temperature_c` while `applianceState=off` still hits API HTTP 500. PR #63 should be extended to the number entity path.

#### Bug 5 — `targetTemperatureC` slider not disabled in fan_only / dry

While `mode=fan_only` (or `dry`), API rejects `targetTemperatureC` commands with HTTP 406 `COMMAND_VALIDATION_ERROR: Capability disabled`. UI should disable the temp slider in those modes (or warn via `extra_state_attributes`). Repro: any temp drag while `mode=fan_only` → 406 toast.

### Updated follow-up list

After this session:

1. Deploy PR #63 branch, run T4/T5, mark PR ready.
2. **Bug 1 (trigger phantom-lies)**: highest priority — fixes #43, eliminates 16-flicker. Open issue + draft PR.
3. **Bug 2 (cache pollution)**: include in PR #63 scope (already partially addressed) or separate small PR.
4. **Bug 3 (`hvac_mode` ignored)**: separate PR; behaviour change, needs tests.
5. **Bug 4 (number off-guard)**: extend PR #63.
6. **Bug 5 (fan_only/dry slider)**: separate PR; capability-aware availability on number entities.
7. Comment #43 with summary of findings (#62 fixes the SSE-side, but #43 root cause is Bug 1, separate PR coming).

