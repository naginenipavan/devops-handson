<a id="top"></a>

# 🛎️ 10 - systemd services

> **Source:** <https://www.escbash.com/topics/linux-systemd-services>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [What is systemd](#lesson-1) | 📖 Lesson |
| 2 | [systemctl basics](#lesson-2) | 📖 Lesson |
| 3 | [journalctl](#lesson-3) | 📖 Lesson |
| 4 | [Create a tiny service](#task-1) | 🛠️ Task |
| 5 | [Inspect a running service](#task-2) | 🛠️ Task |
| 6 | [Scenario: a service that won't start](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: What is systemd

Every long-running program on a modern Linux server is a **service**, and almost every service is managed by **systemd**. At boot it's the first process, and it launches everything else: network, SSH, cron, your database, your web server.

### The first process

The kernel starts exactly one program, the **init system** (systemd on almost every modern distro). It gets **PID 1**:

```bash
ps -p 1 -o pid,comm
```

```
PID COMMAND
  1 systemd
```

Every other process descends from it, so PID 1 is the root of the process tree. If it exits, the machine goes down, which is why it's built to keep running no matter what.

### Why not just `nohup`?

`nohup` keeps a command alive after logout, but it doesn't handle:

- starting automatically on **boot**
- **restarting** if the program crashes
- making sure **dependencies** (like the network) are up first
- collecting **logs** where the rest of the system's logs live

systemd does all of these. To deploy a service you don't run it with `&`, you write a **unit**.

### Units

A **unit** is "a thing systemd manages". Most common is `service`, but there are also timers, sockets, mounts, etc. Names end in their type:

| Unit | What it is |
|------|------------|
| `ssh.service` | the SSH daemon |
| `cron.service` | the cron scheduler |
| `nginx.service` | the nginx web server |
| `backup.timer` | a scheduled job |

Control units with **`systemctl`**, read their logs with **`journalctl`**. That's the whole daily surface.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: systemctl basics

### Life cycle

```bash
systemctl status ssh      # what's going on with ssh?
systemctl start ssh       # start it now
systemctl stop ssh        # stop it now
systemctl restart ssh     # stop then start
systemctl reload ssh      # re-read config, no downtime
```

`reload` only works if the service supports it. If in doubt, `restart` always works (brief downtime).

### enabled vs active

Two **independent** settings:

| Setting | Question |
|---------|----------|
| **active** | is it running right now? |
| **enabled** | will it auto-start at boot? |

A service can be enabled-not-active, active-not-enabled, both, or neither.

```bash
systemctl enable ssh            # auto-start on next boot
systemctl disable ssh           # don't start on boot
systemctl enable --now ssh      # enable AND start immediately
systemctl disable --now ssh     # disable AND stop immediately
```

`--now` is the shortcut you'll type most when setting up or removing a service.

### Reading `systemctl status`

```bash
systemctl status nginx
```

```
● nginx.service - A high performance web server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-07-28 09:14:02 UTC; 2h ago
   Main PID: 431 (nginx)
      Tasks: 2 (limit: 1109)
     Memory: 3.2M
     CGroup: /system.slice/nginx.service
             ├─431 nginx: master process /usr/sbin/nginx
             └─432 nginx: worker process

Jul 28 09:14:02 host systemd[1]: Started nginx.service.
```

| Line | What to read |
|------|--------------|
| `●` dot | green = healthy, red = failed |
| `Loaded` | unit file path and whether it's **enabled** (starts on boot) |
| `Active` | **check first**: `active (running)`, `inactive (dead)`, `failed`, `activating`; `since …` = how long |
| `Main PID` | the service's process ID |
| last lines | most recent journal entries, often enough to spot a problem |

### Listing units

```bash
systemctl list-units --type=service                   # only services
systemctl list-units --type=service --state=failed    # only broken ones
```

On a healthy machine `--state=failed` prints nothing.

### Unit file structure

Your own services go in `/etc/systemd/system/` as `*.service` files:

```ini
[Unit]
Description=My tiny service
After=network.target

[Service]
ExecStart=/usr/local/bin/mytool
Restart=on-failure
Type=simple

[Install]
WantedBy=multi-user.target
```

| Section | Purpose |
|---------|---------|
| `[Unit]` | description and startup ordering (`After=network.target` = start after the network is up) |
| `[Service]` | `ExecStart=` (full path to the command), `Type=simple` (normal foreground program), `Restart=on-failure` (what to do on a crash) |
| `[Install]` | hook into boot; `WantedBy=multi-user.target` = "the normal running state" |

> **❗ Important:** After adding or changing a unit file, systemd doesn't notice on its own:
> ```bash
> sudo systemctl daemon-reload
> ```
> Then `systemctl enable --now myservice.service` starts it and wires it up for future boots.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: journalctl

systemd captures everything a service prints (output and errors) in the **journal**. `journalctl` reads it.

### Per-service logs

```bash
journalctl -u ssh                    # everything ssh has ever logged
journalctl -u ssh --since today      # since midnight
journalctl -u ssh --since '1 hour ago'
journalctl -u ssh -n 100             # last 100 lines
```

`-u` = unit (same names as `systemctl`).

```
Jul 28 09:14:01 host sshd[512]: Server listening on 0.0.0.0 port 22.
Jul 28 09:20:44 host sshd[788]: Accepted password for root from 10.0.0.5
```

Each line: timestamp, hostname, program with PID in brackets, then the message. Newest entries are at the bottom.

### Follow in real time

```bash
journalctl -u ssh -f
```

Like `tail -f` but scoped to a service. Leave it running in another terminal while you change a config.

### Useful filters

| Flag | Effect |
|------|--------|
| `-p err` | errors and worse only |
| `--since '2 hours ago'` | time window |
| `-b` | only the current boot |
| `-o cat` | strip timestamp/hostname, print only the message |

```bash
journalctl -u nginx -p err --since '1 hour ago'     # nginx errors from the last hour
```

If something broke recently on a production server, this is usually the first thing you type.

### Where the journal lives

`/var/log/journal/` as **binary** files, so don't read them directly. On some minimal systems that folder doesn't exist and the journal is **memory only**, wiped on reboot, worth knowing when you can't find yesterday's logs.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Create a tiny service

Wrap a script in a systemd unit.

| File | Requirement |
|------|-------------|
| `/root/mission/heartbeat.sh` | executable, valid shebang, at least one command that never exits (`sleep infinity`) |
| `/etc/systemd/system/heartbeat.service` | `[Unit]`, `[Service]`, `[Install]`; `ExecStart` exactly `/root/mission/heartbeat.sh` |

Afterwards the unit must be **enabled**, **active**, and have **at least one journal entry**.

**Solution (from the site):**

```bash
mkdir -p /root/mission
cat > /root/mission/heartbeat.sh <<'EOF'
#!/bin/bash
sleep infinity
EOF
chmod 755 /root/mission/heartbeat.sh
cat > /etc/systemd/system/heartbeat.service <<'EOF'
[Unit]
Description=Heartbeat service for the systemd lab
After=network.target

[Service]
ExecStart=/root/mission/heartbeat.sh
Type=simple
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now heartbeat.service
```

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Inspect a running service

Most systemd work is inspecting existing services. Practise on `systemd-journald`:

| File | Contents |
|------|----------|
| `/root/journald-status.txt` | `systemctl status` output (use `--no-pager`); must contain `active` |
| `/root/journald-enabled.txt` | `systemctl is-enabled` output (one word: `enabled`, `static`, ...) |
| `/root/journald-recent.txt` | last 10 journal entries for the unit |

**Solution (from the site):**

```bash
systemctl status systemd-journald --no-pager > /root/journald-status.txt
systemctl is-enabled systemd-journald > /root/journald-enabled.txt 2>&1 || true
journalctl -u systemd-journald -n 10 --no-pager > /root/journald-recent.txt
```

> **📝 Note:** `|| true` on `is-enabled` matters: it exits non-zero for some states (like `static`) even while printing the answer you want.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: a service that won't start

**Q:** A colleague deployed a service earlier today. `systemctl status myapp` shows `Active: failed`, but it should be running. What do you check first?

- [x] **`journalctl -u myapp --since '1 hour ago'`**
- [ ] `systemctl status myapp | tail -20`
- [ ] Restart the service repeatedly and hope it comes up
- [ ] Reboot the server

**Why:** a failed unit almost always wrote the reason to the journal just before it died: a config parse error, a missing file, a port already in use, a permission problem. **Read the log first, every time.**

| Option | Problem |
|--------|---------|
| `systemctl status \| tail` | decent shortcut, but can truncate the part you need |
| blind restarts | reproduce the same failure and bury the original timestamps |
| reboot | destroys evidence and takes down every healthy service |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `systemctl status X` | state, enabled/active, last log lines |
| `systemctl start/stop/restart/reload X` | life cycle |
| `systemctl enable --now X` | start on boot **and** now |
| `systemctl disable --now X` | remove from boot **and** stop |
| `systemctl is-enabled X` | enabled at boot? |
| `systemctl list-units --type=service --state=failed` | broken services |
| `systemctl daemon-reload` | after editing unit files |
| `journalctl -u X` | service logs |
| `journalctl -u X -f` | follow live |
| `journalctl -u X -n 100` | last 100 lines |
| `journalctl -u X --since '1 hour ago'` | time window |
| `journalctl -p err` / `-b` / `-o cat` | errors only / this boot / message only |

**Unit file skeleton:** `[Unit]` (Description, After) · `[Service]` (ExecStart, Type, Restart) · `[Install]` (WantedBy=multi-user.target)

<br>

[↑ Back to top](#top)
