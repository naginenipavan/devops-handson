<a id="top"></a>

# 📂 03 - Listing files

> **Source:** <https://www.escbash.com/topics/linux-listing-files>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Listing a folder](#lesson-1) | 📖 Lesson |
| 2 | [Hidden files and useful flags](#lesson-2) | 📖 Lesson |
| 3 | [Inventory /etc and /root](#task-1) | 🛠️ Task |
| 4 | [Getting help](#lesson-3) | 📖 Lesson |
| 5 | [Compare two system folders](#task-2) | 🛠️ Task |
| 6 | [Scenario: looking up a flag you forgot](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: Listing a folder

`ls` prints what's inside a folder. No arguments = current folder; with a path = that folder.

```bash
ls            # current folder
ls /etc       # specifically /etc
```

A fresh home folder can look empty because its only files are **hidden** (names start with a dot). A populated folder like `/etc` shows the difference right away.

Plain output shows names only. Add **`-l`** (lowercase L, "long") to get a row of metadata per entry:

```bash
ls -l /etc
```

### Reading one row

```
drwxr-xr-x 2 root root 4096 Feb 10 14:22 nginx
```

| Column | Value | Meaning |
|--------|-------|---------|
| 1st char | `d` | directory (`-` = regular file) |
| rest | `rwxr-xr-x` | permissions preview (later topic) |
| 2 | `2` | link count, ignore for now |
| 3 | `root` | owner |
| 4 | `root` | group |
| 5 | `4096` | size in bytes |
| 6 | `Feb 10 14:22` | last modified |
| 7 | `nginx` | name |

> **💡 Tip:** For now focus on the **first character** (file or folder?) and the **last columns** (size, date, name). The rest matters once you learn permissions and ownership.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Hidden files and useful flags

### Hidden files

Any name starting with a dot is hidden from plain `ls`. These are **dotfiles**.

```bash
ls        # normal listing, no dotfiles
ls -a     # include hidden entries
```

Dotfiles hold configuration:

| Dotfile | Purpose |
|---------|---------|
| `.bashrc` | shell startup config |
| `.ssh/` | SSH keys and known hosts |
| `.aws/`, `.kube/` | cloud and cluster credentials |
| `.gitignore`, `.env` | project-level config |

As a DevOps engineer you'll edit dotfiles constantly. Learn `-a` early to avoid "where did that file go?" moments.

### Human sizes and time sorting

| Flag | Effect |
|------|--------|
| `-h` | sizes in human units (`4.0K`, `12M`, `3.2G`) instead of raw bytes |
| `-t` | sort newest first, ideal for finding the file that just changed |

### The `ls -lah` combo

The one you'll type hundreds of times a week: **long listing + all files + human sizes**.

```bash
ls -lah
ls -lah ~      # shows .bashrc, .profile that plain ls hides
```

### Pipes and counting lines

A **pipe** (`|`) sends one command's output into another command's input. It's the most useful operator in the shell.

```bash
ls /etc | wc -l      # how many entries live in /etc
```

Read it as: run `ls /etc`, send the output into `wc -l`, which counts lines. (`ls` prints one entry per line when its output isn't a terminal.)

- `wc` = "word count"; `-l` (count lines) is the flag you'll use most.

Filter with `grep`, which keeps only matching lines:

```bash
ls -a ~ | grep '^\.'     # only entries starting with a dot (dotfiles)
```

`grep` gets its own topic later. Recognise the shape: *some command produces lines, grep keeps the ones that match.*

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Inventory /etc and /root

Create `/root/answers` if it doesn't exist, then create three files:

| File | Contents |
|------|----------|
| `/root/answers/etc-count.txt` | total number of entries in `/etc` (a single number) |
| `/root/answers/root-hidden.txt` | hidden entries (dotfiles) in `/root`, one per line, must include `.bashrc` |
| `/root/answers/etc-long.txt` | long listing of `/etc` with hidden files and human-readable sizes |

**Solution (from the site):**

```bash
mkdir -p /root/answers
ls /etc | wc -l > /root/answers/etc-count.txt
ls -a /root | grep '^\.' > /root/answers/root-hidden.txt
ls -lah /etc > /root/answers/etc-long.txt
```

> **📝 Note:** `-a` also lists `.` and `..`; `-A` skips those two. Either works here as long as `.bashrc` appears.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Getting help

Nobody memorises every flag. Linux ships with three ways to look them up, all on the machine you're logged into.

### `--help`

Almost every command accepts it. Prints a short summary and every flag.

```bash
ls --help
```

Fast, searchable when piped through `grep`, and never lies about the installed version.

### `man`

Opens the full manual page. Use when `--help` is too terse.

```bash
man ls
```

| Key | Action |
|-----|--------|
| Arrow keys / `Page Up` / `Page Down` | scroll |
| `/` | search |
| `q` | quit |

### `-h`

Some commands use `-h` as a help shortcut (`docker -h`, `curl -h`). Others use it for something else, e.g. "human-readable" in `ls -h`. When in doubt, try `--help`, it's the convention.

> **💡 Tip:** Reach for `--help` first because it's fast; fall back to `man` for the full picture.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Compare two system folders

Save the contents and counts of `/home` (user homes) and `/bin` (core binaries) side by side. Produce three files:

| File | Contents |
|------|----------|
| `/root/home-contents.txt` | full listing of `/home`, one entry per line. If `/home` is empty, list `/root` instead so the file isn't empty |
| `/root/home-count.txt` | a single number: entries in `/home` |
| `/root/bin-count.txt` | a single number: entries in `/bin` |

**Solution (from the site):**

```bash
if [ -n "$(ls /home)" ]; then ls /home > /root/home-contents.txt; else ls -a /root > /root/home-contents.txt; fi
ls /home | wc -l > /root/home-count.txt
ls /bin | wc -l > /root/bin-count.txt
```

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: looking up a flag you forgot

**Q:** You want to list files in `/var/log` sorted by size, but you don't remember which flag `ls` uses. What do you do?

- [x] **Run `ls --help` and skim the output (or pipe it through `grep`) for the flag you want**
- [ ] Open a browser and search the web for "linux ls sort by size"
- [ ] Guess flags like `-z`, `-m`, `-y` and see what happens
- [ ] Run `man ls` and search inside the manual page

**Why:** `--help` is the fastest first stop on the machine itself. `man ls` also works but is heavier; the lesson's rule is `--help` first, `man` as fallback. Guessing flags is risky and the web isn't needed.

*Confirmed against the site's solution: `--help` is fast, on the machine, and matches the installed version; `man` is the deeper fallback.*

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `ls [path]` | list a folder |
| `ls -l` | long format (type, permissions, owner, group, size, date, name) |
| `ls -a` | include hidden dotfiles |
| `ls -h` | human-readable sizes (with `-l`) |
| `ls -t` | newest first |
| `ls -lah` | long + all + human sizes (the everyday combo) |
| `cmd1 \| cmd2` | pipe output of `cmd1` into `cmd2` |
| `wc -l` | count lines |
| `grep pattern` | keep only matching lines |
| `cmd --help` | quick usage summary |
| `man cmd` | full manual (`/` search, `q` quit) |

<br>

[↑ Back to top](#top)
