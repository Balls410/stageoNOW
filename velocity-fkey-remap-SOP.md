# SOP: Function Key Remapping for Zebra Android Handhelds (Velocity + HighJump Warehouse Advantage)

## Purpose

Configure a Zebra Android handheld (physical keypad, no dedicated F-keys) running
Ivanti Velocity Client so that **Shift + number key** correctly sends F1–F10 to a
HighJump Warehouse Advantage host over VT220 telnet.

## Background

Physical F-keys are produced by the device firmware as **Shift + number**, where
Shift acts as a toggle/latch (press-release arms it, then the next key becomes the
F-key — it does not need to be held). The device's kernel keypad driver already
translates this into a standard Android `KEYCODE_F1`–`KEYCODE_F10` event before
Velocity ever sees it, so no raw scan-code guessing is needed on the device side.

The Key Macro script intercepts these using VT220-style extended scan codes
(`E03B`–`E044`). **These codes are correct as documented below — do not change
them.** If F-keys don't work, the cause is almost always a Host Profile setting,
not the script or the hex codes (see Troubleshooting).

---

## Part 1 — Host Profile Settings (Velocity Console)

Open the Host Profile and confirm these exact values. All four are required —
missing any one of them causes F-keys to fail or produces garbled/corrupted
screen text on the client.

| Section | Setting | Required Value |
|---|---|---|
| Mode | Default Native Mode | **On** |
| Mode | Allow User to Switch Modes | **Off** |
| Mode | Enable Fixed Screen Mode | **On** |
| Keyboard | Keyboard Visibility | **Hide** |

Other settings per the base config guide (`Configuring the Velocity for the
terminal with Android.pdf`):
- Emulation Type: VT220
- Port 23 (Telnet)
- Disable Predictive Formatting: Off
- Screen Rotation: Lock Portrait
- Disable Pinch and Zoom: On

> **Why this matters:** with Native Mode off, or the soft keyboard visible, or
> Fixed Screen Mode off, the device's own VT220 renderer can corrupt the screen
> (garbled repeating characters) during the login screen's redraw and the Key
> Macro script's `WLEvent.on("OnKey<...>")` listeners will not reliably intercept
> physical key events. This was root-caused via live device debugging on
> 2026-09-17 — see `velocity-fkey-remap-debug.md` for the full investigation.

### Native Theme (font size)

Click **Settings → Native Theme**. Set **Screen Properties → Size** to match the
rest of the fleet:

| Setting | Value |
|---|---|
| Native Theme Size (`NativeFontSize`) | **20** |
| Text Size (`TextSize`, under Host Profile) | **100%** |

This matches the `MC93ND.wldep` / `MC93ND_DPT.wldep` fleet-standard deployments
(checked 2026-09-17). The PDF's own step 47 says `24`; fleet devices are actually
running `20` — use `20` for consistency with what's already deployed elsewhere,
not the PDF's number.

---

## Part 2 — Key Macro Script

1. In the Host Profile, click **Script → Library → Key Macro**.
2. Click **Link**, leave Scope as `session`, click OK.
3. Replace the script body with the following (covers all 10 keys in one task —
   no need to create 10 separate linked instances):

```javascript
var keyMap = {
  "E03B": "F1{hex:000D}",
  "E03C": "F2{hex:000D}",
  "E03D": "F3{hex:000D}",
  "E03E": "F4{hex:000D}",
  "E03F": "F5{hex:000D}",
  "E040": "F6{hex:000D}",
  "E041": "F7{hex:000D}",
  "E042": "F8{hex:000D}",
  "E043": "F9{hex:000D}",
  "E044": "F10{hex:000D}"
};

for (var hex in keyMap) {
  (function(hexCode, newValue) {
    var sKeyEvent = "OnKey<" + hexCode + ">";
    WLEvent.on(sKeyEvent, function(event) {
      event.eventHandled = true;
      Device.sendKeys(newValue);
    });
  })(hex, keyMap[hex]);
}

// Optional: one extra custom key, set via Parameters (sHexCode / sNewValue)
if (typeof sHexCode !== "undefined" && sHexCode) {
  var extraHexCode = sHexCode.toUpperCase();
  var extraKeyEvent = "OnKey<" + extraHexCode + ">";
  WLEvent.on(extraKeyEvent, function(event) {
    event.eventHandled = true;
    Device.sendKeys(sNewValue);
  });
}
```

4. Leave the `sHexCode` / `sNewValue` parameters blank unless you need to map one
   additional key beyond F1–F10.
5. Click **Save**.

---

## Part 3 — Deploy

1. Click the **Deploy** (rocket) button in Velocity Console; save the `.wldep`
   file to your PC.

2. **CRITICAL — required every single time you export from Console:** open the
   `.wldep` (it's a zip file) and check `scripts\session.js`. Velocity Console
   always wraps the script in:
   ```javascript
   function KeyMacro(sNewValue, sHexCode) {
     ... script body ...
   }
   ```
   **This wrapper must be removed** — delete the `function KeyMacro(sNewValue,sHexCode) {`
   line and the matching final `}`, so the script body is top-level code again.
   If you skip this step, **all F-keys silently stop working** with no error
   anywhere — the screen just garbles on the next keypress, identical to a Host
   Profile misconfiguration. This is the single most common way this setup breaks
   after it's already been working. See `velocity-fkey-remap-debug.md` (Issue 2)
   for the full root-cause writeup.

   To do this quickly: extract the `.wldep`, edit `scripts\session.js` in a text
   editor to remove the wrapper, then re-zip it back up preserving the same
   folder structure (`hosts\`, `rules\`, `scripts\`, `themes\`).

   **Before (what Console exports — broken):**
   ```javascript
   function KeyMacro(sNewValue,sHexCode) {      ← DELETE this line
   var keyMap = {
     "E03B": "F1{hex:000D}",
     ...
   };

   for (var hex in keyMap) {
     ...
   }

   if (typeof sHexCode !== "undefined" && sHexCode) {
     ...
   }
   }                                              ← DELETE this closing brace (very last line)
   ```

   **After (working):**
   ```javascript
   var keyMap = {
     "E03B": "F1{hex:000D}",
     ...
   };

   for (var hex in keyMap) {
     ...
   }

   if (typeof sHexCode !== "undefined" && sHexCode) {
     ...
   }
   ```

   Only the very first line and the very last line get deleted. Everything in
   between — including indentation — stays exactly as Console exported it; the
   `{`/`}` pair being removed was just the wrapper, and the rest of the script is
   already balanced without it.

3. Copy the fixed `.wldep` file to **Internal Storage\com.wavelink.velocity** on
   the device, named exactly **`wearhousescanners2.wldep`** — Velocity does not
   reliably pick up a `.wldep` under a different filename, even if placed in the
   correct folder.
   - If that folder doesn't exist yet, launch Velocity and tap the host profile
     once — it will create the folder — then copy the file and relaunch.
   - If the folder still isn't visible, reboot the device.
   - **Make sure this is the only `.wldep` in that folder.** Leaving an old copy
     behind (e.g. from testing under a different filename) can cause duplicate
     host profiles to show up in the app. Delete any stray `.wldep` files first.
4. Relaunch Velocity (or reboot) so it picks up the new deployment.

---

## Part 4 — Verification

1. Connect to the host and reach a login/menu screen.
2. Press **Shift + [number]** for each F-key you use and confirm the host
   responds correctly (no garbled/repeating characters, correct action taken).
3. **Not every F-key will visibly do something on every screen** — HighJump
   Warehouse Advantage defines which function keys are active per screen (the
   login screen typically only wires up one or two, e.g. `F5:Version`). Confirm
   the remaining keys once you're past login and on a normal transaction/menu
   screen where more F-keys are expected to be active.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| It was working, then broke after re-editing/re-linking/renaming the profile in Console and redeploying | Console re-wrapped `session.js` in `function KeyMacro(sNewValue,sHexCode) { ... }` on export, which never gets called since it's unlinked from any invocation with those parameters | Strip the wrapper — see Part 3, step 2 |
| Screen fills with repeating garbled characters (e.g. `âââââ...`) after any keypress | Native Mode off, Fixed Screen Mode off, soft keyboard visible, **or** the script wrapper issue above (both produce identical symptoms) | Recheck all 4 settings in Part 1, **and** check for the function wrapper in Part 3 step 2 |
| A specific F-key does nothing | Host doesn't define an action for that key on the current screen | Normal — test on a screen where that key is expected to be used |
| No key works at all, no garbling | Script not deployed / wrong host profile targeted, or Key Macro not linked with scope `session` | Reconfirm deployment reached the device (see Part 3); reconfirm script is linked in Console |
| Need to diagnose a new/different device model | Scan codes may differ | See `velocity-fkey-remap-debug.md` for the adb/getevent-based diagnostic method used to confirm codes on the MC93 |

### How to confirm which one it is

If you have `adb` access to the device, the fastest unambiguous check is the
session log:
1. Temporarily set `EnableSessionLogging` to `true` in the Host Profile, redeploy.
2. Reproduce the keypress, then pull
   `/storage/emulated/0/Android/data/com.wavelink.velocity/files/velocity_session.log`.
3. Look for a `Session 1 - Wrote` entry matching the expected bytes (e.g. `46 32 0D`
   = `F2\r` for the F2 key). If nothing was written at all, the script isn't
   registering — check the function-wrapper issue first, since it's the most
   common cause. Set logging back to `false` before final production deploy.

---

*Last validated: 2026-09-17, on a Zebra MC93 (Android 10), Velocity Client, against
`kobercloud-p` / HighJump Warehouse Advantage 12.14. See `velocity-fkey-remap-debug.md`
for the full live-device debugging session this SOP is based on, including the
Console script-wrapping bug (Issue 2) that took the longest to isolate.*

*Known-good deployment artifacts (script already unwrapped, all settings correct)
are kept alongside this SOP: `wearhousefunctionkeyworking.wldep` (ready to copy
straight to a device) and `wearhousefunctionkey.zip` (source package to import
into Console — remember Part 3 step 2 still applies to anything re-exported from
Console after editing).*
