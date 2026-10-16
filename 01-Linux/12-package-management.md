<a id="top"></a>

# 📦 12 - Package management

> **Source:** <https://www.escbash.com/topics/linux-package-management>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [What package management is](#lesson-1) | 📖 Lesson |
| 2 | [apt basics](#lesson-2) | 📖 Lesson |
| 3 | [Searching and inspecting packages](#lesson-3) | 📖 Lesson |
| 4 | [Install a small tool](#task-1) | 🛠️ Task |
| 5 | [Find which package owns a file](#task-2) | 🛠️ Task |
| 6 | [Scenario: installing nginx on a fresh server](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: What package management is

A **package** is a compiled program plus everything it needs (files, dependencies, config, man page) bundled into one archive installable in one step. A **package manager** installs, updates and removes packages and tracks dependencies for you.

Without one, installing nginx means downloading source and dependencies, compiling, copying files into place, and repeating it all for every security patch. With one, it's a single command.

### The two big families

| Family | Distros | Package format | Tools |
|--------|---------|----------------|-------|
| Debian | Debian / Ubuntu / Mint | `.deb` | `apt` (low level: `dpkg`) |
| Red Hat | RHEL / Fedora / Amazon Linux / CentOS | `.rpm` | `dnf` (older `yum`) |

Commands differ, concepts don't: if you know `apt`, you can learn `dnf` in an hour. This topic covers `apt` (Ubuntu is the usual cloud VM default).

### Where packages come from

`apt` reads **repositories** (URLs where packages live) from `/etc/apt/sources.list` and files in `/etc/apt/sources.list.d/`. Ubuntu ships with its official repos configured, so `apt install nginx` pulls nginx and every dependency in one command.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: apt basics

Five commands cover almost everything.

### `apt update`: refresh the package list

```bash
sudo apt update
```

Installs **nothing**. It downloads a fresh copy of each repo's package index (what's available, at which version). The local copy drifts out of date as repos publish new versions.

- Run it before any install on a box you haven't touched in a while; a stale index can make `apt install` fetch a replaced version and fail with a **404**.
- It needs network access, so it can fail on a machine with no internet.

### `apt install`: add a package

```bash
sudo apt install htop
sudo apt install -y htop                # skip the "Y/n" confirmation
sudo apt install -y htop curl vim       # several at once
```

> **❗ Important:** `-y` is critical in scripts. Without it, apt waits for a keypress a script can't provide.

### Upgrading

```bash
sudo apt upgrade           # upgrade every installed package
sudo apt upgrade htop      # only htop
```

On production you'd usually schedule upgrades, not run them ad hoc.

### Removing: three levels

```bash
sudo apt remove htop       # remove the package, keep its config files
sudo apt purge htop        # remove package AND config files
sudo apt autoremove        # clean up unused dependencies
```

`purge` when you're truly done; `remove` keeps config so a reinstall reuses your settings.

### The update-then-install pattern

Within the first five minutes on any new server:

```bash
sudo apt update
sudo apt install -y htop
```

Refresh first, then install against the fresh list. Often joined with `&&`, but two lines do the same and read more easily while learning.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Searching and inspecting packages

Two kinds of question: *what package do I want?* and *what is this file / what did this package install?*

| Tools | Look at |
|-------|---------|
| `apt search`, `apt show` | the **repo package list** (what's available to install) |
| `dpkg` commands | what's **actually installed** on this machine now |

### `apt search`: find a package by keyword

```bash
apt search nginx
```

Every package whose name or description mentions it, with a one-line summary. Reads the same local list `apt update` refreshes, so a stale list gives stale results.

### `apt show`: details on one package

```bash
apt show nginx
```

Description, version, dependencies, sizes. Look for `Version:` (what apt would install) and `Depends:` (what comes along).

### `dpkg -l`: what's installed here

```bash
dpkg -l | grep nginx
```

```
ii  nginx  1.24.0-...  amd64  small, powerful, scalable web server
```

`ii` = installed and configured.

### `dpkg -L`: which files a package installed

```bash
dpkg -L nginx
```

```
/usr/sbin/nginx
/etc/nginx/nginx.conf
/usr/share/nginx
```

Answers "where does this program's config live?" without guessing.

### `dpkg -S`: which package owns a file

```bash
dpkg -S /usr/sbin/nginx
```

```
nginx: /usr/sbin/nginx
```

The reverse lookup: file path → package name. Useful when a binary shows up and you want its origin.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Install a small tool

Install `tree` with `apt`, then use it.

| Requirement | Detail |
|-------------|--------|
| package | `tree` installed |
| `/root/answers/tree-version.txt` | output of `tree`'s version flag |
| `/root/answers/tree-output.txt` | tree view of `/root`, a couple of levels deep |

Refresh the package lists first and use the non-interactive flag. Run the install even if `tree` is already present; if `apt` can't reach the repos, refresh and retry.

**Solution (from the site):**

```bash
apt-get update
apt-get install -y tree

mkdir -p /root/answers
tree --version > /root/answers/tree-version.txt
tree -L 2 /root > /root/answers/tree-output.txt
```

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Find which package owns a file

Two recurring DevOps questions:

1. "This binary is here. Which package installed it?" → `dpkg -S`
2. "Which files does this package own?" → `dpkg -L`

Use both on `/bin/ls` and its package:

| File | Contents |
|------|----------|
| `/root/who-owns-ls.txt` | `dpkg -S /bin/ls` output, must show `coreutils` |
| `/root/coreutils-files.txt` | every file installed by `coreutils`, including `/bin/ls` or `/usr/bin/ls` |

**Solution (from the site):**

The lab image was unpacked without coreutils' dpkg metadata, so reinstalling restores its entry. Mirrors have off days, so retry:

```bash
for try in 1 2 3; do apt-get update && break; sleep 10; done
```

```bash
for try in 1 2 3; do apt-get install -y --reinstall coreutils </dev/null && break; sleep 10; done
```

`</dev/null` stops dpkg waiting on the terminal for input. Then:

```bash
dpkg -S /bin/ls > /root/who-owns-ls.txt 2>/dev/null || dpkg -S "$(readlink -f /bin/ls)" > /root/who-owns-ls.txt
dpkg -L coreutils > /root/coreutils-files.txt
```

> **📝 Note:** `/bin` is a symlink to `/usr/bin` on modern Ubuntu, so if `/bin/ls` isn't in the database, query the canonical path via `readlink -f`.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: installing nginx on a fresh server

**Q:** You've logged into a brand-new Ubuntu server and need nginx installed and running. What's the correct command sequence?

- [x] **`sudo apt update && sudo apt install -y nginx`**
- [ ] `sudo apt install -y nginx`
- [ ] Download the source tarball and run `./configure && make && make install`
- [ ] `sudo apt install nginx` (no `-y`, no `apt update`)

**Why:** `apt update` refreshes the index so `install` resolves nginx and its dependencies against a **current** list (on a brand-new server it may be empty or months stale → 404s). `-y` avoids a confirmation prompt hanging a provisioning script.

| Option | Problem |
|--------|---------|
| no `apt update` | stale index → 404s |
| build from source | you own every future security patch forever |
| no `-y` | prompt hangs scripts |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `sudo apt update` | refresh package index (installs nothing) |
| `sudo apt install -y pkg` | install without prompting |
| `sudo apt upgrade [pkg]` | upgrade all / one |
| `sudo apt remove pkg` | remove, keep config |
| `sudo apt purge pkg` | remove + config |
| `sudo apt autoremove` | drop unused dependencies |
| `apt search word` | find packages by keyword |
| `apt show pkg` | version, dependencies, sizes |
| `dpkg -l \| grep pkg` | is it installed? (`ii`) |
| `dpkg -L pkg` | files a package installed |
| `dpkg -S /path/file` | package that owns a file |

| Family | Format | Tool |
|--------|--------|------|
| Debian/Ubuntu | `.deb` | `apt` / `dpkg` |
| RHEL/Fedora/Amazon | `.rpm` | `dnf` / `yum` |

<br>

[↑ Back to top](#top)
