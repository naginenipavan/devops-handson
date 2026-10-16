<a id="top"></a>

# ⏰ 15 - Scheduled jobs

> **Source:** <https://www.escbash.com/topics/linux-scheduled-jobs>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [cron format and crontab](#lesson-1) | 📖 Lesson |
| 2 | [System-wide cron jobs](#lesson-2) | 📖 Lesson |
| 3 | [systemd timers](#lesson-3) | 📖 Lesson |
| 4 | [Schedule a cleanup with cron](#task-1) | 🛠️ Task |
| 5 | [Send cron output to a log file](#task-2) | 🛠️ Task |
| 6 | [Scenario: a cron job that isn't running](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: cron format and crontab

`cron` is the classic Linux scheduler. Every DevOps engineer meets it in their first month.

### The five fields

```
┌───── minute        (0 - 59)
│ ┌───── hour          (0 - 23)
│ │ ┌───── day of month  (1 - 31)
│ │ │ ┌───── month         (1 - 12)
│ │ │ │ ┌───── day of week   (0 - 6, Sunday is 0 and 7)
│ │ │ │ │
* * * * *  command to run
```

`*` = "every value".

| Schedule | Meaning |
|----------|---------|
| `0 3 * * *` | every day at 03:00 |
| `*/5 * * * *` | every 5 minutes |
| `0 * * * *` | top of every hour |
| `0 0 * * 0` | every Sunday at midnight |
| `30 2 1 * *` | 02:30 on the 1st of every month |

Stuck? <https://crontab.guru> is a good sanity check.

### Reading a line field by field

```
30 2 1 * * /root/backup.sh
```

`30` minute 30 · `2` hour 2 (→ 02:30) · `1` day 1 of the month · `*` every month · `*` every weekday → run `/root/backup.sh` at **02:30 on the 1st of every month**.

### Editing your crontab

Each user has their own crontab; never edit the file directly.

```bash
crontab -e     # open in $EDITOR (nano/vim)
crontab -l     # list
crontab -r     # remove your ENTIRE crontab (careful!)
```

Save and quit and cron picks it up automatically; no reload needed.

### ⚠️ cron runs with a bare environment

cron does **not** load `.bashrc` / `.profile`, and `PATH` is very short (usually `/usr/bin:/bin`). A command that works by hand can fail under cron because it can't find `docker`, `python3`, `aws`, `node` (often in `/usr/local/bin`), and you never see the "command not found".

Two fixes:

1. Use **absolute paths** in your script: `/usr/local/bin/docker`.
2. Or set `PATH` on the first line of the crontab:

```
PATH=/usr/local/bin:/usr/bin:/bin
```

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: System-wide cron jobs

`crontab -e` is personal. System jobs (log rotation, updates, backups) live under `/etc/`.

### The convenience folders

Drop an executable script in one and it runs on that cadence:

```
/etc/cron.hourly/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/
```

```bash
ls /etc/cron.daily/      # on Ubuntu: apt-compat, logrotate, ...
```

> **⚠️ Warning:** Two rules trip people up:
> 1. The script must be **executable** (`chmod`), or cron skips it.
> 2. The filename must have **no dot**. These folders run through `run-parts`, which ignores files with an extension. Name it `backup`, not `backup.sh`.

### `/etc/cron.d/`: full-schedule files

For a custom schedule. Each line has **one extra field, the user**, between schedule and command:

```
# /etc/cron.d/nightly-backup
30 2 * * * root /usr/local/bin/backup.sh
```

### `/etc/crontab`: the master crontab

Same format as `/etc/cron.d/*`. Modern practice: leave it alone and put jobs in `/etc/cron.d/`, **one file per purpose** (easier to review and remove).

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: systemd timers

cron works, but has no proper logging, no dependency handling, and silently discards output. **systemd timers** are the modern alternative.

### The pair: a service and a timer

A timer never runs a command directly; it triggers a matching **service**. Two files side by side, same name:

```
/etc/systemd/system/backup.service
/etc/systemd/system/backup.timer
```

### Build one end to end

The script:

```bash
echo '#!/bin/bash' > /root/backup.sh
echo 'date >> /root/backup.log' >> /root/backup.sh
chmod +x /root/backup.sh
```

`/etc/systemd/system/backup.service`: `Type=oneshot` because it runs once and exits:

```ini
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
ExecStart=/root/backup.sh
```

`/etc/systemd/system/backup.timer`:

```ini
[Unit]
Description=Run backup daily at 02:30

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

- `OnCalendar=` sets the schedule.
- `Persistent=true` runs a missed job (machine was off at 02:30) as soon as it boots.

### Reading `OnCalendar`

Format: `weekday year-month-day hour:minute:second`; `*` = every. So `*-*-* 02:30:00` = any year/month/day at 02:30:00. Shorthands:

| Value | Meaning |
|-------|---------|
| `hourly` | every hour, on the hour |
| `daily` | every day at 00:00 |
| `weekly` | Monday at 00:00 |
| `Mon *-*-* 09:00:00` | every Monday at 09:00 |

### Load, enable, inspect

```bash
systemctl daemon-reload
systemctl enable --now backup.timer      # enable the TIMER, not the service
systemctl list-timers                    # what's scheduled, last run, next run
```

### cron or systemd timer?

| Use | When |
|-----|------|
| **cron** | quick one-off schedules, personal automation, short and simple |
| **systemd timer** | production jobs, anything logged/monitored, anything depending on other services |

Older systems lean on cron; newer services often ship a `.timer` next to their `.service`.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Schedule a cleanup with cron

Run a cleanup script every 5 minutes.

- `crontab` and the cron service must be available (install the `cron` package if not).
- `/root/mission/cleanup.sh`: executable, proper shebang, removes files matching `/tmp/junk-*`.
- A crontab entry running it every 5 minutes.

**Solution (from the site):**

Install cron if the `crontab` command is missing, and start the service:

```bash
command -v crontab >/dev/null 2>&1 || apt-get update -qq
command -v crontab >/dev/null 2>&1 || DEBIAN_FRONTEND=noninteractive apt-get install -y -qq cron
service cron start >/dev/null 2>&1 || true
```

Create the script:

```bash
mkdir -p /root/mission
cat > /root/mission/cleanup.sh <<'EOF'
#!/bin/bash
rm -f /tmp/junk-*
EOF
chmod +x /root/mission/cleanup.sh
```

`crontab -e` is interactive, so install the schedule non-interactively (print current crontab + new line, pipe into `crontab -`):

```bash
(crontab -l 2>/dev/null | grep -vF '/root/mission/cleanup.sh'; echo '*/5 * * * * /root/mission/cleanup.sh') | crontab -
```

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Send cron output to a log file

cron discards anything a job prints, the #1 reason "my cron job isn't doing anything" is hard to debug. Make every important entry append its output to a log.

1. `/root/mission/heartbeat.sh`: executable, prints at least one line (shebang + `date`).
2. A crontab line running it on any schedule, ending in `>> /var/log/heartbeat.log`.
3. `/var/log/heartbeat.log` exists.

**Solution (from the site):**

```bash
command -v crontab >/dev/null 2>&1 || apt-get update -qq
command -v crontab >/dev/null 2>&1 || DEBIAN_FRONTEND=noninteractive apt-get install -y -qq cron
```

```bash
mkdir -p /root/mission
cat > /root/mission/heartbeat.sh <<'EOF'
#!/bin/bash
date
EOF
chmod +x /root/mission/heartbeat.sh
```

```bash
(crontab -l 2>/dev/null | grep -vF '/root/mission/heartbeat.sh'; echo '* * * * * /root/mission/heartbeat.sh >> /var/log/heartbeat.log 2>&1') | crontab -
touch /var/log/heartbeat.log
```

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: a cron job that isn't running

**Q:** A crontab entry runs `/home/deploy/nightly.sh` every 5 minutes. The script works when run manually with `bash`, but the cron job seems to do nothing. What do you check first?

- [x] **Append `>> /var/log/nightly.log 2>&1` to the schedule line, then read the log after cron runs it**
- [ ] Confirm the cron daemon is running via `systemctl` status
- [ ] Add `PATH=/usr/local/bin:/usr/bin:/bin` at the top of the crontab
- [ ] Reinstall the cron package

**Why:** cron discards a job's output by default (or emails it, which goes nowhere on cloud VMs), so a failing job fails **silently**. Logging stdout and stderr makes the next run visible, and the log usually names the culprit in one glance, very often "command not found" from cron's minimal `PATH`, which the `PATH=` line then fixes.

| Option | Verdict |
|--------|---------|
| check daemon status | a quick sanity check, but not first |
| set `PATH=` | often the eventual fix, but you don't know that yet |
| reinstall cron | a big hammer swung before any diagnosis |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Item | Meaning |
|------|---------|
| `m h dom mon dow cmd` | the five cron fields |
| `*/5 * * * *` | every 5 minutes |
| `crontab -e` / `-l` / `-r` | edit / list / remove (all!) |
| `cmd >> file.log 2>&1` | capture stdout + stderr |
| `PATH=/usr/local/bin:/usr/bin:/bin` | first line of crontab for a sane PATH |
| `/etc/cron.{hourly,daily,weekly,monthly}/` | drop executable scripts, **no dot in name** |
| `/etc/cron.d/file` | custom schedule, extra **user** field |
| `*.service` + `*.timer` | systemd pair (`Type=oneshot`, `OnCalendar=`, `Persistent=true`) |
| `systemctl enable --now x.timer` | activate a timer |
| `systemctl list-timers` | audit scheduled timers |

**Debug rule:** cron job "not running"? Log its output first (`>> log 2>&1`), then check `PATH`, then the daemon.

<br>

[↑ Back to top](#top)
