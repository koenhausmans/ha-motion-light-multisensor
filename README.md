# Smart Lighting Blueprints (Home Assistant)

A pair of Home Assistant automation blueprints that share the same sun-elevation–based
brightness engine (sine curve, separate morning/evening minimums, optional auto-scaling
to today's peak sun):

1. **Motion Light (multi-sensor)** — motion-activated lights for multiple sensors per
   room, with a cancellable off-delay, periodic brightness refresh while occupied, and
   optional manual-override protection.
2. **Switch Light + Scenes** — the same kind of lights driven by a physical button
   (Shelly) or dimmer (Philips Hue / Zigbee2MQTT): single press toggles, double / triple
   press cycle through scenes, and the Hue up / down keys step brightness.

## Repository layout

```
.
├── AGENTS.md                        # guidance for AI agents
├── LICENSE
├── README.md
├── motion_light_multisensor.yaml    # Blueprint 1 – motion-driven
└── switch_light_scenes.yaml         # Blueprint 2 – button/switch-driven
```

The repository lives at
[github.com/koenhausmans/ha-smart-lighting-blueprints](https://github.com/koenhausmans/ha-smart-lighting-blueprints).

---

## Blueprint 1 — Motion Light (multi-sensor)

Turns light(s) ON when ANY selected motion sensor detects motion, and OFF (with a 1s
fade) only once ALL of them have been clear for an optional delay. New motion cancels the
pending off. While occupied, brightness is refreshed on a timer — but only on lights that
are already on, and (with the optional helper) only on lights you haven't manually
changed.

### Install (one-click import)

[![Open your Home Assistant instance and show the blueprint import dialog.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fkoenhausmans%2Fha-smart-lighting-blueprints%2Fmaster%2Fmotion_light_multisensor.yaml)

Or manually: **Settings → Automations & Scenes → Blueprints → Import Blueprint**, and
paste this raw URL:

```
https://raw.githubusercontent.com/koenhausmans/ha-smart-lighting-blueprints/master/motion_light_multisensor.yaml
```

### Requirements

- A `sun.sun` entity (default in HA).
- For per-room manual-override protection: one `input_number` helper per room.

---

## Blueprint 2 — Switch Light + Scenes

Drives the same lights from a physical button instead of motion. Two hardware families
are supported, and you can use either or both (leave the other device empty):

| Press | Shelly (`single_push` button) | Hue / Zigbee2MQTT dimmer (`action`) |
|---|---|---|
| Single / ON | **Toggle** — off → on at sun-based brightness; on → off | `on_press` → on at sun-based brightness |
| Double | Next scene: advance the input_select, then activate that scene | `on_press` ×2 → next scene *(needs multi-press helper)* |
| Triple | Advance two scenes, then activate the result | `on_press` ×3 → two scenes forward *(needs multi-press helper)* |
| Off | — | `off_press` → off |
| Up / Down | — | `up_press` / `down_press` → step brightness ±step% |

Whenever the lights are turned **off** (Shelly toggle-off or Hue `off_press`), the scene
`input_select` is reset to its **first** option.

- **The Shelly cycles scenes natively** (a single-button Shelly exposes single/double/triple
  push). **The Hue ON key cycles scenes via _virtual_ multi-press** — the dimmer has no
  native double/triple, so presses are counted in software. This is **opt-in**: it only
  activates when you assign the multi-press `input_text` helper below. Leave the helper
  empty and the Hue ON key fires a single press instantly, exactly as before.
- **Multi-press latency:** when the helper is configured, a single and a double ON press
  act only after the configured max-delay window (default 500 ms), since the automation
  must wait to see whether more presses follow. A triple fires immediately on the third
  press. The Shelly is unaffected (its multi-press is native).
- **Brightness stepping is Hue-only** (it has dedicated up/down keys). This mirrors the
  hardware.
- The scene `input_select`'s options must be scene **entity_ids** (e.g.
  `scene.bedroom_relax`). Cycling selects the next option, then calls `scene.turn_on` on
  that value.
- **`shelly_button` subtype gotcha:** the push event's button identifier varies by model.
  Single-channel Shellys are usually `button1`; some report `button`. Set the input to
  match what your device exposes.

### Install (one-click import)

[![Open your Home Assistant instance and show the blueprint import dialog.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fkoenhausmans%2Fha-smart-lighting-blueprints%2Fmaster%2Fswitch_light_scenes.yaml)

Or manually paste this raw URL into the Import Blueprint dialog:

```
https://raw.githubusercontent.com/koenhausmans/ha-smart-lighting-blueprints/master/switch_light_scenes.yaml
```

### Requirements

- A `sun.sun` entity (default in HA).
- An `input_select` helper whose options are scene entity_ids (e.g.
  `input_select.bedroom_scene_entities`).
- A Shelly button and/or a Hue (MQTT / Zigbee2MQTT) dimmer device.
- *(Optional, for Hue scene cycling)* an `input_text` helper — **one per automation
  instance** — assigned to the multi-press helper input. It stores the last ON event so
  double/triple presses can be detected. Leave it empty to keep the Hue ON key instant
  (single press only).

---

## How the sun-based brightness (elevation) system works

Both blueprints compute a turn-on brightness from the sun's current elevation, so the
lights are dim around dawn/dusk and brighter at midday. The math (see the
`target_brightness` template in either YAML):

1. **Read the sun.** Take `sun.sun`'s `elevation`, and its `rising` attribute to decide
   *morning vs evening*. The minimum brightness and the low-elevation threshold are split
   on this — morning and evening can ramp differently
   (`brightness_min_morning` / `brightness_min_evening`,
   `elevation_low_morning` / `elevation_low_evening`).
2. **Normalize.** `frac` is where the current elevation sits within
   `[elevation_low, elevation_high]`, clamped to `0…1`. Below `elevation_low` → `0`
   (minimum brightness); at/above `elevation_high` → `1` (maximum brightness).
3. **Smooth.** A cosine ease — `factor = (1 − cos(frac · π)) / 2` — turns the linear
   fraction into a soft S-curve so brightness eases in and out rather than ramping
   linearly.
4. **Scale.** `brightness = brightness_min + (brightness_max − brightness_min) · factor`,
   rounded and clamped to `1…100 %`.

**Which setting controls what:**

| Setting | Controls |
|---|---|
| `brightness_min_morning` / `brightness_min_evening` | Brightness floor at/below `elevation_low` |
| `brightness_max` | Brightness ceiling at/above `elevation_high` |
| `elevation_low_morning` / `elevation_low_evening` | Sun angle where the ramp *starts* |
| `elevation_high` | Sun angle where the ramp *reaches max* (fixed mode) |
| `auto_elevation_high` | If on, sets the high angle to today's solar-noon elevation (`90 − \|latitude − declination\|`) so midday is full brightness year-round; `elevation_high` is then ignored |
| `dynamic_brightness` / `fixed_brightness` | Turn the whole sun engine off and use a flat brightness instead |

### Tuning `elevation_high`

Worth tuning: **`elevation_high`**. Peak sun is very seasonal — in the Netherlands /
Central Europe solar noon is roughly **60° midsummer** but only **~14° midwinter**. With
the default `elevation_high: 40`, a winter noon only reaches `frac ≈ 0.35`, so the light
stays fairly dim even at midday — which may be exactly what you want (dark days → softer
light), or not. For full midday brightness every day year-round, drop `elevation_high` to
around **15** (or enable **auto-scale**, which tracks it for you). The `elevation_low`
default of `0°` means the ramp tracks actual daylight; set it to **−6** to start
brightening during civil twilight.

---

## Optional: update notifications

To get "new version available" notifications for these blueprints, use the
[Blueprints Updater](https://github.com/luuquangvu/blueprints-updater) integration:

1. **Install it through HACS.** Search for *Blueprints Updater*. If it isn't in the
   default store, add `https://github.com/luuquangvu/blueprints-updater` as a custom
   repository with category **Integration**, then download it and restart Home
   Assistant.
2. **Add the integration:** **Settings → Devices & Services → Add Integration →
   Blueprints Updater**, and pick your update options (auto-update, interval, etc.).

No URL needs to be entered manually: the integration automatically detects any blueprint
that carries a `source_url` in its metadata, which Home Assistant records for you when you
import a blueprint from a raw URL above. Detected updates show up as native `update`
entities.

> **Note on the repo rename:** Home Assistant keys an imported blueprint by its
> `source_url`. If you previously imported `motion_light_multisensor.yaml` from the old
> `ha-motion-light-multisensor` URL, re-importing from the new
> `ha-smart-lighting-blueprints` URL is treated as a *new* blueprint, not an update —
> re-import once and remove the old copy.

## Versioning

Version = the GitHub release tag, e.g. `v0.2.0`:

```bash
git tag v0.2.0
git push --tags
```

Then publish it as a Release on GitHub.
