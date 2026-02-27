# Planned Features

Design principle: extend, don't fork. All changes should be additive to the existing codebase so we can
easily rebase on new upstream releases from `lwouis/alt-tab-macos`.

---

## 1. Filter/Close Windows by Domain

Filter the window list by a web domain (e.g. `github.com`) and close all matching browser windows/tabs at once.

- Should work across browsers that expose tab URLs via accessibility APIs
- Add CLI commands: `--list-domain=<domain>`, `--close-domain=<domain>`

---

## 2. YAML-based Settings (bridge, not replacement)

A YAML config file (`~/.config/alt-tab-macos/config.yaml`) that maps to UserDefaults.

- Reads YAML on launch (and on file change via FSEvents watcher)
- Writes values into UserDefaults through existing `Preferences.set()` so all existing
  preference validation, caching (`CachedUserDefaults`), and change notification
  (`PreferencesEvents`) continue to work
- `--export-config` CLI command to dump current UserDefaults to YAML
- `--import-config=<path>` CLI command to apply a YAML file
- Benefit: version-controllable, portable, scriptable config

---

## 3. Extended CLI: Space-filtered Window Switching

Extend `CliEvents.swift` / `CliServer` with new commands (same CFMessagePort protocol):

- `--show-space=<N>` — show the alt-tab UI filtered to windows on space N only.
  Implemented by setting a transient space filter before calling `App.showUi()`,
  so `Windows.refreshWhichWindowsToShowTheUser()` only includes windows whose
  `spaceIndexes` contains N.
- `--cycle-space=<N>` — if UI is already showing for space N, cycle to next window;
  otherwise show UI filtered to space N.
- `--list-space=<N>` — JSON list of windows on space N.

---

## 4. yabai + skhd Integration (orchestrator script)

A helper script/config generator that wires together yabai, skhd, and alt-tab-macos.

### F1-F12 space switching + cycling workflow

The workflow for each key (e.g. F1 for space 1):

1. **First press**: yabai switches to space N (`yabai -m space --focus N`)
2. **Second press** (already on space N): alt-tab shows window cycler filtered to space N
   (`AltTab --show-space=N` or `AltTab --cycle-space=N`)

Implementation options for space switching (F1-F12 → spaces 1-12):

- **Option A: skhd + yabai** — skhd binds F1-F12, a small shell script checks current
  space via `yabai -m query --spaces`, and either switches space or calls alt-tab CLI
- **Option B: macOS Mission Control shortcuts** — a setup script writes keyboard shortcut
  preferences to `com.apple.symbolichotkeys.plist` mapping F1-F12 to "Switch to Desktop N".
  Includes `--backup-shortcuts` / `--restore-shortcuts` to save/revert previous settings.
- **Option C: hybrid** — use macOS native shortcuts for space switching (no yabai/SIP
  dependency), skhd for the "already on this space → cycle windows" logic

### Config format (YAML)

```yaml
spaces:
  1: { key: f1 }
  2: { key: f2 }
  3: { key: f3 }
  # ...
  12: { key: f12 }

behavior:
  first_press: switch_space    # via yabai or native Mission Control shortcut
  second_press: cycle_windows  # via alt-tab --cycle-space=N
```

The orchestrator reads this and generates:
- skhdrc entries
- Optionally sets macOS Mission Control keyboard shortcuts (with backup/restore)
- alt-tab UserDefaults (via YAML bridge or CLI)

---

## Implementation Order

1. **Extended CLI** (#3) — smallest change, highest leverage, all in `CliEvents.swift`
2. **YAML bridge** (#2) — new file, reads YAML → writes UserDefaults, no existing code modified
3. **Orchestrator script** (#4) — external to alt-tab, wires everything together
4. **Domain filtering** (#1) — needs accessibility research for browser tab URL extraction
