# Bug: Mute state not updated after RC5 command on AV receivers

## Description

When using `set_mute()` on devices that don't support direct mute write (AV40, AVR series, etc.), the internal `_state[CommandCodes.MUTE]` is never updated after the command completes.

## Affected Devices

All devices NOT in `MUTE_WRITE_SUPPORTED`:
- AV40, AV41
- AVR series (AVR5, AVR10, AVR11, AVR20, AVR21, AVR30, AVR31, AVR390, AVR450, AVR550/600, AVR750, AVR850, AVR860)
- FMJ AVR series

These use the RC5 IR simulation path instead of direct MUTE command.

## Root Cause

In `state.py`, the `set_mute()` method:

```python
async def set_mute(self, mute: bool) -> None:
    if self._api_model in MUTE_WRITE_SUPPORTED:
        # Direct write - response updates _state[MUTE] via _listen
        await self._client.request(self._zn, CommandCodes.MUTE, bytes([...]))
    else:
        # RC5 simulation - response updates _state[SIMULATE_RC5_IR_COMMAND]
        # BUT _state[MUTE] is NEVER updated!
        await self._client.request(
            self._zn, CommandCodes.SIMULATE_RC5_IR_COMMAND, command
        )
```

The `_listen` callback updates `_state[packet.cc]` with the response, but for RC5 commands, `packet.cc` is `SIMULATE_RC5_IR_COMMAND`, not `MUTE`.

Meanwhile, `get_mute()` reads from `_state[CommandCodes.MUTE]` which remains stale.

## Symptoms

- First mute/unmute works (initial state from `update()` is correct)
- Subsequent mute toggles appear to work (device responds) but `get_mute()` returns stale state
- Home Assistant shows wrong mute status after first toggle

## Proposed Fix

After sending an RC5 mute command, explicitly query the mute state:

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
        # Query mute state after RC5 command to update _state[MUTE]
        try:
            data = await self._client.request(
                self._zn, CommandCodes.MUTE, bytes([0xF0])
            )
            self._state[CommandCodes.MUTE] = data
        except Exception:
            pass  # State will update on next poll

```

## Environment

- Home Assistant 2026.2.1
- arcam_fmj library version: (latest)
- Device: Arcam AV40
