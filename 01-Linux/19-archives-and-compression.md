<a id="top"></a>

# 🗜️ 19 - Archives and compression

> **Source:** <https://www.escbash.com/topics/linux-archives-and-compression>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Archives and compression are two different jobs](#lesson-1) | 📖 Lesson |
| 2 | [Creating and listing archives](#lesson-2) | 📖 Lesson |
| 3 | [Extracting where you meant to](#lesson-3) | 📖 Lesson |
| 4 | [Package a release artifact](#task-1) | 🛠️ Task |
| 5 | [Inspect and unpack a delivery](#task-2) | 🛠️ Task |
| 6 | [Scenario: a tarball from a vendor](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: Archives and compression are two different jobs

`nginx-1.27.tar.gz` is **two tools stacked**; confusing them is why people guess at flags for years.

| Job | Tool | What it does |
|-----|------|--------------|
| **Bundling** | `tar` | many files/folders → one file, preserving names, permissions, timestamps and structure. Does **not** shrink anything ("tape archive") |
| **Squeezing** | `gzip` | rewrites **one** file smaller. Doesn't bundle; knows nothing about folders |

### Bundling with `tar`

```bash
mkdir -p /root/demo/config
echo "print('hello')" > /root/demo/app.py
echo "port=8080" > /root/demo/config/settings.conf
tar -cf /root/demo.tar /root/demo
ls -l /root/demo.tar
```

One file now holds the folder; size ≈ the sum of what went in, plus bookkeeping.

### Squeezing with `gzip`

```bash
gzip /root/demo.tar          # replaces demo.tar with demo.tar.gz
ls -l /root/demo.tar.gz
gunzip /root/demo.tar.gz     # reverses it
```

`gzip` **replaces** its input and handles a single file only, which is why the two tools are used together.

### Reading the file names

| Name | What it is |
|------|------------|
| `backup.tar` | a bundle, not compressed |
| `app.log.gz` | one compressed file, no bundle |
| `release.tar.gz` | a bundle that was then compressed |
| `release.tgz` | same, shorter name |
| `source.tar.bz2`, `source.tar.xz` | same idea, different compressor |

In practice you never run them separately: `tar` calls the compressor for you (the `z` flag).

### Why DevOps is full of these

- **Release artifacts** ship as one `.tar.gz` so a deploy downloads one thing.
- **Backups** become one movable file.
- **Rotated logs** are gzipped in place (`app.log.1.gz`); `zcat` reads them without unpacking.
- **Container image layers** are tarballs underneath.

Text compresses extremely well: logs or source routinely shrink to ~1/5, meaning a fifth of the transfer time and storage cost.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Creating and listing archives

`tar` is really **four verbs plus a few options**.

### The four verbs (pick exactly one)

| Flag | Job |
|------|-----|
| `-c` | **c**reate an archive |
| `-x` | e**x**tract |
| `-t` | lis**t** contents without extracting |
| `-r` | append to an existing archive |

### Options

| Flag | Meaning |
|------|---------|
| `-f FILE` | the archive file, nearly always required |
| `-z` | run through gzip |
| `-v` | verbose: print each file |
| `-C DIR` | change into DIR first |

> **⚠️ Warning:** `-f` must come **last** among bundled letters, because the filename follows it. `tar -czf out.tar.gz src` works; `tar -cfz out.tar.gz src` doesn't (`z` is read as the filename).

### Creating a compressed archive

```bash
mkdir -p /root/site/css
echo "<h1>esc bash</h1>" > /root/site/index.html
echo "body { margin: 0 }" > /root/site/css/style.css

tar -czvf /root/site.tar.gz -C /root site
```

```
site/
site/index.html
site/css/
site/css/style.css
```

Read it as: *create, gzip, verbose, into `/root/site.tar.gz`, having first changed into `/root`, archiving `site`.*

### Why `-C` matters when creating

`tar` stores paths exactly as typed:

```bash
tar -czf /root/bad.tar.gz /root/site
tar -tzf /root/bad.tar.gz
# root/site/
# root/site/index.html
```

Every entry carries a `root/` prefix, so extracting gives a stray `root/` folder. **Habit:** `cd` (or `-C`) to the parent and name the folder relatively, so the archive has one clean top-level directory.

### Look before you leap: `-t`

```bash
tar -tzf /root/site.tar.gz
```

Cheap, safe, reads nothing onto disk. Do it **every time** before extracting something you didn't create. It answers: *one tidy top-level folder?* and *any absolute paths or `../`?*

### Excluding files

Logs, caches and `.git` rarely belong in a release:

```bash
tar -czf /root/site.tar.gz --exclude='*.log' --exclude='.git' -C /root site
```

Quote the pattern, or the shell expands `*.log` before `tar` sees it.

### Compressing a single file

```bash
gzip /root/site/index.html           # → index.html.gz, original gone
gunzip /root/site/index.html.gz      # back again
gzip -k /root/site/index.html        # -k keeps the original too
```

Read gzipped text without unpacking (great for rotated logs):

```bash
zcat /var/log/app.log.1.gz | tail -20
zgrep "ERROR" /var/log/app.log.1.gz
```

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Extracting where you meant to

Creating is forgiving; **extracting is where servers get messy**: `tar -xzf` writes into your *current* directory and will scatter hundreds of files across it.

### The basic extract

```bash
tar -xzf /root/site.tar.gz
```

Everything lands relative to where you're standing. Run from `/`, you just wrote into `/`.

### Always name the destination with `-C`

```bash
mkdir -p /var/www
tar -xzf /root/site.tar.gz -C /var/www
```

Contents land under `/var/www` no matter where you are. The directory **must already exist**: `tar` won't create it. Make `-C` a reflex and the "where did all these files come from?" class of incident disappears.

### The tarball with no top-level folder

A well-made archive:

```
site/
site/index.html
site/css/style.css
```

A badly made one holds files **loose** at the top:

```
index.html
css/style.css
README.md
```

Extract that into a folder you care about and the files mix in, silently overwriting collisions: a **tarball bomb**. Hence `tar -tzf` first. If the listing looks loose, extract into an empty directory of your own:

```bash
mkdir -p /tmp/unpack
tar -xzf /root/mystery.tar.gz -C /tmp/unpack
ls /tmp/unpack
```

Look, then move it where it belongs.

### Dropping a wrapper folder: `--strip-components`

An archive's contents sit inside a version-stamped folder you don't want:

```
nginx-1.27.0/
nginx-1.27.0/conf/
nginx-1.27.0/src/
```

```bash
tar -xzf nginx-1.27.0.tar.gz -C /opt/nginx --strip-components=1
```

Removes the first path element from every entry, so `conf/` and `src/` land directly in `/opt/nginx`. Very common in Dockerfiles and install scripts.

### Extracting one file

Name the path exactly as in the listing:

```bash
tar -tzf /root/site.tar.gz
tar -xzf /root/site.tar.gz -C /tmp site/index.html
```

Handy for pulling one config out of a 2 GB backup.

### Extracting doesn't consume the archive

Unlike bare `gzip`, `tar -x` leaves the `.tar.gz` where it was. Deleting it is a separate, deliberate decision.

### The habit, in three lines

```bash
tar -tzf archive.tar.gz | head                    # 1. look inside
mkdir -p /path/to/destination                     # 2. make the destination
tar -xzf archive.tar.gz -C /path/to/destination   # 3. extract there
```

**Look, prepare, extract.** That order is the whole lesson.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Package a release artifact

Turn the app folder into a single compressed artifact, without runtime files.

**Set up first:**

```bash
mkdir -p /root/app/config
echo "print('hello from esc bash')" > /root/app/app.py
echo "port=8080" > /root/app/config/settings.conf
echo "2026-08-11 request served" > /root/app/app.log
```

**Requirements:**

1. Create `/root/releases`.
2. Build a gzip archive `/root/releases/app-v1.tar.gz` containing the `app` folder.
3. Leave `app.log` **out**: logs belong to the running machine, not the release.
4. Paths must start at `app/`, **not** `root/app/` (use `-C /root` and name `app` relatively).
5. Leave `/root/app` untouched.

Check: `tar -tzf /root/releases/app-v1.tar.gz` should show `app/`, `app/app.py`, `app/config/`, `app/config/settings.conf` and no `app.log`.

**Solution (from the site):**

```bash
mkdir -p /root/app/config /root/releases
echo "print('hello from esc bash')" > /root/app/app.py
echo "port=8080" > /root/app/config/settings.conf
echo "2026-08-11 request served" > /root/app/app.log
tar -czf /root/releases/app-v1.tar.gz --exclude='app/app.log' -C /root app
tar -tzf /root/releases/app-v1.tar.gz
```

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Inspect and unpack a delivery

A colleague drops a website build on the server. Look **before** you extract, then put it where the web server expects it.

**Set up first** (stages `/root/incoming/site.tar.gz`):

```bash
mkdir -p /root/incoming /tmp/build/site/css
echo "<h1>esc bash</h1>" > /tmp/build/site/index.html
echo "body { margin: 0 }" > /tmp/build/site/css/style.css
tar -czf /root/incoming/site.tar.gz -C /tmp/build site
rm -rf /tmp/build
```

**Do:**

1. List the archive **without extracting** and save it to `/root/incoming/site-listing.txt`. Is there one tidy top-level folder, or loose files?
2. Create `/var/www`.
3. Extract so you get `/var/www/site/index.html` and `/var/www/site/css/style.css`.
4. Nothing may be extracted into `/root` (clean up any stray folder).
5. Leave `/root/incoming/site.tar.gz` in place.

**Check your understanding:** you ran the extract from `/root` without a destination and got `/root/site`. Why? **Because `tar -x` writes relative to the current directory; the flag `-C /var/www` would have sent it to `/var/www` regardless.**

**Solution (from the site):**

```bash
tar -tzf /root/incoming/site.tar.gz > /root/incoming/site-listing.txt
mkdir -p /var/www
tar -xzf /root/incoming/site.tar.gz -C /var/www
```

(The site's full solution also recreates the archive and removes any stray `/root/site` first, as above.)

Expected: `cat /var/www/site/index.html` → `<h1>esc bash</h1>`.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: a tarball from a vendor

**Q:** A vendor emails `agent-bundle.tar.gz` and says "just unpack it into `/etc/vendor`". You've never seen it, and you're root on a production box. What do you do first?

- [x] **`tar -tzf agent-bundle.tar.gz | head -30`, then extract with `-C` into the destination once the listing looks safe**
- [ ] `mkdir -p /tmp/vendor && tar -xzf agent-bundle.tar.gz -C /tmp/vendor`, look at what appeared, then move it into `/etc/vendor`
- [ ] `cd /etc/vendor && tar -xzf ~/agent-bundle.tar.gz`
- [ ] `cd / && tar -xzf ~/agent-bundle.tar.gz`, since the archive surely stores full paths like `etc/vendor/...`

**Why:** **look, then act.** Listing is read-only and answers the two questions that matter: *one tidy top-level folder?* and *any absolute paths or `../`?* Then `tar -xzf ... -C /etc/vendor` puts it exactly where intended wherever you're standing.

The reflex to build: **look → make the destination → extract with `-C`**, in that order, whenever you didn't create the archive.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `tar -czf out.tar.gz -C /parent folder` | create gzip archive with clean relative paths |
| `tar -czf out.tar.gz --exclude='*.log' -C /parent folder` | skip files (quote the pattern) |
| `tar -tzf archive.tar.gz` | **list** contents (safe, read-only) |
| `tar -xzf archive.tar.gz -C /dest` | extract into an existing directory |
| `tar -xzf a.tar.gz -C /dest --strip-components=1` | drop the wrapper folder |
| `tar -xzf a.tar.gz -C /tmp path/in/archive` | extract one file |
| `gzip f` / `gunzip f.gz` / `gzip -k f` | compress / decompress / keep original |
| `zcat f.gz` / `zgrep pat f.gz` | read a compressed file without unpacking |

| Flag | Meaning |
|------|---------|
| `-c` `-x` `-t` `-r` | create / extract / list / append |
| `-f` | archive filename (put last in bundled letters) |
| `-z` | gzip |
| `-v` | verbose |
| `-C` | change directory first |

**Safe-extract habit:** `tar -tzf` → `mkdir -p dest` → `tar -xzf ... -C dest`.

<br>

[↑ Back to top](#top)
