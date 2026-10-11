# Weather-based Salt Cell (SWC) Output Automation

## Goal
Let the user set a "sunny" salt cell output % and a "cloudy" output %. The driver
picks one based on current weather and pushes it to the chlorinator.

## Findings
- Driver currently has no SWC control at all (only reads `pool_salinity`).
- Read API (confirmed by an open-source probe): `session.json?command=get_swc_config`
  returns `poolSWCSP`, `spaSWCSP`, `boostStatus`.
- Write API: **not publicly documented.** Best candidate is `set_swc_config` with
  `poolSWCSP=<pct>`, but this must be verified against a live account
  (e.g. capture the request the iAqualink web/app sends when changing output).
- Hubitat *drivers* cannot subscribe to other devices (only apps can), so the
  weather source must either be fetched by the driver or pushed in via a command.

## Decisions (from user)
- Weather source: **both**. Open-Meteo by default, plus a `setWeatherCondition` override command.
- SWC write: **best guess** `set_swc_config` with full response logging + post-write re-read.

## Plan
- [x] Preferences: enable toggle, sunny %, cloudy %, cloud-cover threshold % (runs on existing update interval)
- [x] Attributes: `swc_pool_output`, `swc_spa_output`, `weather_condition`, `cloud_cover`
- [x] Read current output via `get_swc_config` each update cycle (only when automation enabled)
- [x] Weather: Open-Meteo `current=cloud_cover,is_day`; night = cloudy; 10-pt hysteresis
- [x] `setWeatherCondition(auto|sunny|cloudy)` override command
- [x] `setSaltOutput(pct)` command (clamped 0-100); spa output sent back unchanged
- [x] Only write when target differs from current output
- [x] Skip while boost is active (spa mode doesn't matter: only the pool setpoint is changed)
- [x] README section
- [x] Verify Groovy compiles + logic tests

## Review
- Compiled the driver with Groovy 4 and ran 15 scenario checks against stubbed Hubitat APIs:
  sunny/cloudy, no redundant writes, hysteresis both directions, night, boost, override
  persist/resume, clamping, ignored-write warning, no-SWC system, unset preference defaults. All pass.
- **Not verified on real hardware:** the `set_swc_config` command and the `boostStatus` value format.
  Open-Meteo is blocked from the dev sandbox, so the request shape follows its documented API.
- Revision 2 (user feedback): removed "night = cloudy"; added Cloudy Delay (0-24 h, 0 = immediate).
  Delay applies only sunny→cloudy, resets when it clears, and doesn't apply to manual overrides.
  Hysteresis now tracks the raw condition in `state.rawWeather`, separate from the delayed
  `weather_condition` attribute. Test suite expanded to 23 checks, all passing.
- Weather failures are caught in `updateStates()`, so they never break normal status updates.
