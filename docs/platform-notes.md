# Platform notes

For installation/build dependencies see [README](../README.md) and
[BUILD.md](../BUILD.md). This guide describes implementation choices, not a
compatibility guarantee for every desktop or target app.

## Text delivery

[clipboard.rs](../src-tauri/src/clipboard.rs) selects the paste method;
[input.rs](../src-tauri/src/input.rs) handles Enigo input primitives.

| Method            | Behavior                                                                                |
| ----------------- | --------------------------------------------------------------------------------------- |
| `ctrl_v`          | Clipboard text plus paste chord (Command-V on macOS).                                   |
| `ctrl_shift_v`    | Clipboard text plus modified paste chord, useful for compatible terminals.              |
| `shift_insert`    | Clipboard text plus Shift-Insert.                                                       |
| `direct`          | Type text through an available platform tool/Enigo rather than a clipboard paste chord. |
| `none`            | No automatic typing/paste; clipboard handling remains a separate preference.            |
| `external_script` | Execute configured script with transcript text as an argument.                          |

The default is direct typing on Linux, `ctrl_v` elsewhere. External scripts must
be trusted: they receive transcript text and run as your user. A missing script
path is an error; Handy does not create the script for you.

Clipboard handling can preserve prior content or leave the transcript copied.
Legacy clipboard paste uses pre/post delays (both default to 60 ms); slow target
apps may not read the clipboard before restoration. Auto-submit is separately
disabled by default; enabling it can send Enter or a modified Enter to the target
app. Test in a safe text field before using it in a terminal or chat application.

[paste_tx/](../src-tauri/src/paste_tx/mod.rs) offers optional, debug-gated
"reliable paste" on macOS/Windows. It publishes text lazily, waits for a read
after chord injection and a quiet period, and avoids restoring over a newer user
copy. It has bounded timeouts and falls back to legacy paste when unavailable.
It does not prove which application read the clipboard and is not a Linux path.

## Linux: tools, Wayland and overlays

The direct-typing selection logic in [clipboard.rs](../src-tauri/src/clipboard.rs)
checks the session and available executables:

- **KDE Wayland:** prefer `kwtype` (KDE Fake Input protocol).
- **Other Wayland:** prefer `wtype` (virtual keyboard protocol).
- **Wayland fallback:** `dotool`, then `ydotool`.
- **X11:** `xdotool`, then `ydotool`.
- If no automatic tool is selected, the direct path can fall back to Enigo;
  Wayland compatibility remains compositor-dependent.

The `typing_tool` setting also permits an explicit choice. Automatic selection
based on availability does not mean Handy retries every tool after a chosen
tool fails. `wtype` is not suitable for all compositors; KDE is explicitly
handled separately. `ydotool` needs its daemon and usable uinput permissions;
`dotool` likewise depends on system input permissions. Follow your distribution's
tool setup rather than granting broad device access blindly. Clipboard-chord
injection has its own tool routing; do not assume every direct-typing backend
supports every paste chord. The code probes `ydotool key --help` to accommodate
symbolic and raw-keycode versions.

Linux defaults to Tauri shortcuts. Where the compositor does not expose usable
global shortcuts, bind `handy --toggle-transcription`,
`handy --toggle-post-process`, or `handy --cancel` in your desktop/window manager.
See [CLI implementation](../src-tauri/src/cli.rs),
[single-instance dispatch](../src-tauri/src/lib.rs), and
[signals](../src-tauri/src/signal_handle.rs). On Linux, do not reuse old SIGUSR1
bindings: WebKit uses that signal internally; use the CLI for post-processing.

The overlay defaults to **style `none`**, with position `bottom`. Visibility is
not an `OverlayPosition::None` variant anymore. Overlay focus interactions can
affect the paste target. To disable it, choose overlay style None, not position.
[overlay.rs](../src-tauri/src/overlay.rs) includes GTK layer-shell support;
`HANDY_NO_GTK_LAYER_SHELL=1` disables that path, not the linked runtime dependency.
See root troubleshooting for distro packages and WebKit startup workarounds.

## macOS: permissions and Secure Input

[App.tsx](../src/App.tsx) and
[AccessibilityPermissions.tsx](../src/components/AccessibilityPermissions.tsx)
check microphone and accessibility permission. Grant requested access in system
privacy settings; a working settings window does not prove background recording
or input injection has permission. Default shortcuts use Option-Space and
Option-Shift-Space, and the default backend is handy-keys.

[secure_input.rs](../src-tauri/src/secure_input.rs) monitors Secure Input, which
can suppress keyed shortcuts. It emits status/diagnostic information and enables
Carbon fallbacks for supported bindings when appropriate. Coverage may be
degraded or absent for some bindings; the fallback is not a way to bypass secure
text entry. The UI warning and tray status help identify sustained interference.
Close the offending secure-entry session or application when appropriate rather
than repeatedly changing recognition models.

Apple Intelligence post-processing is compiled for macOS ARM64 and checked for
availability when invoked. Its presence in settings does not mean the operating
system has an available model; see [post-processing](post-processing.md).

## Windows and portable data

Windows defaults to handy-keys shortcuts and clipboard Ctrl-V delivery. Engine
acceleration depends on the build, selected backend and installed drivers; see
[Cargo.toml](../src-tauri/Cargo.toml) and
[accelerator enumeration](../src-tauri/src/managers/transcription.rs). The
receipt-sequenced paste implementation uses delayed clipboard rendering and
`WM_RENDERFORMAT` ([windows.rs](../src-tauri/src/paste_tx/windows.rs)).

[portable.rs](../src-tauri/src/portable.rs) recognizes a `portable` marker next to
the executable whose content starts with `Handy Portable Mode`, redirecting
app-owned data to adjacent `Data/`. The main window's WebView data directory is
also redirected in [lib.rs](../src-tauri/src/lib.rs). Keep that directory writable
and protect it like a settings/history backup: it can contain credentials,
recordings and transcripts. HF shared cache storage is distinct from app-owned
models; see [models](models.md#storage-and-downloads).
