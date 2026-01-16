---
title: "[Snippet] Monitor Active Deep Scrubbing (Real-time)"
public: true
---

## 1. Overview
This script provides real-time monitoring of Placement Groups (PGs) currently undergoing **Deep Scrubbing** within the Ceph cluster. 
It calculates the scrubbing progress (%) based on the number of objects and refreshes the view every 30 seconds..

> **Note:** This script requires the `jq` package. Ensure it is installed on your bastion or monitor node (`sudo apt install jq` or `yum install jq`).

## 2. One-Liner Command
Copy and paste the following command into your terminal to start monitoring.

```bash
while true; do 
  clear
  echo "=== Active Deep Scrubbing Monitor (Refresh: 30s) ==="
  
  # Fetch PG stats in JSON format and parse with jq
  sudo ceph pg dump pgs --format=json | jq -r '
  
  # Define the header columns for the output table
  def header: "PG\tOSDs\tSTATE\tOBJECTS\tSCRUBBED\tTRIMMED\tPROG%\tSCRUB_SINCE\tDUR(s)\tSCHEDULE\tLAST_ACTIVE";

  # 1. Print Header and Filter Data
  [ header ] + (
    .pg_stats
    | map(select(.state | contains("scrubbing+deep"))) 
    | map(
        [
          .pgid,
          (.acting | tostring),
          .state,
          (.stat_sum.num_objects // 0),
          (.objects_scrubbed // 0),
          (.objects_trimmed // 0),
          
          # 2. Calculate Progress Percentage
          # Formula: (objects_scrubbed / total_objects) * 100
          (
            if (.stat_sum.num_objects // 0) > 0
            then (((.objects_scrubbed // 0) / (.stat_sum.num_objects)) * 100 | floor | tostring + "%")
            else "0%"
            end
          ),
          
          # 3. Format Timestamps and Duration
          (.last_scrub_stamp | split("T")[0]),
          (.last_scrub_duration // 0 | tostring),
          (.scrub_schedule // "N/A"),
          ((.last_became_active // "N/A") | split("T")[0])
        ] | @tsv
      )
  )
  | .[]
  ' | column -t -s $'\t'
  
  sleep 30
done
```

## 3. Output Description
| Column | Description |
| :--- | :--- |
| **PG** | Placement Group ID (e.g., `1.1f`) |
| **OSDs** | List of OSD IDs currently acting for this PG (Acting Set) |
| **STATE** | Current state of the PG (e.g., `active+clean+scrubbing+deep`) |
| **OBJECTS** | Total number of objects stored in the PG |
| **SCRUBBED** | Number of objects verified so far in the current cycle |
| **TRIMMED** | Number of objects removed during snapshot trimming |
| **PROG%** | Scrubbing progress percentage (`SCRUBBED` / `OBJECTS`) |
| **SCRUB_SINCE** | Timestamp of the last completed scrub execution |
| **DUR(s)** | Duration of the last scrub session in seconds |
| **SCHEDULE** | Scheduled timestamp for the next scrubbing |
| **LAST_BECAME_ACTIVE** | Timestamp when the PG last entered the `active` state |

## 4. Output
```bash
PG      OSDs                STATE                        OBJECTS  SCRUBBED  TRIMMED  PROG%  SCRUB_SINCE  DUR(s)  SCHEDULE                  LAST_BECAME_ACTIVE
42.1bb  [163,64,124]        active+clean+scrubbing+deep  7716     3772      1741     48%    2026-01-08   2641    deep scrubbing for 1167s  2025-10-23
25.bf   [145,8,87]          active+clean+scrubbing+deep  13855    12380     0        89%    2026-01-08   6566    deep scrubbing for 5514s  2025-09-06
25.9f   [113,13,121]        active+clean+scrubbing+deep  14060    10827     1        77%    2026-01-08   4189    deep scrubbing for 5202s  2025-09-06
44.b    [69,92,112]         active+clean+scrubbing+deep  18784    16169     0        86%    2026-01-08   20      deep scrubbing for 7684s  2025-09-06
30.38   [129,24,102,59,34]  active+clean+scrubbing+deep  52868    13259     0        25%    2026-01-08   48      deep scrubbing for 1680s  2025-10-21
30.44   [76,79,133,84,25]   active+clean+scrubbing+deep  52745    51524     0        97%    2026-01-08   24      deep scrubbing for 9059s  2025-10-21
25.5e   [6,104,81]          active+clean+scrubbing+deep  14085    10887     18       77%    2026-01-08   7074    deep scrubbing for 5388s  2025-10-21
30.70   [55,39,152,125,30]  active+clean+scrubbing+deep  52554    35759     0        68%    2026-01-08   44      deep scrubbing for 7142s  2025-09-06
30.7d   [105,31,88,8,128]   active+clean+scrubbing+deep  52765    30299     0        57%    2026-01-08   57      deep scrubbing for 6803s  2025-10-21
```