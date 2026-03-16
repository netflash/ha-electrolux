# Release Notes v3.6.3

### Translations
- **Spanish (es.json)** — fixed exception message placeholders to match English template variable names:
  - `appliance_offline`: `{estado}` → `{state}`
  - `not_supported_by_program`: `{programa}` → `{program}`
  - `food_probe_not_supported_by_program`: `{programa}` → `{program}`
  - `food_probe_locked_by_program`: `{valor}`, `{programa}` → `{value}`, `{program}`
  - `invalid_preset_mode`: `{modo}`, `{modos}` → `{mode}`, `{modes}`
- **All other language files** — fixed placeholder mismatches to align with English template:
  - Czech, Danish, Dutch, Finnish, German, Hungarian, Luxembourg, Norwegian, Polish, Romanian, Russian, Slovak, Swedish — corrected placeholder names in exception messages to use English identifiers (`{state}`, `{program}`, `{value}`, `{mode}`, `{modes}`) instead of translated names
  - Italian — fixed `invalid_preset_mode` to use `{modes}` instead of duplicate `{mode}`