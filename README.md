# ATTENTION: Work continues within <https://github.com/RellHaiser/HD2_SabreSquadron>.

# IJoy.dll source port

Clean C++17 rewrite of `IJoy.dll` (Hidden & Dangerous 2, 2004, MSVC6 release build, 73,728 bytes):
the game's joystick and force-feedback layer over DirectInput 7. It builds a drop-in replacement
with the same export, vtables and behaviour, checked against the original by a differential test.

## What the DLL does

One export, `_CreateJoyBase@20` (ordinal 1), and three interfaces (see `include/ijoy.h`):

| Interface    | Class      | Size   | Role |
|--------------|------------|--------|------|
| `IJoyBase`   | `JoyBase`  | 0x2c   | DirectInput object, the attached joysticks with their objects and caps, open joysticks |
| `IJoy`       | `Joy`      | 0xf4c  | one device: polling, axis scaling/dead zone/saturation/modes, buttons with "pressed" edges, POV hats, FF device control |
| `IJoyEffect` | `JoyEffect`| 0x128  | one force-feedback effect: the 11 standard DirectInput effects or a set from an `.ffe` file |

The game (`HD2_SabreSquadron.exe`, call at 0x6dca31) calls `CreateJoyBase(&g_joyBase, hInstance,
hwnd, nullptr, 0)`, `GetNumJoys`, `CreateJoy(&joy, i, 0)` per joystick and `SetAxisMode(1, invert)`
when the setting asks for it; the rest it reaches through the stored interfaces.

Everything is reference counted; global counters (`g_leaks`, 0x1000fe60) can report unreleased
objects in a message box when the DLL unloads (`IJOY_REPORT_LEAKS`).

## Layout

```
include/ijoy.h          public interfaces, structs and constants (for the game port)
src/ijoy_internal.h     implementation classes, original member offsets in comments
src/joy_base.cpp        CreateJoyBase, JoyBase, device enumeration, leak report
src/joy.cpp             Joy
src/joy_effect.cpp      JoyEffect
src/data_format.cpp     c_dfDIJoystick2 as the original linked it (generated)
src/layout_check.cpp    static_asserts: class sizes and member offsets match the original
ijoy.def                export with the original ordinal
lib/dinput.def          import library source for DINPUT.dll (DirectInputCreateEx)
tests/compare.cpp       differential test against the original DLL
_re/                    Ghidra export (decomp.json, user_decomp.txt, user_asm.txt), tools
```

Every function carries its original address in a `// 0x1000xxxx` comment.

## Build

```
python build.py            # build/IJoy.dll (+ .lib, .pdb), MSVC x86
python build.py compare    # build/compare.exe
```

MSVC only: the interfaces are MSVC vtables (one virtual-destructor slot, `__thiscall`, bool in AL,
float in ST0); g++ emits two destructor slots. The Windows SDK no longer ships `dinput.lib`, so the
build makes one from `lib/dinput.def` with MinGW `dlltool --kill-at` (found on PATH or in
`C:\nim-2.2.6\dist\mingw32\bin`). Intermediates go to `%TEMP%\ijoy_build` (Google Drive locks
fresh files in the project folder).

## Verification

`tests/compare.cpp` loads the original and the port in one process and drives both through the
same random call sequences against a scripted fake DirectInput 7 (fake `IDirectInput7A`, devices
and effects: no joystick needed, since `CreateJoyBase` accepts an existing `IDirectInput7A`). Every
call into DirectInput (with its full arguments: properties, data format, effect parameters) and
every result coming back (floats as bit patterns) is logged; the logs must be identical.

```
copy H:\HD2_Unmodified\IJoy.dll to a work folder; copy build\IJoy.dll there as IJoy_port.dll
build\compare.exe WORK\IJoy.dll WORK\IJoy_port.dll 3000 600 [stats]
```

Result (2026-10-30): 3000 seeds, 1966 of them with a JoyBase, 4.03 million log lines per side,
0 differences. All 11 effect types, file effects, every setter/commit, lost-device updates, raw/
relative/inverted/unipolar axes and failure injection on every DirectInput call are exercised
(`stats` prints the event histogram). `_re/tools/mutate.py ORIGINAL_DLL` rebuilds the port with
one deliberate deviation at a time: 17 of 18 are caught; the 18th removes a check that is dead
in the original as well.

The test sets the FPU to 24-bit precision after loading the DLLs, as the game runs (Direct3D
sets it; the original's CRT start-up resets it on load). In that mode the original's x87 float
code rounds like the port's SSE floats.

## Behaviour kept from the original

The port reproduces observable quirks rather than fixing them (each marked in the source):

- Axis range fallback: when a device refuses the -10000..10000 range, the constructor reads the
  Y axis' range for every axis except X, and centres a 0-based range on `-(max + half)`.
  `SetAxisMode` (raw → calibrated) stores the re-read range into axis X's scaling.
- Saturation of the last axis (slider 1) writes its positive clamp into slider 0.
- POV 0 counts a change only when moved and not centred; POVs 1-3 count any move or any
  uncentred update.
- On a lost device with `IJOY_UPDATE_KEEP_LOST`, a loop running to 8 over two 4-entry arrays sets
  `buttons[0]` to 0x01010101 and axis X's dead zone to 0.
- `IsButtonDown` / `IsButtonPressed`: the switch over button groups has no breaks, so later groups
  are tested with the same bit.
- x87 compare semantics: NaN axes (zero device range) saturate to -saturation; clamps turn NaN
  into the lower bound.
- `SetAxes` with an invalid axis resets the first two pending axes but not their count.
- A failed `Joy` constructor still releases the `JoyBase` (which it never AddRef'd); the open-joystick
  list keeps a dangling pointer to a joystick released to zero; a failed `JoyBase` constructor
  still counts the object down. The device enumeration stops (and leaks the entry) when a device
  cannot be created, so a JoyBase needs every attached joystick to open; with none attached
  `CreateJoyBase` fails.
- `GetStatus` of a file effect queries the (null) single-effect pointer once per file effect.
  `GetAxisInfo` zeroes `*info` before its null check.

## Deviations

- Object lookups by name (`GetAxisName` & co.) tolerate a device without objects (the original
  dereferences the empty list; DirectInput always reports objects for a joystick).
- `SetCondition` clamps in locals instead of two global scratch arrays.
- Allocation failures throw (`new`) instead of being checked; fields the original left
  uninitialised start at zero.
- The data format table is a source file instead of being linked from DirectX 7's `dinput.lib`
  (same 164 entries, including that library's slider offsets for velocity/acceleration/force).
- Modern static CRT (`/MT`) instead of the MSVC6 one.

## Ghidra

Program `IJoy.dll` in the shared project: auto-analysis had missed 48 functions reached only
through vtables or the CRT's atexit list (created), and had made a function of the data-format
table at 0x10005ee0 (deleted). `_re/tools/apply_names.py` names all 94 user functions after the
port (`Class::Method`); the CRT (0x10006920 onwards) is left as is. `_re/tools/gh.py` is the HTTP
client (127.0.0.1:8089, `program=IJoy.dll`), `_re/tools/export_all.py` refreshes `_re/*.json`.
