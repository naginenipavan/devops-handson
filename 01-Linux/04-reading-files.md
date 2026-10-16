<a id="top"></a>

# 📄 04 - Reading files

> **Source:** <https://www.escbash.com/topics/linux-reading-files>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [cat, head, tail, and wc](#lesson-1) | 📖 Lesson |
| 2 | [less for big files](#lesson-2) | 📖 Lesson |
| 3 | [Following a log with tail -f](#lesson-3) | 📖 Lesson |
| 4 | [Summarize the /etc/passwd file](#task-1) | 🛠️ Task |
| 5 | [Triage a large system file](#task-2) | 🛠️ Task |
| 6 | [Scenario: finding errors in a huge log](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: `cat`, `head`, `tail`, and `wc`

Four commands do most of the work when you need to look at a text file.

### `cat`: print the whole thing

```bash
cat /etc/hostname
cat /etc/os-release
```

Great for short files. For anything long, use the tools below, otherwise the terminal fills up faster than you can scroll.

### `head` and `tail`: peek at either end

- `head` prints the **first 10** lines by default; `tail` prints the **last 10**.
- Both take `-n` to change the count.

```bash
head /etc/os-release
head -n 3 /etc/os-release     # first 3 lines only
tail -n 5 /etc/passwd         # last 5 lines
```

Perfect for logs: the interesting bit is usually near the top (startup) or the bottom (most recent).

### `wc`: count lines, words, bytes

Most of the time you'll use `-l` for the line count.

```bash
wc -l /etc/passwd
```

Piped from another command, `wc -l` becomes "how many things did that produce":

```bash
ls /etc | wc -l      # how many entries live in /etc
```

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: `less` for big files

`cat` prints everything at once, a problem when "everything" is a million lines of log. `less` opens a **scrollable viewer** that only loads what you need.

```bash
less /etc/services
```

You're now inside the viewer, not the shell. Keys you'll use:

| Key | Action |
|-----|--------|
| `Space` or `f` | page down |
| `b` | page up |
| `g` | jump to the top |
| `G` (capital) | jump to the bottom |
| `/word` | search forward; `n` for next match |
| `?word` | search backward |
| `q` | quit and return to the shell |

Open a file straight at the bottom:

```bash
less +G /var/log/dpkg.log
```

> **💡 Tip:** Not sure if a file is big? Use `less`, not `cat`. It opens a 2 GB file instantly, whereas `cat` would flood your terminal for minutes.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Following a log with `tail -f`

Watching a log update in real time is a daily activity on any real server. That's `tail -f`.

The lab runs a real nginx server, and every request lands in its access log. Generate a few lines first:

```bash
curl -s localhost > /dev/null
curl -s localhost > /dev/null
tail -f /var/log/nginx/access.log
```

`-f` ("follow") keeps the command running and prints each new line the moment it's appended. When something breaks in production, this is where the truth lives.

### Variations

```bash
tail -n 100 -f /var/log/nginx/access.log   # last 100 lines, then follow
tail -F /var/log/nginx/access.log          # keep watching even if the file is rotated
```

| Flag | Use |
|------|-----|
| `-n 100 -f` | get context before the live stream starts |
| `-F` (capital) | keep watching across **log rotation** (old log renamed, fresh one created) |

- Press **`Ctrl+C`** to stop following.
- System-wide log is usually `/var/log/syslog`; on systemd machines use `journalctl -f`. Tailing works the same way.
- "Just tail the log" = this.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Summarize the /etc/passwd file

A quick look at `/etc/passwd` on an unfamiliar machine tells you how many accounts exist, which are system accounts (first entries), and which were added most recently (last entries).

Produce `/root/passwd-summary.txt` containing, in any order:

1. The total number of accounts (a single line with just the number).
2. The first three lines of `/etc/passwd`.
3. The last three lines of `/etc/passwd`.

Hint from the site: one command to count lines, one for the top, one for the bottom. Redirect the first into the file, then **append** the other two.

**Solution (from the site):**

```bash
wc -l < /etc/passwd > /root/passwd-summary.txt
head -n 3 /etc/passwd >> /root/passwd-summary.txt
tail -n 3 /etc/passwd >> /root/passwd-summary.txt
```

> **📝 Note:** `wc -l < file` prints just the number. `wc -l file` also prints the filename, which would break "a single line with just the number".

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Triage a large system file

Three questions cover most of what you need for an unfamiliar file: how big is it, what does the top look like, what does the bottom look like? `/etc/services` (every well-known TCP/UDP port and service name, a few hundred lines) is the target.

Produce `/root/services-triage.txt` containing, in any order:

1. The total line count of `/etc/services` (a single line with just the number).
2. The first 10 lines.
3. The last 10 lines.

**Solution (from the site):**

```bash
wc -l < /etc/services > /root/services-triage.txt
head -n 10 /etc/services >> /root/services-triage.txt
tail -n 10 /etc/services >> /root/services-triage.txt
```

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: finding errors in a huge log

**Q:** A service is misbehaving. `/var/log/app.log` is 2 GB. You need to find recent database errors. What do you do?

- [ ] `cat /var/log/app.log`
- [ ] `tail -n 1000 /var/log/app.log | grep -i error`
- [x] **`grep -i error /var/log/app.log | tail -n 50`**
- [ ] Open the log in `less`, press `/error` to search and `G` to jump to the end

**Why:** `grep` streams through the whole file without loading it into memory and keeps only matching lines; the trailing `tail` keeps the 50 most recent matches, giving **full coverage plus recency** in one pipeline. "This is the command DevOps engineers actually type at 3 AM."

| Option | Trade-off |
|--------|-----------|
| `cat` on 2 GB | floods the terminal with everything you didn't ask for |
| `tail -n 1000 \| grep` | fast, but can miss errors older than the last 1000 lines |
| `less` + `/error` | works, but slower and manual |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `cat file` | print the whole file (short files only) |
| `head file` / `head -n N file` | first 10 / first N lines |
| `tail file` / `tail -n N file` | last 10 / last N lines |
| `wc -l file` | count lines (`wc -l < file` for just the number) |
| `less file` | scrollable viewer (`Space`/`b` page, `g`/`G` top/bottom, `/word` search, `q` quit) |
| `less +G file` | open at the bottom |
| `tail -f file` | follow new lines live (`Ctrl+C` to stop) |
| `tail -n 100 -f file` | last 100 lines, then follow |
| `tail -F file` | follow across log rotation |
| `journalctl -f` | follow logs on systemd machines |
| `grep -i error file \| tail -n 50` | 50 most recent error lines in a huge log |

<br>

[↑ Back to top](#top)
