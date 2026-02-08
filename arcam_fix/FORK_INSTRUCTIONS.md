# Fork-basierte Lösung für Arcam FMJ

## Schritt 1: Library forken

1. Gehe zu https://github.com/elupus/arcam_fmj
2. Klick **Fork** (oben rechts)
3. Erstelle den Fork in deinem Account

## Schritt 2: Fix im Fork implementieren

```bash
# Clone deinen Fork
git clone https://github.com/DEIN_USERNAME/arcam_fmj.git
cd arcam_fmj

# Branch erstellen
git checkout -b fix-mute-state-rc5

# Datei bearbeiten
nano src/arcam/fmj/state.py
```

**Finde die `set_mute` Methode und ändere zu:**

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

```bash
# Commit und Push
git add -A
git commit -m "Fix: Query mute state after RC5 command for AV receivers"
git push origin fix-mute-state-rc5
```

## Schritt 3: PR zum Original-Repo erstellen

1. Gehe zu https://github.com/elupus/arcam_fmj
2. Klick **Pull Requests** → **New Pull Request**
3. Wähle "compare across forks"
4. Base: `elupus/arcam_fmj` `master` ← Head: `DEIN_USERNAME/arcam_fmj` `fix-mute-state-rc5`
5. Erstelle den PR mit der Beschreibung aus `GITHUB_ISSUE.md`

## Schritt 4: Fork in HA nutzen (während PR pending)

### Option A: Package Override (einfach)

In HA Container/SSH:

```bash
# Installiere deinen Fork anstelle des offiziellen Packages
pip install git+https://github.com/DEIN_USERNAME/arcam_fmj.git@fix-mute-state-rc5

# HA neu starten
ha core restart
```

**Nachteil:** Wird bei HA-Updates überschrieben.

### Option B: requirements_override (dauerhaft)

Erstelle `/config/requirements_override.txt`:
```
arcam-fmj @ git+https://github.com/DEIN_USERNAME/arcam_fmj.git@fix-mute-state-rc5
```

Dann in `configuration.yaml`:
```yaml
# Nicht direkt unterstützt - erfordert manuelles pip install nach Updates
```

### Option C: Custom Component mit Fork-Dependency (beste Lösung)

1. Erstelle `config/custom_components/arcam_fmj_fixed/`

2. Kopiere alle Dateien von der Original-Integration

3. Ändere `manifest.json`:
```json
{
  "domain": "arcam_fmj_fixed",
  "name": "Arcam FMJ (Fixed)",
  "version": "1.0.0",
  "requirements": ["arcam-fmj @ git+https://github.com/DEIN_USERNAME/arcam_fmj.git@fix-mute-state-rc5"],
  ...
}
```

4. Entferne die Original-Integration und nutze die Fixed-Version

---

## Schnellster Weg zum Testen

Während du auf PR-Merge wartest:

```bash
# SSH/Terminal in HA
pip install git+https://github.com/DEIN_USERNAME/arcam_fmj.git@fix-mute-state-rc5
ha core restart
```

Dann Blueprint testen - Mute sollte jetzt funktionieren!
