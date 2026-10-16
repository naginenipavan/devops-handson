<a id="top"></a>

# 💾 13 - Disks and filesystems

> **Source:** <https://www.escbash.com/topics/linux-disks-and-filesystems>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Disks, partitions, and mount points](#lesson-1) | 📖 Lesson |
| 2 | [Disk usage with df and du](#lesson-2) | 📖 Lesson |
| 3 | [Finding large folders with du](#lesson-3) | 📖 Lesson |
| 4 | [Write a disk usage report](#task-1) | 🛠️ Task |
| 5 | [Estimate folder sizes for a backup plan](#task-2) | 🛠️ Task |
| 6 | [Scenario: a full disk at 3 AM](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: Disks, partitions, and mount points

Storage stacks up in layers:

| Layer | What it is | Example |
|-------|------------|---------|
| **Disk** | raw hardware (or virtual disk on a VM) | `/dev/sda`, `/dev/nvme0n1`, `/dev/vda` |
| **Partition** | a slice of a disk | `/dev/sda1` (first partition of `sda`) |
| **Filesystem** | what actually understands files and folders | `ext4`, `xfs`, `btrfs` |
| **Mount point** | the folder in the tree where a filesystem is attached | `/`, `/home`, `/var` |

Writing to `/var/log/app.log`: Linux finds which filesystem holds `/var`, sends the write there, and it lands on that filesystem's partition on the disk.

### `lsblk`: see the whole stack

```bash
lsblk
```

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0  100G  0 disk
├─sda1   8:1    0   99G  0 part /
└─sda2   8:2    0    1G  0 part [SWAP]
```

- `sda` = the disk (100 GB)
- `sda1` = main partition (99 GB) mounted at `/`
- `sda2` = small swap partition

`lsblk` is your first stop to see what storage exists before worrying about how full it is. A small cloud VM often shows a single device like `vda` with one partition (or none), and that's normal.

### `/etc/fstab`, briefly

Tells Linux what to mount where at boot:

```
/dev/sda1  /  ext4  defaults  0  1
```

> **⚠️ Warning:** You'll rarely edit it, but a broken `/etc/fstab` can stop the machine booting cleanly. Check your work twice.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Disk usage with `df` and `du`

Two commands, two different questions.

### `df`: how full each filesystem is

```bash
df -h
```

`-h` = human-readable (K, M, G).

```
Filesystem  Size  Used  Avail  Use%  Mounted on
/dev/sda1    99G   14G    80G   15%  /
tmpfs       2.0G     0   2.0G    0%  /dev/shm
```

| Column | Meaning |
|--------|---------|
| Filesystem | device behind the mount |
| Size / Used / Avail | capacity / taken / free |
| **Use%** | used space as a percentage, **the number you watch** |
| Mounted on | folder the filesystem lives at |

| Use% | Action |
|------|--------|
| > 80% | start planning |
| > 95% | act now |

`tmpfs` rows are memory-backed, usually not worth worrying about. `df` never tells you **which folder** ate the space; use `du`.

### `du`: how big a folder is

```bash
du -sh /var/log      # 28M    /var/log
du -sh /etc          # 5.8M   /etc
du -h /var/log       # drop -s: a size for every subfolder
```

| Flag | Effect |
|------|--------|
| `-s` | summarise the whole path into one number (otherwise a line per subfolder) |
| `-h` | human units |

### Why `df` and `du` sometimes disagree

| Cause | Effect |
|-------|--------|
| a process still holds a **deleted** file open | `df` counts the space, `du` can't see the file |
| **sparse** files | `du` shows logical size, `df` the physical size on disk |

"`df` says 90% full, `du` says 30%" usually means someone deleted a huge log while an app was still writing to it. **Restart the app** and the space frees up.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Finding large folders with `du`

`df` says a filesystem is full; `du` says which folder is at fault.

### The top-level scan

Start at the full mount and drill down one level at a time:

```bash
sudo du -h --max-depth=1 /var 2>/dev/null | sort -h
```

| Part | Role |
|------|------|
| `--max-depth=1` | one line per direct subfolder, not every file |
| `sort -h` | sort by human size, smallest → largest |
| `2>/dev/null` | discard "permission denied" noise |

The biggest folder is at the **bottom**. Repeat inside it:

```bash
sudo du -h --max-depth=1 /var/log 2>/dev/null | sort -h
```

Two or three iterations find the guilty folder or file.

### `ncdu`: interactive alternative

Not installed by default:

```bash
sudo apt install -y ncdu
ncdu /var
```

Arrow keys drill into folders; `d` deletes the selected entry. Faster than repeated `du` during triage. On an offline machine `du | sort -h` is the reliable fallback.

> **⚠️ Warning:** **Never `rm -rf` in `/var` (or anywhere) on a hunch.** Confirm what's safe first. Active processes are writing right now, and deleting those files triggers the "df and du disagree" mystery without actually freeing space.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Write a disk usage report

Produce a single snapshot at `/root/disk-report.txt` containing, in any order:

1. a block-device listing
2. human-readable filesystem usage (including the root filesystem row)
3. a per-subfolder size breakdown of `/var`, largest at the bottom

**Solution (from the site):**

```bash
lsblk > /root/disk-report.txt
df -h >> /root/disk-report.txt
du -h --max-depth=1 /var 2>/dev/null | sort -h >> /root/disk-report.txt
```

`>` creates the file with the first section; `>>` appends the rest.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Estimate folder sizes for a backup plan

Before a backup, check how big the folders are. Produce `/root/size-estimate.txt` with one line per folder (`du -sh` output: `SIZE<whitespace>PATH`) for:

- `/etc` (system configuration)
- `/root` (root's home)
- `/var/log` (log data)

**Solution (from the site):**

```bash
du -sh /etc 2>/dev/null > /root/size-estimate.txt
du -sh /root 2>/dev/null >> /root/size-estimate.txt
du -sh /var/log 2>/dev/null >> /root/size-estimate.txt
```

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: a full disk at 3 AM

**Q:** An alert wakes you at 3 AM: `/var` is 97% full on a production API server and requests are starting to error. What's your first move?

- [x] **`sudo du -h --max-depth=1 /var 2>/dev/null | sort -h`**
- [ ] `sudo find /var/log -name '*.log' -size +100M`
- [ ] `sudo rm -rf /var/log/*.log`
- [ ] Reboot the server to reset it

**Why:** **diagnose before deleting.** One command maps which subfolder ate the space, with the culprit at the bottom. Then you know whether it's logs, cache, database files or a core dump, and can pick a safe fix.

| Option | Problem |
|--------|---------|
| `find` for big logs | assumes an answer you haven't verified |
| `rm -rf` logs | files held open don't free space until the writer restarts (the df-vs-du trap) **and** you destroy evidence; truncating in place with `>` is the safe variant |
| reboot | frees nothing, the same files are still there |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `lsblk` | disks → partitions → mount points |
| `df -h` | free space per filesystem (watch **Use%**) |
| `du -sh path` | total size of a folder |
| `du -h path` | size of every subfolder |
| `du -h --max-depth=1 path \| sort -h` | one-level breakdown, biggest last |
| `2>/dev/null` | hide permission-denied noise |
| `ncdu path` | interactive explorer (`apt install ncdu`) |
| `/etc/fstab` | boot-time mounts (edit with care) |

**Full-disk playbook:** `df -h` (which filesystem?) → `du --max-depth=1 | sort -h` (which folder?) → repeat inside the biggest → fix safely. If `df` ≫ `du`, restart the app holding a deleted file.

<br>

[↑ Back to top](#top)
