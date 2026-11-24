# puppeTiered

Exposure framework for everything local to macOS.

**Version `0.10.1` (Build `12`)** • **macOS 14.0+ Universal (`arm64` + `x86_64`)**  
**Created by Ivar Limpens Richaards (ILR / Lampekapje)** • **Bundle ID:** `com.ilr.puppetiered`

A native macOS control deck for watching and controlling the things you run on your own machine — services, processes, system settings, containers, VMs, diagnostics, and config files — grouped into modular **views** declared in a lightweight `.toml` DSL or built with the built-in visual Builder.

### The "Dark Magic" Notebook Problem

We all had that battered physical notebook (or sprawling `.txt` file) stuffed with obscure, dark-magic terminal incantations we used once 15 years and 25 clients ago — across half a dozen OSes and network gear (*shudder*) — don't pretend you didn't. Add to that everything modern macOS hides seven clicks deep in System Settings or behind a four-line shell pipeline just to check an IP, flush a cache, or flip a system toggle.

Puppetiered turns that notebook into a **declarative `.toml` DSL for live macOS UIs**. Instead of hardcoding a single-purpose dashboard, each view file is a self-contained manifest where you can mix and match **any combination** of live process/service switches, 2-state or multi-state toggles, parameterized one-shot actions, live diagnostic readouts, and validated config-file editors inside a single screen — organized into custom named sections that update automatically with zero boilerplate code.

---

## Why It Exists & Key Guarantees

- **Zero External Runtime or Third-Party Dependencies:** Puppetiered is a standalone native Swift/AppKit/SwiftUI universal binary (`arm64` + `x86_64`). It requires no Python, no Node, no Homebrew, and no background helper daemons, and uses its own hardened, [zero-leak](MEMORY_BENCHMARK_v0.9.6.md) native TOML parser that only accepts the exact syntax and schema fields Puppetiered works with — nothing else even exists inside the parser.
- **Zero Telemetry or Online Shenanigans:** No analytics, no crash reporters, no update pings, no accounts. The app makes zero network requests of its own; it only ever runs the local commands declared in your views.
- **Native OS Speed:** Process checks (`pgrep -x`) run in-process via Darwin `sysctl(KERN_PROC_ALL)` + `libproc` with zero subprocess spawning, device GPU utilization / VRAM and per-PID GPU time are read in-process via unprivileged `IOKit` (`IOAccelerator`), process exits are tracked immediately via kernel `kqueue` (`EVFILT_PROC` / `NOTE_EXIT`), and off-screen views automatically relax or pause polling when the window is hidden.
- **Bring Your Own Commands (or Language):** Built-in and starter views use native macOS/POSIX tools (`ps`, `awk`, `sed`, `grep`, `networksetup`, `lsof`, `launchctl`), while your own custom `.toml` views are free to call shell pipelines, Python scripts, database CLIs, or anything else on your machine.

---

## Installation

1. Download **`Puppetiered.dmg`** (or copy **`Puppetiered.app`** directly).
2. Drag **`Puppetiered.app`** into **`/Applications`**.
3. **First Launch (Gatekeeper):** Because this release is ad-hoc signed (`com.ilr.puppetiered`), macOS will block the very first launch. Allow it once via either:
   - **Terminal (fastest):**
     ```sh
     xattr -dr com.apple.quarantine /Applications/Puppetiered.app
     ```
   - **System Settings:** Try opening Puppetiered once → open **System Settings → Privacy & Security** → scroll to *Security* → click **Open Anyway**.

---

## What's Inside

| Screen | Purpose |
| :--- | :--- |
| **Overview** | At-a-glance status across all active views, system CPU load and live `IOKit` GPU/VRAM cards, and attention banners. |
| **Views** | Live control decks rendered from `.toml` files in `~/Library/Application Support/Puppetiered/Views/`. Supports `dashboard` views (live-monitored with custom `section` cards, optional `show_gpu = true` GPU/VRAM header pills, and a top `X/Y Running` Overview card with click-to-scroll), `browser` views (on-demand master-detail lists that never poll in the background), and `notebook` views (`style = "notebook"`, zero polling, zero auto-run command reference books). Dashboard and browser views display a clickable `<view-id>.sqlite` history pill (`on` / `off` / `problem`) and mirrored recording toggle. |
| **Notebook** | Your permanently accessible **Engineer's Little Black Book** in the sidebar footer. Includes a one-click starter template, multi-notebook or `"All Notebooks"` filtering, SQLite FTS5 + literal command/tag/notes search, freeform drag-and-drop groups (`section`) and entry reordering, clickable tags, `Quick Add` capture, interactive parameter workbench (auto-assembled on change or triggered manually via **Assemble Command**, configurable in **Settings → Notebook**), non-executing `zsh -n` syntax & binary validation, and smart pipeline/`sudo`-aware `man` page lookups. |
| **Starter Views** | One-click ready-made views for Docker, VirtualBox, and Python environments. |
| **Builder** | Visual editor for creating, modifying, or repairing any view (`dashboard`, `browser`, or `notebook` — including `notes`, `tags`, `source`, and `copy_only` action metadata, malformed or incomplete `.toml` files that failed to load, inline field validation highlights, and raw TOML fallback) without hand-writing TOML, complete with crash-safe draft autosave, a top **All Views** back button (`⌘[` / `Esc`), **Open View**, and `⌘S` / `⌃S` save shortcuts. |
| **Settings** | Drag-and-drop views within or across sidebar groups (cross-group moves surgically update `group` in `.toml` with Undo), hide/archive/delete views, configure **Notebook** command auto-assembly, toggle per-view SQLite history recording, manage archiving & calendar by-date pruning, copy CLI hooks, configure poll intervals, and switch between **System** and **Phosphor** (retro green-on-black CRT) themes. |

**Keyboard Navigation & Shortcuts:** `⌘1–⌘5` jump between Overview, Notebook, Builder, Starter Views, and Examples (`⌘,` Settings, `⌘R` Reload Views); `⌘K` / `⇧⌘K` focus search; `↑`/`↓` + `Return` browse and open rows; `Esc` clears filters, closes sheets, or exits the Builder editor (`⌘[`); and `⌘S` / `⌃S` save in Builder, Notebook Quick Add, inline Workbench edits, and Config File editors.

---

## Notebook — Your Engineer's Little Black Book

Have a half-page `csrutil`, `defaults write`, `xattr`, or `lsof` pipeline you only need once a year? Puppetiered pins a dedicated **Notebook** workspace permanently in the sidebar footer alongside Settings and Builder (`style = "notebook"`, zero background polling, zero auto-run):

- **Start in one click:** Generate the built-in **Little Black Book** starter notebook (postponing macOS update nags, SIP checks, quarantine removal, directory ownership, firewall status) or keep multiple topic notebooks searched together via **All Notebooks**.
- **SQLite FTS5 + literal symbol search:** Ranked full-text search (Porter stemming + BM25) across titles, tags, freeform groups, notes, commands, parameters, and sources — paired with literal substring matching for flags like `-dr`, plist domains, and pipes.
- **Freeform groups & drag-and-drop reordering:** Group entries into any `section` name you type (with inline group rename), click tags to filter, and drag-and-drop to reorder or move entries across groups — plus drag-and-drop reordering for views and sidebar groups directly in the sidebar and Settings.
- **`+ Quick Add` capture:** Save new snippets, notes, tags, sources, or `copy_only` commands in seconds, or open any entry in the Builder for parameters (`[[item.param]]`) and confirmations.
- **Editable Command Workbench:** Fill parameters live (auto-assembled as you type, or on demand with **Assemble Command** via **Settings → Notebook**), tweak the resulting command inline (**Reset** / **Save to View**), run non-executing `/bin/zsh -n` syntax + missing-binary validation, pop open smart `sudo`/pipeline-aware `man` pages, **Copy** / **Copy w/ sudo**, or **Run** directly from Puppetiered.

```toml
id = "little-black-book"
name = "Little Black Book"
icon = "book"
group = "notebook"
style = "notebook"

[[item]]
id = "lbb-defer-updates"
label = "Postpone Major macOS Update Nags"
kind = "action"
section = "System Updates"
run_cmd = "defaults write com.apple.SoftwareUpdate MajorOSUserNotificationDate -date {{until}}"
requires_sudo = true
tags = ["softwareupdate", "updates", "macos"]
notes = "Pushes major OS upgrade notifications out until the chosen UTC date."

[[item.param]]
name = "until"
prompt = "Defer until (UTC)"
type = "date"
```

---

## Anatomy of a `.toml` View (The View DSL)

Views live in `~/Library/Application Support/Puppetiered/Views/`. Drop a `.toml` file into that folder and Puppetiered loads it immediately — no restart or recompile required.

Because the `.toml` schema acts as a composable UI + command DSL, a single view isn't locked to one type of tool or resource. You can combine background daemons, GUI apps, kernel/network toggles, parameterized shell actions, live log/table blocks, and guarded configuration files in any order and any combination, grouped under custom `section = "..."` cards:

```toml
id = "my-stack"
name = "My Stack"
icon = "server"
group = "dev"

[[item]]
id = "api"
label = "API Server"
kind = "process"
section = "Core Services"
status_cmd = "pgrep -x my-api-binary"
start_cmd = "./run-api.sh"
stop_cmd = "pkill -x my-api-binary"
cwd = "/tmp"
port = 8000

[[item]]
id = "ipv6"
label = "IPv6 (Wi-Fi)"
kind = "toggle"
section = "Network"
status_cmd = "networksetup -getinfo Wi-Fi | awk -F': ' '/^IPv6/{print $2}'"
cwd = "/tmp"
requires_sudo = true

[[item.state]]
value = "Off"
set_cmd = "networksetup -setv6off Wi-Fi"

[[item.state]]
value = "Automatic"
set_cmd = "networksetup -setv6automatic Wi-Fi"
```

### The Five Item Kinds

1. **`process`** — A daemon, app, or service with live running/stopped status, PID badge, inline elapsed uptime + CPU/GPU/RAM usage, a top-of-view `X/Y Running` summary (with CPU/RAM and optional/auto GPU/VRAM pills) with click-to-scroll, and a 1-second live process inspector (CPU/GPU/memory, open files, and network sockets).
2. **`toggle`** — A 2-state switch (`on_cmd` / `off_cmd`) or multi-state segmented/dropdown selector (`[[item.state]]`).
3. **`action`** — A one-off command button, optionally with `confirm` prompts or parameterized `[[item.param]]` inputs.
4. **`info`** — A live read-only status readout (single-line output renders inline; multi-line output automatically renders in a monospaced diagnostic block), with optional `copyable = true` and `requires_sudo = true` sticky admin snapshot refresh.
5. **`config_file`** — An in-app file editor with automatic timestamped backups (up to 20 kept), `validate_cmd` syntax checking (`{{file}}`), and `apply_cmd` hooks (`{{file}}`).

---

## Per-View SQLite History & CLI Hooks

Every view can opt in (either from **Settings → View List** or from the `.sqlite` toggle pill right in the view's header) to record state transitions, CPU/GPU/memory deltas, and executed actions into its own standalone database:

- **One `.sqlite` per view:** `~/Library/Application Support/Puppetiered/History/<view-id>.sqlite` (backed directly by macOS system `libsqlite3` in WAL mode — zero external connectors or heavy UI viewers bundled in the app).
- **Flat `history` & `state_changes` SQL views:** Open any `.sqlite` file in `sqlite3`, TablePlus, DB Browser, DuckDB, Python, or your own message broker and `SELECT * FROM history` or `SELECT * FROM state_changes` without manual joins.
- **No automatic caps:** Unrecorded views keep full `pauseWhenHidden` zero-footprint behavior; recorded views continue background logging on a separate `10s–300s` cadence (`30s` default) while hidden. **Settings → History & Archiving** includes calendar by-date pruning (`DatePicker` + `7d`/`30d`/`90d`/`1y` jumps), **Archive & Start New** WAL rotation into `History/Archive/`, and Trash-safe clearing.
- **Built-in CLI Hooks:**
  ```sh
  # Invoke directly from the app bundle (or symlink `puppetiered-history` from bin/):
  /Applications/Puppetiered.app/Contents/MacOS/Puppetiered history list
  /Applications/Puppetiered.app/Contents/MacOS/Puppetiered history status my-stack
  /Applications/Puppetiered.app/Contents/MacOS/Puppetiered history tail my-stack --changes-only --json
  /Applications/Puppetiered.app/Contents/MacOS/Puppetiered history query my-stack "SELECT * FROM state_changes ORDER BY sample_id DESC LIMIT 20"
  /Applications/Puppetiered.app/Contents/MacOS/Puppetiered history record my-stack on
  /Applications/Puppetiered.app/Contents/MacOS/Puppetiered history prune my-stack --before 30d --yes
  /Applications/Puppetiered.app/Contents/MacOS/Puppetiered history archive my-stack
  ```

---

## Authoring Views with an AI Harness (`puppetiered-view-authoring`)

Included in the DMG and [`puppetiered-view-authoring/`](puppetiered-view-authoring/) is a self-contained AI skill (`SKILL.md` + `references/schema-reference.md`) for Claude Code, Codex, or Antigravity.

Drop `puppetiered-view-authoring` into your AI agent's skills folder (or pass `SKILL.md` as context) and ask your harness:
> *"Using the `puppetiered-view-authoring` skill, build a Puppetiered `.toml` view for [my local services / Xcode processes / network toggles] using native macOS commands."*

---

## Official Warranty, License & Copyright

> **OFFICIAL WARRANTY:** This software is guaranteed to **“work on my machine. All others best beware.”**  
> Provided strictly **AS-IS** with zero external dependencies, zero telemetry, zero support channels, and zero ways to reach the author.

### The Obligatory 40-Day Shareware Notice
Puppetiered is distributed as a **40-day shareware evaluation copy**. After 40 days, you must either buy a license or do what everyone has done with WinRAR since 1995: open **About Puppetiered**, click **“Remind Me in 40 Years”**, and keep using 100% of its features forever at full speed.

**Copyright © 2026 Ivar Limpens Richaards (ILR / Lampekapje). All rights reserved.**  
Poking under the hood for your own amusement is fair game; redistributing or modifying the binary for others is prohibited (and since there are zero ways to ask for permission, the answer is already no).
