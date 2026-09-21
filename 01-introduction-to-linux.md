<a id="top"></a>

# 🐧 01 - Introduction to Linux

> **Source:** <https://www.escbash.com/topics/linux-shell-basics>
> **Contents:** 5 lessons · 2 hands-on tasks · 2 quizzes

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [What is Linux](#lesson-1) | 📖 Lesson |
| 2 | [Your first commands](#lesson-2) | 📖 Lesson |
| 3 | [`pwd`, `ls`, and `cat`](#lesson-3) | 📖 Lesson |
| 4 | [`mkdir` and `cd`](#lesson-4) | 📖 Lesson |
| 5 | [Redirecting output with `>`](#lesson-5) | 📖 Lesson |
| 6 | [Collect machine information](#task-1) | 🛠️ Task |
| 7 | [Prepare a working folder](#task-2) | 🛠️ Task |
| 8 | [Quick check: what kernel is this?](#quiz-1) | ❓ Quiz |
| 9 | [Scenario: logging into an unfamiliar server](#quiz-2) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: What is Linux

Linux is an operating system (like Windows and macOS) with a different philosophy: **free, open source, designed for servers first**.

- It is the default on AWS, Azure and Google Cloud VMs, and underlies Docker, Kubernetes, most CI runners and most production databases.
- **Goal for DevOps:** sit at a Linux machine, **look around, read a config, restart a service, and get out**. You don't need to be a kernel developer.
- Day-to-day you use dozens of commands, not thousands.

### Distributions

Linux is a family of **distributions**: Ubuntu, Debian, CentOS, Red Hat, Amazon Linux, Alpine.

- They mostly behave the same.
- **This course uses Ubuntu**, the usual cloud VM default.

### The shell

- The **shell** is a text program that reads what you type and prints the result. No mouse.
- On a server, the shell *is* the interface.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Your first commands

On Ubuntu the shell is **bash**. It greets you with a prompt:

```
root@abc123:~#
```

| Part | Meaning |
|------|---------|
| `root` | your user |
| `abc123` | machine name |
| `~` | current folder (shorthand for home) |
| `#` | prompt symbol (`#` = root, `$` = regular user) |

### Safe read-only commands

These only print information, so there is nothing to break:

```bash
whoami        # which user are you?
hostname      # what's this machine called?
date          # what time does it think it is?
uname -a      # kernel and architecture details
echo hello    # print "hello" back at you
```

### Flags change what a command does

Flags (options) are letters prefixed with `-`.

```bash
uname         # kernel name only (short)
uname -s      # same as above, explicit form
uname -a      # everything: kernel, host, version, architecture
```

> [!TIP]
> Not sure which flag to use? Try `--help` after the command.

### Keyboard shortcuts

| Keys | Action |
|------|--------|
| `Ctrl+C` | cancel whatever the shell is running |
| `Ctrl+L` (or `clear`) | wipe the screen |
| `↑` Up arrow | bring back the previous command; keep pressing to go further back |

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: `pwd`, `ls`, and `cat`

Three commands answer three questions: *where am I, what's in this folder, what's in this file?*

### `pwd`: print the current folder

```bash
pwd
```

- Prints the current folder as a full path, e.g. `/root`.
- Home folder: `/root` for root, `/home/alice` for user `alice`.
- Commands act on files in the current folder unless you give a **full path** (starting with `/`).

### `ls`: list a folder's contents

```bash
ls          # entries in the current folder
ls /        # entries at the top of the filesystem
ls /etc     # entries in /etc
```

No argument = current folder. With a path = that folder's contents.

### `cat`: print a file's contents

```bash
cat /etc/hostname      # the machine's name, as stored on disk
cat /etc/os-release    # which Linux distribution and version
```

Good for short files; longer files have better tools (covered in a later topic).

### Tab completion

Type part of a command or path and press `Tab`; the shell completes it.

| You type | Press Tab | Result |
|----------|-----------|--------|
| `who` | `Tab` | `whoami` |
| `cat /et` | `Tab` | `cat /etc/` |

- Press `Tab` twice when several things match to see the options.

> [!TIP]
> Learn this now. It saves keystrokes and prevents typos in long paths.

<br>

---

<br>

<a id="lesson-4"></a>

## 📖 Lesson 4: `mkdir` and `cd`

Two commands you'll use many times a day.

### `mkdir`: create a directory

```bash
mkdir notes              # create in the current directory
mkdir /root/projects     # create at an absolute path
```

Plain `mkdir` fails if the parent doesn't exist. **`-p` creates every missing parent:**

```bash
mkdir -p /root/projects/api/src
```

- Creates `projects`, then `projects/api`, then `projects/api/src`.
- `-p` is also quiet if the directory already exists, so it is **safe to run twice**.

### `cd`: change directory

```bash
cd /etc     # jump using an absolute path
cd notes    # step into `notes` in the current folder
cd ..       # up one level (parent directory)
cd ~        # jump to your home folder
cd          # bare cd = cd ~
```

After `cd`, `pwd` shows the new location and later commands act on that folder. `cd` is the most-used navigation command in Linux.

### Typical workflow

```bash
mkdir notes
cd notes
# ...do things here...
cd          # back home
```

<br>

---

<br>

<a id="lesson-5"></a>

## 📖 Lesson 5: Redirecting output with `>`

By default a command prints to the screen. `>` sends the output into a file instead.

```bash
whoami > user.txt      # write your username into user.txt
hostname > host.txt    # write the machine name into host.txt
date > when.txt        # write the current date and time
```

- If the file doesn't exist, `>` **creates** it.
- If it exists, `>` **overwrites** it (previous contents are gone).

> [!WARNING]
> `>` silently replaces existing content. Double-check the filename first.

Read it back:

```bash
cat user.txt
```

### Relative vs full paths

After `cd` into a folder, a plain filename lands in that folder. Full paths always work:

```bash
whoami > /root/answers/user.txt
```

| Style | Use when |
|-------|----------|
| Relative name (`user.txt`) | shorter; you're already in the right folder |
| Full path (`/root/answers/user.txt`) | safer; you're not sure where you are |

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Collect machine information

**Set up:** create `/root/answers` and `cd` into it.

Create these five files inside `/root/answers`:

| File | Contents |
|------|----------|
| `user.txt` | current username |
| `host.txt` | machine hostname |
| `kernel.txt` | kernel name (should be `Linux`) |
| `root-listing.txt` | listing of the top-level `/` directory |
| `os.txt` | contents of `/etc/os-release` |

**Worked answer** (written from the lesson commands, not copied from the site's hidden solution):

```bash
mkdir -p /root/answers
cd /root/answers
whoami > user.txt
hostname > host.txt
uname -s > kernel.txt
ls / > root-listing.txt
cat /etc/os-release > os.txt
```

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Prepare a working folder

Create `/root/work` with three files:

| File | Contents |
|------|----------|
| `me.txt` | your username (single line: `root`) |
| `today.txt` | output of `date` |
| `here.txt` | output of `hostname` |

**Worked answer:**

```bash
mkdir -p /root/work
cd /root/work
whoami > me.txt
date > today.txt
hostname > here.txt
```

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Quick check: what kernel is this?

**Q:** Run `uname -s`. What does it print?

- [ ] Windows
- [ ] BSD
- [ ] Darwin
- [x] **Linux**

*The site checks this against your own lab, so it needs the lab running. Answer taken from the lesson.*

<br>

---

<br>

<a id="quiz-2"></a>

## ❓ Quiz 2: Scenario: logging into an unfamiliar server

**Q:** You've logged in to a server you've never used, and other people depend on it. Nothing is broken. What's your first command?

- [ ] `sudo reboot`
- [ ] `rm -rf` with a wildcard on `/tmp`
- [x] **`whoami` and `hostname`**
- [ ] `ls` in your home folder

**Why:** confirm **which server you're on and which user you're acting as** before doing anything else.

| Option | Why it's wrong |
|--------|----------------|
| `sudo reboot` | needlessly disrupts a live server |
| `rm -rf` on `/tmp` | destructive, and nothing needed cleaning |
| `ls` in home | harmless, but doesn't confirm where or who you are |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `whoami` | current user |
| `hostname` | machine name |
| `date` | current date/time |
| `uname -s` / `uname -a` | kernel name / full system info |
| `echo text` | print text |
| `pwd` | current folder |
| `ls [path]` | list contents |
| `cat file` | print file |
| `mkdir [-p] dir` | create directory (`-p` = with parents) |
| `cd [dir]` | change directory (`..` up, `~` or bare = home) |
| `cmd > file` | write output to file (overwrites) |
| `Tab` / `Ctrl+C` / `Ctrl+L` / `↑` | complete / cancel / clear / previous command |
| `cmd --help` | show options |

[↑ Back to top](#top)
