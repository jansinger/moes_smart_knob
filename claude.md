# MOES Smart Knob Blueprint Project

## Project Goal

Create a Home Assistant blueprint for the **MOES Smart Knob (Tuya ERS-10TZBVK-AA)** via **Zigbee2MQTT** to control:

1. **Media Player volume** (Arcam AV40) via knob rotation
2. **Home Cinema On script** via single press
3. **Home Cinema Off script** via double press

The home cinema scripts control: Arcam AV40 media player, JVC Beamer, Apple TV.

## Target Environment

- **Home Assistant:** 2026.2.1
- **Zigbee2MQTT:** latest (2.x series)
- **Device:** Tuya ERS-10TZBVK-AA (MOES Smart Knob)
- **Controlled device:** Arcam AV40 via `arcam_fmj` integration

---

## Device Technical Reference

### MOES Smart Knob (ERS-10TZBVK-AA)

**Source:** https://www.zigbee2mqtt.io/devices/ERS-10TZBVK-AA.html

**Exposed properties:** `action`, `action_step_size` (0-255), `action_transition_time`, `action_rate` (0-255), `battery` (0-100%), `operation_mode` ("command" | "event")

### Operation Modes

The knob has two modes, switched by **triple-click**:

#### EVENT Mode (recommended for this project)

| User Action    | `action` value |
|----------------|----------------|
| Rotate Left    | `rotate_left`  |
| Rotate Right   | `rotate_right` |
| Single Click   | `single`       |
| Double Click   | `double`       |
| Hold (>3s)     | `hold`         |

- `action_step_size`, `action_transition_time`, `action_rate` are **null** in event mode

#### COMMAND Mode

| User Action                   | `action` value                  |
|-------------------------------|---------------------------------|
| Rotate Left                   | `brightness_step_down`          |
| Rotate Right                  | `brightness_step_up`            |
| Single Click                  | `toggle`                        |
| Push+Hold >3s                 | `hue_move`                      |
| Release after hold            | `hue_stop`                      |
| Push+Hold + Rotate Left       | `color_temperature_step_down`   |
| Push+Hold + Rotate Right      | `color_temperature_step_up`     |

- `action_step_size` is 0-255 in command mode (indicates rotation amount)
- `action_rate` is 0-255

### MQTT Topics

- **State topic:** `zigbee2mqtt/<FRIENDLY_NAME>` (JSON payload with all properties)
- **Action topic:** `zigbee2mqtt/<FRIENDLY_NAME>/action` (plain text action value)
- **Set operation mode:** publish `{"operation_mode": "event"}` to `zigbee2mqtt/<FRIENDLY_NAME>/set`
- **Read operation mode:** publish `{"operation_mode": ""}` to `zigbee2mqtt/<FRIENDLY_NAME>/get`

### Example MQTT Payload (state topic)

```json
{
  "action": "single",
  "action_rate": null,
  "action_step_size": null,
  "action_transition_time": null,
  "battery": 100,
  "brightness": 182,
  "operation_mode": "event",
  "linkquality": 109
}
```

### Known Issues

- **Group 0 toggle problem:** The toggle action in command mode may control unexpected devices because manufacturers place devices in group 0 by default. Fix: create a Z2M group with a different ID and add the knob to it.
- **action_step_size always zero:** Some users report `action_step_size` staying at 0 regardless of configuration. This is unreliable for volume scaling.
- **Fast rotation detection:** Multiple quick rotations may register as single increments. The blueprint should handle this gracefully.

---

## Arcam AV40 Integration

**Source:** https://www.home-assistant.io/integrations/arcam_fmj

- Integration: `arcam_fmj`
- Entity type: `media_player` (zone-based entities)
- Communication: Local Polling (IoT class)
- Setup: Config flow (auto-discovery or manual via Settings > Devices & Services)

### Power Control Limitation

Arcam receivers **turn off their network port in standby**. The integration retries connection every 5 seconds. Workarounds for power-on:

1. **Newer models (AV40):** Enable "HDMI & IP On" under HDMI Settings
2. **IR blaster:** Send RC5 codes (Zone 1: Device 16, Function 123) via `arcam.turn_on` event
3. **Serial gateway:** Most reliable communication method

### Relevant Services

| Service                        | Parameters                           |
|--------------------------------|--------------------------------------|
| `media_player.volume_set`      | `volume_level` (float 0.0-1.0)      |
| `media_player.volume_up`       | (none)                               |
| `media_player.volume_down`     | (none)                               |
| `media_player.volume_mute`     | `is_volume_muted` (bool)            |
| `media_player.turn_on`         | (none)                               |
| `media_player.turn_off`        | (none)                               |
| `media_player.media_play`      | (none)                               |
| `media_player.media_pause`     | (none)                               |

---

## Blueprint Design Decisions

### Trigger Strategy: Direct MQTT Topic

Use `trigger: mqtt` subscribing to the `/action` topic. This is the **most reliable and blueprint-friendly** approach.

**Rationale:**
- MQTT Device Triggers (recommended by Z2M docs) **cannot be used in blueprints** because they require a specific `device_id` that cannot be templated.
- Event entities (Z2M 2.0 experimental) are still subject to change and require `homeassistant: {experimental_event_entities: true}`.
- Direct MQTT trigger on the `/action` topic is proven, reliable, and works across Z2M versions.
- The `/action` topic publishes plain text action names (not JSON), making payload matching simple.

### Operation Mode: EVENT

Use **event mode** for the blueprint:
- Provides `single`, `double`, `rotate_left`, `rotate_right` - exactly what we need
- `double` is only available in event mode (not in command mode)
- Trade-off: no `action_step_size` in event mode, so volume step must be configured as a blueprint input

### Volume Control Approach

Use `media_player.volume_set` with calculated volume level:

```yaml
# Pattern for volume adjustment
{% set current = state_attr(entity_id, 'volume_level') | float(0) %}
{% set step = step_percent / 100 %}
{% set new_vol = current + step %}  # or - step for volume down
{{ [[ new_vol, 0.0 ] | max, 1.0] | min }}
```

**Why `volume_set` instead of `volume_up`/`volume_down`:**
- `volume_set` provides precise, configurable step sizes
- `volume_up`/`volume_down` use a fixed device-defined increment
- For the Arcam AV40, fine-grained control is preferred

### Automation Mode: `restart`

Use `mode: restart` so rapid knob rotations cancel any in-progress actions and apply the latest rotation immediately. This prevents queuing issues with fast knob turns.

---

## Blueprint Schema Best Practices (HA 2026.2)

**Source:** https://www.home-assistant.io/docs/blueprint/schema/

### Required Blueprint Fields

```yaml
blueprint:
  name: "Short descriptive name"
  description: "Markdown-supported description"
  domain: automation  # or script, template
  homeassistant:
    min_version: "2024.6.0"  # if using input sections
  author: "Author Name"
  input:
    # input definitions
```

### Input Selectors

| Selector Type | Use Case                        |
|---------------|---------------------------------|
| `entity`      | Select HA entity                |
| `device`      | Select HA device                |
| `action`      | Define action sequences          |
| `number`      | Numeric slider/box              |
| `text`        | Text input                      |
| `select`      | Dropdown options                |
| `target`      | Entity/device/area targeting    |

### Input Sections (v2024.6.0+)

Group related inputs visually with collapsible sections:

```yaml
input:
  section_name:
    name: "Section Title"
    icon: mdi:icon-name
    collapsed: true
    description: "Section description"
    input:
      my_input:
        name: "Input Name"
        selector:
          entity: {}
```

### Trigger Syntax (Modern)

Use `trigger:` keyword (not `platform:` at root level):

```yaml
trigger:
  - trigger: mqtt       # modern syntax
    topic: "..."
```

Note: `platform: mqtt` still works but `trigger: mqtt` is the newer style.

### trigger_variables

Use `trigger_variables` for values needed in trigger templates (evaluated at trigger setup time, not at runtime):

```yaml
trigger_variables:
  my_var: !input some_input
trigger:
  - trigger: mqtt
    topic: "{{ my_var }}"
```

---

## Reference Blueprints

### 1. pbergman Z2M Blueprint (generic, all actions)

**Source:** https://github.com/pbergman/ha-blueprints/blob/3fc7a18cd8ea34a82f5a955beea13c02a5f805d7/ERS-10TZBVK-AA.yaml

**Key patterns:**
- Uses `trigger_variables` for MQTT topic construction
- Dual trigger: MQTT `/action` topic + state entity for mode changes
- `choose` block maps payload strings to `!input` action sequences
- Handles both event and command mode actions
- `mode: single`, `max_exceeded: silent`

### 2. Existing ZHA Media Blueprint (in this repo)

**File:** `blueprint-zha-media.yaml`

**Key patterns (to adapt from ZHA to Z2M):**
- Volume control via `media_player.volume_set` with calculated level
- Step percent as configurable input (default 10%)
- Checks media player is on before adjusting volume
- Uses `service_template` (deprecated - use `service` or `action` instead)

**Issues to fix in Z2M version:**
- Replace `service_template` with `action` (modern HA syntax)
- Replace `data_template` with `data` (modern HA syntax)
- Remove unnecessary `repeat` loop (`repeat.index < 2` runs only once)
- Add double press support (not available in ZHA version)
- Use direct MQTT trigger instead of `zha_event`

### 3. Improved Light Control Blueprint

**Source:** https://community.home-assistant.io/t/zigbee2mqtt-control-light-entity-including-press-turn-with-tuya-moes-smart-knob-ers-10tzbvk-aa-v1-1/787779

**Key patterns:**
- Extracts `action_step_size` from `trigger.payload_json` with fallback
- Uses multiplier for sensitivity tuning
- Clamps values to prevent over/underflow

---

## Blueprint Implementation Plan

### File: `blueprint-z2m-media.yaml`

**Inputs:**
- `mqtt_device_name` - Z2M friendly name of the knob
- `base_topic` - Z2M base topic (default: `zigbee2mqtt`)
- `media_player` - Target media player entity (Arcam AV40)
- `volume_step` - Volume change per rotation tick (default: 5, range 1-20, as percentage)
- `single_press_action` - Action for single press (home cinema ON script)
- `double_press_action` - Action for double press (home cinema OFF script)

**Trigger:** MQTT on `{{ base_topic }}/{{ mqtt_device_name }}/action`

**Actions mapped:**
| Knob Action    | Blueprint Action                     |
|----------------|--------------------------------------|
| `single`       | Run single_press_action input        |
| `double`       | Run double_press_action input        |
| `rotate_right` | Volume up by step_percent            |
| `rotate_left`  | Volume down by step_percent          |
| `hold`         | (optional: mute toggle)              |

**Volume calculation:**
```yaml
action: media_player.volume_set
target:
  entity_id: !input media_player
data:
  volume_level: >-
    {% set current = state_attr(media_player_id, 'volume_level') | float(0) %}
    {% set step = volume_step / 100 %}
    {% set new_vol = current + step %}
    {{ [[new_vol, 0.0] | max, 1.0] | min }}
```

---

## Important Syntax Notes for HA 2026.2

1. **Use `action:` instead of `service:`** - `service:` is deprecated in favor of `action:` for calling services in automations
2. **Use `data:` instead of `data_template:`** - templates are auto-detected in `data:`
3. **Use `trigger: mqtt` instead of `platform: mqtt`** - newer trigger syntax
4. **Use `!input` references in `trigger_variables`** - not directly in trigger config
5. **Blueprint `min_version`** - set if using features like input sections

---

## Sources

- [Zigbee2MQTT ERS-10TZBVK-AA Device Page](https://www.zigbee2mqtt.io/devices/ERS-10TZBVK-AA.html)
- [Home Assistant Blueprint Schema](https://www.home-assistant.io/docs/blueprint/schema/)
- [Home Assistant MQTT Trigger Docs](https://www.home-assistant.io/docs/automation/trigger/#mqtt-trigger)
- [Home Assistant Media Player Services](https://www.home-assistant.io/integrations/media_player/)
- [Arcam FMJ Integration](https://www.home-assistant.io/integrations/arcam_fmj)
- [pbergman Z2M Blueprint (GitHub)](https://github.com/pbergman/ha-blueprints/blob/3fc7a18cd8ea34a82f5a955beea13c02a5f805d7/ERS-10TZBVK-AA.yaml)
- [SmartHomeCircle MOES Knob Guide](https://smarthomecircle.com/moes-zigbee-smart-knob-with-homeassistant)
- [Z2M 2.0 Action Events Discussion](https://community.home-assistant.io/t/using-the-new-action-events-in-zigbee2mqtt-2-0/821709)
- [Z2M HA Integration Guide](https://www.zigbee2mqtt.io/guide/usage/integrations/home_assistant.html)
- [Improved Light Blueprint](https://community.home-assistant.io/t/zigbee2mqtt-control-light-entity-including-press-turn-with-tuya-moes-smart-knob-ers-10tzbvk-aa-v1-1/787779)
- [MOES Knob Blueprint Exchange Thread](https://community.home-assistant.io/t/zigbee2mqtt-tuya-moes-smart-knob-ers-10tzbvk-aa/419989)
- [Universal Smart Knob Blueprint](https://community.home-assistant.io/t/zha-deconz-zigbee2mqtt-tuya-ers-10tzbvk-aa-smart-knob-universal-blueprint-all-actions-double-click-events-control-lights-media-players-and-more-with-hooks/870899)
- [Z2M MQTT Topics and Messages](https://www.zigbee2mqtt.io/guide/usage/mqtt_topics_and_messages.html)
- [action_step_size Discussion](https://community.home-assistant.io/t/tuya-ers-10tzbvk-aa-and-setting-action-step-size/757172)
- [Improved Tuya Smart Knob Blueprint](https://community.home-assistant.io/t/tuya-smart-knob-improved-blueprint-for-zigbee2mqtt/799168)
