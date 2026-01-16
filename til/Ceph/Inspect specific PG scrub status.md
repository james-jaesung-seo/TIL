---
title: "[Snippet] Inspect Specific PG Scrub Status & Schedule"
public: true
---

## 1. Overview
This script performs a detailed inspection of a **specific Placement Group (PG)**. 
It combines data from `ceph pg dump` (general stats) and `ceph pg query` (internal state) to provide a comprehensive view of:
1.  **Scrubbing Progress:** Calculated percentage of scrubbed objects.
2.  **Schedule:** Exact timestamp or queue status for the next scrub.
3.  **Blocking Status:** Checks if the scrub is active or waiting on specific OSDs (useful for debugging stuck scrubbing).

> **Note:** This script requires `jq` and `sudo` privileges.

## 2. One-Liner Command
Run the following command and enter the target **PG ID** (e.g., `40.bec`) when prompted.

```bash
date; read -p "Enter PG ID: " pgid && \
json=$(sudo ceph pg dump pgs --format=json) && \
dump_info=$(echo "$json" | jq -r --arg pgid "$pgid" '.pg_stats[] | select(.pgid == $pgid) | ["--- PG Dump Info ---", "PGID: \(.pgid)", "State: \(.state)", "Objects: \(.stat_sum.num_objects // 0)", "Scrubbed: \(.objects_scrubbed // 0)", "Progress: " + (if (.stat_sum.num_objects // 0) > 0 then (((.objects_scrubbed // 0) / (.stat_sum.num_objects // 1)) * 100 | floor | tostring + "%") else "0%" end), "Last Scrub: \(.last_scrub_stamp // "N/A")"] | .[]') && \
echo -e "$dump_info" && \
schedule=$(echo "$json" | jq -r --arg pgid "$pgid" '.pg_stats[] | select(.pgid == $pgid) | .scrub_schedule // "N/A"') && \
if [[ "$schedule" =~ ([0-9]+)s$ ]]; then seconds=${BASH_REMATCH[1]}; printf "Schedule: %s (%dd %dh %dm %ds)\n" "$schedule" "$((seconds/86400))" "$((seconds%86400/3600))" "$((seconds%3600/60))" "$((seconds%60))"; else echo "Schedule: $schedule"; fi && \
echo -e "\n--- PG Query Info ---" && \
sudo ceph pg "$pgid" query --format=json | jq -r '"State: \(.state)\nActing Primary: osd.\(.info.stats.acting_primary)\nScrub Active: \(.scrubber.active)\nWaiting on OSDs: \(.scrubber.waiting_on_whom // [])"'
```

## 3. Output Description
The output is divided into two sections: Dump Info (General Stats) and Query Info (Internal State).
| Section | Field | Description |
| :--- | :--- | :--- |
| **PG Dump Info** | **PGID** | The target Placement Group ID entered (e.g., `40.bec`). |
| | **State** | Current state of the PG (e.g., `active+clean`, `scrubbing`). |
| | **Objects** | Total number of objects stored in this PG. |
| | **Scrubbed** | Number of objects verified so far in the current cycle. |
| | **Progress** | Scrubbing progress percentage (`Scrubbed` / `Objects` * 100). |
| | **Last Scrub** | Timestamp of the last successfully completed scrub session. |
| | **Schedule** | Next scheduled scrub time or queue status (e.g., `queued for scrub`). |
| **PG Query Info** | **Acting Primary** | The OSD ID currently responsible for coordinating the scrub. |
| | **Scrub Active** | `true` if the scrubber is actively running, `false` otherwise. |
| | **Waiting on OSDs** | List of OSDs that are delaying the scrub (Key for debugging stuck scrubs). |

## 4. Sample Output
```bash
Fri Jan 16 02:07:20 AM UTC 2026
Enter PG ID: 40.bec
dumped pgs

--- PG Dump Info ---
PGID: 40.bec
State: active+clean
Objects: 151
Scrubbed: 151
Progress: 100%
Last Scrub: 2026-01-14T23:01:55.434783+0000
Schedule: queued for scrub

--- PG Query Info ---
State: active+clean
Acting Primary: osd.72
Scrub Active: false
Waiting on OSDs: []
```
