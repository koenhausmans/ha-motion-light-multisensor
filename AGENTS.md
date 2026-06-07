# AGENTS.md

Guidance for AI agents working in this repository.

## Project

A single Home Assistant **automation blueprint** (`motion_light_multisensor.yaml`):
multi-sensor motion lighting with sun-elevation–based brightness. There is no
application code, no build step, and no test framework — the deliverable is one
YAML file plus its README.

## Repository structure

```
.
├── AGENTS.md
├── README.md
└── motion_light_multisensor.yaml   # the blueprint (the whole product)
```

Do not add `hacs.json` or a `custom_components/` tree unless the task is explicitly
to wrap this blueprint in a custom integration. HACS has **no blueprint category**,
so a bare blueprint repo is installed via Home Assistant's native blueprint import,
not HACS.

## Validating a change

There is no CI. Before committing an edit to the blueprint:

1. **Structure check.** Standard YAML linters choke on Home Assistant's custom
   `!input` tag. Validate structure with a loader that ignores unknown `!` tags:
   ```bash
   python3 -c "import yaml; \
   yaml.SafeLoader.add_multi_constructor('!', lambda l,s,n: None); \
   yaml.load(open('motion_light_multisensor.yaml'), Loader=yaml.SafeLoader); \
   print('ok')"
   ```
2. **Template check.** Paste the `target_brightness` (and `tick_targets`) Jinja into
   Home Assistant → Developer Tools → Template and confirm it renders a number / a
   list of entity_ids before trusting it.
3. **Behavior check.** Create an automation from the blueprint and exercise: single
   sensor on/off, multiple sensors (off only when ALL clear), re-entry during the
   off window, and a manual brightness change followed by a tick.

## Critical invariants — do NOT break these

These encode bugs that were deliberately engineered away. "Simplifying" any of them
reintroduces a real failure.

- **`mode: restart` is mandatory.** It is what makes a returning-motion event cancel
  a pending turn-off.
- **The off-delay lives in the `all_clear` template trigger's `for:`, never as an
  in-action `delay` step.** With `mode: restart`, the periodic `tick` trigger would
  restart the automation and silently drop an in-action delay, leaving lights on.
  For the same reason, turn-off uses `transition: 1` (a bulb-side fade), not a delay.
- **"All clear" = template over all sensors**, not a multi-entity state trigger with
  `for` (that fires when *any one* sensor clears, not all). Keep
  `expand(...) | selectattr('state','eq','on') | list | count == 0`.
- **The `tick` branch must only touch lights that are already on** (via `tick_targets`)
  — it must never turn an off light back on.
- **Manual-override protection** compares a light's current brightness to the last
  automatic value stored in the optional `input_number` helper, within
  `override_tolerance`. The tolerance exists because `brightness_pct` ↔ 0–255
  `brightness` round-trips can be off by ~1; do not set the default to 0.
- **Morning vs evening** is decided by `sun.sun`'s `rising` attribute. Both the
  minimum brightness and the low-elevation threshold are split on it.
- **Auto-scaled high threshold** uses `90 - |latitude - declination|` from
  `zone.home` latitude and day-of-year. When it is enabled, low thresholds must stay
  at/below 0° or the elevation span collapses in winter and brightness pins to min.

## Home Assistant templating rules to respect

- `!input` cannot be used inside a Jinja template. Capture it in `variables:` first
  (and in `trigger_variables:` for templates used inside triggers).
- Inside the `variables:` block, templated variables should reference only literal
  `!input` variables, not other templated variables (evaluation-order safety).
  `tick_targets` follows this; keep it that way.

## Conventions

- One automation blueprint per repo; `blueprint.domain: automation`.
- Distribution: native blueprint import via the `my.home-assistant.io` badge in the
  README. Keep the file path stable — HA keys imported blueprints by `source_url`,
  so renaming the file or repo creates a duplicate instead of an update.
- Versioning: GitHub **release tags** only (`vMAJOR.MINOR.PATCH`, currently `v0.1.0`).
  There is no version field inside the YAML. Cut a release for each user-facing change.
- Keep `README.md` input descriptions in sync with the blueprint's `input:` block.
