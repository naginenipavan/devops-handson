<a id="top"></a>

# 🚀 Linux Project #2: Capture an incident health snapshot

> **Source:** <https://www.escbash.com/topics/linux-project-2>
> **Contents:** 1 hands-on project task (no lessons or quizzes)
> **Skills used:** [09 Processes](09-processes.md) · [11 Networking essentials](11-networking-essentials.md) · [13 Disks and filesystems](13-disks-and-filesystems.md) · [14 Monitoring and performance](14-monitoring-and-performance.md)

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Capture an incident health snapshot](#project) | 🛠️ Project |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="project"></a>

## 🛠️ Project: Capture an incident health snapshot

You're on call at esc bash. An alert fired: the web server feels slow, and the channel needs a clear picture of the machine **before anyone changes anything**. Capture a health snapshot into one file the team can read at a glance.

### Set up the project

```bash
mkdir -p /var/log/escbash-app
echo "service started" > /var/log/escbash-app/app.log
echo "request handled" >> /var/log/escbash-app/app.log
echo "request handled" >> /var/log/escbash-app/app.log
```

### What to capture

Create `/root/health-report.txt` containing, in any order, real output for:

| # | Section | Command from |
|---|---------|--------------|
| 1 | uptime and load average | chapter 14 |
| 2 | memory summary (total, used, available) | chapter 14 |
| 3 | snapshot of running processes | chapters 9 / 14 |
| 4 | disk usage for each filesystem | chapter 13 |
| 5 | largest folders under `/var` | chapter 13 |
| 6 | ports currently listening | chapter 11 |

Append as you go (`>>`) so nothing gets overwritten. The site runs 7 checks.

### Worked answer

> **📝 Note:** The site shows **no solution** for the project tasks, so this is my own answer built from earlier chapters. I haven't run it in the lab; treat it as a starting point.

```bash
uptime                                                > /root/health-report.txt
free -h                                              >> /root/health-report.txt
ps aux                                               >> /root/health-report.txt
df -h                                                >> /root/health-report.txt
du -h --max-depth=1 /var 2>/dev/null | sort -h       >> /root/health-report.txt
ss -tlnp                                             >> /root/health-report.txt
```

- First line uses `>` to create/overwrite the file; every other line uses `>>` to append.
- For process snapshots `top -b -n1 | head -20` is an alternative to `ps aux`.
- Optionally add a header before each section, e.g. `echo "== DISK ==" >> /root/health-report.txt`, so the report is easier to scan.

Verify:

```bash
wc -l /root/health-report.txt
grep -E 'load average|Mem:|Filesystem|LISTEN' /root/health-report.txt
```

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Question | Command |
|----------|---------|
| How busy / how long up? | `uptime` |
| Memory pressure? | `free -h` (read **available**) |
| What's running? | `ps aux` or `top -b -n1 \| head -20` |
| Which filesystem is full? | `df -h` |
| Which folder ate the space? | `du -h --max-depth=1 /var \| sort -h` |
| What's listening? | `ss -tlnp` |

**Incident rule:** capture state first, change nothing, restart last.

<br>

[↑ Back to top](#top)
