---
title: "Snippet] Inspect Detailed PG State-Recovery History"
public: true
---

## 1. Overview
This script executes a deep inspection of a specific Placement Group (PG) using the internal query map. Unlike general stats, this command retrieves the detailed JSON representation of the PG to provide insight into:

* **OSD Mapping:** Visualizes Up Set (CRUSH mapping) vs Acting Set (Actual IO handlers).
* **Scrubber Internals:** Checks specific scrubber states and waiting lists.
* **Recovery Context:** Tracks the current recovery machine state (e.g., Started/Primary/Active) and history (Last Clean Epoch).

> **Note:** This script requires `jq` and `sudo` privileges to parse the raw Ceph query output.

## 2. One-Liner Command
Run the following command and enter the target **PG ID** (e.g., `40.bec`) when prompted.

```bash
read -p "Enter PG ID: " pgid && \
sudo ceph pg "$pgid" query --format=json | jq -r '
"--- PG Status ---",
"State:\t\t\(.state)",
"Up OSDs:\t\t\(.up)",
"Acting OSDs:\t\t\(.acting)",
"Primary OSD:\t\tosd.\(.acting_primary)",
"\n--- Scrubber Status ---",
"Scrub Active:\t\t\(.scrubber.active)",
"Scrub State:\t\t\(.scrubber.state)",
"Waiting on OSDs:\t\(.scrubber.waiting_on_whom // [])",
"\n--- Recovery & History ---",
"Recovery State:\t\t\(.recovery_state[0].name // "N/A")",
"  Enter Time:\t\t\(.recovery_state[0].enter_time // "N/A")",
"Object Count:\t\t\(.info.stats.stat_sum.num_objects)",
"Last Clean Epoch:\t\(.last_clean_epoch // "N/A")"
'
```

## 3. Output Description
The output is categorized into three logical sections: General Status, Scrubber details, and Recovery History.

| Section | Field | Description |
| :--- | :--- | :--- |
| **PG Status** | **State** | Current state of the PG (e.g., `active+clean`, `peering`). |
| | **Up OSDs** | The set of OSDs selected by CRUSH map for this PG. |
| | **Acting OSDs** | The set of OSDs actually handling I/O for this PG (should match Up set in healthy clusters). |
| | **Primary OSD** | The OSD responsible for coordination and replication. |
| **Scrubber Status** | **Scrub Active** | `true` if a scrub is currently executing. |
| | **Scrub State** | Internal state of the scrubber state machine. |
| | **Waiting on OSDs** | List of OSDs preventing the scrub from proceeding. |
| **Recovery & History** | **Recovery State** | Current state in the Peering state machine (e.g., `Started/Primary/Active`). |
| | **Enter Time** | Timestamp when the PG entered the current recovery state. |
| | **Object Count** | Total number of objects known to the PG metadata. |
| | **Last Clean Epoch** | The last map epoch version where the PG was fully clean. |

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
