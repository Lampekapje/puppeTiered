---
name: puppetiered-view-authoring
description: Write, edit, or validate a Puppetiered view file (.toml manifest in ~/Library/Application Support/Puppetiered/Views/ describing a sidebar view of services, multi-state toggles, actions, info monitors, or config files). Use whenever asked to create or update a Puppetiered view, convert terminal commands or scripts into a view, or debug/validate a view .toml file.
---

# Writing a Puppetiered View (v0.9.6+)

A **View** is a single `<id>.toml` file in `~/Library/Application Support/Puppetiered/Views/`.
Puppetiered watches that directory via FSEvents and hot-reloads on both atomic saves and in-place edits — no app restart or rebuild is needed. View manifests are parsed in-process by Puppetiered's own hardened, zero-leak native TOML parser (`ILRTOML`), which enforces strict syntax bounds and reads **only** the exact whitelisted schema keys defined below (nothing else exists inside the parser). If a file has a syntax or schema error, Puppetiered surfaces a load-issue card in **Overview** and **Settings** with direct click-through to **Open in Builder**, **Edit .toml**, or **Reveal in Finder** (plus manual **Reload Views** buttons in the sidebar and Settings).

This skill is **self-contained**: the complete v0.9.6 schema, capability matrix, and runtime guardrails are below. An exhaustive field-by-field reference is also bundled in `references/schema-reference.md` next to this file.

---

## 1. Exact Top-Level TOML Shape (No `[meta]`, No `[[items]]`)

> **CRITICAL SYNTAX RULE:**
> - View metadata keys (`id`, `name`, `icon`, `group`, `style`) live at the **top level** of the TOML file — **NEVER** wrap them in `[meta]` or `[view]`.
> - Item tables use the **singular** key `[[item]]` (and `[[item.param]]`, `[[item.state]]`) — **NEVER** `[[items]]`.

```toml
id = "xcode-device-sync"       # required: unique slug across ALL installed views
name = "Xcode Device Sync"     # required: sidebar display name
icon = "network"               # optional (default "folder"): see 8 valid icons below
group = "system"               # optional (default "modules"): sidebar section heading
style = "dashboard"            # optional: "dashboard" (default) | "browser"

[defaults]                     # optional: shared default values for action params by name
service = "Wi-Fi"

[[item]]
id = "usbmuxd-daemon"          # required: unique across ALL installed views
label = "usbmuxd (USB Mux)"    # required: row title
kind = "process"               # required: process | toggle | action | info | config_file
status_cmd = "pgrep -x usbmuxd"
start_cmd = "launchctl kickstart -k system/com.apple.mobile.usbmuxd"
stop_cmd = "killall -9 usbmuxd"
cwd = "/private/etc"
requires_sudo = true
```

### View-Level Fields

| Key | Type | Required | Default | Rules & Behavior |
|:---|:---|:---|:---|:---|
| `id` | string | **yes** | — | Lowercase hyphenated slug. Must be unique across **all** installed `.toml` files (cross-file collisions are rejected). |
| `name` | string | **yes** | — | Human-readable title shown in the sidebar and view header. |
| `icon` | string | no | `"folder"` | Must be one of the **8 built-in icon names**: `"server"`, `"shield"`, `"network"`, `"folder"`, `"package"`, `"container"`, `"vm"`, `"code"`. Any other string silently falls back to a generic grid icon. |
| `group` | string | no | `"modules"` | Freeform sidebar section header (`"system"`, `"network"`, `"dev"`, etc.). **Special value:** `"examples"` hides the view from the main sidebar, Overview, and Settings, showing it only on the **Examples** screen. |
| `style` | string | no | `"dashboard"` | `"dashboard"` (all items visible, grouped by kind, live-polled every 2s) or `"browser"` (left category list + right on-demand detail pane; **nothing polls**, supports `info` and `action` items only). |
| `[defaults]` | table | no | `{}` | String-to-string map pre-filling `[[item.param]]` starting values by matching param `name` across multiple actions in the view. |
| `[[item]]` | array of tables | **yes** | — | Zero or more items (see the 5 item kinds below). |

---

## 2. The Five Item Kinds & How to Maximize Them

Every `[[item]]` requires `id`, `label`, and `kind`, and supports an optional `section = "..."` string.
- **Cross-view uniqueness:** Every `item.id` must be unique not only within its own file, but **across all installed views**. Prefix item IDs with the domain (e.g. `ktrace-artrace` instead of `status`) so they never collide with another view.
- **Custom Named Sections (`section = "..."`):** On `style = "dashboard"` views, setting `section = "Core Build Engine"` or `section = "Ancillary Tool — Simulator"` on any `[[item]]` groups items sharing that `section` name into their own titled card (in the order sections first appear in the `.toml` file) instead of lumping all items under a single `"Services"` / `"Info"` / `"Actions"` card. Section names are also searchable in the view's search bar and editable in the visual Builder.

### A. `kind = "process"` — Start/Stop Services, Live Elapsed/CPU/RAM Chips, Top Overview Card & Drill-Down

| Field | Required | Notes |
|:---|:---|:---|
| `status_cmd` | **yes** | Exit `0` = running; non-zero with empty stderr = stopped; non-zero with stderr or timeout = Unavailable. **Must print the numeric PID as its first token** (see below). |
| `start_cmd` | **yes** | Command to start/kickstart the service. |
| `stop_cmd` | **yes** | Command to stop/kill the service. |
| `cwd` | **yes** | Working directory (use `"/private/etc"` or `"/tmp"` for system daemons, or the project path for local services). |
| `section` | no | Custom section card title (defaults to `"Services"` when omitted). |
| `port` | no | Integer port badge displayed next to the service name. |
| `requires_sudo` | no (`false`) | Routes `start_cmd` and `stop_cmd` through the macOS admin prompt (`osascript`). `status_cmd` **never** prompts. |

> **Maximizing `process` Features (Zero-Subprocess `pgrep -x` + Overview Card + Inline Telemetry + Live Process Inspector):**
> - **Prefer `pgrep -x <name>` (Zero-Subprocess `libproc` Fast-Path):** When `status_cmd` is `pgrep -x <name>` (including `||`-chained alternatives like `pgrep -x swift-frontend || pgrep -x swiftc` or quoted names like `pgrep -x 'Icon Composer'`), Puppetiered resolves the PID **in-process via Darwin `libproc` (`proc_listallpids` / `proc_pidpath`) with zero shell subprocesses**. Use `pgrep -x` whenever possible instead of `pgrep -f`.
> - **Top-of-View Overview Card with Click-to-Scroll:** Any dashboard view with `process` items automatically renders an **Overview** card at the top showing `X/Y Running`, aggregate CPU% & RAM across active processes in the view, and a clickable list of currently running items. Clicking any running item in the Overview card smooth-scrolls straight down to its section/row and highlights it (or click `ⓘ` to jump straight into the live process inspector).
> - **Inline Elapsed Uptime & Mild System Usage (`⏱ elapsed` · `CPU% · RAM`):** Puppetiered automatically batch-polls `ps` for all running service PIDs in the active view in a single subprocess and displays live `⏱ elapsed` and `CPU% · RAM` chips directly on each running service row.
> - **Click-Through Live Process Inspector:** Clicking any **running** `process` row title opens a dedicated 1-second drill-down screen showing live **CPU %**, **Resident Memory (MB & %)**, **Elapsed Time**, **Cumulative CPU Time**, **Open Files** (`lsof -p <pid> -F ftn`), and **Network Connections** (`lsof -a -p <pid> -i -F ftn`).
> - **Rule 1 (PID Output):** Puppetiered extracts the live PID from the **first integer token** printed by `status_cmd` (e.g., `pgrep -x mydaemon` or `lsof -t -i:9002 | head -1`). If `status_cmd` exits `0` without printing the numeric PID first, the row cannot capture a PID — which disables inline elapsed/CPU/RAM chips, Overview metrics, and click-through drill-down!
> - **Rule 2 (User vs. Root Daemons):** The live process inspector runs `lsof -p <pid>` unprivileged. For user-owned services (`UID 501`), click-through automatically shows all open files and sockets. For `root` / system daemons (`usbmuxd`, `remoted`), macOS hides root file descriptors from unprivileged `lsof` — so pair root `process` items with a `requires_sudo = true` `info` or `action` item when the user needs to inspect root sockets/files.

---

### B. `kind = "toggle"` — Classic On/Off Switch OR Multi-State Picker

#### 1. Classic Two-State Toggle (no `[[item.state]]` tables)

| Field | Required | Notes |
|:---|:---|:---|
| `status_cmd` | **yes** | Read-only command polled every 2s. |
| `status_parse` | **yes** | Must be `"colon_tail"` (text after last `:` is `on`/`off`), `"exit_code"` (`0` = on, non-zero with empty stderr = off), or `"contains:TEXT"` (stdout contains `TEXT`). |
| `on_cmd` | **yes** | Command to turn the setting on. |
| `off_cmd` | **yes** | Command to turn the setting off. |
| `cwd` | **yes** | Working directory. |
| `requires_sudo` | no (`false`) | Gates `on_cmd` and `off_cmd`. |

#### 2. Multi-State Toggle Picker (2+ `[[item.state]]` tables)
Use this whenever a setting has **more than two states** (or two named modes like `Automatic` / `Off` / `Link-local`). Up to 4 states render as an inline segmented control; 5+ states render as a dropdown menu.

```toml
[[item]]
id = "wifi-ipv6-mode"
label = "IPv6 (Wi-Fi)"
kind = "toggle"
status_cmd = "networksetup -getinfo Wi-Fi | awk -F': ' '/^IPv6/{print $2}'"
status_parse = "line"          # optional: "line" (default) or "colon_tail"
cwd = "/private/etc"
requires_sudo = true

[[item.state]]
value = "Off"                  # required: exact case-insensitive text status_cmd outputs
set_cmd = "networksetup -setv6off Wi-Fi"

[[item.state]]
value = "Automatic"
label = "Automatic (SLAAC)"    # optional: UI label (defaults to value)
set_cmd = "networksetup -setv6automatic Wi-Fi"

[[item.state]]
value = "Link-local"
label = "Link-local only"
set_cmd = "networksetup -setv6LinkLocal Wi-Fi"
```
- Requires at least 2 `[[item.state]]` entries, each with `value` and `set_cmd` (optional `label`).
- `on_cmd` and `off_cmd` are omitted/ignored on multi-state toggles.
- `status_parse` must be `"line"` (default) or `"colon_tail"` (`exit_code` and `contains:` are rejected on multi-state toggles).

---

### C. `kind = "action"` — One-Shot Commands, Group Resets & Parameterized Runs

| Field | Required | Notes |
|:---|:---|:---|
| `run_cmd` | **yes** | Command executed on click. May contain `{{name}}` holes matching `[[item.param]]` entries. |
| `cwd` | **yes** | Working directory. |
| `requires_sudo` | no (`false`) | Routes `run_cmd` through the macOS admin password dialog. |
| `confirm` | no | Shows a confirmation dialog with this exact message before executing (use for destructive resets; omit for harmless/reversible actions). |
| `[[item.param]]` | no | Array of parameter tables (`name`, optional `prompt`, optional `default`, optional `type = "text" | "date"`). |

- **Parameter quoting:** For `type = "text"` (default), user input is split on whitespace and **each token is POSIX-shell-quoted individually** (so `"1.1.1.1 1.0.0.1"` becomes `'1.1.1.1' '1.0.0.1'`, safe against shell injection). Do not wrap `{{param}}` in extra quotes inside `run_cmd`.
- **Calendar dates (`type = "date"`):** Renders a native macOS date picker (with `+1y`/`+5y`/`+10y` jump buttons) and quotes the formatted timestamp (`yyyy-MM-dd HH:mm:ss Z`) as a **single** shell token.
- **Hole validation:** Every `{{name}}` in `run_cmd` must have a matching `[[item.param]] name = "name"` and vice versa (orphaned holes and unused params are rejected by the Builder validator).

---

### D. `kind = "info"` — Live Read-Only Status, Tables & Sticky Admin Snapshots

| Field | Required | Notes |
|:---|:---|:---|
| `status_cmd` | **yes** | Read-only command. Single-line output renders inline; multi-line output renders in a monospaced block. |
| `cwd` | **yes** | Working directory. |
| `copyable` | no (`false`) | Adds a one-click Copy button next to the output (set `true` for IPs, UDIDs, diagnostic tables, or commands). |
| `requires_sudo` | no (`false`) | **Never prompts during the automatic 2s poll.** Instead, if the unprivileged poll fails (`Unavailable`), it reveals a manual **"Refresh with Admin"** button that fetches a **sticky** elevated snapshot on click. |

---

### E. `kind = "config_file"` — In-App Config Editor with Validate, Apply & 20 Backups

| Field | Required | Notes |
|:---|:---|:---|
| `path` | **yes** | Absolute path to the real config file on disk (e.g. `"/etc/pf.conf"`). |
| `validate_cmd` | **yes** | Must contain `{{file}}` — run against a temporary scratch copy of the draft before touching `path`. |
| `apply_cmd` | **yes** | Must contain `{{file}}` — run after backing up `path` and writing the validated draft. |
| `requires_sudo` | no (`false`) | Runs backup + write + `apply_cmd` inside a **single** admin prompt. |

> **Note:** `config_file` is the **only** item kind that does **not** take `cwd` (its working directory is derived automatically from `path`'s parent folder).

---

## 3. Critical macOS & Puppetiered Runtime Pitfalls (Must Check!)

1. **The `requires_sudo = true` (`osascript`) Root UID Trap (`$(id -u)` is `0`, NOT `501`!)**
   - Puppetiered runs `requires_sudo = true` commands via `/usr/bin/osascript -e 'do shell script "..." with administrator privileges'`.
   - Inside that subshell, `$(id -u)` evaluates to **`0` (`root`)** and `$HOME` is `/var/root`.
   - Whenever a `requires_sudo = true` command needs the logged-in console user's UID or username (e.g., `launchctl kickstart -k gui/<uid>/...`, `su - <user>`, or fixing ownership on `/var/db/lockdown/RemotePairing/user_<uid>`), **ALWAYS use**:
     ```sh
     CONSOLE_UID=$(stat -f '%u' /dev/console)
     CONSOLE_USER=$(stat -f '%Su' /dev/console)
     ```
     **Never** use `$(id -u)` or `$USER` inside a `requires_sudo = true` command when you mean the logged-in desktop user!

2. **Non-Interactive Subshell / No-TTY Quirks**
   - Commands run with `stdin` closed and no controlling TTY.
   - CLIs that behave differently without a TTY **must** be given explicit non-interactive flags:
     - `xcrun devicectl`: **Always** pass `--timeout 5` (or `--timeout 30`); without `--timeout`, `devicectl` immediately exits `64` (`Error: Please specify a 'timeout' value`) when spawned from a non-interactive pipe!
     - `lsof`: Always pass `-n -P` so DNS and port-name resolution don't stall the 2s poll timeout.
     - `top`: Always pass `-l 1` (or `-l 2 -n 10`).

3. **Native OS Tools First (Zero Unnecessary Dependencies)**
   - **Always prefer built-in macOS / Unix / POSIX CLI tools** (`pgrep -x`, `ps`, `awk`, `sed`, `grep`, `stat`, `sysctl`, `networksetup`, `scutil`, `lsof`, `nc`, `netstat`, `tmutil`, `diskutil`, `osascript`) for service, file, network, and process monitoring.
   - Do **not** invoke `python3`, `node`, `jq`, or other external runtimes for tasks that native Unix pipelines (`ps -raxo ... | awk '...'`) solve natively — spawning an interpreter every 2 seconds wastes significant CPU and adds external runtime dependencies.
   - Only use `python3` or third-party CLI tools when there is genuinely no built-in OS tool for the job (e.g. querying a specific database table or parsing complex nested JSON where `plutil`/`awk` cannot), or when the user explicitly asks to use that tool.

4. **Respect the 2-Second Dashboard Poll Budget (< 0.5s)**
   - In `style = "dashboard"`, the currently visible view polls its `process`, `toggle`, and `info` items every 2 seconds (while off-screen views relax to a 20s background safety-net cadence).
   - Never put slow or heavy commands (`brew`, `system_profiler`, `xcrun devicectl`, `ktrace` dumps) in a dashboard `info` or `toggle` `status_cmd`. Put them in `kind = "action"` (on-demand button) or make the view `style = "browser"`.

5. **Multi-Line Shell Scripts in TOML**
   - Use TOML multi-line basic strings (`""" ... """`) or multi-line literal strings (`''' ... '''`).
   - Prefer `''' ... '''` for multi-line shell and `awk` scripts so backslashes (`\n`, `\t`, regexes) don't need double-escaping.

---

## 4. Pre-Delivery Validation Checklist

Before finishing any `.toml` view file:
1. **Top-level keys:** `id` and `name` present at root; no `[meta]` wrapper; items use `[[item]]` (singular).
2. **Icon & style:** `icon` is one of `server`, `shield`, `network`, `folder`, `package`, `container`, `vm`, `code`; `style` is `dashboard` or `browser` (`browser` contains only `info`/`action`).
3. **Cross-view ID uniqueness:** Check existing files in `~/Library/Application Support/Puppetiered/Views/*.toml` so neither the view `id` nor any `[[item]] id` collides with another installed view.
4. **Per-kind required fields:**
   - `process`: `status_cmd` (prints PID first!), `start_cmd`, `stop_cmd`, `cwd`.
   - `toggle` (classic): `status_cmd`, `status_parse` (`colon_tail` | `exit_code` | `contains:TEXT`), `on_cmd`, `off_cmd`, `cwd`.
   - `toggle` (multi-state): `status_cmd`, `cwd`, `status_parse` (`line` | `colon_tail`), and ≥ 2 `[[item.state]]` tables with unique `value` and `set_cmd`.
   - `action`: `run_cmd`, `cwd`, and 1-to-1 match between `{{hole}}` tokens in `run_cmd` and `[[item.param]]` names.
   - `info`: `status_cmd` (fast & read-only), `cwd`.
   - `config_file`: `path`, `validate_cmd` (has `{{file}}`), `apply_cmd` (has `{{file}}`), no `cwd`.
5. **Privilege check:** Any `requires_sudo = true` command that references the desktop user uses `$(stat -f '%u' /dev/console)` instead of `$(id -u)`.
