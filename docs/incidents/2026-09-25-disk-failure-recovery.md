# 🧯 Incident report: Disk 2 failure and Immich database recovery

| | |
|---|---|
| **Date of recovery** | 25 September 2026 |
| **System** | Unraid NAS (home lab) |
| **Severity** | Medium: no data lost, one service (Immich) unavailable |
| **Status** | ✅ Resolved. Disk 2 replacement is deferred and tracked as a follow-up action |
| **Author** | Damian |

---

## 📋 Summary

Data Disk 2 failed with a controller/firmware fault and was removed from the array. Parity was then resynced across the six healthy disks. When the server came back up, the array started cleanly. Unraid automatically repaired the two NTFS data disks as it mounted them, and every disk passed SMART. All shares were available, and every Docker container except **immich-postgres** started. **immich-server** depends on the database, so it failed too.

Postgres was failing because it could not read its own config file inside the container: `Permission denied` on `/etc/postgresql/postgresql.conf`. The database data itself was intact. The fault was in the container's **writable layer**, which Docker stores on NTFS Disk 1. An unclean shutdown, followed by the automatic NTFS repair on mount, damaged that layer.

I backed up the database directory first, then recreated the container from its existing Unraid template. That gave it a fresh writable layer with the same image, settings and data mapping. Both Immich containers came back healthy. The image did not need re-pulling and the database did not need repairing.

The underlying risk is that Docker's system storage and appdata sit on an NTFS disk. That migration is now a high-priority follow-up.

---

## 🖥️ Environment

| Component | Detail |
|---|---|
| OS | Unraid 7.3.2 |
| CPU / RAM | Intel Core i5-4460 / 8 GB DDR3 |
| Array | 1 × 1 TB parity + data disks, all 1 TB Seagate 2.5" drives |
| Filesystems | Disks 1 and 3: NTFS · Disks 4–7: XFS |
| Docker storage | Docker system storage and appdata both on **Disk 1 (NTFS)** |
| Networking | Containers on a custom bridge network |

**Containers (11, eight services):** Immich (server, machine-learning, redis, postgres), Jellyfin, Pi-hole, Nginx Proxy Manager, Grafana, Prometheus, Tailscale, Uptime Kuma.

---

## 🕒 Timeline

### Background: earlier failure

| When | Event |
|---|---|
| 31 Jul | immich-postgres first runs from its current data directory |
| 22 Aug | Last scheduled parity check completes with **0 errors** |
| 20 Sep | immich-postgres logs a clean `database system is shut down` (its last successful run) |
| Before 25 Sep | Disk 2 fails with a controller/firmware fault (see [Diagnosis](#-diagnosis)) |
| Before 25 Sep | Disk 2 is removed with **New Config**, keeping the other assignments. Parity is resynced across the six healthy disks |
| Before 25 Sep | The server has an unclean shutdown, so the NTFS disks are left marked dirty |

### 25 September: recovery session

| Step | Action / observation |
|---|---|
| 1 | Server powered on. Array stopped and showing **"Configuration valid"**. Disk 2 slot unassigned. All drives detected at 29–31 °C |
| 2 | Checked for the replacement drive. The Disk 2 dropdown offered no new device. The syslog was logging `problem getting id` about once a second for the old Disk 2 device, so the dead drive is **still physically connected** |
| 3 | **Decision:** defer the Disk 2 replacement and run with the slot empty. This is safe because parity was already resynced without it |
| 4 | Array started in Normal mode. Unraid ran `ntfsfix -d` on Disk 1 and then Disk 3 before mounting them. Both disks mounted, parity was updated alongside the writes, and no parity check was triggered |
| 5 | Health checks passed: SMART on all 7 disks, shares, notifications and schedules (see [Verification](#-verification)) |
| 6 | Containers started. **immich-postgres** exited with code 1, and **immich-server** failed because it depends on the database |
| 7 | Read the Postgres logs, inspected the container's mounts and permissions, and checked where the filesystem was mounted |
| 8 | **Backup taken:** copied the full Postgres data directory (442 MB) to an XFS disk with `cp -a` and saved the old container log |
| 9 | Inspected the image in a throwaway container. The image was healthy, so the fault was in the old container's writable layer |
| 10 | Recreated immich-postgres from its Unraid template |
| 11 | immich-postgres and immich-server both reported healthy after several minutes. Incident closed |

---

## 🔎 Symptoms

| Symptom | Where seen |
|---|---|
| Disk 2 slot offered no replacement drive | Unraid Main tab |
| `problem getting id` about once a second for the old Disk 2 device | Syslog |
| `ntfsfix -d` run automatically on Disks 1 and 3 at mount time | Array start log |
| immich-postgres exits with code 1 | Docker tab |
| immich-server fails to start | Docker tab (dependency on the database) |
| `could not open configuration file "/etc/postgresql/postgresql.conf": Permission denied` followed by `FATAL` | immich-postgres container log |

### Disk status after array start

| Slot | Filesystem | State | SMART | Notes |
|---|---|---|---|---|
| Parity | — | ✅ Valid | ✅ Pass | Load-cycle count 170k+, still above the failure threshold |
| Disk 1 | NTFS | ✅ Mounted | ✅ Pass | `ntfsfix -d` on mount. Load-cycle count 170k+. Holds Docker system storage and appdata |
| Disk 2 | — | ⚪ Empty slot | — | Failed drive removed from the array, still physically connected |
| Disk 3 | NTFS | ✅ Mounted | ✅ Pass | `ntfsfix -d` on mount |
| Disk 4 | XFS | ✅ Mounted | ✅ Pass | |
| Disk 5 | XFS | ✅ Mounted | ✅ Pass | |
| Disk 6 | XFS | ✅ Mounted | ✅ Pass | |
| Disk 7 | XFS | ✅ Mounted | ✅ Pass | |

All seven disks showed 0 reallocated, 0 pending, 0 offline-uncorrectable and 0 CRC errors.

### Container status before the fix

| Container | State |
|---|---|
| immich-postgres | ❌ Exited (1) |
| immich-server | ❌ Failed (database unavailable) |
| immich-machine-learning | ✅ Running |
| immich-redis | ✅ Running |
| Jellyfin | ✅ Running |
| Pi-hole | ✅ Running |
| Nginx Proxy Manager | ✅ Running |
| Grafana | ✅ Running |
| Prometheus | ✅ Running |
| Tailscale | ✅ Running |
| Uptime Kuma | ✅ Running |

---

## 🧪 Diagnosis

### Disk 2 (earlier failure)

The evidence pointed to a failure inside the drive itself, not in the cabling or the SATA link:

- **Device-side errors:** `Emask 0x1` / `AC_ERR_DEV`. The drive itself reported the error.
- **`SErr 0x0`:** the SATA link recorded no errors, which rules out the cable and port.
- **Corrupted capacity reporting:** the drive reported the wrong size, a typical sign of controller or firmware failure.
- **Sustained I/O errors at sector 0:** the drive could not reliably read its first sector.

Swapping the cable would not have fixed this. The drive needed to come out.

### Immich database (this session)

I followed the evidence from the error message outwards, one step at a time:

1. **Postgres logs.** The last run on 20 Sep ended with a clean `database system is shut down`. The database was not mid-write when the server went down, so data corruption was unlikely from the start. Today's error named a **config file**, not a data file.
2. **`docker inspect`.** The container had one host mapping: `appdata/immich/postgres` → `/var/lib/postgresql/data`. **`/etc/postgresql` is not mapped from the host.** It lives inside the container.
3. **`ls -ln` on the data directory.** Every file was owned by `999:999` with `drwx------` / `-rw-------`, which is exactly what Postgres expects.
4. **`mount`.** Disk 1 was mounted as `ntfs3`, and Docker's overlayfs container layers were stored under the system share on Disk 1.
5. **Backup before any change.** I copied the data directory (442 MB) to an XFS disk with `cp -a` to preserve ownership and modes, and saved the old container log.
6. **Checked the image in a throwaway container.** In the same image, `/etc/postgresql` exists and is world-writable with the sticky bit (`drwxrwxrwt`), so UID 999 can read it. The directory is empty in the image. The Immich entrypoint script writes `postgresql.conf` at startup (the log shows `Using SSD storage`).

Together these checks showed that the image and data directory were fine. The broken piece was the file the entrypoint had written into **that specific container's writable layer**.

### 🚫 Theories ruled out

I considered each of these theories and ruled it out. They are recorded here because ruling them out shaped the fix.

| Theory | Verdict | Evidence |
|---|---|---|
| "`ntfsfix` is destroying Disk 1's data / losing Windows writes" | **Overstated** | Windows had not touched these disks. Unraid's own NTFS driver had been using them. Emptying the journal only discards writes that were in flight at crash time, and the MFT and MFT mirror both checked OK. I let the mount finish rather than stop the array mid-mount, which could have caused a hang or an unclean stop |
| "Postgres can't run on NTFS because it can't get 700 permissions / UID 999 ownership" | **Ruled out** | The data files had exactly those permissions and that ownership. Postgres had run from that folder from 31 Jul to 20 Sep. The error named a file **inside the container**, not in the data directory |
| "The config file is missing from the image" (first test) | **Bad test, corrected** | My first check ran `cat` on the config with the entrypoint overridden and got `No such file`. That proved nothing, because the entrypoint generates the file at runtime and I had bypassed it. I replaced it with an `ls` of the directory, which showed the permissions that actually mattered |
| "Recreating the container probably won't help" | **Proven wrong** | Recreating the container was the fix |

---

## 🎯 Root cause

**The immich-postgres container's writable layer on NTFS Disk 1 was damaged by the unclean shutdown and the NTFS repair that followed on mount.**

Docker's overlayfs keeps each container's writable layer under the system share, which lives on Disk 1 (NTFS, mounted via `ntfs3`). The Immich entrypoint writes `postgresql.conf` into that layer. After the unclean shutdown and the `ntfsfix -d` pass, Postgres (running as UID 999) could no longer read that file, so it exited with `FATAL`.

**Contributing factor:** Docker system storage and appdata live on an NTFS disk. NTFS under Linux copes badly with unclean shutdowns, so one crash became a service outage.

---

## 🛠️ Fix

1. **Backup first:** copied the full Postgres data directory (442 MB) to an XFS disk with `cp -a` and saved the old container log.
2. **Recreated the container** from the Unraid Docker template: **Edit → make a no-op change → Apply**. Unraid removed the old container and created a new one with:
   - the same image
   - the same settings and environment
   - the same data directory mapping
   - a **fresh writable layer**

The image was not re-pulled and the database was not repaired. No other changes were made.

---

## ✅ Verification

| Check | Result |
|---|---|
| immich-postgres | ✅ Healthy after several minutes |
| immich-server | ✅ Healthy after several minutes |
| All other containers | ✅ Running |
| SMART, all 7 disks | ✅ Pass: 0 reallocated / 0 pending / 0 offline-uncorrectable / 0 CRC |
| Shares | ✅ All available |
| Notifications | ✅ None outstanding |
| Last parity check (22 Aug) | ✅ 0 errors |
| Scheduled parity check | ✅ Monthly, 1st of the month at 02:00, no corrections |
| Docker settings | ✅ "Preserve user defined networks" enabled |
| Autostart | ✅ On for immich-server and Uptime Kuma |

---

## 📝 Lessons learned

- **Keep Docker system storage and appdata on a Linux-native filesystem (XFS or btrfs).** NTFS under Linux copes badly with unclean shutdowns, and here it turned one crash into a container failure.
- **Read the actual error before theorising.** The path in the error message pointed away from the data directory straight away, and following it would have saved time on the NTFS-permissions theory.
- **Take a backup before any change**, even when the fix looks low-risk. The backup took a few seconds and would have made a wrong move recoverable.
- **Check that a diagnostic test tests what you think it does.** Overriding the entrypoint also skipped the step that creates the file I was looking for.
- **Remove failed hardware promptly.** A dead drive left connected floods the syslog and hides useful messages.

---

## 📌 Follow-up actions

| Priority | Action | Status |
|---|---|---|
| 🔴 High | Migrate the appdata and system shares off NTFS Disk 1 onto XFS, then convert Disks 1 and 3 to XFS | ⏳ Open |
| 🔴 High | Scheduled Immich database backup script (logical dump to a separate disk) | ⏳ Open |
| 🟠 Medium | Unplug the failed drive and install the replacement 1 TB Disk 2 | ⏳ Open |
| 🟠 Medium | SMART monitoring script that tracks load-cycle counts (parity and Disk 1 are at 170k+) | ⏳ Open |
| 🟡 Low | Add a 4 TB drive as the new parity disk (parity must be at least as large as the largest data disk), then reuse the old 1 TB parity as a data disk | ⏳ Open |
| 🟡 Low | UPS for clean shutdown on power loss, to prevent the unclean shutdown that started this incident | ⏳ Open (already on roadmap) |
