<a id="top"></a>

# 🧭 02 - Paths and navigation

> **Source:** <https://www.escbash.com/topics/linux-paths-and-navigation>
> **Contents:** 2 lessons · 2 hands-on tasks · 2 quizzes

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [The Linux filesystem tree](#lesson-1) | 📖 Lesson |
| 2 | [Absolute and relative paths](#lesson-2) | 📖 Lesson |
| 3 | [Set up a small project folder](#task-1) | 🛠️ Task |
| 4 | [Set up a config-per-environment layout](#task-2) | 🛠️ Task |
| 5 | [Quick check: which path is absolute?](#quiz-1) | ❓ Quiz |
| 6 | [Scenario: reading /etc/hosts from your home folder](#quiz-2) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: The Linux filesystem tree

Every file and folder lives inside **one tree** with exactly one root, written `/`. No `C:`, no `D:`, no separate drives.

```
/
├── etc/       # system-wide configuration
├── var/       # variable data: logs, spools, caches
├── home/      # regular users' home folders
├── root/      # the root user's home
├── tmp/       # temporary files, often wiped on reboot
├── usr/       # installed software
└── bin/       # essential command binaries
```

| Folder | What it holds |
|--------|---------------|
| `/etc` | system-wide configuration |
| `/var` | data that changes while the system runs: logs, mail queues, package caches. You'll open `/var/log` most, where every service writes its logs |
| `/home` | one folder per regular user (`alice` gets `/home/alice` and typically owns everything inside) |
| `/root` | home of the root (administrator) user, kept separate from `/home` so root still works if `/home` is on a broken disk |
| `/tmp` | scratch space, any process can write here, usually wiped on reboot. **Never store anything you want to keep** |
| `/usr` | installed software: programs in `/usr/bin`, libraries in `/usr/lib`, docs in `/usr/share`. Package managers put most files here |
| `/bin` | essential binaries needed early in boot and recovery (`ls`, `cp`, `bash`). On modern Linux usually a symlink to `/usr/bin` |

Others exist too: `/dev`, `/proc`, `/sys`, `/opt`, `/srv`, `/mnt`, `/media`. You'll meet them as they come up.

### Where you are in the tree

Run `pwd` to see where the shell has placed you (probably `/root`). Every path you write is either an **address starting from `/`** or a **shortcut from wherever `pwd` reports**.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Absolute and relative paths

A path is an address for a file or folder. You can use either its **absolute** path or its **relative** path.

### Absolute paths

Imagine `nginx.conf` inside `nginx`, inside `etc`:

```
/
└── etc
    └── nginx
        └── nginx.conf
```

An absolute path **starts with `/`** and gives the full route from the top of the tree:

```
/etc/nginx/nginx.conf
```

Read the slashes as "inside". More examples:

```
/var/log/syslog
/home/alice/notes.txt
```

Absolute paths work from anywhere. Standing in `/tmp`, `/etc/hosts` still means the same file.

### Relative paths

A relative path **doesn't start with `/`**. It is read from wherever you're currently standing.

If your current folder is `/home/alice`:

| Relative path | Means |
|---------------|-------|
| `notes.txt` | `/home/alice/notes.txt` |
| `photos/summer.jpg` | `/home/alice/photos/summer.jpg` |
| `../bob/notes.txt` | `/home/bob/notes.txt` |

Two shortcuts:

| Symbol | Meaning |
|--------|---------|
| `.` | the folder you are in now |
| `..` | the folder one level up |

> **💡 Tip:** Use an **absolute** path when the command must work from anywhere; use a **relative** path when you're already standing next to the file.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Set up a small project folder

Build this exact layout under `/root/projects`:

```
/root/projects/
|-- webapp/
|   |-- README.txt      (contents: webapp)
|   \-- src/
|       \-- main.txt    (contents: src)
\-- api/
    \-- README.txt      (contents: api)
```

- Three folders, three text files.
- Each file contains **exactly the single word** shown (no extra spaces, no other lines).
- You can use full paths from home, or `cd` into each folder and use short names (closer to real project work).

**Solution (from the site):**

```bash
mkdir -p /root/projects/webapp/src /root/projects/api
echo webapp > /root/projects/webapp/README.txt
echo src > /root/projects/webapp/src/main.txt
echo api > /root/projects/api/README.txt
```

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Set up a config-per-environment layout

One settings file per environment, under `/root/deploy`:

```
/root/deploy/
|-- dev/
|   \-- settings.txt      (contents: dev)
|-- staging/
|   \-- settings.txt      (contents: staging)
\-- prod/
    \-- settings.txt      (contents: prod)
```

Each `settings.txt` contains only the word matching its environment (no other text, no trailing lines).

**Solution (from the site):**

```bash
mkdir -p /root/deploy/dev /root/deploy/staging /root/deploy/prod
echo dev > /root/deploy/dev/settings.txt
echo staging > /root/deploy/staging/settings.txt
echo prod > /root/deploy/prod/settings.txt
```

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Quick check: which path is absolute?

**Q:** Which of these is an absolute path?

- [ ] `~/notes.txt`
- [ ] `notes.txt`
- [x] **`/home/alice/notes.txt`**
- [ ] `../notes.txt`

**Why:** absolute paths start with `/`. `~/notes.txt` starts with `~` (shorthand for home), and the other two are relative.

<br>

---

<br>

<a id="quiz-2"></a>

## ❓ Quiz 2: Scenario: reading /etc/hosts from your home folder

**Q:** You're logged in as root, currently in `/root`. You need to see the contents of `/etc/hosts`. What's the simplest way?

- [x] **`cat /etc/hosts`**
- [ ] `cd .. && cd .. && cd etc && cat hosts`
- [ ] `cd /etc && cat hosts`
- [ ] `find / -name hosts`

**Why:** an absolute path works from anywhere, so one command does it. The others do extra work (`cd` chains, changing your folder, searching the whole disk).

*Confirmed against the site's solution: absolute paths work from anywhere, so one command with no side effects.*

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Concept | Remember |
|---------|----------|
| Root of the tree | `/` (one tree, no drive letters) |
| Absolute path | starts with `/`, works from anywhere |
| Relative path | no leading `/`, read from the current folder |
| `.` / `..` | current folder / parent folder |
| `~` | your home folder |
| `/etc` | configuration |
| `/var/log` | service logs |
| `/home/<user>` | regular user's home |
| `/root` | root's home |
| `/tmp` | temporary, don't keep anything here |
| `/usr`, `/bin` | installed software, essential binaries |
| `mkdir -p a/b/c` | create nested folders in one go |

<br>

[↑ Back to top](#top)
