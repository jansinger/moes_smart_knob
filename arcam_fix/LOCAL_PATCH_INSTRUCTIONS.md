# Lokale Patch-Anleitung für Arcam FMJ Integration

## Option 1: HA Container / HA OS

### Schritt 1: Finde die Library

```bash
# SSH in HA oder Terminal Add-on öffnen
docker exec -it homeassistant bash

# Oder bei HA OS:
ha core exec

# Finde die arcam library:
find /usr/local/lib -name "state.py" | grep arcam
# Typischer Pfad: /usr/local/lib/python3.12/site-packages/arcam/fmj/state.py
```

### Schritt 2: Backup erstellen

```bash
cp /usr/local/lib/python3.12/site-packages/arcam/fmj/state.py \
   /usr/local/lib/python3.12/site-packages/arcam/fmj/state.py.backup
```

### Schritt 3: Datei bearbeiten

```bash
vi /usr/local/lib/python3.12/site-packages/arcam/fmj/state.py
```

Finde die `set_mute` Methode (ca. Zeile 100-120) und ändere:

**VORHER:**
```python
    async def set_mute(self, mute: bool) -> None:
        if self._api_model in MUTE_WRITE_SUPPORTED:
            bool_to_hex = 0x00 if mute else 0x01
            await self._client.request(
                self._zn, CommandCodes.MUTE, bytes([bool_to_hex])
            )
        else:
            command = self.get_rc5code(RC5CODE_MUTE, mute)
            await self._client.request(
                self._zn, CommandCodes.SIMULATE_RC5_IR_COMMAND, command
            )
```

**NACHHER:**
```python
    async def set_mute(self, mute: bool) -> None:
        if self._api_model in MUTE_WRITE_SUPPORTED:
            bool_to_hex = 0x00 if mute else 0x01
            await self._client.request(
                self._zn, CommandCodes.MUTE, bytes([bool_to_hex])
            )
        else:
            command = self.get_rc5code(RC5CODE_MUTE, mute)
            await self._client.request(
                self._zn, CommandCodes.SIMULATE_RC5_IR_COMMAND, command
            )
            # FIX: Query mute state after RC5 command to update _state[MUTE]
            try:
                data = await self._client.request(
                    self._zn, CommandCodes.MUTE, bytes([0xF0])
                )
                self._state[CommandCodes.MUTE] = data
            except Exception:
                pass  # State will update on next poll
```

### Schritt 4: HA Core neu starten

```bash
ha core restart
```

---

## Option 2: Custom Component Override

Alternativ kannst du eine Custom Component erstellen die die Original-Integration überschreibt.

### Schritt 1: Verzeichnis erstellen

In deinem HA config Ordner:
```
config/
└── custom_components/
    └── arcam_fmj/
        ├── __init__.py
        ├── manifest.json
        └── media_player.py
```

### Schritt 2: Dateien kopieren

Kopiere alle Dateien von:
https://github.com/home-assistant/core/tree/dev/homeassistant/components/arcam_fmj

nach `custom_components/arcam_fmj/`

### Schritt 3: Version in manifest.json erhöhen

Ändere die Version in `manifest.json` auf etwas höheres.

### Schritt 4: media_player.py patchen

Ändere die `async_mute_volume` Methode:

```python
async def async_mute_volume(self, mute: bool) -> None:
    """Send mute command."""
    await self._state.set_mute(mute)
    # Force refresh mute state after RC5 command
    await asyncio.sleep(0.1)  # Small delay for device to process
    await self._state.update()  # Refresh all state
    self.async_write_ha_state()
```

(Füge `import asyncio` oben hinzu falls nicht vorhanden)

---

## Option 3: Workaround im Blueprint

Falls du die Library nicht patchen willst, können wir den Blueprint anpassen:

1. Nach dem Mute-Befehl kurz warten (100ms)
2. Dann `homeassistant.update_entity` aufrufen um den State zu refreshen

Dies ist weniger elegant aber erfordert keine Code-Änderungen.

---

## Nach dem Patch

1. HA neu starten
2. Mute testen - Status sollte jetzt korrekt aktualisiert werden
3. Blueprint neu laden und testen

**Achtung:** Der Patch wird bei HA-Updates überschrieben (Option 1) oder muss manuell synchron gehalten werden (Option 2).
