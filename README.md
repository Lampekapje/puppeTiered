# puppeTiered

Exposure framework for everything local to macOS.

**Version `0.9.6` (Build `7`)** • **macOS 14.0+ Universal (`arm64` + `x86_64`)**  
**Created by Ivar Limpens Richaards (ILR / Lampekapje)** • **Bundle ID:** `com.ilr.puppetiered`

A native macOS control deck for watching and controlling the things you run on your own machine — services, processes, system settings, containers, VMs, diagnostics, and config files — grouped into modular **views** declared in a lightweight `.toml` DSL or built with the built-in visual Builder.

### The "Dark Magic" Notebook Problem

We all had that battered physical notebook (or sprawling `.txt` file) stuffed with obscure, dark-magic terminal incantations we used once 15 years and 25 clients ago — across half a dozen OSes and network gear (*shudder*) — don't pretend you didn't. Add to that everything modern macOS hides seven clicks deep in System Settings or behind a four-line shell pipeline just to check an IP, flush a cache, or flip a system toggle.

Puppetiered turns that notebook into a **declarative `.toml` DSL for live macOS UIs**. Instead of hardcoding a single-purpose dashboard, each view file is a self-contained manifest where you can mix and match **any combination** of live process/service switches, 2-state or multi-state toggles, parameterized one-shot actions, live diagnostic readouts, and validated config-file editors inside a single screen — organized into custom named sections that update automatically with zero boilerplate code.

---

## Why It Exists & Key Guarantees

- **Zero External Runtime or Third-Party Dependencies:** Puppetiered is a standalone native Swift/AppKit/SwiftUI universal binary (`arm64` + `x86_64`). It requires no Python, no Node, no Homebrew, and no background helper daemons, and uses its own hardened, [zero-leak](MEMORY_BENCHMARK_v0.9.6.md) native TOML parser that only accepts the exact syntax and schema fields Puppetiered works with — nothing else even exists inside the parser.
- **Zero Telemetry or Online Shenanigans:** No analytics, no crash reporters, no update pings, no accounts. The app makes zero network requests of its own; it only ever runs the local commands declared in your views.
- **Native OS Speed:** Process checks (`pgrep -x`) run in-process via Darwin `sysctl(KERN_PROC_ALL)` + `libproc` with zero subprocess spawning, process exits are tracked immediately via kernel `kqueue` (`EVFILT_PROC` / `NOTE_EXIT`), and off-screen views automatically relax or pause polling when the window is hidden.
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
| **Overview** | At-a-glance status across all active views, system load/memory bar, and attention banners. |
| **Views** | Live control decks rendered from `.toml` files in `~/Library/Application Support/Puppetiered/Views/`. Supports `dashboard` views (live-monitored with custom `section` cards and a top `X/Y Running` Overview card with click-to-scroll) and `browser` views (on-demand master-detail lists that never poll in the background). |
| **Starter Views** | One-click ready-made views for Docker, VirtualBox, and Python environments. |
| **Builder** | Visual editor for creating or modifying views without hand-writing TOML, complete with crash-safe draft autosave. |
| **Settings** | Reorder views and sidebar groups, hide/archive/delete views, configure poll interval and pause-when-hidden behavior, and switch between **System** and **Phosphor** (retro green-on-black CRT) themes. |

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

1. **`process`** — A daemon, app, or service with live running/stopped status, PID badge, inline elapsed uptime + CPU/RAM usage, a top-of-view `X/Y Running` summary with click-to-scroll, and a 1-second live process inspector (CPU/memory, open files, and network sockets).
2. **`toggle`** — A 2-state switch (`on_cmd` / `off_cmd`) or multi-state segmented/dropdown selector (`[[item.state]]`).
3. **`action`** — A one-off command button, optionally with `confirm` prompts or parameterized `[[item.param]]` inputs.
4. **`info`** — A live read-only status readout (single-line output renders inline; multi-line output automatically renders in a monospaced diagnostic block), with optional `copyable = true` and `requires_sudo = true` sticky admin snapshot refresh.
5. **`config_file`** — An in-app file editor with automatic timestamped backups (up to 20 kept), `validate_cmd` syntax checking (`{{file}}`), and `apply_cmd` hooks (`{{file}}`).

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
