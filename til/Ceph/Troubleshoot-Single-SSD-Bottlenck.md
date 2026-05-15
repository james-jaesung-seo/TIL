---
title: "[Troubleshooting] Single SATA SSD Bottleneck Cascading into Cluster-Wide Slow OSDs"
public: true
---

## Problem

A single SATA SSD hitting 100% utilization (`%util`) can cascade into a cluster-wide performance issue. Because SATA SSDs have a limited NCQ depth of 32, once the I/O queue saturates (`aqu-sz` > 200), read latency (`r_await`) spikes to 200–400ms. Since Ceph routes all client I/O through the **Primary OSD**, any PG whose Primary lives on that overloaded disk becomes slow — and because `commit_latency` includes waiting for **replica acknowledgments**, even OSDs on healthy disks appear slow if they share a replica set with the bottlenecked OSD.

### How to Detect

```bash
# Cluster-wide disk utilization snapshot (via SSH from a jump host)
# Sort by %util descending, filter disks above 10%
iostat -xp 5 2 | awk '/^sd[a-z]+ / && NR>30 {print}' | sort -k22,22nr

# Map high-latency OSDs to physical disks
sudo ceph osd perf | awk '$2>=50' | sort -k2,2nr
sudo ceph device ls | grep "osd.51"
```

Key indicators of a single-disk bottleneck:
- `aqu-sz` > 100 on one disk while others are near 0
- `r_await` > 100ms (normal SSD: < 1ms)
- `%util` pinned at 100%
- `ceph osd perf` shows rotating high-latency OSDs (different OSD each check)

## Immediate Response

### Option A: Lower Primary Affinity (safest, no data movement)

```bash
# Remove the overloaded OSD from Primary role
# Client reads shift to another replica instantly — zero data migration
sudo ceph osd primary-affinity osd.51 0

# Restore after the situation resolves
sudo ceph osd primary-affinity osd.51 1
```

**Primary Affinity** controls which OSDs are eligible to be the Primary for a PG.
- All client reads and writes go through the Primary OSD only
- Scrub, deep scrub, and recovery are also coordinated by the Primary
- Setting affinity to `0` offloads **all** of these from the target OSD
- The OSD remains in the acting set as a replica (still receives write replication)
- No data movement occurs — only metadata changes

### Option B: Mark OSD Out (heavier, triggers backfill)

```bash
sudo ceph osd set noout   # prevent automatic out marking first
sudo ceph osd out osd.51

# After resolution
sudo ceph osd in osd.51
sudo ceph osd unset noout
```

> **Warning**: `osd out` triggers PG remapping and backfill. If the OSD is `out` for a long time (weeks+), `osd in` will trigger **another** full backfill back. In that case, `ceph osd purge` + re-add as a new OSD is more efficient.

### Option C: Throttle Recovery / Client QoS

```bash
# Limit recovery impact on client I/O
sudo ceph config set osd osd_max_backfills 1
sudo ceph config set osd osd_recovery_max_active 1
sudo ceph config set osd osd_recovery_sleep 0.5
```

## Root Cause Prevention

### Separate NVMe and SATA SSD with Device Classes

```bash
# Create device-class-specific CRUSH rules
ceph osd crush rule create-replicated nvme_rule default host nvme
ceph osd crush rule create-replicated ssd_rule default host ssd

# Assign pools to specific device classes
ceph osd pool set vm-fast crush_rule nvme_rule
ceph osd pool set vm-bulk crush_rule ssd_rule
```

### Use Primary Affinity to Route I/O to NVMe

In a mixed NVMe + SATA cluster, set SATA OSDs to never be Primary:

```bash
for osd_id in $(ceph osd ls-tree ssd); do
  ceph osd primary-affinity osd.${osd_id} 0
done
```

All client I/O and scrub/recovery coordination goes to NVMe; SATA serves only as replica storage.

### Enable mclock QoS Scheduler

```bash
ceph config set osd osd_op_queue mclock_scheduler
ceph config set osd osd_mclock_profile high_client_ops
```

Prioritizes client I/O over background operations (scrub, recovery) within each OSD.

## The Hidden Cost of `osd out` / `osd in` Round-Trip

Using `osd out` for temporary load relief seems straightforward, but the return path is expensive due to how CRUSH works.

### CRUSH Is Deterministic

CRUSH computes PG placement using a hash of the PG ID and the current OSD map. The same inputs always produce the same output:

```
osd.74 is in:   CRUSH(PG 16.5) → [74, 12, 56]    # osd.74 is Primary
osd.74 is out:  CRUSH(PG 16.5) → [88, 12, 56]    # osd.88 replaces osd.74
osd.74 is in:   CRUSH(PG 16.5) → [74, 12, 56]    # back to osd.74 as Primary
```

### Double Backfill Problem

When `osd.74` is marked `out`:
1. CRUSH removes osd.74 from all PGs
2. New OSDs (e.g., osd.88) are assigned as replacements
3. **Backfill #1**: Remaining replicas (osd.12) → read data → write to osd.88

When `osd.74` is marked `in` again:
1. CRUSH puts osd.74 back into its original PGs
2. osd.88 is removed from those PGs
3. **Backfill #2**: osd.12 → read data → write to osd.74 (stale data from the `out` period must be overwritten)
4. osd.88's now-unnecessary copy is deleted

The data on osd.74 from before the `out` period cannot be reused as-is because writes may have occurred during the `out` window, making osd.74's copy stale.

### Recommended Approach by Duration

| `out` Duration | Recommended Action |
|----------------|-------------------|
| Minutes to hours | `osd in` — incremental recovery via PG log |
| Days | `osd in` — monitor backfill load carefully |
| Weeks+ | `ceph osd purge` + re-add as new OSD |

For extended outages, purging and re-adding avoids the double backfill entirely. The new OSD (with a fresh ID) receives a balanced share of PGs from the whole cluster rather than pulling back every PG that the old OSD owned.

```bash
# Purge the old OSD and re-create on the same disk
ceph osd purge osd.74 --yes-i-really-mean-it
cephadm osd create <hostname> --data /dev/sdh
```

### Why `primary-affinity 0` Avoids All of This

| | `primary-affinity 0` | `osd out` + `osd in` |
|---|---|---|
| Data movement on apply | **None** | Full backfill |
| Data movement on revert | **None** | Full backfill (again) |
| Time to take effect | Seconds | Minutes to hours |
| Risk | None | Backfill can saturate cluster |
| Use case | Temporary load relief | Permanent removal / disk replacement |

## Key Takeaways

- SATA SSD NCQ depth (32) is the fundamental bottleneck — once `aqu-sz` exceeds this, latency explodes
- `ceph osd perf` reports an EWMA (exponentially weighted moving average), not instantaneous latency — it takes minutes to reflect current state; use `iostat` for real-time diagnosis
- `commit_latency` includes replica wait time — a single slow disk can make unrelated OSDs appear slow
- `primary-affinity 0` is the fastest and safest emergency response: no data movement, instant effect, trivially reversible
- For mixed SSD/NVMe clusters, device class separation + primary affinity is the best long-term strategy
