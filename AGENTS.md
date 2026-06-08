# AGENTS.md

Guidance for AI agents working in this repository.

## Project

Two Home Assistant **automation blueprints** that share one sun-elevation–based
brightness engine. There is no application code, no build step, and no test framework —
the deliverable is two YAML files plus their README.

- `motion_light_multisensor.yaml` — multi-sensor motion lighting.
- `switch_light_scenes.yaml` — button/switch-driven lighting (Shelly + Hue/Z2M) with
  scene cycling.

## Repository structure

```
.
├── AGENTS.md
├── README.md
├── motion_light_multisensor.yaml   # Blueprint 1 – motion-driven
└── switch_light_scenes.yaml        # Blueprint 2 – button/switch-driven
```

Do not add `hacs.json` or a `custom_components/` tree unless the task is explicitly
to wrap this blueprint in a custom integration. HACS has **no blueprint category**,
so a bare blueprint repo is installed via Home Assistant's native blueprint import,
not HACS.

## Validating a change

There is no CI. Before committing an edit to the blueprint:

1. **Structure check.** Standard YAML linters choke on Home Assistant's custom
   `!input` tag. Validate structure with a loader that ignores unknown `!` tags (run for
   each blueprint you touched):
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

### Shared (both blueprints)

- **Sun-based brightness** (`target_brightness` template) is duplicated verbatim in both
  YAML files. Keep them in sync — change one, change the other. **Morning vs evening** is
  decided by `sun.sun`'s `rising` attribute; both the minimum brightness and the
  low-elevation threshold are split on it. The **auto-scaled high threshold** uses
  `90 - |latitude - declination|` from `zone.home` latitude and day-of-year; when it is
  enabled, low thresholds must stay at/below 0° or the elevation span collapses in winter
  and brightness pins to min.

### `switch_light_scenes.yaml`

- **Triggers are keyed by `id`, not by order.** The `choose:` branches dispatch on
  `trigger.id` (`toggle`, `light_on`, `light_off`, `scene_next`, `scene_next2`,
  `bright_up`, `bright_down`). Keep ids and branches aligned.
- **Shelly single push is a toggle; Hue ON/OFF are dedicated.** `single_push` checks
  whether any light is on and turns all off (else on). Hue `on_press` always turns on,
  `off_press` always turns off — do not collapse them into a toggle.
- **Every off-path resets the scene `input_select` to its first option**
  (`input_select.select_first`): the Shelly toggle's off branch and the Hue `off_press`
  branch. Turning *on* must NOT reset it.
- **Scene cycling advances first, then activates.** Double = one `select_next`; triple =
  two `select_next`; both then `scene.turn_on` with `{{ states(scene_select) }}` (the
  input_select's options are scene entity_ids).
- **Up/down use `brightness_step_pct`** (`+step` / `-step`), not absolute brightness, so
  HA clamps and applies the delta.
- **`mode: parallel` (with `max`)** — NOT `queued` and NOT `restart`. The Hue virtual
  multi-press counts overlapping runs (see below): `queued` serialises them so the second
  press can't start until the first run's delay ends (double/triple never detected), and
  `restart` would cancel an in-flight Shelly scene activation. Parallel keeps "rapid
  presses are never dropped" while letting the counting runs overlap.
- **Virtual multi-press is Hue-ON-only and opt-in.** The Hue ON key has no native
  multi-press, so it is counted in software using an optional `input_text` helper
  (`helper_last_controller_event`) storing `{"a","n","t"}` and a max-delay input
  (`helper_double_press_delay`). The `on_count` variable is self-contained (parses the
  helper itself, referencing only `!input`s per the evaluation-order rule) and increments
  the prior ON streak only when the previous event was `on_press` within the delay window,
  capped at 3. **Triple acts immediately** at count 3;
  **single/double act after the delay**, gated by a post-delay re-check that the helper's
  `t` still equals the `this_ts` this run wrote (a later press overwrites it → the stale
  run aborts — this is what makes parallel mode correct without `restart`). When enabled,
  **every Hue key stamps the helper first** (off/up/down too) so a following press
  invalidates a pending ON single/double. When the helper is empty (`dp_enabled` false)
  the ON key turns on instantly and no key touches the helper. The Shelly keeps its
  **native** single/double/triple and never touches the helper.
- **Device inputs are optional** (`shelly_device` / `hue_device` default to empty) so a
  user with one hardware family isn't forced to select the other. The `shelly_button`
  subtype varies by model (`button1` vs `button`) and is a configurable input.

### `motion_light_multisensor.yaml`

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

(See **Shared** above for the sun-based brightness invariants.)

## Home Assistant templating rules to respect

- `!input` cannot be used inside a Jinja template. Capture it in `variables:` first
  (and in `trigger_variables:` for templates used inside triggers).
- Inside the `variables:` block, templated variables should reference only literal
  `!input` variables, not other templated variables (evaluation-order safety).
  `tick_targets` follows this; keep it that way.

## Conventions

- All blueprints are `blueprint.domain: automation`; one automation per YAML file.
- Distribution: native blueprint import via the `my.home-assistant.io` badges in the
  README. Keep each file path stable — HA keys imported blueprints by `source_url`, so
  renaming a file or the repo creates a duplicate instead of an update. The repo was
  renamed from `ha-motion-light-multisensor` to `ha-smart-lighting-blueprints`; that
  rename already reset the `source_url` for existing motion-blueprint users (a one-time
  re-import). Avoid further renames.
- Versioning: GitHub **release tags** only (`vMAJOR.MINOR.PATCH`). There is no version
  field inside the YAML. Cut a release for each user-facing change.
- Keep `README.md` input descriptions in sync with each blueprint's `input:` block.
