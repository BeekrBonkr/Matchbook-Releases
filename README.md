# Matchbook (MBedwars Add-on)

Matchbook is an add-on for **MBedwars (Paper/Purpur 1.21+)** that records **per-match Bedwars stats** for competitive play and esports review.  
It tracks **kills, deaths, and bed breaks** using authoritative MBedwars events/API (no chat parsing).

Matchbook generates a short **match code** (format: `XXXX-XXXX`) so players/referees can easily write it down and submit it for scoring.

---

## Requirements
- **Paper/Purpur 1.21+**
- **MBedwars 5.5.6+**
- Optional: **PlaceholderAPI** (placeholders)

---

## Install
1. Download the latest jar from Releases.
2. Put it here:
   ```
   plugins/MBedwars/add-ons/Matchbook-<version>.jar
   ```
3. Restart the server to generate config.

Config location (typical):
```
plugins/MBedwars/add-ons/Matchbook/config.yml
```

---

## What gets saved
A match is only saved if at least one of these happened:
- **Kills**
- **Deaths**
- **Bed breaks**

This prevents “empty” matches from cluttering storage.

---

## Commands
- `/mb matches` — open your match history GUI
- `/mb all` — open all matches GUI
- `/mb view <matchcode>` — view a specific match
- `/mb export <matchcode[,matchcode...]>` — export CSV (single or combined)
- `/mb migrate yaml2mysql [--dry-run]` — migrate YAML → MySQL (admin)
- `/mb migrate mysql2yaml [--dry-run]` — migrate MySQL → YAML (admin)

Aliases: `/matchbook`, `/mb`

---

## Storage
Matchbook supports **YAML** or **MySQL**.

### YAML
- Saves match files under an addon data folder (matches + user index).
- Simple and portable.

### MySQL
- Stores the match document as YAML text in SQL for consistency with YAML mode.
- Includes a player→match index for fast history lookups.

---

## Example config
```yml
storage:
  type: yaml  # yaml or mysql

  mysql:
    host: "localhost"
    port: 3306
    database: "matchbook"
    username: "root"
    password: "password"
    params: "useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=UTC"
    table_prefix: "matchbook_"

match:
  announce_match_code: false

# Matches only save when these stats occur:
tracked_activity:
  count_kills: true
  count_deaths: true
  count_bed_breaks: true

logging:
  debug: false
```

---

## PlaceholderAPI
If PlaceholderAPI is installed:
- `%matchbook_match_code%` — your current match code (if in a tracked match)

---

## Notes for competitive use
- Match codes are short and easy to submit (`XXXX-XXXX`).
- Matchbook avoids cross-match stat contamination (players leaving and joining new matches won’t affect prior match records).
- Designed for review workflows: history GUI + match details + CSV export.

---

## Support / Troubleshooting
If you run into issues, grab:
- your server `latest.log`
- Matchbook config
- MBedwars version number

and include a quick repro (what you clicked/typed, what arena, what happened).
