# Windows Port Status

Checked: 2026-06-30, after rebasing onto upstream `main` commit `444ccf9`.

Local sync note:

- Local `main` now contains upstream through `444ccf9` plus the local
  `AGENTS.md` build-note commit (`7656b7b` after rebase).

## Summary

Neru is modularized for Windows. The port now has basic support in-tree, not
just scaffolding.

Architecture evidence:

- Core behavior lives behind ports in
  [internal/core/ports/](../internal/core/ports/).
- Platform implementations live behind adapters in
  [internal/core/infra/](../internal/core/infra/).
- OS selection happens through build-tagged factories such as
  [factory_windows.go](../internal/core/infra/platform/factory_windows.go).
- Windows has first-class file slots for system, hotkeys, event tap,
  accessibility, overlays, config, systray, IPC, and app startup.
- Shared app startup wires `SystemPort`, accessibility, overlays, hotkeys,
  event tap, and IPC uniformly in
  [app_initialization_steps.go](../internal/app/app_initialization_steps.go).

## Implemented

These pieces are now in the merged code.

Build and packaging:

- `just build` builds a native Windows `bin/neru.exe` with `CGO_ENABLED=0`.
- `just build-windows` and `just release-ci-windows` build Windows artifacts.
- CI runs Windows lint, vet, test, and build jobs.
- Release publishing includes `neru-windows-amd64.zip` and
  `neru-windows-arm64.zip`.
- `build-windows` and `release-ci-windows` embed app icon/resources via
  `generate-winres`; plain `just build` does not run that resource step.

Runtime foundation:

- Windows AppData paths are implemented.
- Focused app PID/name/path lookup uses Win32 foreground-window APIs.
- Active screen bounds, screen names, focused window bounds, cursor position,
  and cursor movement are implemented.
- Dark-mode registry detection exists, and the app polls for theme changes.
- Native MessageBox alerts exist for config validation, onboarding, and generic
  alert paths.

Input and modes:

- Global hotkeys use `RegisterHotKey`.
- Active-mode keyboard capture uses `WH_KEYBOARD_LL`.
- Non-modifier mode keys are consumed so they do not type into focused apps.
- Ctrl/Alt/Win modified shortcuts are allowed through while still notifying
  Neru mode dispatch.
- Sticky modifier key-down/key-up events are emitted.
- Left, right, and middle click, left mouse down/up, cursor movement, and
  vertical scroll use `SendInput`/`SetCursorPos`.

Overlays:

- Native overlay rendering uses layered Win32 windows and GDI-style software
  drawing.
- Grid, recursive grid, hints, hint search input, mode indicator, sticky
  modifier indicator, and mouse action indicator render on Windows.
- Hints render atomically for correct z-index behavior.
- Theme-aware overlay styling and Windows font aliases exist.

Accessibility:

- Initial UI Automation integration exists.
- The current foreground HWND seeds UIA traversal.
- Clickable descendants are collected with bounds, UIA control type/role,
  accessible name, and offscreen state. Value text, enabled state, focus state,
  and process ID still need explicit extraction.
- UIA control types are mapped to Neru clickable roles.
- Hint selection performs coordinate-based mouse actions.

IPC and shell:

- CLI-to-daemon IPC uses `\\.\pipe\neru` via `github.com/Microsoft/go-winio`.
- Windows doctor/status paths use the named pipe.
- Official `just` recipes run on this host when invoked through Git Bash:
  `just --shell C:\opt\Git\bin\sh.exe --shell-arg -cu build`.

Systray:

- Windows systray support exists.
- The Windows daemon host runs the systray loop.

## Remaining Work

Windows has basic support now, but it is not full macOS parity. Keep new work
small; each item below is intended to be a practical PR-sized task.

### 1. Fix Stale Unsupported-Platform Warning

Problem:

- [main.go](../cmd/neru/main.go) still prints the old non-Darwin warning:
  "Most features ... are stubs" and "Only macOS is currently supported."
- That is false for current Windows basic support.

Tasks:

- Change the launch warning to distinguish Linux and Windows support levels.
- Avoid warning Windows users that working features are stubs.
- Keep caveats honest about missing notifications, app watcher, and full parity.

Done when:

- Launching on Windows no longer prints obsolete support text.
- Linux still gets accurate caveats.

### 2. Finish Mouse Action Parity

Problem:

- Basic click, drag primitives, and vertical scroll exist.
- Windows still ignores horizontal scroll, click modifiers, and restore-cursor
  intent.

Likely files:

- [input.go](../internal/core/infra/platform/windows/input.go)
- [element_windows.go](../internal/core/infra/accessibility/element_windows.go)
- [action_service.go](../internal/app/services/action_service.go)
- [scroll_service.go](../internal/app/services/scroll_service.go)

Tasks:

- Add horizontal scroll using `MOUSEEVENTF_HWHEEL`.
- Apply configured action modifiers around click/down/up actions.
- Honor restore-cursor behavior for point actions.
- Implement or honestly report Windows event-tap config APIs that are still
  no-op/trivial today: modifier passthrough, intercepted modifier keys,
  synthetic modifier posting, and keyboard layout selection.
- Add unit tests for generated input flags where possible.
- Add `integration && windows` coverage only where real input is safe.

Done when:

- Left/right horizontal scroll works in apps that support it.
- Modified clicks match macOS/Linux action semantics.
- Cursor restoration matches user configuration.

### 3. Validate Multi-Monitor And DPI Behavior

Problem:

- Monitor APIs and overlay bounds are implemented.
- Left-of-primary, above-primary, and mixed-DPI layouts still need explicit
  validation.

Likely files:

- [system.go](../internal/core/infra/platform/windows/system.go)
- [manager_windows.go](../internal/ui/overlay/manager_windows.go)
- [overlay.go](../internal/core/infra/platform/windows/overlay.go)
- [system_windows_integration_test.go](../internal/core/infra/platform/windows/system_windows_integration_test.go)

Tasks:

- Test negative monitor origins.
- Test mixed-DPI monitor layouts.
- Verify grid, recursive grid, hints, mode indicator, sticky indicator, mouse
  action indicator, and click targeting on each monitor.
- Fix coordinate conversion, overlay sizing, or window placement if mismatches
  appear.

Done when:

- Overlay geometry and click targets line up on all attached monitors.
- No Windows path assumes primary monitor origin `(0,0)`.

### 4. Add App And Display Watchers

Problem:

- App watcher capability is still a stub.
- Foreground app changes and display/DPI changes are not watched through a
  Windows event source.

Likely files:

- [platform_other.go](../internal/core/infra/appwatcher/platform_other.go)
- [factory_windows.go](../internal/core/infra/platform/factory_windows.go)
- [manager_windows.go](../internal/ui/overlay/manager_windows.go)

Tasks:

- Use `SetWinEventHook(EVENT_SYSTEM_FOREGROUND)` for focused app changes.
- Refresh app-specific config and hotkeys on foreground changes.
- Listen for `WM_DISPLAYCHANGE` and `WM_DPICHANGED` through a hidden message
  window.
- Refresh overlay bounds after monitor or DPI changes.

Done when:

- App-specific exclusions and hotkeys react when focus changes.
- Overlays resize correctly after monitor and DPI changes.
- `WindowsCapabilities().AppWatcher` can honestly report supported.

### 5. Fix Dark Mode Capability Reporting

Problem:

- Windows dark-mode registry detection and theme polling exist.
- `WindowsCapabilities()` still reports dark mode detection as a stub.

Likely files:

- [theme.go](../internal/core/infra/platform/windows/theme.go)
- [theme_observer_windows.go](../internal/app/theme_observer_windows.go)
- [capability_presets.go](../internal/core/ports/capability_presets.go)

Tasks:

- Confirm registry behavior on supported Windows 10/11 versions.
- Update capability reporting to supported if current implementation is enough.
- Add a focused unit test or documented manual check.

Done when:

- Capability text matches runtime behavior.
- `neru doctor` no longer reports dark mode as unimplemented on Windows.

### 6. Harden Named-Pipe IPC

Problem:

- Windows IPC uses `go-winio` named pipes.
- `ListenPipe(path, nil)` relies on default pipe security.

Likely files:

- [transport_windows.go](../internal/core/infra/ipc/transport_windows.go)
- [SECURITY.md](../SECURITY.md)
- [ARCHITECTURE.md](ARCHITECTURE.md)
- [CLI.md](CLI.md)

Tasks:

- Verify `go-winio` license and module path casing.
- Decide whether the pipe needs an explicit local-user security descriptor.
- Check pipe-squatting and cross-user access behavior.
- Document the Windows IPC transport and security model.

Done when:

- CLI-to-daemon IPC is limited to the intended local user boundary.
- Security docs describe the Windows transport.

### 7. Add Windows Toast Notifications

Problem:

- Native MessageBox alerts exist.
- `ShowNotification` is still a no-op, so systray feedback notifications do not
  surface on Windows.

Likely files:

- [system.go](../internal/core/infra/platform/windows/system.go)
- [systray.go](../internal/app/components/systray/systray.go)
- [capability_presets.go](../internal/core/ports/capability_presets.go)

Tasks:

- Implement Windows toast notifications or another appropriate native
  notification path.
- Decide behavior for unpackaged `.exe` builds.
- Update notification capability reporting.

Done when:

- Systray actions that call `ShowNotification` produce visible Windows
  feedback.
- `WindowsCapabilities().Notifications` can honestly report supported or a
  clearly scoped partial status.

### 8. Deepen UI Automation Coverage

Problem:

- UIA hints have initial coverage.
- Deeper UIA operations and richer element discovery are still missing.

Likely files:

- [uia_windows.go](../internal/core/infra/accessibility/uia_windows.go)
- [tree_windows.go](../internal/core/infra/accessibility/tree_windows.go)
- [element_windows.go](../internal/core/infra/accessibility/element_windows.go)
- [platform_other.go](../internal/core/infra/accessibility/platform_other.go)

Tasks:

- Extract value/current text, enabled state, focused state, and process ID from
  UIA where available.
- Add UIA cache requests or incremental tree updates for large applications.
- Add UIA pattern execution where useful: Invoke, Toggle, ExpandCollapse,
  SelectionItem, Value, ScrollItem.
- Improve popup/context-menu/tooltip enumeration.
- Tune roles for browsers, Electron, Office, terminals, and Windows Settings.
- Implement Windows `platformActiveScreenBounds`; the non-Darwin fallback
  currently returns an empty rectangle.
- Implement currently stubbed helper lookups only when a caller needs them:
  `ApplicationByPID`, `ApplicationByBundleID`, `ElementAtPosition`.

Done when:

- Hints are reliable in large and complex Windows apps.
- Coordinate clicks remain fallback, not the only action strategy.

### 9. Improve Rendering And Interaction Polish

Tasks:

- Consider Direct2D/DirectWrite if current rendering quality or performance
  becomes limiting.
- Add incremental overlay redraw instead of full-window repaint where useful.
- Improve recursive-grid animation and virtual pointer polish.
- Add screen-share hiding if Windows exposes a reliable signal.

### 10. Add Native Windows Product Packaging

Tasks:

- Validate Windows CLI UX with `-H windowsgui`: current Windows builds use the
  GUI subsystem, and terminal subcommands may need separate validation or
  split CLI/daemon packaging.
- Add service/autostart support through Task Scheduler, Startup folder, or a
  Windows service.
- Add installer packaging.
- Improve keyboard layout and IME handling beyond current virtual-key mapping.

### 11. Clean Up Windows Documentation And Comments

Problem:

- Some repo docs and comments still reflect the pre-merge Windows state.

Tasks:

- Fix [CROSS_PLATFORM.md](CROSS_PLATFORM.md) release recipe examples; current
  text omits the required architecture argument for `release-ci-windows`.
- Add Windows install/download guidance now that Windows release artifacts
  exist; README install docs are still macOS/Homebrew-oriented.
- Remove or reword stale "Windows stub" comments in overlay component files
  where manager-backed rendering is real.
- Update stale Windows integration-test comments: CI selects integration tests
  through `just test-all`, even though many real-session tests may skip in
  headless runners.
- Align client-side Windows doctor output with the full capability set,
  including notifications, app watcher, and dark-mode rows when no daemon is
  running.

Done when:

- Contributor docs, user install docs, code comments, and doctor output match
  the merged Windows support level.

## Current Capability Reporting

Reported supported in [capability_presets.go](../internal/core/ports/capability_presets.go):

- process
- screen
- cursor
- accessibility
- overlay
- global hotkeys
- keyboard event tap

Reported stub:

- notifications
- app watcher
- dark mode detection

The dark-mode row is stale relative to runtime behavior and should be updated
after confirming Windows 10/11 registry behavior.

## Best Next Slice

Highest-value next work:

1. Fix the stale non-Darwin launch warning.
2. Finish mouse action parity: horizontal wheel, action modifiers, restore
   cursor.
3. Correct dark-mode capability reporting.
4. Harden and document Windows named-pipe IPC security.
5. Add app/display watchers.
