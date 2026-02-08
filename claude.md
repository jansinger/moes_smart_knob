# MOES Smart Knob Blueprint Project

## Project Goal

Create a Home Assistant blueprint for the **MOES Smart Knob (Tuya ERS-10TZBVK-AA)** via **Zigbee2MQTT** to control:

1. **Media Player volume** (Arcam AV40) via knob rotation
2. **Home Cinema toggle** via single press (based on JVC projector state)

The home cinema scripts control: Arcam AV40 media player, JVC Beamer, Apple TV.

## Current Implementation

**File:** `blueprint-z2m-command-mode.yaml` (v1.2.0)

**Mode:** COMMAND mode (triple-click to switch)

| Knob Action | MQTT Action | Result |
|-------------|-------------|--------|
| Rotate right | `brightness_step_up` | Volume up (scaled by action_step_size) |
| Rotate left | `brightness_step_down` | Volume down (scaled by action_step_size) |
| Single press | `toggle` | Cinema ON if JVC=standby, OFF if JVC=on |
| Hold release | `hue_stop` | Shows current volume notification |

**User's Entity IDs:**
- Media player: `media_player.arcam_...`
- JVC power sensor: `sensor.jvc_projector_energiestatus`
- Cinema ON script: `script.heimkino_an`
- Cinema OFF script: `script.heimkino_aus`

---

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

#### COMMAND Mode (recommended - used in current implementation)

| User Action                   | `action` value                  |
|-------------------------------|---------------------------------|
| Rotate Left                   | `brightness_step_down`          |
| Rotate Right                  | `brightness_step_up`            |
| Single Click                  | `toggle`                        |
| Push+Hold >3s                 | `hue_move`                      |
| Release after hold            | `hue_stop`                      |
| Push+Hold + Rotate Left       | `color_temperature_step_down`   |
| Push+Hold + Rotate Right      | `color_temperature_step_up`     |

- `action_step_size` is 0-255 for rotation actions (indicates rotation speed/amount)
- `action_step_size` is **null** for non-rotation actions (toggle, hue_stop, etc.)
- `action_rate` is 0-255

**Advantages:**
- `action_step_size` allows intensity-aware volume control
- Single blueprint, no helpers required
- Simpler setup

**Trade-off:**
- No `double` press available (only in EVENT mode)

#### EVENT Mode (alternative)

| User Action    | `action` value |
|----------------|----------------|
| Rotate Left    | `rotate_left`  |
| Rotate Right   | `rotate_right` |
| Single Click   | `single`       |
| Double Click   | `double`       |
| Hold (>3s)     | `hold`         |

- `action_step_size`, `action_transition_time`, `action_rate` are **null** in event mode
- Requires counter helper + accumulator pattern for reliable fast rotation

### MQTT Topics

- **State topic:** `zigbee2mqtt/<FRIENDLY_NAME>` (JSON payload with all properties)
- **Action topic:** `zigbee2mqtt/<FRIENDLY_NAME>/action` (plain text action value)
- **Set operation mode:** publish `{"operation_mode": "command"}` to `zigbee2mqtt/<FRIENDLY_NAME>/set`

### Example MQTT Payload (COMMAND mode rotation)

```json
{
  "action": "brightness_step_up",
  "action_rate": 50,
  "action_step_size": 43,
  "action_transition_time": null,
  "battery": 100,
  "operation_mode": "command",
  "linkquality": 109
}
```

### Example MQTT Payload (COMMAND mode toggle)

```json
{
  "action": "toggle",
  "action_rate": null,
  "action_step_size": null,
  "action_transition_time": null,
  "battery": 100,
  "operation_mode": "command",
  "linkquality": 109
}
```

---

## Key Lessons Learned

### 1. COMMAND vs EVENT Mode

**COMMAND mode is simpler** for volume control because:
- `action_step_size` provides rotation intensity (0-255)
- No need for counter helpers or accumulator patterns
- Single automation handles everything

**EVENT mode requires two automations:**
- Accumulator (mode: queued) to count rotations
- Volume Applier (mode: restart) to apply debounced volume

### 2. Null Handling in Jinja2

**Critical:** In COMMAND mode, `action_step_size` is explicitly `null` for non-rotation actions.

```yaml
# WRONG - default() only catches MISSING keys, not null values
raw_step: "{{ trigger.payload_json.action_step_size | default(13) | int }}"
# Error: int got invalid input 'None'

# CORRECT - default(value, true) also replaces null/None values
raw_step: "{{ trigger.payload_json.action_step_size | default(13, true) | int }}"
```

### 3. Automation Mode for Rotation Events

**Use `mode: queued`** (not `restart`) for rotation handling:
- `mode: restart` drops events - only the last rotation is processed
- `mode: queued` processes every rotation event in order
- Set `max: 100` and `max_exceeded: silent` to handle rapid rotations

### 4. MQTT Topic for Blueprints

Use the **state topic** (JSON), not the `/action` topic (plain text):
- State topic: `zigbee2mqtt/<FRIENDLY_NAME>` → JSON with all properties
- Allows access to `action_step_size` from `trigger.payload_json`

### 5. Volume Calculation Pattern

```yaml
# Clamp volume between 0.0 and 1.0
new_vol: "{{ [[current_vol + step_percent, 0.0] | max, 1.0] | min }}"
```

### 6. action_step_size Reliability

**Community-Konsens:** `action_step_size` kann unzuverlässig sein!
- Manche User berichten, dass es immer 0 bleibt
- Nicht über Z2M konfigurierbar
- **Empfehlung:** Immer mit Minimum-Floor absichern

**Implementierte Lösung:**
```yaml
raw_step: "{{ trigger.payload_json.action_step_size | default(13, true) | int }}"
step_size: "{{ [raw_step, 13] | max }}"  # Minimum 13 garantiert
```

### 7. MQTT QoS für zuverlässige Zustellung

MQTT QoS 1 garantiert, dass Nachrichten mindestens einmal zugestellt werden:
```yaml
trigger:
  - trigger: mqtt
    topic: "{{ base_topic ~ '/' ~ mqtt_device_name }}"
    qos: 1  # Garantierte Zustellung
```

---

## Troubleshooting & Optimierung

### Verpasste Rotations-Events

Wenn nicht jeder Rotationsschritt ankommt, prüfe folgende Ursachen:

| Ursache | Prüfung | Lösung |
|---------|---------|--------|
| **Z2M Debounce** | Z2M → Device → Settings → Debounce | Auf 0 setzen oder leer lassen |
| **Z2M Throttle** | Z2M → Device → Settings → Throttle | Nicht setzen / deaktivieren |
| **Schlechtes Signal** | Z2M → Device → LQI-Wert | Sollte > 50 sein |
| **Queue Overflow** | HA → Automation Traces → max_exceeded | max erhöhen (aktuell: 100) |
| **Geräte-Firmware** | Nicht prüfbar | Nicht änderbar |

### Z2M Geräte-Einstellungen (empfohlen)

In Zigbee2MQTT → Devices → Smart Knob → Settings (Zahnrad):

| Einstellung | Empfehlung | Grund |
|-------------|------------|-------|
| `debounce` | **Leer (0)** | Verhindert Zusammenfassen von Events |
| `throttle` | **Nicht setzen** | Würde Events verwerfen |
| `filtered_cache` | **Deaktiviert** | Erlaubt wiederholte gleiche Werte |
| `qos` | **1 oder 2** | Garantierte Message-Zustellung |
| `retain` | **false** | Keine alten Nachrichten beim Start |

### Zigbee Netzwerk-Qualität

- **LQI (Link Quality Indicator):** Sollte > 50 sein, idealerweise > 100
- **Prüfung:** Z2M → Devices → Smart Knob → "About" Tab
- **Verbesserung:**
  - Knob näher an Router/Coordinator positionieren
  - Zigbee Router (z.B. smarte Steckdosen) als Repeater verwenden
  - Interferenzen mit WLAN (Kanal 11) vermeiden

### Automation Traces prüfen

1. HA → Einstellungen → Automationen
2. Automation auswählen → Traces
3. Prüfen auf:
   - `max_exceeded` Warnungen → Queue zu klein
   - Fehlende Trigger → Zigbee/MQTT Problem
   - Condition-Fehler → action leer

---

## Arcam AV40 Integration

**Source:** https://www.home-assistant.io/integrations/arcam_fmj

- Integration: `arcam_fmj`
- Entity type: `media_player` (zone-based entities)
- Communication: Local Polling (IoT class)

### Power Control Limitation

Arcam receivers **turn off their network port in standby**. Workarounds:
1. **AV40:** Enable "HDMI & IP On" under HDMI Settings
2. **IR blaster:** Send RC5 codes via `arcam.turn_on` event

### Relevant Services

| Service                        | Parameters                           |
|--------------------------------|--------------------------------------|
| `media_player.volume_set`      | `volume_level` (float 0.0-1.0)      |
| `media_player.volume_up`       | (none)                               |
| `media_player.volume_down`     | (none)                               |

---

## Blueprint Design Decisions

### Trigger Strategy: Direct MQTT State Topic

Use `trigger: mqtt` subscribing to the state topic (not `/action`):

```yaml
trigger_variables:
  base_topic: !input base_topic
  mqtt_device_name: !input mqtt_device_name

trigger:
  - trigger: mqtt
    topic: "{{ base_topic ~ '/' ~ mqtt_device_name }}"

condition:
  - "{{ trigger.payload_json.action | default('') != '' }}"
```

**Rationale:**
- State topic provides JSON with `action_step_size`
- MQTT Device Triggers cannot be templated in blueprints
- Condition filters out state updates without action changes

### Cinema Toggle Logic

Toggle based on JVC projector state to prevent double-triggers:

```yaml
choose:
  - conditions: "{{ states(jvc_sensor_id) == 'on' }}"
    sequence: # Turn cinema OFF
  - conditions: "{{ states(jvc_sensor_id) == 'standby' }}"
    sequence: # Turn cinema ON
  # Default: do nothing (warming, cooling, error)
```

---

## Blueprint Schema Best Practices (HA 2026.2)

### Modern Syntax

1. **Use `action:` instead of `service:`** - deprecated
2. **Use `data:` instead of `data_template:`** - templates auto-detected
3. **Use `trigger: mqtt` instead of `platform: mqtt`** - newer syntax
4. **Use `!input` in `trigger_variables`** - not directly in trigger config
5. **Use `default(value, true)`** - to also replace null values

### Input Sections (v2024.6.0+)

```yaml
input:
  section_name:
    name: "Section Title"
    icon: mdi:icon-name
    collapsed: true
    input:
      my_input:
        selector:
          entity: {}
```

---

## Archived Blueprints

Previous EVENT mode implementation (in `archive/` folder):
- `blueprint-z2m-knob-accumulator.yaml` - Part 1: Event capture with counter
- `blueprint-z2m-volume-applier.yaml` - Part 2: Debounced volume application
- `blueprint-z2m-media.yaml` - Legacy single-file EVENT mode

These required helpers (counter, input_number) and were more complex.

---

## Sources

### Device Documentation
- [Zigbee2MQTT ERS-10TZBVK-AA Device Page](https://www.zigbee2mqtt.io/devices/ERS-10TZBVK-AA.html)
- [Z2M MQTT Topics and Messages](https://www.zigbee2mqtt.io/guide/usage/mqtt_topics_and_messages.html)
- [Z2M HA Integration Guide](https://www.zigbee2mqtt.io/guide/usage/integrations/home_assistant.html)

### Home Assistant
- [Home Assistant Blueprint Schema](https://www.home-assistant.io/docs/blueprint/schema/)
- [Home Assistant MQTT Trigger Docs](https://www.home-assistant.io/docs/automation/trigger/#mqtt-trigger)
- [Home Assistant Media Player Services](https://www.home-assistant.io/integrations/media_player/)
- [Arcam FMJ Integration](https://www.home-assistant.io/integrations/arcam_fmj)

### Community Blueprints & Discussions
- [pbergman Z2M Blueprint (GitHub)](https://github.com/pbergman/ha-blueprints/blob/3fc7a18cd8ea34a82f5a955beea13c02a5f805d7/ERS-10TZBVK-AA.yaml)
- [Improved Light Blueprint](https://community.home-assistant.io/t/zigbee2mqtt-control-light-entity-including-press-turn-with-tuya-moes-smart-knob-ers-10tzbvk-aa-v1-1/787779)
- [MOES Knob Blueprint Exchange Thread](https://community.home-assistant.io/t/zigbee2mqtt-tuya-moes-smart-knob-ers-10tzbvk-aa/419989)
- [Universal Smart Knob Blueprint](https://community.home-assistant.io/t/zha-deconz-zigbee2mqtt-tuya-ers-10tzbvk-aa-smart-knob-universal-blueprint-all-actions-double-click-events-control-lights-media-players-and-more-with-hooks/870899)
- [action_step_size Discussion](https://community.home-assistant.io/t/tuya-ers-10tzbvk-aa-and-setting-action-step-size/757172)
- [Improved Tuya Smart Knob Blueprint](https://community.home-assistant.io/t/tuya-smart-knob-improved-blueprint-for-zigbee2mqtt/799168)
- [Z2M 2.0 Action Events Discussion](https://community.home-assistant.io/t/using-the-new-action-events-in-zigbee2mqtt-2-0/821709)

### Guides
- [SmartHomeCircle MOES Knob Guide](https://smarthomecircle.com/moes-zigbee-smart-knob-with-homeassistant)
