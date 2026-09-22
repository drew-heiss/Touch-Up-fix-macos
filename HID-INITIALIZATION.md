# CoolTouch touchscreen initialization

The Planar SLP75-T controller identifies as KasaTech / CoolTouch System,
VID `0x2575`, PID `0x6738`. It exposes separate mouse and digitizer HID interfaces.

During testing, Touch Up received more than 1,000 mouse reports (`0x03`) and no
digitizer reports, even with Input Monitoring access working. Reading its Windows
certification feature (`0xff00/0xc5`, report `0x44`) succeeded but did not fix it.

The decisive change was explicitly **setting Device Mode to 2 (multiple input)**
through its standard Device Configuration feature. Reading that feature already
returned 2; skipping the SET because of that value would leave this controller
emitting mouse reports. The SET succeeded, digitizer reports (`0x01`, 49 bytes)
arrived, and the user confirmed touch circles in Test.

## Retained code

`TouchUpCore/HIDInterpreter.c` discovers the standard Device Mode and Device
Identifier elements from the descriptor, verifies their collection and two-field
report layout, then sends mode 2 while preserving Device Identifier. IOKit composes
the report; no controller name, VID/PID, or report ID is hard-coded.

Initialization runs after input callbacks are installed, off the UI thread. The
device reference stays retained until the request finishes. Failed device opens
and null input queues are handled without proceeding into unusable input state.

The certification request and temporary diagnostic observers/logging were removed.
Parsing, display mapping, gestures, and scrolling were not changed.

The cleaned Release build passed compilation and signature verification. After
disconnecting and reconnecting USB, the user confirmed that Test still displays
touch circles with this mode-only initialization.

The mode meanings are documented by Microsoft in
[Using Report Descriptors to Support Capability Discovery](https://learn.microsoft.com/en-us/windows-hardware/design/component-guidelines/using-report-descriptors-to-support-capability-discovery).
This controller required more than the certification read discussed in
[upstream issue #34](https://github.com/shueber/Touch-Up/issues/34).

## Build and transfer

The universal Release build includes both `arm64` and `x86_64`, targets macOS 12
or later, and is signed with the user's Apple Development identity. The transferable
archive is `build/Touch Up-fixed-universal.zip` (ignored by Git).

Copy the ZIP to the other Mac, extract it, move **Touch Up.app** to **Applications**,
and open it normally. Xcode is not needed. Grant **Input Monitoring** and
**Accessibility**, then quit and reopen the app. It runs in the menu bar, with no
Dock icon. Quit any older Touch Up instance before launching this one.

This local build is not notarized. If macOS blocks it, use the app-specific
**Open Anyway** option in System Settings → Privacy & Security, as described in
[Apple's instructions](https://support.apple.com/102445).

Keep the signing identity consistent when rebuilding. During debugging, switching
between an ad hoc build and an Apple Development build with the same bundle ID
caused TCC signature mismatches. That was separate from the touchscreen's mode
initialization issue.
