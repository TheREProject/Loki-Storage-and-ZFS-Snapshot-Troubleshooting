# Loki Storage and ZFS Snapshot Troubleshooting

## Overview

Loki appeared to be consuming several GB of storage even after reducing log retention from one week to three days.

The investigation initially focused on Loki's chunk storage and retention configuration, but the actual cause turned out to be **ZFS snapshots inherited through pool replication**.

This was a useful example of troubleshooting storage utilization across multiple layers:

**Loki → Docker → filesystem → ZFS dataset → ZFS snapshots → virtual disk**

---

## Environment

- TrueNAS
- ZFS storage pool
- Docker container running Grafana Loki
- Loki version: `3.7.7`
- Loki storage: filesystem
- Loki schema: TSDB v13
- Replication factor: `1`
- Tenant: `fake`
- Original retention: `7 days`
- Temporary test retention: `72h`
- Final retention: `7 days`

Docker mounts:

```text
/mnt/pool/Loki/config → /etc/loki
/mnt/pool/Loki/data   → /loki
```

---

## Initial Problem

The Loki chunks directory was several GB in size and contained logs/chunks that appeared to be months old.

Retention was reduced to three days:

```yaml
limits_config:
  retention_period: 72h
```

However, the older data appeared to remain.

The initial assumption was that Loki retention or compaction was not functioning correctly.

---

## Loki Configuration Investigation

The original configuration used paths such as:

```yaml
common:
  path_prefix: /Loki

  storage:
    filesystem:
      chunks_directory: /data/chunks
      rules_directory: /data/rules

compactor:
  working_directory: /data/compactor
```

However, `/data` was not a persistent Docker mount.

The container had:

```text
/mnt/pool/Loki/data → /loki
```

The storage paths were therefore changed to use the persistent `/loki` mount:

```yaml
common:
  path_prefix: /loki

  replication_factor: 1

  ring:
    kvstore:
      store: inmemory

  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  delete_request_store: filesystem
  retention_delete_delay: 2h

limits_config:
  allow_structured_metadata: false
  max_entries_limit_per_query: 50000
  max_query_series: 100000
  retention_period: 72h
```

---

## Verifying Loki Retention

Loki logs showed the compactor running and applying retention.

Metrics included:

```text
loki_compactor_apply_retention_operation_total{status="success"} 7
```

After restarting Loki, the counter reset and subsequently reached:

```text
loki_compactor_apply_retention_operation_total{status="success"} 6
```

This confirmed that retention operations were succeeding.

Other useful metrics showed:

```text
loki_compactor_oldest_pending_delete_request_age_seconds 0

loki_compactor_pending_delete_requests_count 0
```

There were no outstanding deletion requests.

### Conclusion

Loki retention was functioning correctly.

---

## Checking Actual Filesystem Usage

The persistent Loki chunks directory was checked with:

```bash
du -sh /mnt/pool/Loki/data/chunks
```

After the configuration change and retention processing, the directory was **much smaller than it had previously been**.

This was the first major indication that Loki itself was successfully deleting old data.

However, the virtual disk was still reporting substantially more space in use.

---

## Investigating the Storage Layer

The entire dataset was checked:

```bash
du -sh /mnt/pool
```

Result:

```text
~8.7G
```

The virtual disk, however, was:

```text
64 GB
```

This created a significant discrepancy.

A reboot of TrueNAS did not change the reported disk usage, ruling out simple stale runtime accounting.

---

## Investigating ZFS

ZFS dataset accounting was checked:

```bash
zfs list -o name,used,avail,refer,usedbysnapshots,usedbydataset
```

The Loki dataset showed approximately:

```text
pool/Loki

USED               37.7G
AVAIL               9.01G
REFER                4.84G
USEDBYSNAPSHOTS     32.8G
USEDBYDATASET        4.8G
```

This was the critical clue.

The live filesystem contents were relatively small, while a large amount of space was being retained by ZFS snapshots.

More precise accounting later showed:

```text
usedbysnapshots   34.9G
usedbydataset      1.92G
used              36.8G
```

---

## Understanding `du` vs. ZFS

`du` only reports files currently visible in the live filesystem.

ZFS snapshots can retain the blocks belonging to files that have since been deleted.

Therefore:

```text
Loki deletes old chunk
        ↓
File disappears from live filesystem
        ↓
du no longer counts it
        ↓
ZFS snapshot still references the underlying blocks
        ↓
Disk space remains allocated
```

This explained why Loki's retention appeared to work while the disk remained heavily utilized.

---

## Identifying the Snapshots

The relevant snapshots were:

```text
pool/Loki@auto-2026-09-13_10-00
pool/Loki@auto-2026-09-15_19-34
```

Their individual accounting showed large `REFER` values, but relatively modest `USED` values:

```text
Snapshot                         USED     REFER

auto-2026-09-13_10-00           1.38G    33.1G
auto-2026-09-15_19-34           1.81G    34.1G
```

This highlighted an important ZFS concept:

> **`REFER` is not the same thing as additional disk space consumed by the snapshot.**

`USED` is the more useful value when determining how much space can be reclaimed by destroying a snapshot.

---

## Dry-Run Snapshot Deletion

Before deleting anything, ZFS dry runs were performed:

```bash
zfs destroy -nvp pool/Loki@auto-2026-09-13_10-00
```

and:

```bash
zfs destroy -nvp pool/Loki@auto-2026-09-15_19-34
```

The estimated reclaimable space was approximately:

```text
Snapshot 1: 1.39 GB
Snapshot 2: 1.95 GB
```

This demonstrated that the large `REFER` values should not be interpreted as directly reclaimable space.

The snapshots were ultimately removed through the **TrueNAS GUI** because they were no longer needed.

---

## Final Diagnosis

The apparent Loki storage problem was **not caused by Loki retention failing**.

The actual chain of events was:

1. Loki had accumulated a large amount of historical data.
2. The pool was replicated from another virtual disk.
3. ZFS snapshots were brought over as part of that replication.
4. Loki's retention policy subsequently deleted old files from the live filesystem.
5. The ZFS snapshots continued to preserve blocks belonging to the old data.
6. `du` therefore showed only the current, relatively small Loki dataset.
7. ZFS continued reporting tens of GB of allocated snapshot data.
8. Removing the unnecessary snapshots released the previously retained storage.

---

## Result

After deleting the unnecessary snapshots:

- Loki's current storage footprint was small.
- Loki retention was confirmed to be functioning.
- The large unexplained ZFS allocation was resolved.
- The virtual disk regained the expected free space.

The temporary three-day retention test was therefore no longer necessary.

---

## Final Loki Retention

Retention can safely be returned to the desired seven-day period:

```yaml
limits_config:
  retention_period: 168h
```

or:

```yaml
limits_config:
  retention_period: 7d
```

After changing the configuration, restart Loki and verify that the compactor continues successfully applying retention.

---

## Useful Troubleshooting Commands

### Check Loki dataset

```bash
zfs list -o name,used,avail,refer,usedbysnapshots,usedbydataset
```

### Check detailed ZFS properties

```bash
zfs get used,usedbysnapshots,usedbydataset pool/Loki
```

### Check snapshots

```bash
zfs list -t snapshot -r pool/Loki -o name,used,refer,creation
```

### Dry-run snapshot deletion

```bash
zfs destroy -nvp pool/Loki@SNAPSHOT_NAME
```

### Check pool utilization

```bash
zpool list
```

### Check visible filesystem usage

```bash
du -sh /mnt/pool
du -sh /mnt/pool/Loki
```

### Check filesystem-level usage

```bash
df -h
```

### Check Docker container storage

```bash
docker ps -s --filter name=loki
```

### Check Docker storage usage

```bash
docker system df
```

---

## Lessons Learned

### 1. `du` is not the whole storage picture

For ZFS-backed storage:

```text
du ≠ ZFS USED
```

`du` reports current files, while ZFS also accounts for snapshots and other filesystem-level allocations.

### 2. `REFER` is not equivalent to disk consumption

A snapshot can have a very large `REFER` value without consuming that amount of additional disk space.

Use `USED`, `USEDBYSNAPSHOTS`, and dry-run deletion estimates when investigating snapshot storage.

### 3. Check snapshots before blaming application retention

An application can correctly delete old data while ZFS continues retaining the underlying blocks through snapshots.

### 4. Replication can bring historical storage with it

When moving or replicating a ZFS pool/dataset, snapshots can carry historical data that is invisible from the live filesystem but still consumes storage.

### 5. Troubleshoot from the application down to the storage layer

The useful progression was:

```text
Loki
  ↓
Docker
  ↓
Filesystem
  ↓
ZFS dataset
  ↓
ZFS snapshots
  ↓
Virtual disk
```

The key was following the discrepancy instead of assuming the original application was responsible.

---

## Infrastructure / IT Skills Demonstrated

This troubleshooting exercise involved several real-world infrastructure skills:

- Linux filesystem administration
- Docker container administration
- Persistent container storage
- Loki/observability administration
- Log retention and TSDB management
- ZFS dataset administration
- ZFS snapshot management
- Storage capacity analysis
- Virtual disk troubleshooting
- Replication awareness
- Evidence-based infrastructure troubleshooting
