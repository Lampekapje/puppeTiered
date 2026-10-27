# Puppetiered — Full `.toml` Schema Reference (v0.9.6)

This is the complete field reference for Puppetiered `.toml` view manifests (`~/Library/Application Support/Puppetiered/Views/<id>.toml`), covering every top-level key, item kind, parameter table, multi-state toggle table, and validation rule supported in v0.9.6.

> **Hardened Parser Guarantee:** View files are parsed by Puppetiered's built-in, zero-dependency, zero-leak C99 parser (`ILRTOML`). It enforces strict bounds (no embedded NUL bytes, capped key/array nesting depth, 16 MB input ceiling, zero side effects) and extracts **only** the exact whitelisted schema keys documented below — any unknown key is ignored and nothing else exists inside the parser.

## View-Level Fields

| key | type | required | default | validation & behavior |
|---|---|---|---|---|
| `id` | string | yes | — | must be unique across *all* installed view files, not just this one |
| `name` | string | yes | — | sidebar and header display title |
| `icon` | string | no | `"folder"` | any string is accepted; the 8 built-in icon names below render dedicated icons, while any other string renders a generic grid glyph |
| `group` | string | no | `"modules"` | freeform sidebar section heading (`"apps"`, `"network"`, `"system"`, `"dev"`, etc.). Special value: `"examples"` is excluded from the main sidebar, Overview, and Settings and shown only on the dedicated Examples screen. |
| `style` | string | no | `"dashboard"` | must be `"dashboard"` or `"browser"`; anything else falls back to `"dashboard"` and reports a load issue naming the bad value |
| `[[item]]` | array of tables | yes (can be empty) | — | zero or more items (see below) |
| `[defaults]` | table of string→string | no | `{}` | pre-fills an `action` param's starting value by *name* — e.g. set `service = "Wi-Fi"` once here instead of repeating `default = "Wi-Fi"` in every action that has a `service` param. An action's own per-param `default` still wins if a param name here doesn't match anything that action declares. |

### Built-In Icon Vocabulary

| `icon` value | SF Symbol shown |
|---|---|
| `"server"` | `server.rack` |
| `"shield"` | `checkmark.shield` |
| `"network"` | `network` |
| `"folder"` | `folder` |
| `"package"` | `shippingbox` |
| `"container"` | `cube.box` |
| `"vm"` | `desktopcomputer` |
| `"code"` | `chevron.left.forwardslash.chevron.right` |
| anything else | `square.grid.2x2` (generic fallback) |

## Fields Every `[[item]]` Has, Regardless of Kind

| key | type | required |
|---|---|---|
| `id` | string | yes — unique within the file (in-file duplicates are skipped with a load issue) and across all installed views |
| `label` | string | yes |
| `kind` | string | yes — must be exactly one of `process` / `toggle` / `action` / `info` / `config_file`; anything else is skipped with a load issue |
| `section` | string | no — optional custom section card heading on dashboard views (trimmed; empty string normalized to `nil`). Items within each kind sharing the same `section` string are grouped into their own titled card in order of first appearance; items without `section` use the default kind heading (`"Services"`, `"Toggles"`, `"Actions"`, `"Info"`, `"Config Files"`). Also indexed by in-view search. |

## `kind = "process"`

| key | type | required | notes |
|---|---|---|---|
| `status_cmd` | string | yes | exit 0 + parseable output = running; **first integer token printed is captured as the live PID** to enable: (1) inline `⏱ elapsed` and `CPU% · RAM` badges on the row, (2) the top-of-view **Overview** card (`X/Y Running`, aggregate CPU/RAM, and one-click smooth scroll to the item's section), and (3) click-through to the 1-second live process inspector (CPU/mem stats, open files, network sockets). **Zero-subprocess fast-path:** when `status_cmd` is `pgrep -x <name>` (including `||`-chained alternatives), Puppetiered resolves the PID in-process via Darwin `sysctl(KERN_PROC_ALL)` + `proc_pidpath` without spawning a shell. |
| `start_cmd` | string | yes | command to start the service |
| `stop_cmd` | string | yes | command to stop the service |
| `cwd` | string | yes | required on every kind that can contain a relative path — no default, no guessing |
| `port` | integer | no | display badge shown next to the service name — not used to derive status |
| `requires_sudo` | bool | no, default `false` | gates `start_cmd`/`stop_cmd` through the macOS admin prompt; `status_cmd` never prompts, regardless of this flag |

## `kind = "toggle"`

A toggle is either **classic** (an on/off switch) or **multi-state** (a picker over any
number of states). It is multi-state whenever it has `[[item.state]]` tables.

**Classic:**

| key | type | required | notes |
|---|---|---|---|
| `status_cmd` | string | yes | read-only status command |
| `status_parse` | string | yes | one of `"colon_tail"`, `"exit_code"`, or `"contains:TEXT"` (see below) — anything else reads as Unavailable |
| `on_cmd` | string | yes | command to enable the toggle |
| `off_cmd` | string | yes | command to disable the toggle |
| `cwd` | string | yes | working directory |
| `requires_sudo` | bool | no, default `false` | gates `on_cmd`/`off_cmd`; `status_cmd` never prompts |

**`status_parse` values (classic):**
- `colon_tail` — take the text after the last `:` on the first non-empty
  line, lowercase, must equal `on` or `off` (anything else → Unavailable
  with the actual value in the reason). Matches `networksetup
  -getairportpower`/`-getinfo`-style output.
- `exit_code` — exit 0 → on; non-zero → off (or Unavailable if stderr has
  content, since a real failure isn't the same as a genuine "off").
- `contains:TEXT` — on if stdout contains the literal `TEXT`; if the
  command fails outright (empty stdout + non-empty stderr), it reports
  Unavailable rather than silently reporting "off".

**Multi-state** (two or more states):

```toml
[[item]]
id = "ipv6-wifi"
label = "IPv6 (Wi-Fi)"
kind = "toggle"
status_cmd = "networksetup -getinfo Wi-Fi | awk -F': ' '/^IPv6/{print $2}'"
cwd = "/private/etc"
requires_sudo = true

[[item.state]]
value = "Off"
set_cmd = "networksetup -setv6off Wi-Fi"

[[item.state]]
value = "Automatic"
label = "Automatic (SLAAC)"   # optional; defaults to the value
set_cmd = "networksetup -setv6automatic Wi-Fi"

[[item.state]]
value = "Link-local"
label = "Link-local only"
set_cmd = "networksetup -setv6LinkLocal Wi-Fi"
```

| key | type | required | notes |
|---|---|---|---|
| `status_cmd` | string | yes | prints the current state's `value` |
| `cwd` | string | yes | working directory |
| `status_parse` | string | no, default `"line"` | `"line"` — the first non-empty line is the state; `"colon_tail"` — the text after its last `:`. `exit_code` and `contains:` are **rejected** on multi-state toggles |
| `requires_sudo` | bool | no, default `false` | gates every `set_cmd`; `status_cmd` never prompts |
| `on_cmd`, `off_cmd` | — | **not used** | ignored if present |
| `[[item.state]]` | array of tables | yes, **at least 2** | no upper bound |

| key (per state) | type | required | notes |
|---|---|---|---|
| `value` | string | yes | what `status_cmd` prints when this is the current state. Matched ignoring case and surrounding space; values must be unique ignoring case |
| `set_cmd` | string | yes | puts the setting into this state |
| `label` | string | no | what the picker shows; defaults to `value` |

An item that breaks a rule is skipped with a named reason while the rest of the file loads. If
the status output matches no state it reads as **unrecognized** (never guessed into the
first state); if there's no output it reads **unavailable**. Up to four states show as a
segmented control, five or more as a dropdown.

## `kind = "action"`

| key | type | required | notes |
|---|---|---|---|
| `run_cmd` | string | yes | may contain `{{name}}` holes matching declared params |
| `cwd` | string | yes | working directory |
| `requires_sudo` | bool | no, default `false` | routes `run_cmd` through the macOS admin prompt |
| `confirm` | string | no | if present, a confirmation dialog shows this exact text before running; only meaningful on `action` |
| `[[item.param]]` | array of tables | no | parameter definitions for `{{name}}` holes in `run_cmd` |

**`[[item.param]]` fields** (nested under an `action` item):

| key | type | required | default |
|---|---|---|---|
| `name` | string | yes | — matches a `{{name}}` hole in `run_cmd` |
| `prompt` | string | no | falls back to `name` itself |
| `default` | string | no | falls back to `""` |
| `type` | string | no | `"text"` — the only other recognized value is `"date"`, which renders a native calendar picker instead of a text field when the action runs |

Every param is `"text"` by default — one freeform field. Typed text is
split on whitespace and each token is POSIX-shell-quoted individually
before substitution (so one param can hold several space-separated values,
like a DNS server list, and can never break out of its own argument via
shell metacharacters).

`type = "date"` quotes the picked date as a *single* token instead (its spaces
are part of one value — e.g. `"2027-06-01 00:00:00 +0000"`, the format
`defaults write ... -date` expects).

## `kind = "info"`

| key | type | required | notes |
|---|---|---|---|
| `status_cmd` | string | yes | read-only command; single-line output renders inline and multi-line output renders in a monospaced block |
| `cwd` | string | yes | working directory |
| `requires_sudo` | bool | no, default `false` | **does not gate the automatic poll** (which always runs unprivileged so a view never triggers a password prompt on a timer) — instead, it enables a manual **"Refresh with Admin"** button that captures a sticky elevated snapshot when the unprivileged read is Unavailable |
| `copyable` | bool | no, default `false` | shows a copy-to-clipboard button next to the value (whichever is currently shown — the live auto-polled one or a sticky elevated snapshot) |

## `kind = "config_file"`

| key | type | required | notes |
|---|---|---|---|
| `path` | string | yes | absolute path to the real file this item edits |
| `validate_cmd` | string | yes | must contain a `{{file}}` hole — substituted with a temporary copy of the draft before touching `path` |
| `apply_cmd` | string | yes | must contain a `{{file}}` hole — substituted with the real `path`, run only after validation and automatic timestamped backup+write succeed |
| `requires_sudo` | bool | no, default `false` | gates the single Apply operation (validate + backup + write + `apply_cmd`, one privileged call); Validate-alone also routes through this flag |
| `cwd` | — | **not a field on this kind** | the working directory for both commands is derived automatically as `path`'s parent directory |

## Ignored / Kind-Specific Fields

Fields that belong to one item kind are ignored if written on another kind:
- `confirm` and `[[item.param]]` — only consumed on `action`.
- `port` — only consumed on `process`.
- `copyable` — only consumed on `info`.
- `on_cmd`, `off_cmd`, `status_parse`, and `[[item.state]]` — only consumed on `toggle`.
- `path`, `validate_cmd`, and `apply_cmd` — only consumed on `config_file`.

## Whole-File & Cross-View Validation Rules

- **Duplicate `id` within one file** → later item is skipped with a load issue; the rest of the file still loads.
- **Duplicate view `id` or item `id` across files** → flagged across installed views and rejected in the Builder.
- **Missing required field for the declared `kind`** → that item is skipped with a named reason; the rest of the file still loads.
- **Unknown `kind` string** → item is skipped with a load issue naming the bad value.
- **Unknown `style` string** → view still loads, falls back to `"dashboard"`, and reports a load issue naming the bad value.
- **Orphaned `{{name}}` holes in `run_cmd` or unused `[[item.param]]` declarations** → rejected by the Builder validator and authoring checklist.
