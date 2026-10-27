# Puppetiered v0.9.6 (Build 7) — Memory & Leak Benchmark Data

- **Target:** `Puppetiered.app` (`com.ilr.puppetiered`)
- **Version:** `0.9.6` (Build `7`)
- **Architecture:** Universal (`arm64` + `x86_64`)
- **OS:** macOS 15.7.3 (`24G419`)
- **Tools:** `/usr/bin/leaks`, `/usr/bin/footprint`, `ps`, `getrusage(RUSAGE_SELF)`

---

## 1. Build 6 vs. Build 7 Comparison

| Metric | v0.9.5 (Build 6) | v0.9.6 (Build 7) |
| :--- | :--- | :--- |
| **Parser Backend** | Third-party C++ TOML library | Custom C99 arena parser |
| **Allocation Strategy** | Heap per-node + ref-counted wrappers | Paged `8 KB` arena (`O(1)` single-pass teardown) |
| **Leaked Nodes (Reload Stress)** | `4,567` leaks | **`0` leaks** |
| **Leaked Bytes (Reload Stress)** | `277,728 bytes` (`271.2 KB`) | **`0 bytes`** |
| **Root Retain Cycles** | `423` cycles | **`0` cycles** |
| **Post-Warmup RSS Trajectory** | Linear growth (`O(N)` per reload) | Flat plateau (`0.0 KB` delta) |

---

## 2. Stress Tier Summary

| Tier | Workload | Iterations | Initial RSS | Plateau RSS | Post-Warmup Delta | Leaks (`leaks`) |
| :--- | :--- | ---: | ---: | ---: | ---: | :--- |
| **Tier 1** | C99 Parser Parse + Free (`~6.0 KB` view) | `250,000` | `1,392 KB` | `1,536 KB` | `0.0 KB` (cycles 40k–200k) | `0 leaks (0 B)` |
| **Tier 2** | Full View Load + Model Bridge (`5` views) | `100,000` | `21,952 KB` | `25,168 KB` | `0.0 KB` (cycles 20k–100k) | `0 leaks (0 B)` |
| **Tier 3** | Malformed / Hostile Payload Recovery (`10` inputs) | `50,000` | `21,904 KB` | `25,056 KB` | `0.0 KB` (cycles 10k–50k) | `0 leaks (0 B)` |
| **Tier 4** | Live GUI `FSEvents` Hot-Reload (`10` views × `100`) | `1,000` | `44,096 KB` | `46,272 KB` | `0.0 KB` (cycles 40–100) | `0 leaks (0 B)` |

---

## 3. Tier 1 — Native C99 Parser Parse/Free Loop (250,000 Cycles)

- **Input:** `~6.0 KB` multi-section `.toml` view manifest (nested arrays of tables, inline tables, multi-line strings, booleans, integers).
- **Execution:** `200,000` continuous parse + arena-free cycles (`getrusage` telemetry), followed by `50,000` cycles under `/usr/bin/leaks --atExit`.

### RSS Progression

| Cycle | Max RSS (`KB`) | Delta from Warmup (`KB`) |
| ---: | ---: | ---: |
| `0` (Initial) | `1,392` | — |
| `40,000` | `1,536` | `0.0` |
| `80,000` | `1,536` | `0.0` |
| `120,000` | `1,536` | `0.0` |
| `160,000` | `1,536` | `0.0` |
| `200,000` | `1,536` | `0.0` |

### `leaks --atExit` Output (50,000 Cycles)

```text
Process:         c_stress [29457]
Path:            /private/tmp/c_stress
Load Address:    0x1013ad000
Identifier:      c_stress
Version:         0
Code Type:       X86-64
Platform:        macOS

Physical footprint:         2400K
Physical footprint (peak):  2400K
Idle exit:                  untracked
----

leaks Report Version: 4.0, multi-thread, stacks
Process 29457: 133 nodes malloced for 16 KB
Process 29457: 0 leaks for 0 total leaked bytes.
```

---

## 4. Tier 2 — Full View Loader & Model Bridge (100,000 Cycles)

- **Input:** `5` distinct `.toml` view manifests (`20,000` full load + validation + Swift struct mapping cycles each = `100,000` total loads).

### RSS Progression

| Cycle | RSS (`KB`) | Delta from Warmup (`KB`) |
| ---: | ---: | ---: |
| `0` (Initial) | `21,952` | — |
| `20,000` | `25,168` | `0.0` |
| `40,000` | `25,168` | `0.0` |
| `60,000` | `25,168` | `0.0` |
| `80,000` | `25,168` | `0.0` |
| `100,000` | `25,168` | `0.0` |

### `leaks --atExit` Summary

```text
Physical footprint:         25.2M
Physical footprint (peak):  25.2M
Process 28966: 1253 nodes malloced for 79 KB
Process 28966: 0 leaks for 0 total leaked bytes.
```

---

## 5. Tier 3 — Malformed & Hostile Input Recovery (50,000 Cycles)

- **Input:** `10` malformed/hostile `.toml` payloads (`5,000` iterations each = `50,000` failed parses):
  1. Unterminated string literal
  2. Duplicate key definition
  3. Unclosed table header (`[[items]`)
  4. Unclosed array literal
  5. Invalid numeric literal (`interval = 12abc`)
  6. Invalid escape sequence (`\z`)
  7. Embedded NUL byte / `\u0000` escape rejection
  8. Deeply nested array (`> 32` depth cap rejection)
  9. Valid syntax missing required schema fields
  10. Valid syntax with unsupported item `kind`

### RSS Progression

| Cycle | RSS (`KB`) | Delta from Warmup (`KB`) |
| ---: | ---: | ---: |
| `0` (Initial) | `21,904` | — |
| `10,000` | `25,056` | `0.0` |
| `20,000` | `25,056` | `0.0` |
| `30,000` | `25,056` | `0.0` |
| `40,000` | `25,056` | `0.0` |
| `50,000` | `25,056` | `0.0` |

### `leaks --atExit` Summary

```text
Process 28966: 1253 nodes malloced for 79 KB
Process 28966: 0 leaks for 0 total leaked bytes.
```

---

## 6. Tier 4 — Live GUI `FSEvents` Hot-Reload (1,000 View Transitions)

- **Target Process:** `/Applications/Puppetiered.app/Contents/MacOS/Puppetiered` (PID `24146`)
- **Workload:** `100` batch cycles installing and removing `10` view manifests in `~/Library/Application Support/Puppetiered/Views/` (`1,000` live directory reload transitions + SwiftUI view-tree diff updates).

### Live Process RSS Progression

| Cycle | View Transitions | RSS (`KB`) | `%MEM` | Delta from Cycle 40 (`KB`) |
| ---: | ---: | ---: | ---: | ---: |
| `0` (Baseline) | `0` | `44,096` | `0.3%` | — |
| `20` | `200` | `46,256` | `0.3%` | — |
| `40` | `400` | `46,272` | `0.3%` | `0.0` |
| `60` | `600` | `46,272` | `0.3%` | `0.0` |
| `80` | `800` | `46,272` | `0.3%` | `0.0` |
| `100` | `1,000` | `46,272` | `0.3%` | `0.0` |

### Live `/usr/bin/leaks` Output

```text
Process:         Puppetiered [24146]
Path:            /Applications/Puppetiered.app/Contents/MacOS/Puppetiered
Load Address:    0x104c50000
Identifier:      com.ilr.puppetiered
Version:         0.9.6 (7)
Code Type:       X86-64
Platform:        macOS

Physical footprint:         45.3M
Physical footprint (peak):  51.4M
Idle exit:                  untracked
----

leaks Report Version: 4.0, multi-thread, stacks
Process 24146: 182760 nodes malloced for 27072 KB
Process 24146: 0 leaks for 0 total leaked bytes.
```

### Live `/usr/bin/footprint` Breakdown

```text
======================================================================
Puppetiered [24146]: 64-bit    Footprint: 45.3 MB (4096 bytes per page)
======================================================================

    Dirty      Clean  Reclaimable    Regions    Category
    -----      -----  -----------    -------    --------
    26 MB        0 B          0 B        178    Malloc (zone Breakdown)
  4072 KB        0 B          0 B         55    IOKit
  1332 KB    2936 KB          0 B        246    __DATA
  1056 KB    5092 KB          0 B        470    __DATA_CONST
   656 KB     308 KB          0 B        119    __DATA_DIRTY
   164 KB        0 B          0 B         10    Stack
   120 KB        0 B          0 B         15    CoreUI image data
    52 KB        0 B          0 B          2    CoreAnimation
    36 KB        0 B          0 B          2    CG image
   ...
    -----      -----  -----------    -------    --------
  33.9 MB    50.8 MB          0 B       2812    TOTAL
```
