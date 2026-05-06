# Release Notes v3.6.5

### AC Fixes

- Correct HVAC mode detection for Electrolux ACs  
Electrolux air conditioners (Bogong, AC, CA, Azul, Panther, Telica) often report applianceState = "OFF" even while actively running in a valid mode (e.g. HEAT, AUTO).
The integration previously treated applianceState as the power source of truth, causing the climate entity to remain visually “Off” even though the AC was operating.
HVAC mode is now derived exclusively from the mode attribute, which correctly reflects the active operating mode for these devices.

- Climate entity now tracks mode changes reliably  
When changing HVAC mode (e.g. to HEAT, COOL, AUTO), the integration now sends the appropriate executeCommand and mode commands and optimistically updates the local mode state, so the Home Assistant UI reflects the new mode immediately instead of appearing stuck in “Off”.

#### Behavioral improvements

- AC climate entities no longer appear “stuck Off” after changing mode.

- Mode changes (HEAT/COOL/AUTO/DRY/FAN_ONLY) are now consistently reflected in the UI.

- Users no longer need to rely on the raw executeCommand entity to “unstick” the climate state.

#### Scope

- Changes apply only to AC‑type appliances (AC, CA, Azul, Panther, Bogong, Telica).

- No behavior changes for ovens, washers, dryers, dishwashers, or other non‑climate entities.