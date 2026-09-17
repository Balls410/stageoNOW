# Velocity Console — Function Key Remap Debugging

## Goal
Remap physical scan codes on a Zebra Android handheld (running Ivanti Velocity Client,
connected to a "HighJump Software Warehouse Advantage" VT220 telnet host) so that
pressing **Shift + number key** sends the correct F-key sequence (e.g. Shift+2 → F2)
to the host session.

The device's physical keypad does not have dedicated F1–F10 keys — F-key input is
produced by holding the blue **Shift/Fn** button and pressing a number key.

## Environment
- **Console:** Ivanti Velocity Console (Windows desktop app)
- **Client:** Ivanti Velocity Client for Android
- **Device:** Zebra rugged handheld (e.g. MC93xx-class device, physical numeric keypad + Shift/Fn key)
- **Project:** "Test (TE)" project, Host Profile name: `kobercloud-p`
- **Host:** Telnet, port 23, Host address `172.17.32.41`, Emulation Type VT220
- **Host application:** HighJump Software Warehouse Advantage, Version 12.14
- **Task type:** Advanced Configuration → Task (formerly called "Scripts"), type = **Key Macro**
- **Scope:** Task's Default scope is `session`

## Current script (single task, handles F1–F10 in a loop)

```javascript
// Built-in mappings for F1–F10
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

The `E03B`–`E044` hex codes are the *standard/assumed* Zebra scan codes for F1–F10
(F10 = E044 is documented by Ivanti as the example). These have **not been confirmed**
as correct for this specific device/keypad combination — the device uses a Shift+number
combo, not dedicated F-keys, so the actual scan codes it emits are unverified.

## Original single-key template (before consolidating into the loop above)

This is the underlying Key Macro script template that Velocity Console's Key Macro
task type ships with. Originally we created one task per F-key, each with two
Script Parameters (`sHexCode`, `sNewValue`) filled in via the UI:

```javascript
var hexCode = sHexCode.toUpperCase();
var sKeyEvent = "OnKey<" + hexCode + ">";
WLEvent.on( sKeyEvent, function(event) { event.eventHandled = true; Device.sendKeys( sNewValue ); } );
```

We abandoned the "10 separate tasks with parameters" approach in favor of the single
looped script above, to avoid creating 10 tasks × 2 parameters × required Description
fields manually through the UI.

## What we've tried / observed

1. **Confirmed** the task's Default scope is `session` (visible in the task Details tab).
2. **Confirmed** Default Native Mode is ON in the Host Profile's Mode settings (per
   original setup steps) — VT220 emulation may be consuming/handling function keys
   itself before our script's listener sees them. We have NOT yet tested with Native
   Mode toggled off via the device's on-device menu option "Turn Native Mode Off."
3. Looked for a device on-screen "Enable Key Test mode" feature (mentioned in the Key
   Macro task's own built-in Description text: *"From the application menu choose
   Enable Key Test mode. Press a key sequence to determine its scan code. For
   Example, F10 will return E044."*) — **this option does NOT appear** in the actual
   on-device Velocity menu (confirmed via screenshot). The menu only shows: Add
   Session, session list, Turn Native Mode Off, Session Details, About, Exit.
4. Tested pressing **Shift+2** (intended to trigger F2 / hex E03C) while connected to
   the HighJump login screen (`USER ID` prompt, with `F5:Version` shown at bottom of
   screen as a host-side hint that F5 is expected to trigger something).
   - **Result both times** (once sending `F2{hex:000D}`, once sending a diagnostic
     `HELLO{hex:000D}` in place of F2): the screen filled with garbled, repeating
     characters (looked like "ааааааааааааааааа4" type repeating glyphs) spanning
     multiple lines (overwriting what had been "Warehouse" and "Version 12.14" lines),
     rather than a single clean value being sent once.
   - This happened **identically for both `F2{hex:000D}` and `HELLO{hex:000D}`**,
     which argues against a "sending F2 re-triggers the F2 key event" recursion theory
     (since "HELLO" text shouldn't re-trigger an OnKey<E03C> listener) and suggests
     either:
     (a) the key event is firing many times per single press (repeat/bounce), each
         firing appends text + Enter, cycling through fields, producing the garbled
         multi-line output, OR
     (b) the listener never actually caught the intended key at all, and what's on
         screen is raw/garbled input from the physical key itself leaking straight
         through to the terminal (unrelated to our script).

## Important documentation finding — likely root cause

Found in Ivanti's VScript docs for `WLEvent.onKey()`:

> "This only handles key values that are mapped. If the Velocity Client key test only
> shows a code and does not show a value for the key, you must use
> `Keyboard.mapKeypress()` to map the key first."

We are using `WLEvent.on("OnKey<HEXCODE>", ...)` (string event name form) rather than
`WLEvent.onKey(keyCode, funcRef, scope)` (the documented dedicated function). It's
unconfirmed whether these two forms behave identically, or whether the string-based
`WLEvent.on("OnKey<...>")` approach requires the key to already be "mapped" via
`Keyboard.mapKeypress()` before it will fire — which would mean our current script
**never actually intercepts the key at all**, and the garbled output we're seeing is
raw, unmapped physical-key input passing straight through to the terminal, untouched
by our script.

## Next steps / things to check

1. **Try `Keyboard.mapKeypress()` first**, then attach the `WLEvent` listener, e.g.:
   ```javascript
   Keyboard.mapKeypress(0xE03C, /* mapped value TBD */);
   ```
   Need to check exact signature/behavior of `Keyboard.mapKeypress()` in the Velocity
   Scripting API docs (VScript 1.2 / Velocity 2.1 admin docs on help.ivanti.com).

2. **Try the dedicated `WLEvent.onKey()` function** instead of the string-based
   `WLEvent.on("OnKey<...>")` form, e.g.:
   ```javascript
   WLEvent.onKey(0xE03C, function(event) {
     event.eventHandled = true;
     Device.sendKeys("F2{hex:000D}");
   }, "session");
   ```
   (Note: docs show `keyCode` as Integer, not hex string — may need numeric form
   instead of the `"E03C"` string we've been using.)

3. **Toggle "Turn Native Mode Off"** on the device (visible in the on-device Velocity
   menu) and retest Shift+2, to rule out VT220 Native Mode intercepting the key before
   our script does.

4. **Enable Logging** (Host Profile → Logging in Velocity Console) to capture the raw
   key event(s) generated by a single Shift+2 press, so we can see the *actual* scan
   code(s)/event(s) fired rather than assuming E03C is correct. Log file location on
   device not yet confirmed — likely under Internal Storage, possibly
   `Android/data/com.wavelink.velocity/files` on Android 10 devices (per Ivanti docs),
   or `Downloads/com.wavelink.velocity` on newer Android versions.

5. **Test with a single, quick tap** rather than holding the key down, to rule out
   key-repeat/bounce causing multiple rapid-fire events per press.

## Files/values for reference

Full F1–F10 scan code table currently assumed (unverified for this device):

| Key | Hex Code | Intended sendKeys value |
|-----|----------|--------------------------|
| F1  | E03B     | `F1{hex:000D}`  |
| F2  | E03C     | `F2{hex:000D}`  |
| F3  | E03D     | `F3{hex:000D}`  |
| F4  | E03E     | `F4{hex:000D}`  |
| F5  | E03F     | `F5{hex:000D}`  |
| F6  | E040     | `F6{hex:000D}`  |
| F7  | E041     | `F7{hex:000D}`  |
| F8  | E042     | `F8{hex:000D}`  |
| F9  | E043     | `F9{hex:000D}`  |
| F10 | E044     | `F10{hex:000D}` |

---

## RESOLVED — Live device debugging session (2026-09-17)

Root cause fully identified via live `adb`/`getevent`/`logcat` debugging on a Zebra
MC93 against `kobercloud-p` / HighJump Warehouse Advantage 12.14. Two separate,
compounding issues were found and fixed. **Both must be addressed — fixing only one
leaves F-keys broken.**

### Issue 1 — Host Profile settings drift

The `E03B`–`E044` hex table above is **correct and does not need to change**. The
original failures (garbled/repeating screen characters) were caused by four Host
Profile settings that didn't match this document's own PDF companion guide
(`Configuring the Velocity for the terminal with Android.pdf`):

| Setting | Broken value found | Required value |
|---|---|---|
| Default Native Mode | Off | **On** |
| Allow User to Switch Modes | On | **Off** |
| Enable Fixed Screen Mode | Off | **On** |
| Keyboard Visibility | Show | **Hide** |

With Native Mode off, VT220 physical function keys (Shift+number on this device's
keypad, which the kernel keylayout already translates to real Android
`KEYCODE_F1`–`KEYCODE_F10` events) fall through to the client's default handling,
which corrupts the screen during the login redraw. Fixing these 4 settings alone
resolved the corruption and let the Key Macro script intercept keys correctly
**the first time** — but see Issue 2 for why it then appeared to stop working after
being re-deployed through Velocity Console.

### Issue 2 — Velocity Console wraps exported Key Macro scripts in a function that never gets called

**This is the critical, easy-to-hit gotcha.** After confirming the fix worked, the
corrected profile was re-imported into Velocity Console, and (for convenience) the
profile was renamed from `kobercloud-FKEYTEST` to a production name. Re-exporting
from Console at that point broke the F-keys again — with symptoms identical to
Issue 1 (garbled screen), even though the exported `hostprofile.xml` and
`session.js` were byte-for-byte equivalent in every field that matters.

Root cause: Velocity Console's Key Macro script library item has:
```xml
<FunctionName>KeyMacro</FunctionName>
<Parameters>
  <ScriptParameter><VariableName>sNewValue</VariableName>...</ScriptParameter>
  <ScriptParameter><VariableName>sHexCode</VariableName>...</ScriptParameter>
</Parameters>
```
When Console **exports** a `.wldep`, it wraps the script's `ScriptText` in:
```javascript
function KeyMacro(sNewValue, sHexCode) {
  ... your script body ...
}
```
Our script registers all its `WLEvent.on(...)` listeners as **top-level code**,
expecting to run immediately when the script loads. Wrapped inside `KeyMacro(...)`,
none of that code executes unless something actually calls `KeyMacro()` — and since
the Link step's `sHexCode`/`sNewValue` parameters were left blank (not needed; the
script has its own internal F1–F10 table), Velocity's runtime does not appear to
invoke the function. Net effect: **zero listeners get registered, silently.** No
error, no log — the key event just falls through to the client's default handling,
producing the exact same garbled-screen symptom as Issue 1.

**Fix:** after exporting a `.wldep` from Velocity Console for this script, open the
archive and strip the wrapper from `scripts\session.js` — delete the first line
(`function KeyMacro(sNewValue,sHexCode) {`) and the matching final closing `}`, so
the script body runs as top-level code again. This was verified to fix it reliably,
repeatedly, on live hardware.

**Confirmed NOT the cause** (ruled out during this session, in case future
debugging retreads this ground): zip entry order inside the `.wldep`,
`EnableSessionLogging`, `DisablePredictive`, app data / cache state (`pm clear`),
device reboot, and the exact host profile `Name` string all made no difference —
only the function-wrapper wrapping/unwrapping of `session.js` correlated with
working vs. broken behavior, tested many times over.

### Practical implication

Every time this Key Macro script is edited or re-linked inside Velocity Console and
re-deployed, **the exported `session.js` will be re-wrapped in `function
KeyMacro(sNewValue,sHexCode) { ... }` again**, and will need the wrapper stripped
again before deploying. See `velocity-fkey-remap-SOP.md` for the exact procedure.
The known-good, already-unwrapped deployment artifacts are kept at
`wearhousefunctionkeyworking.wldep` (ready-to-deploy `.wldep`) and
`wearhousefunctionkey.zip` (Console-importable source package) in this folder.

### Font size (Native Theme) — matched to fleet standard

While a second MC93 device (serial `...278`) was connected for comparison, its
deployed `MC93ND.wldep` / `MC93ND_DPT.wldep` (identical files, byte-for-byte) were
checked for their Native Theme settings:

- `NativeFontSize`: **20**
- `TextSize`: **100%**

The `kobercloud-p` profile was originally deployed with `NativeFontSize: 18`. This
was updated to `20` to match the rest of the fleet (`themes\nativetheme.xml` in
both `wearhousefunctionkeyworking.wldep` and `wearhousefunctionkey.zip`). Note this
differs from the companion PDF's step 47, which says to use `24` — the actual
fleet-standard value observed on deployed devices is `20`, not `24`.

### Stray duplicate `.wldep` caused duplicate host profiles

At one point the device's `files` folder ended up with two files:
`wearhousescanners2.wldep` (the correct, currently-imported one) and a stale
`wearhousefunctionkeyworking.wldep` left over from testing before the exact
expected filename was confirmed (see Issue 2 investigation above — Velocity
appears to require the filename `wearhousescanners2.wldep` specifically to import
correctly). Having both present caused two host profiles to show up in the app.
Fix: delete any `.wldep` file in that folder that isn't the current
`wearhousescanners2.wldep` before relaunching Velocity.
