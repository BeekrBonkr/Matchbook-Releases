# Matchbook (MBedwars Add-on) — Competitive Match Stats Recorder

Matchbook is an add-on plugin for **MBedwars** (Paper/Purpur 1.21+) that records **per-match Bedwars stats** with a strong focus on **accuracy, reliability, and data integrity**.

It records match stats by taking **authoritative snapshots from the MBedwars API at match start and match end** — **not** by parsing chat.

It supports **YAML** or **MySQL** as a storage backend, with an admin migration command to move data between backends.

---

## Table of contents

- [Key goals](#key-goals)
- [Requirements](#requirements)
- [Install](#install)
- [Configuration](#configuration)
  - [Storage: YAML vs MySQL](#storage-yaml-vs-mysql)
  - [Match capture settings](#match-capture-settings)
  - [Tracked stat keys](#tracked-stat-keys)
  - [Logging](#logging)
- [Commands](#commands)
- [Permissions](#permissions)
- [GUIs](#guis)
  - [Match history GUI](#match-history-gui)
  - [All matches GUI](#all-matches-gui)
  - [Match details GUI](#match-details-gui)
- [Exports (CSV)](#exports-csv)
  - [Single match export](#single-match-export)
  - [Combined export (multi-match)](#combined-export-multi-match)
- [PlaceholderAPI](#placeholderapi)
- [Storage & data formats](#storage--data-formats)
  - [Match IDs](#match-ids)
  - [YAML match document format](#yaml-match-document-format)
  - [MySQL schema](#mysql-schema)
  - [Indexes](#indexes)
- [Migration](#migration)
  - [YAML → MySQL](#yaml--mysql)
  - [MySQL → YAML](#mysql--yaml)
  - [Dry-run](#dry-run)
  - [Common migration pitfalls](#common-migration-pitfalls)
- [Accuracy / integrity notes](#accuracy--integrity-notes)
- [Troubleshooting](#troubleshooting)
  - [“Storage init failed (MYSQL) … Communications link failure”](#storage-init-failed-mysql--communications-link-failure)
  - [Invalid plugin.yml](#invalid-pluginyml)
  - [Tab completion not working](#tab-completion-not-working)
  - [Match appears missing in GUI](#match-appears-missing-in-gui)
- [FAQ](#faq)
- [Roadmap ideas](#roadmap-ideas)
- [Contributing](#contributing)
- [License](#license)

---

## Key goals

Matchbook was built for competitive environments where correctness matters:

- **Accurate per-match stats** (snapshot start/end; diff computed)
- **Human-readable collision-safe match IDs** (`match_id`)
- **Storage backend selectable** (YAML or MySQL)
- **GUI browsing + match detail review**
- **CSV exports** (single match or combined across matches)
- **Minimal assumptions**: avoids relying on chat messages or timing quirks
- **Forensic-friendly**: supports warnings/partial data fallback when needed

---

## Requirements

- **Paper/Purpur 1.21+**
- **MBedwars 5.5.6+** (Matchbook is an MBedwars add-on)
- Optional:
  - **PlaceholderAPI** (for placeholders)

---

## Install

1. Download the latest jar from the releases page.

2. Copy the jar into the MBedwars add-ons folder:
   ```
   plugins/MBedwars/add-ons/Matchbook-<version>.jar
   ```

3. Restart the server.

4. Configure Matchbook:
   - Config file location depends on your project layout, but typically:
     ```
     plugins/MBedwars/add-ons/Matchbook/config.yml
     ```
   - After first boot, Matchbook should generate or load its config.

---

## Configuration

Example `config.yml`:

```yaml
storage:
  # yaml or mysql
  type: yaml

  mysql:
    host: "laredo.bloom.host"
    port: 3306
    database: "db
    username: "username"
    password: "REDACTED"
    # Optional: JDBC URL params (no leading '?')
    params: "useUnicode=true&characterEncoding=utf8&useSSL=true&requireSSL=true&verifyServerCertificate=false&allowPublicKeyRetrieval=true&serverTimezone=UTC"
    table_prefix: "matchbook_"
    pool:
      maximum_pool_size: 10
      minimum_idle: 2
      connection_timeout_ms: 10000
      idle_timeout_ms: 300000
      max_lifetime_ms: 1800000

match:
  enforce_win_loss_from_result: true
  announce_match_code: false

  # Timing controls for snapshot capture
  start_snapshot_delay_ticks: 20
  end_snapshot_delay_ticks: 80
  snapshot_timeout_ticks: 80

  tracked_keys:
    - "bedwars:kills"
    - "bedwars:final_kills"
    - "bedwars:final_deaths"
    - "bedwars:beds_destroyed"
    - "bedwars:wins"
    - "bedwars:loses"
    - "bedwars:beds_lost"

logging:
  debug: false
```

### Storage: YAML vs MySQL

#### YAML mode
- `storage.type: yaml`
- Matches are saved as YAML files under:
  ```
  <addonDataFolder>/matches/<dayFolder>/<filename>.yml
  ```
- Player history is indexed under:
  ```
  <addonDataFolder>/users/<uuid>.yml
  ```

#### MySQL mode
- `storage.type: mysql`
- Matches are stored in SQL, but the authoritative match document is still stored as **YAML text**, byte-for-byte consistent with YAML mode.
- Player history uses a DB index table for fast lookups.

### Match capture settings

- `start_snapshot_delay_ticks`  
  Delay after match start event before capturing the “start snapshot”. Helps ensure MBedwars stats are initialized.

- `end_snapshot_delay_ticks`  
  Delay after match end event before capturing the “end snapshot”. Helps ensure MBedwars has written final values.

- `snapshot_timeout_ticks`  
  If a snapshot request stalls (API callback doesn’t return), Matchbook will complete with partial data and log a warning.

### Tracked stat keys

`tracked_keys` controls which MBedwars stat keys are recorded and exported.

Common keys:
- `bedwars:kills`
- `bedwars:final_kills`
- `bedwars:final_deaths`
- `bedwars:beds_destroyed`
- `bedwars:beds_lost`
- `bedwars:wins`
- `bedwars:loses`

Add more keys if MBedwars exposes them; Matchbook stores whatever it retrieves.

### Logging

- `logging.debug: true` enables more verbose output.
- Matchbook also emits warnings if:
  - snapshot timeouts occur
  - matches are saved in “ABORTED” state due to shutdown/flush
  - match documents appear inconsistent

---

## Commands

Root command: `/matchbook` (alias: `/mb`)

### Player / coach commands

- `/mb help`  
  Shows only the commands you can use based on permissions.

- `/mb matches`  
  Opens a GUI showing **your** match history (most recent first).

- `/mb all`  
  Opens a GUI showing **all matches** on the server (most recent first).

- `/mb view <matchcode>`  
  Opens the match details GUI for the given match code.

- `/mb export <matchcode[, matchcode...]>`  
  Exports CSV:
  - single match export to `exports/<matchcode>.csv`
  - multi-match export to `exports/combined_<timestamp>.csv` (aggregated by UUID)

### Admin commands

- `/mb migrate yaml2mysql [--dry-run]`
- `/mb migrate mysql2yaml [--dry-run]`

Migration moves stored matches between backends. See [Migration](#migration).

---

## Permissions

Matchbook uses a **per-command permission model** with grouping.

### Group permissions

- `mb.command.use`  
  Allows using `/mb` at all.  
  Default: `true`

- `mb.command.default`  
  Grants common commands.  
  Default: `true`  
  Includes:
  - `mb.command.matches`
  - `mb.command.all`
  - `mb.command.view`
  - `mb.command.export`
  - `mb.command.help`

- `mb.command.admin`  
  Grants admin commands.  
  Default: `op`  
  Includes:
  - `mb.command.migrate`

### Per-command permissions

- `mb.command.matches`
- `mb.command.all`
- `mb.command.view`
- `mb.command.export`
- `mb.command.help`
- `mb.command.migrate`

### Legacy permissions (backwards compatibility)

Matchbook also accepts older nodes:
- `matchbook.admin`
- `matchbook.matches`
- `matchbook.migrate`

If you’re using LuckPerms, you can assign either the new nodes or legacy nodes.

---

## GUIs

### Match history GUI

Command:
- `/mb matches`

Shows:
- match list for the player (most recent first)
- click a match → open details GUI

### All matches GUI

Command:
- `/mb all`

Shows:
- all matches saved on the server (most recent first)
- click a match → open details GUI

### Match details GUI

Commands:
- click a match in a GUI
- `/mb view <matchcode>`

Shows:
- match header (arena, start/end time, result)
- per-player stats displayed using **colored wool** by team
- pagination for large matches

---

## Exports (CSV)

### Single match export

```bash
/mb export <matchcode>
```

Saves:
```
<addonDataFolder>/exports/<matchcode>.csv
```

Includes:
- per-player row
- UUID, username, team
- tracked stat diffs

### Combined export (multi-match)

```bash
/mb export <match1>,<match2>,<match3>
```

Saves something like:
```
<addonDataFolder>/exports/combined_<timestamp>.csv
```

Combined exports:
- aggregate by **UUID**
- sums each tracked stat diff across the selected matches

This is useful for:
- tournament series totals
- coach review across multiple rounds
- event recap reporting

---

## PlaceholderAPI

If PlaceholderAPI is installed, Matchbook registers placeholders.

Supported placeholders:

- `%matchbook_match_id%`
- `%matchbook_match_code%`

Both return the **current match’s internal match ID** for a player in a tracked match, otherwise an empty string.

Older variations may also be accepted depending on build:
- `%matchbook_matchid%`
- `%matchbook_matchcode%`

---

## Storage & data formats

### Match IDs

Each match has a `match_id` that is:
- human-readable
- collision-safe (intended to be unique)
- stored inside match YAML and MySQL rows

Matchbook GUIs, exports, and placeholders all use this ID.

### YAML match document format

In YAML mode, Matchbook saves each match as a YAML file with this general shape:

```yaml
match:
  match_id: "<id>"
  start_unix: 1700000000
  end_unix: 1700000123
  arena: "MyArena"
  result: "RED" # or "BLUE", "TIE", "ABORTED", etc.
  start_snapshot_taken_unix: 1700000001
  participants:
    - "uuid-1"
    - "uuid-2"

players:
  <uuid-1>:
    username: "PlayerOne"
    team: "RED"
    start_taken_unix: 1700000001
    start:
      bedwars:kills: 10
    end:
      bedwars:kills: 12
    diff:
      bedwars:kills: 2
```

In MySQL mode, the **exact same YAML text** is stored in a `LONGTEXT` column.

### MySQL schema

Matchbook uses two tables (with a configurable prefix):

- `${prefix}matches` — match metadata + YAML document text
- `${prefix}player_matches` — player → match index for history lookups

---

## Migration

Matchbook supports migrating match data between YAML and MySQL.

### YAML → MySQL

```bash
/mb migrate yaml2mysql
```

### MySQL → YAML

```bash
/mb migrate mysql2yaml
```

### Dry-run

```bash
/mb migrate yaml2mysql --dry-run
/mb migrate mysql2yaml --dry-run
```

---

## Troubleshooting

### Storage init failed (MYSQL) … Communications link failure

Usually one of:
- DB host/port blocked
- TLS required but `useSSL=false`
- config keys not being read due to incorrect path

Try:
- verify connectivity (`nc -vz host 3306`)
- enable SSL in params
- check startup logs for printed JDBC URL

### Invalid plugin.yml

YAML needs correct indentation and quoting for strings with colons:
```yaml
description: "Legacy: Allows ..."
```

---

## Roadmap ideas

Useful competitive features:
- match “reviewed” state + referee notes
- integrity hashes/signatures per match document
- webhook to Discord for match completion
- team summaries (per match, per series)
- coach dashboard exports (last N matches)

---

## License

TBD
