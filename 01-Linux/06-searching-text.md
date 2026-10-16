<a id="top"></a>

# 🔎 06 - Searching text

> **Source:** <https://www.escbash.com/topics/linux-searching-text>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [grep for finding text in a file](#lesson-1) | 📖 Lesson |
| 2 | [grep across many files](#lesson-2) | 📖 Lesson |
| 3 | [find for locating files](#lesson-3) | 📖 Lesson |
| 4 | [Search /etc with grep and find](#task-1) | 🛠️ Task |
| 5 | [Locate all network config files under /etc](#task-2) | 🛠️ Task |
| 6 | [Scenario: locating a setting in a config folder](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: `grep` for finding text in a file

`grep` searches for a pattern inside a file and prints every line that matches. Logs run to thousands of lines, configs are full of settings you don't care about, and command output scrolls away. `grep` pulls out only the lines that matter.

- Service broke? First move: **grep the logs for `error`**.
- Need to know how a setting is configured? **grep the config file for it**.

### The basic form

```bash
grep pattern file
grep root /etc/passwd      # every line of /etc/passwd containing "root"
```

### Prepare a demo log

```bash
mkdir -p /tmp/grep-demo
cd /tmp/grep-demo
echo "Starting web service on port 8080" > app.log
echo "User alice logged in" >> app.log
echo "ERROR failed to connect to database" >> app.log
echo "Retrying database connection" >> app.log
echo "error disk space low" >> app.log
echo "Service healthy" >> app.log
```

```bash
grep error app.log
```

```
error disk space low
```

Only one line: the `ERROR` line did **not** match, because by default `grep` is case-sensitive.

### The four flags you'll use daily

| Flag | Meaning | Example | Output |
|------|---------|---------|--------|
| `-i` | ignore case | `grep -i error app.log` | both `ERROR ...` and `error ...` lines |
| `-n` | show line numbers | `grep -in error app.log` | `3:ERROR failed to connect to database` / `5:error disk space low` |
| `-c` | count matching lines | `grep -ic error app.log` | `2` |
| `-v` | invert (lines that do **not** match) | `grep -iv error app.log` | the four non-error lines |

Notes:

- `-i` is almost always what you want on logs, since you rarely know the capitalisation.
- `-n` is handy for opening the file at that spot in an editor.
- `-c` answers "how many errors?" without scrolling.
- `-v` is common for stripping comments from a config:

```bash
grep -v '^#' /etc/hosts      # leaves only the actual settings
```

### Combining flags

Flags stack and order doesn't matter: `grep -in`, `grep -ni` and `grep -i -n` are identical.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: `grep` across many files

`-r` (recursive) walks every file under a folder.

```bash
grep -r "listen" /etc/nginx/
```

Every match prints as `filename:line`: the answer to "which file mentioned this?".

### Narrowing the search

| Flag | Effect |
|------|--------|
| `-l` | print only the **filenames** with at least one match |
| `--include='*.conf'` | restrict to files matching a glob |

```bash
grep -r -l "PermitRootLogin" /etc/            # which files mention this setting
grep -r --include='*.conf' "listen" /etc/     # only .conf files under /etc
```

### Piping into `grep`

`grep` filters **any** stream of text:

```bash
ps aux | grep nginx              # any nginx processes running?
history | grep ssh               # ssh commands from history (empty on a new shell)
ls /etc | grep -i network        # entries in /etc whose name mentions "network"
```

> **💡 Tip:** Pattern to remember: *something produces lines, `grep` keeps the ones that matter.*

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: `find` for locating files

`grep` searches **inside** files. `find` locates the **files themselves**.

```bash
find <path> [conditions]
```

| Condition | Meaning |
|-----------|---------|
| `-name 'pattern'` | filename, case-sensitive (wildcards work but **must be quoted**) |
| `-iname 'pattern'` | filename, case-insensitive |
| `-type f` | regular files only |
| `-type d` | directories only |

### Everyday recipes

```bash
find /etc -name 'nginx.conf'            # a specific file under /etc
find /etc -name '*.conf'                # every .conf file under /etc
find / -type d -name 'nginx'            # a directory anywhere on the machine
find /var/log -type f -name '*.log'     # regular log files under /var/log
```

### Always quote the pattern

If unquoted, the shell expands the glob **before** `find` sees it: you miss matches or get an error.

```bash
find /etc -name '*.conf'     # correct
find /etc -name *.conf       # wrong, the shell expanded *.conf first
```

### `find` vs `grep`

| Question | Tool |
|----------|------|
| "Where is a file called X?" | `find` |
| "Which files contain the string X?" | `grep -r` |

Sometimes you want both: `find` locates candidates, `grep` searches inside them.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Search /etc with grep and find

Create `/root/answers` if needed, then three files:

| File | Contents |
|------|----------|
| `/root/answers/nologin-count.txt` | a single number: lines in `/etc/passwd` containing `nologin` (accounts that can't log in interactively) |
| `/root/answers/ubuntu-files.txt` | full paths of files under `/etc/apt` that mention `ubuntu`, one per line |
| `/root/answers/conf-count.txt` | a single number: files ending in `.conf` anywhere under `/etc` |

Use `grep` for content searches and `find` for filename searches.

**Solution (from the site):**

```bash
mkdir -p /root/answers
grep -c nologin /etc/passwd > /root/answers/nologin-count.txt
grep -rl ubuntu /etc/apt > /root/answers/ubuntu-files.txt
find /etc -name '*.conf' | wc -l > /root/answers/conf-count.txt
```

- `grep -c` counts matching lines.
- `grep -rl` prints only the names of matching files.
- A quoted `find -name` piped into `wc -l` counts files.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Locate all network config files under /etc

You've taken over a machine and want to know which config files relate to networking. Every file under `/etc` whose name contains `network` (any case: `network`, `NetworkManager`, ...) is a candidate.

Write every matching path to `/root/network-configs.txt`, one per line, restricted to `/etc`.

**Solution (from the site):**

```bash
find /etc -iname '*network*' > /root/network-configs.txt
```

- Start at `/etc` so every result is an `/etc/` path.
- `-iname` makes the match case-insensitive.
- Quote the pattern so the shell doesn't expand it.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: locating a setting in a config folder

**Q:** You're on a server you didn't build and need to find where nginx's `access_log` path is set. The config lives under `/etc/nginx/` but you don't know which file. What's the fastest move?

- [ ] Open `/etc/nginx/nginx.conf` in `less` and press `/access_log`
- [ ] `grep -r access_log /`
- [ ] Print `/etc/nginx/nginx.conf` with `cat` and eyeball it
- [x] **`grep -r "access_log" /etc/nginx/`**

**Why:** it searches every file under the folder in one shot and prints filename plus matching line, so the setting turns up even when it lives in an include file (e.g. under `sites-available/` or `conf.d/`).

| Option | Problem |
|--------|---------|
| `less` / `cat` on one file | only works if you guessed the right file |
| `grep -r ... /` | scans the entire filesystem for a setting whose folder you already know |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `grep pattern file` | lines matching pattern |
| `grep -i` | ignore case |
| `grep -n` | show line numbers |
| `grep -c` | count matching lines |
| `grep -v` | invert: non-matching lines |
| `grep -r pattern dir/` | search recursively (`file:line` output) |
| `grep -rl pattern dir/` | list only the matching filenames |
| `grep -r --include='*.conf' pattern dir/` | only files matching a glob |
| `cmd \| grep pattern` | filter any command's output |
| `find path -name 'x'` | find by name (case-sensitive) |
| `find path -iname 'x'` | find by name (case-insensitive) |
| `find path -type f` / `-type d` | files only / directories only |
| `find ... \| wc -l` | count matching files |

**Rules of thumb:** always quote `find` patterns · `find` = *where is the file*, `grep -r` = *which files contain this*.

<br>

[↑ Back to top](#top)
