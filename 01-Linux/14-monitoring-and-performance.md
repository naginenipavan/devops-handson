<a id="top"></a>

# 📈 14 - Monitoring and performance

> **Source:** <https://www.escbash.com/topics/linux-monitoring-and-performance>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Uptime and load average](#lesson-1) | 📖 Lesson |
| 2 | [Memory with free](#lesson-2) | 📖 Lesson |
| 3 | [top and htop](#lesson-3) | 📖 Lesson |
| 4 | [Write a machine health report](#task-1) | 🛠️ Task |
| 5 | [Memory pressure snapshot](#task-2) | 🛠️ Task |
| 6 | [Scenario: a suddenly slow server](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: Uptime and load average

```bash
uptime
```

```
19:15:42 up 3 days, 4:12, 2 users, load average: 0.42, 0.55, 0.61
```

| Piece | Meaning |
|-------|---------|
| `up 3 days, 4:12` | how long since the last reboot |
| `2 users` | users currently logged in |
| `load average` | **the useful part**: three numbers |

### What load average means

The three numbers are the average count of processes wanting the CPU over the last **1, 5 and 15 minutes**. Compare them to the **number of CPU cores**:

| Load vs cores | Meaning |
|---------------|---------|
| below | spare capacity |
| around | fully utilised |
| above | processes are queuing for CPU |

A load of `4.0` on an 8-core box = 50%, healthy. The same load on a 2-core box = overloaded.

```bash
nproc      # core count in one number
```

### Reading the three numbers together

| Pattern | Meaning |
|---------|---------|
| 1-min rising, 15-min low | something just started spiking; may resolve itself |
| 1-min low, 15-min high | was bad, calming down |
| all three high | sustained overload, investigate |

Load average says the machine is busy, **not what's making it busy**. That's `top` (below).

### The raw source: `/proc/loadavg`

```bash
cat /proc/loadavg
```

```
0.42 0.55 0.61 2/312 4567
```

The three load averages, then runnable/total processes, then the last PID created. Scripts read this file instead of parsing `uptime`.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Memory with `free`

```bash
free -h       # -h = human-readable units
```

```
        total   used   free  shared  buff/cache  available
Mem:     2.0Gi  200Mi  1.2Gi  0.0Ki      500Mi      1.7Gi
Swap:       0B     0B     0B
```

Six columns, but only two matter most:

| Column | Meaning |
|--------|---------|
| `used` | memory held by processes |
| `free` | memory the kernel hasn't touched at all (usually small, and that's fine) |
| **`available`** | memory that could go to a new process, **including cache the kernel would release** |

> **❗ Important:** Look at **available**, not free. 200 MiB free but 1.7 GiB available = plenty of memory.

### `buff/cache`: borrowed, not lost

Linux uses spare RAM to cache recently read files and releases it the moment a process needs it. That's why "free" often looks tiny on an idle box. Linux treats unused RAM as wasted RAM.

### Swap

The `Swap:` row is disk-backed memory. On modern cloud VMs it's often zero/disabled, which is normal. Swap **in active use** while RAM is available means something is misconfigured.

### Extract the available value

```bash
free -h | grep '^Mem:' | sed 's/.* //'
```

`grep '^Mem:'` keeps the memory row; `sed 's/.* //'` deletes everything up to the last space, leaving the last column (available).

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: `top` and `htop`

`top` is the interactive dashboard of what's running now.

```bash
top
```

Top block: load average, CPU%, memory, task counts. Below: one row per process, sorted by CPU. `q` quits.

### Keys inside `top`

| Key | Action |
|-----|--------|
| `M` | sort by memory |
| `P` | sort by CPU (default) |
| `k` | kill a process (asks for PID) |
| `1` | per-CPU stats instead of aggregate |
| `h` | built-in help |

### `htop`: a better `top`

Coloured, scrollable, mouse-friendly. Not installed by default (needs network):

```bash
sudo apt install -y htop
htop
```

Sort by clicking column headers, kill with `F9`. If you're offline, plain `top` is always there.

### `top` in a script

```bash
top -b -n1 | head -20
```

| Part | Role |
|------|------|
| `-b` | batch mode: plain text, no interactive screen |
| `-n1` | one iteration, then exit |
| `head -20` | summary block + busiest processes |

### Two more tools

| Tool | Purpose | Notes |
|------|---------|-------|
| `vmstat 1` | virtual memory and CPU activity, a fresh line every second | already installed |
| `iostat` | disk read/write stats | from `sysstat`: `sudo apt install -y sysstat` |

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Write a machine health report

Attach a snapshot of machine vitals to an incident ticket: `/root/health-report.txt`, containing the output of:

1. `uptime`
2. `free` with the human-readable flag
3. a non-interactive `top` snapshot (first 20 lines is fine)

**Solution (from the site):**

```bash
uptime > /root/health-report.txt
free -h >> /root/health-report.txt
top -b -n1 | head -20 >> /root/health-report.txt
```

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Memory pressure snapshot

| File | Contents |
|------|----------|
| `/root/mem-summary.txt` | output of `free -h` (the `Mem:` row must appear) |
| `/root/swap-info.txt` | every `/proc/meminfo` line whose name starts with `Swap` (SwapTotal, SwapFree, SwapCached), filtered with `grep` |

**Solution (from the site):**

```bash
free -h > /root/mem-summary.txt
grep Swap /proc/meminfo > /root/swap-info.txt
```

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: a suddenly slow server

**Q:** A web service that normally responds in 50 ms now takes 5 seconds. You SSH into the box. What's your first check?

- [x] **Run `top` and watch it for a minute**
- [ ] Run `uptime && free -h && df -h`
- [ ] Tail the service's own logs first
- [ ] Restart the service and see if it helps

**Why:** **establish system state before touching anything.** `top` shows load, memory pressure and the busiest processes on one live screen; a minute of watching reveals the culprit (runaway process, swap thrashing, CPU pinned) or rules out the machine and points you at the service.

| Option | Verdict |
|--------|---------|
| `uptime && free -h && df -h` | same triage as a snapshot, fine over flaky SSH, just less alive |
| service logs | can wait: a slow service is often the *victim* of something else on the box |
| restart first | worst order: erases the signals you came to read, and the cause remains |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `uptime` | uptime, users, load average (1/5/15 min) |
| `nproc` | CPU core count (compare to load) |
| `cat /proc/loadavg` | raw load numbers |
| `free -h` | memory (**read `available`**) |
| `grep Swap /proc/meminfo` | swap details |
| `top` | live dashboard (`M` memory, `P` CPU, `1` per-CPU, `q` quit) |
| `top -b -n1 \| head -20` | scriptable snapshot |
| `htop` | nicer `top` (`apt install htop`) |
| `vmstat 1` | memory/CPU activity each second |
| `iostat` | disk I/O (`apt install sysstat`) |

**Slow-server triage:** `top` / `uptime` → `free -h` → `df -h` → *then* the service's logs → restart last.

<br>

[↑ Back to top](#top)
