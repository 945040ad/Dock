# Core Dock: Design Spec

Date: 2026-11-08
Status: Awaiting review
Scope: Piece 1 of 3 of a Windows 11 taskbar replacement

## 1. Goal

A macOS-style dock for Windows 11, written in Rust, used daily on the author's own PC. It shows pinned and running apps with cursor-driven magnification, and can replace the Windows taskbar.

### Success criteria

- The author can work a full day using the dock instead of the taskbar for launching and switching apps.
- Idle CPU averages under 0.1% over 60 seconds when the dock is not hovered.
- Magnification keeps up with the display refresh rate, with no visible stutter.
- The taskbar returns after any dock failure, including a force-kill.
- Icons are each program's real Windows icon. No Apple artwork is used.

### Project breakdown

| Piece | Content | Status |
|---|---|---|
| 1. Core dock | This spec | In design |
| 2. System tray | Other apps' tray icons, with working clicks and menus | Later, own spec |
| 3. Status area | Clock, calendar, volume, network, battery, buttons for Quick Settings and Notification Center | Later, own spec |

Windows' own Start menu, search and notification pop-ups remain in use. `dock_model` must allow extra non-app regions to be added after the app items later, so pieces 2 and 3 can sit on the dock. Nothing for those regions is built now.

### Out of scope for this piece

Auto-hide, multiple monitors, themes, a settings window, an installer, auto-update and code signing. Starting with Windows is done by the author placing a shortcut to `dock-guard.exe` in the Startup folder.

## 2. Architecture

A Cargo workspace with three crates:

| Crate | Type | Purpose |
|---|---|---|
| `dock-model` | Library | Pure logic. No dependency on the `windows` crate |
| `dock` | Binary | The dock application |
| `dock-guard` | Binary | Crash supervisor and the entry point the user runs |

### Modules

| Module | Location | Responsibility | Built on |
|---|---|---|---|
| `dock_model` | `dock-model` | Item list, ordering, window grouping, pin logic, magnification and layout maths, animation state | None |
| `render` | `dock` | Borderless window and a composition visual tree: one frosted shelf visual and one visual per icon | `windows` crate, `Windows.UI.Composition`, `DesktopWindowTarget`, host backdrop brush, `WS_EX_NOREDIRECTIONBITMAP` |
| `windows_tracker` | `dock` | Tracks taskbar-eligible windows and groups them by app | `SetWinEventHook`, `RegisterShellHookWindow`, AppUserModelID |
| `icons` | `dock` | Loads and caches icons on a background thread | `IShellItemImageFactory` |
| `actions` | `dock` | Launch, focus, minimize, cycle, close, new instance | Shell and Win32 APIs |
| `taskbar_guard` | `dock` | Hides the taskbar when enabled, re-hides it, reserves screen space | AppBar API (`SHAppBarMessage`), `Shell_TrayWnd`, `TaskbarCreated` message |
| `config` | `dock` | Loads and saves settings and pins | TOML at `%APPDATA%\Dock\config.toml` |

### Threading and data flow

The UI thread runs the Windows message loop and owns the model and all visuals. The dock does not poll. It sleeps until an event arrives.

Flow, in one direction only:

1. Windows events, mouse input or a background-completion message arrives.
2. `dock_model` is updated.
3. The new layout is computed.
4. Layout is applied to visuals as position and size changes. Icons are not redrawn.

Background threads load icons and save config. They post a message to the UI thread and never touch the model or visuals directly.

An animation tick runs at display rate only while something is moving, and stops when all animation has settled.

### Window sizing and click-through

At rest, the window is exactly the size of the shelf. When the cursor enters, the window grows upward to fit fully magnified icons.

The hover region is the current shelf rectangle plus the current bounds of each icon. When the cursor leaves that region the dock collapses, and the window shrinks back when collapse finishes, within 250 ms. Clicks landing on the enlarged window outside the hover region during that time are ignored.

### Window eligibility and grouping

A window appears on the dock when it is visible, not cloaked (`DWMWA_CLOAKED`), not a tool window (`WS_EX_TOOLWINDOW`), and either unowned or marked `WS_EX_APPWINDOW`.

Windows are grouped by AppUserModelID, read from the window's property store or the owning package. If there is none, they are grouped by executable path. Store apps hosted in `ApplicationFrameHost` are resolved to the hosted app's AppUserModelID.

### Crash supervision

The user runs `dock-guard.exe`. It starts `dock.exe` and waits on its process handle. On any dock exit other than a clean quit:

1. It restores the taskbar and confirms reserved screen space is released.
2. It restarts the dock, unless the dock has crashed 3 times within 60 seconds, in which case it leaves the taskbar restored and stops.

The dock also waits on the guard's process handle. If the guard exits unexpectedly, the dock restores the taskbar and stays running with taskbar hiding turned off for that session.

Restoring the taskbar is safe to repeat, and is applied whether or not hiding was active.

## 3. Behaviour

### Layout

Left to right: Start button, pinned apps, a thin separator, running apps that are not pinned. The separator is shown only when unpinned running apps exist. The dock is bottom-centre on the primary monitor.

The Start button is a generic grid glyph that opens the Windows Start menu by sending the Windows key.

### Magnification

- Icon base size is 48 px, and maximum is 96 px. Both are scaled for display DPI.
- For an icon at distance `d` from the cursor, measured in resting icon widths, with influence radius `R = 3`:
  - If `d < R`, `scale = 1 + (max/base - 1) * cos²(π·d / (2R))`.
  - Otherwise, `scale = 1`.
- The shelf widens with the icons and stays centred.
- Entering and leaving are smoothed with a critically damped spring.

### Icon actions

| Input | App state | Result |
|---|---|---|
| Left-click | Not running | Launch, with a short bounce |
| Left-click | One window, not focused | Focus it, restoring it if minimized |
| Left-click | One window, focused | Minimize it |
| Left-click | Several windows | Focus the most recent. Further clicks cycle through the app's windows |
| Middle-click | Any | Open a new instance |
| Right-click | Any | Native menu: open windows, New window, Keep in Dock or Remove from Dock, Close all |
| Drag | Pinned or running | Reorder. A dragged unpinned running app that is dropped among the pins becomes pinned |
| Drag off the dock | Pinned | Unpin |

### Indicators

- Running apps show a small dot under the icon.
- An app requesting attention (shell hook flash notification) bounces until it is focused.
- Hovering shows a label with the app name above the icon.

### System integration

- The dock hides when the foreground app is fullscreen (`ABN_FULLSCREENAPP`) and returns afterwards.
- On `TaskbarCreated`, the dock re-registers its AppBar and re-hides the taskbar if hiding is enabled.
- The process is per-monitor DPI aware (v2) and repositions on DPI, resolution and monitor changes.
- The dock reserves screen space through the AppBar API, so maximized windows stop above it.
- With `hide_taskbar = false`, the dock sits directly above the taskbar.

### Configuration

`hide_taskbar` defaults to `false`, because the clock and tray are not available until pieces 2 and 3 exist.

```toml
hide_taskbar = false
icon_size = 48
max_icon_size = 96

[[pinned]]
app_id = "Microsoft.WindowsTerminal_8wekyb3d8bbwe!App"

[[pinned]]
path = "C:\\Program Files\\Example\\example.exe"
```

Each pin has exactly one of `app_id` (AppUserModelID) or `path`. A shortcut (`.lnk`) path is allowed.

## 4. Error handling

| Failure | Response |
|---|---|
| Dock crashes or is killed | Guard restores the taskbar, then restarts the dock up to the limit in section 2 |
| Guard exits unexpectedly | Dock restores the taskbar and runs without hiding for that session |
| GPU device lost, including after sleep/resume or a driver update | Rebuild visuals from the model |
| App has no usable icon | Generic placeholder icon |
| Pinned app no longer exists | Icon stays greyed out; right-click allows removal |
| Launch fails | Icon shakes briefly; reason is logged |
| Config missing | Start with defaults and write a new file |
| Config corrupt | Rename it to `config.toml.bak`, start with defaults, log it |
| Focus is refused, including for elevated apps | Flash the window with `FlashWindowEx`; log it |

Logs are written to `%APPDATA%\Dock\dock.log`, and kept to a bounded size.

## 5. Testing

### Pre-implementation spike (throwaway)

Run first, on the author's PC (Windows build 26300). Code from the spike is not kept.

| Question | Pass | If it fails |
|---|---|---|
| Can a Rust window combine host backdrop blur with per-pixel transparent visuals? | Blur shows behind a rounded shelf; surrounding area is fully transparent | Use a plain translucent shelf with no blur |
| Does hiding the taskbar hold, and does guard restoration work? | Stays hidden through opening Start, Win+Tab and an Explorer restart; returns after a force-kill; reserved space is released | Revisit the hiding method before building `taskbar_guard` |

### Automated

Unit tests on `dock-model` cover the magnification formula and its boundaries, layout, ordering, window grouping, pin and unpin rules, and drag outcomes. Unit tests also cover config parsing, round-trips and corrupt-file handling.

### Manual checklist (real app)

- Force-kill `dock.exe`, then the guard, with hiding on, and confirm the taskbar returns.
- Crash the dock repeatedly and confirm restarts stop at the limit.
- Restart Explorer.
- Run a fullscreen game and a fullscreen video.
- Sleep and resume.
- Change display scaling and resolution.
- Use a Store app, a Win32 app and an elevated app.
- Open several windows of one app and cycle them.
- Maximize a window and confirm it stops above the dock.

### Performance

Measure idle CPU over 60 seconds against the success criterion, and check magnification smoothness by eye on the real display.
