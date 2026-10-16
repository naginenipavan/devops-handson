<a id="top"></a>

# 🚀 Linux Project #1: Automate log cleanup

> **Source:** <https://www.escbash.com/topics/linux-project-1>
> **Contents:** 1 hands-on project task (no lessons or quizzes)
> **Skills used:** [05 Managing files](05-managing-files.md) · [07 Editing files](07-editing-files.md) · [15 Scheduled jobs](15-scheduled-jobs.md)

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Automate log cleanup](#project) | 🛠️ Project |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="project"></a>

## 🛠️ Project: Automate log cleanup

You're a DevOps engineer at esc bash. The app server keeps filling its disk with old logs and nobody clears them by hand. Set up a cleanup that **runs on a schedule and leaves a record of each run**.

### Set up the project

```bash
mkdir -p /var/log/escbash-app
echo "service started" > /var/log/escbash-app/app.log
echo "old entries" > /var/log/escbash-app/app.log.1.old
echo "old entries" > /var/log/escbash-app/app.log.2.old
echo "old entries" > /var/log/escbash-app/app.log.3.old
```

`/var/log/escbash-app/` now has one live log (`app.log`) and old rotated logs ending in `.old`.

### What to build

| # | Requirement |
|---|-------------|
| 1 | Script `/root/bin/cleanup.sh` that deletes the `*.old` files in `/var/log/escbash-app/`, **prints one line each run**, leaves `app.log` alone, and is **executable** |
| 2 | A **crontab** entry that runs it on a schedule and **appends output** to `/var/log/escbash-app/cleanup.log` |
| 3 | `/var/log/escbash-app/cleanup.log` exists |
| 4 | Run the script **once yourself** so the old logs are cleared now |

The site runs 7 checks on this.

### Worked answer

> **📝 Note:** The site shows **no solution** for the project tasks, so this is my own answer built from chapters 5, 7 and 15. I haven't run it in the lab; treat it as a starting point.

**1. The script**

```bash
mkdir -p /root/bin
cat > /root/bin/cleanup.sh <<'EOF'
#!/bin/bash
rm -f /var/log/escbash-app/*.old
echo "$(date): cleanup ran"
EOF
chmod +x /root/bin/cleanup.sh
```

**2. Make sure cron exists** (from chapter 15), then schedule non-interactively:

```bash
command -v crontab >/dev/null 2>&1 || { apt-get update -qq; DEBIAN_FRONTEND=noninteractive apt-get install -y -qq cron; }
service cron start >/dev/null 2>&1 || true

(crontab -l 2>/dev/null | grep -vF '/root/bin/cleanup.sh'; echo '0 * * * * /root/bin/cleanup.sh >> /var/log/escbash-app/cleanup.log 2>&1') | crontab -
```

`0 * * * *` = top of every hour. Any schedule works as long as the line ends by appending to `cleanup.log`.

**3. Create the log and run once**

```bash
touch /var/log/escbash-app/cleanup.log
/root/bin/cleanup.sh >> /var/log/escbash-app/cleanup.log
```

**4. Verify**

```bash
ls /var/log/escbash-app/        # app.log and cleanup.log only
crontab -l
cat /var/log/escbash-app/cleanup.log
```

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Piece | Why |
|-------|-----|
| `rm -f dir/*.old` | glob only the old files; `-f` doesn't fail if none exist |
| `echo "$(date): ..."` | one log line per run = an audit trail |
| `>> file 2>&1` in the cron line | cron discards output otherwise |
| `(crontab -l ...; echo ...) \| crontab -` | edit the crontab without an interactive editor |
| run it once by hand | verifies the script before waiting for the schedule |

<br>

[↑ Back to top](#top)
