<a id="top"></a>

# ⚙️ 09 - Processes

> **Source:** <https://www.escbash.com/topics/linux-processes>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [What is a process](#lesson-1) | 📖 Lesson |
| 2 | [Finding and stopping processes](#lesson-2) | 📖 Lesson |
| 3 | [Background and persistent jobs](#lesson-3) | 📖 Lesson |
| 4 | [Run, find, and stop a process](#task-1) | 🛠️ Task |
| 5 | [Kill all sleep processes at once](#task-2) | 🛠️ Task |
| 6 | [Scenario: stopping a long-running script](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: What is a process

A **process** is a running program. Every command you type spawns one; every service is one. Some are short (`ls` starts, prints, exits), some run for weeks (`nginx`, `sshd`, a database).

### Listing processes

```bash
ps aux
```

```
USER   PID %CPU %MEM    VSZ   RSS TTY STAT START TIME COMMAND
root     1  0.0  0.4 168944 12016 ?   Ss   09:12 0:01 /sbin/init
root   533  0.0  0.2  55208  5400 ?   Ss   09:12 0:00 nginx: master process
```

| Column | Meaning |
|--------|---------|
| `USER` | who owns the process |
| `PID` | process ID, unique per process |
| `%CPU`, `%MEM` | resources it's using |
| `COMMAND` | the command line that launched it |

**PID 1** (`/sbin/init`) is special: the first thing the kernel starts and the ancestor of everything else.

### Everything is referred to by PID

Start a throwaway process to get a real PID:

```bash
sleep 300 &        # shell prints something like [1] 4821  ← the PID
kill 4821          # use the number your shell printed
echo $$            # your own shell's PID
```

### Parent processes

Every process was started by a **parent**. `ps -ef` shows a `PPID` column:

```bash
ps -ef | head
```

In bash, bash is the parent and your command is the child. This matters when chasing which service started which helper, or when a process keeps coming back (its parent is respawning it).

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Finding and stopping processes

### Finding by name

```bash
ps aux | grep nginx                    # also matches the grep itself
ps aux | grep nginx | grep -v grep     # drop the noise
```

Cleaner alternatives:

```bash
pgrep nginx       # just PIDs
pgrep -a nginx    # PIDs plus command lines
pidof nginx       # exact program name, PIDs on one line
```

Use `pgrep` for pattern matching, `pidof` when you know the exact name.

### The live view: `top`

`ps` is a snapshot; `top` updates continuously (sorted by CPU).

| Key | Action |
|-----|--------|
| `q` | quit |
| `M` | sort by memory |
| `P` | sort by CPU |

`htop` is a nicer, coloured version worth installing on machines you use a lot.

### Stopping a process: `kill`

`kill` sends a **signal**.

```bash
kill 1234       # SIGTERM: "please stop"
kill -9 1234    # SIGKILL: "stop now, no cleanup"
```

| Signal | No. | Meaning |
|--------|-----|---------|
| `SIGTERM` | 15 | shut down cleanly (the default) |
| `SIGKILL` | 9 | force immediate stop, no cleanup |
| `SIGHUP` | 1 | reload config (many daemons) |
| `SIGINT` | 2 | interrupt, what `Ctrl+C` sends |

> **💡 Tip:** Try plain `kill PID` first, wait a couple of seconds, and use `kill -9` only if the process is truly stuck. SIGTERM lets it close files and flush buffers; SIGKILL can leave half-written files.

### Killing by name

```bash
sleep 300 &
sleep 300 &
pkill sleep       # pattern match (like pgrep)
killall sleep     # exact command name
```

Both accept signal flags (`pkill -9 sleep`).

> **⚠️ Warning:** `pkill sleep` stops **every** `sleep` on the machine, including other users' or scripts'. If you know the exact PID, prefer that.

### Inspecting one process: `ps -p`

```bash
ps -p 1234      # only PID 1234
```

### Command substitution: `$(...)`

`$(command)` runs `command` and substitutes its output into the surrounding line. Use it when a command takes a PID as an **argument**, not on stdin (`kill`, `ps -p`).

```bash
kill $(pgrep myapp)                 # kill the PID pgrep printed
ps -p $(cat /var/run/app.pid)       # inspect the PID stored in a file
```

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Background and persistent jobs

### `&`: run in the background

```bash
sleep 300 &       # prints [1] 12345 = job number and PID
```

The shell hands you the prompt back immediately.

### A never-ending sleep

```bash
sleep infinity    # never returns; Ctrl+C to stop
```

The standard way to keep a script or container running until something else stops it (minimal Docker images, placeholder systemd services).

### Job control

```bash
jobs         # list background jobs
fg %1        # bring job 1 to the foreground
fg           # most recent job
```

Pause a foreground command and push it to the background:

| Keys | Effect |
|------|--------|
| `Ctrl+Z` | pause it |
| `bg` | resume it in the background |

### Background jobs die with your shell

A background job is still a child of your shell. Close the SSH session and it dies, fine for a `sleep`, disastrous for a two-hour backup.

### `nohup`: survive logout

```bash
nohup sleep 300 &      # stand-in for a long task
```

- The process is detached from your shell.
- Output that would go to the terminal goes to `nohup.out` in the current directory.
- Find it later: `ps aux | grep sleep`.

For auto-restart, log rotation and startup ordering, use **systemd** (next topics).

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Run, find, and stop a process

Create `/root/answers`, then:

1. Start a long `sleep` (e.g. 600 s) in the background.
2. Save its PID (just the number) in `/root/answers/sleep-pid.txt`.
3. Save the `ps` entry for that PID in `/root/answers/sleep-info.txt` (must mention `sleep`).
4. Send it a termination signal so it no longer runs.

**Solution (from the site):**

```bash
mkdir -p /root/answers
sleep 600 &
SLEEP_PID=$!
echo "$SLEEP_PID" > /root/answers/sleep-pid.txt
ps -p "$SLEEP_PID" > /root/answers/sleep-info.txt
kill "$SLEEP_PID"
wait "$SLEEP_PID" 2>/dev/null || true
```

- `$!` is the PID of the most recent background job.
- The final `wait` lets the shell reap the killed process so it's fully gone.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Kill all sleep processes at once

Combine `pgrep` with command substitution so one `kill` handles all of them.

1. Start at least three `sleep` processes in the background.
2. Save their PIDs, one per line, in `/root/sleeps-before.txt`.
3. Kill every `sleep` with a single command that finds them by name.
4. Save the (empty) output of `pgrep sleep` in `/root/sleeps-after.txt` (file must exist but be empty).

**Solution (from the site):**

```bash
pkill -x sleep 2>/dev/null || true
sleep 600 &
sleep 600 &
sleep 600 &
sleep 1
pgrep -x sleep > /root/sleeps-before.txt
kill $(pgrep -x sleep) 2>/dev/null || true
wait
pgrep -x sleep > /root/sleeps-after.txt || true
```

- `-x` matches the exact name.
- `|| true` keeps the script succeeding, since `pgrep` exits non-zero when nothing matches.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: stopping a long-running script

**Q:** A nightly backup script has been running three hours, twice as long as expected. You want to stop it. What's the right thing to do?

- [x] **`kill <PID>` (plain SIGTERM), wait a few seconds, and check that it exited**
- [ ] Force-kill it immediately with `kill -9`
- [ ] `killall backup.sh`
- [ ] Reboot the server

**Why:** the escalation ladder is **SIGTERM to the exact PID first, wait, then SIGKILL only if it ignored you.** SIGTERM lets the backup finish its current write, close file handles and exit cleanly, so the archive stays usable.

| Option | Problem |
|--------|---------|
| `kill -9` | skips cleanup, risks a corrupt half-written file |
| `killall` | can take out unrelated processes sharing the name |
| reboot | trades one stuck script for an outage of every service |

*Precision first, force later, never the whole box.*

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `ps aux` / `ps -ef` | all processes / with PPID |
| `ps -p PID` | one process |
| `pgrep name` / `pgrep -a name` | PIDs / PIDs + command lines |
| `pidof name` | PIDs by exact name |
| `top` | live view (`q`, `M`, `P`) |
| `kill PID` | SIGTERM (polite) |
| `kill -9 PID` | SIGKILL (last resort) |
| `pkill name` / `killall name` | signal all matching processes |
| `$(cmd)` | substitute cmd's output |
| `cmd &` | run in background |
| `jobs`, `fg`, `bg`, `Ctrl+Z` | job control |
| `nohup cmd &` | survive logout |
| `$!` / `$$` | PID of last background job / of this shell |

<br>

[↑ Back to top](#top)
