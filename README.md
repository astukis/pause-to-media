# pause-to-media ⏸️

Remap the **Pause/Break** key to **Media Play/Pause** on Windows using the registry `Scancode Map` value.

No background process. No keyboard hooks. No installers. No AutoHotkey, no SharpKeys. One registry value, set once, permanent.

| Press | Result |
|---|---|
| `Pause` | Toggles media playback |

The remap happens at the keyboard-driver level, so every application sees an ordinary media key. Nothing runs in the background afterwards.

---

## Why this needs a repo

Every popular remapping tool **cannot do this**, and SharpKeys doesn't even list Pause as a selectable key. The reason is a real hardware quirk:

Normal keys send a **two-byte** scancode. Pause/Break does not — it sends a **three-byte** sequence:

```
E1 1D 45   (make)
E1 9D C5   (break)
```

SharpKeys' data model is entirely two-byte scancodes, so Pause isn't representable. Its "Type Key" capture misreads the sequence and reports `Special: Num Lock (00_45)` - that's the *tail* of Pause's sequence, not Pause itself. Remapping from there silently does nothing.

The `Scancode Map` value *does* support an `E1` prefix, so this is possible - but only if you also account for that trailing `45` byte, which Windows otherwise interprets as a **NumLock** press. Without the second entry, every Pause press also toggles NumLock.

---

## Install

1. Download **`install-pause-to-media.reg`**
2. Double-click it
3. **Yes** at the UAC prompt
4. Reboot or sign out (some systems apply it immediately)

That's it.

---

## Uninstall

1. Download **`uninstall-pause-to-media.reg`**
2. Double-click it
3. **Yes** at the UAC prompt
4. Reboot or sign out

Pause returns to normal. The value is deleted entirely, not just reset.

---

## Verify

```
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Keyboard Layout" /v "Scancode Map"
```

Or just press Pause and see if your music stops.

---

## How it works

A `REG_BINARY` value at:

```
HKLM\SYSTEM\CurrentControlSet\Control\Keyboard Layout\Scancode Map
```

| Offset | Bytes | Meaning |
|---|---|---|
| `0x00` | `00 00 00 00` | Version — must be zero |
| `0x04` | `00 00 00 00` | Flags — must be zero |
| `0x08` | `03 00 00 00` | Entry count = **3** (2 mappings + null terminator) |
| `0x0C` | `22 E0 1D E1` | New `E0_22` (Media Play/Pause) ← Old `E1_1D` (Pause) |
| `0x10` | `00 00 45 00` | New `00_00` (nothing) ← Old `00_45` (stray NumLock byte) |
| `0x14` | `00 00 00 00` | Null terminator |

Each entry is 4 bytes: **new scancode (2, little-endian), then old scancode (2, little-endian)**. The count field includes the terminator.

---

## Caveats

**NumLock.** The second entry swallows the stray byte from Pause's sequence. On keyboards where the physical NumLock key emits `00_45`, this disables NumLock as a side effect - it does *not* affect every keyboard. Test after installing.

If NumLock stops working and you need it, remove the `00 00 45 00` entry and change the count from `03` to `02`:

```
"Scancode Map"=hex:00,00,00,00,00,00,00,00,02,00,00,00,22,e0,1d,e1,00,00,00,00
```

The trade-off: every Pause press will then also toggle NumLock.

**Existing remaps.** This writes the *entire* `Scancode Map` value, so it overwrites any mappings you already have. Back up first:

```
reg export "HKLM\SYSTEM\CurrentControlSet\Control\Keyboard Layout" scancode-backup.reg
```

Then merge your entries into the `.reg` and bump the count accordingly.

**All keyboards.** It's driver-level, so it applies to every keyboard you plug in - not just the built-in one.

**Permanent.** Survives reboots and Windows updates. Lost only if the value is removed or Windows is reinstalled.

---

## Alternatives considered

| Method | Works for Pause? | Runtime cost |
|---|---|---|
| **Scancode Map** (this repo) | ✅ | none |
| SharpKeys | ❌ key not in its list | none |
| AutoHotkey (`Pause::Send "{Media_Play_Pause}"`) | ✅ | one running process |
| PowerShell `WH_KEYBOARD_LL` hook | ✅ | one running process |

If you're fine with a small background process, AutoHotkey is one line and much less fragile than hand-written registry bytes. This repo exists for people who want nothing running at all.

---
