# Mythic Engine with `EnableMouseInPointer` support (Civilization VI mouse fix)

A fork of [MythicApp/wine](https://github.com/MythicApp/wine) (the Wine source behind
[Mythic](https://getmythic.app), CrossOver-derived, for running Windows games on macOS)
with a small patch that makes mouse input work in games that use the Windows **Pointer
Input API**.

## The symptom

You launch the game through Mythic. It starts, the cursor moves, the keyboard works,
and clicking anywhere triggers "click-anywhere" actions (skipping an intro video, for
example) — but **buttons never highlight on hover and clicks never register on them.**
In Civilization VI this means you are stuck on the intro screen: the `Continue` button
does nothing, no matter where on the screen you click.

## The cause

Civilization VI calls [`EnableMouseInPointer`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-enablemouseinpointer)
and from then on listens for `WM_POINTER*` messages instead of the classic
`WM_MOUSEMOVE` / `WM_LBUTTONDOWN` ones.

Wine does not implement this. In Mythic's engine, `NtUserEnableMouseInPointer`
(`dlls/win32u/input.c`) is a stub that fails with `ERROR_CALL_NOT_IMPLEMENTED`, and
`WM_POINTERUPDATE` is never sent anywhere. So the game's UI receives no pointer
position and no clicks — while anything still wired to ordinary mouse messages keeps
working, which is why the intro video can be skipped but the button cannot be pressed.

This is **not** a resolution, window focus, Retina, or macOS Game Mode problem. Those
are the usual suspects and none of them are responsible.

## The fix

Valve solved this in Proton. This branch applies the same idea to Mythic's tree, in two
files:

- `dlls/win32u/input.c` — `NtUserEnableMouseInPointer` records the flag and returns
  `TRUE`; `NtUserIsMouseInPointerEnabled` reports it back.
- `dlls/win32u/message.c` — in `process_mouse_message`, ordinary mouse messages also
  emit the matching `WM_POINTERUPDATE` / `WM_POINTERWHEEL` / `WM_POINTERHWHEEL`.

No header changes are needed: the required constants already exist in
`include/winuser.rh`.

### Scope, honestly

This is a **rudimentary implementation** — a semi-stub. It synthesises pointer messages
from mouse messages. It does *not* implement the rest of the pointer API
(`GetPointerInfo`, touch/pen input, separate `WM_POINTERDOWN`/`WM_POINTERUP`, device
enumeration). Applications that genuinely need those will still not work.

- **Verified:** Sid Meier's Civilization VI (Epic build) — clicking works.
- **Expected but untested:** other titles hitting the same wall. The bug is widely
  reported for **Unity games**, which call the same API. Civilization VI is not Unity,
  so at least two different engines are affected.

## Installing a build

The GitHub Actions workflow produces an `Engine` artifact — a complete Mythic Engine,
including GPTK/D3DMetal.

```sh
# quit Mythic and any running game first
ENGINE=~/Library/"Application Support"/Mythic/Engine

cp -a "$ENGINE" "$ENGINE.backup"          # keep a way back
rm -rf "$ENGINE" && mkdir -p "$ENGINE"
tar -xf Engine.tar.xz -C "$ENGINE"

# the build does not produce these two; carry them over
cp -a "$ENGINE.backup/verbs.txt" "$ENGINE.backup/winetricks" "$ENGINE"
```

Two things worth knowing before you do this:

- This branch builds **Mythic Engine 3.0.0 (wine-9.0)**, which is ahead of the released
  2.6.1 (wine-7.7). It is not a Mythic release build.
- wine-9.0 will **upgrade your existing wine prefix, and that is one-way.** Back up
  `~/Library/Containers/xyz.blackxfiied.Mythic/Containers/Default` first if you might
  want to go back.

## Credits

- The original fix is Valve's, from Proton.
- Adapted to Wine 7.7 / Whisky by [IdyllicHappiness](https://github.com/IdyllicHappiness)
  in [Whisky issue #1169](https://github.com/Whisky-App/Whisky/issues/1169); that fork's
  build artifacts have long expired, which is why this exists.
- Upstream engine and build recipe: [MythicApp/wine](https://github.com/MythicApp/wine).

Wine is licensed under the LGPL; see [`LICENSE`](LICENSE) and [`COPYING.LIB`](COPYING.LIB).
