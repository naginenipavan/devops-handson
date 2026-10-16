<a id="top"></a>

# 🗂️ 05 - Managing files

> **Source:** <https://www.escbash.com/topics/linux-managing-files>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Creating, copying, and moving files](#lesson-1) | 📖 Lesson |
| 2 | [Removing files safely](#lesson-2) | 📖 Lesson |
| 3 | [Wildcards and globs](#lesson-3) | 📖 Lesson |
| 4 | [Organize a project folder](#task-1) | 🛠️ Task |
| 5 | [Rename and archive a backup folder](#task-2) | 🛠️ Task |
| 6 | [Scenario: cleaning temp files from a folder](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: Creating, copying, and moving files

Three commands cover most of the "put a file here" moves.

### `touch`: create an empty file

```bash
touch /root/notes.txt
```

If the file already exists, `touch` only updates its modification time and leaves the contents alone.

### `cp`: copy

```bash
cp /etc/hostname /tmp/hostname-backup
cp /etc/hostname /tmp/                 # destination is a folder: same name inside it
cp -r /etc/apt /tmp/apt-backup         # copying a folder needs -r (recursive)
```

Without `-r`, `cp` refuses to copy directories.

### `mv`: move **and** rename

Linux doesn't distinguish the two: renaming is just moving to a new name in the same folder.

```bash
mv /tmp/hostname-backup /tmp/hostname.txt   # rename
mv /tmp/hostname.txt /root/                 # move to /root
```

Unlike `cp`, `mv` handles folders **without `-r`**. Moving a folder just updates a pointer, so it's fast even for huge trees.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Removing files safely

`rm` deletes files. It's the simplest command in this topic and the most dangerous.

### The basics

```bash
rm /root/notes.txt                       # delete a file
rm /tmp/hostname /root/hostname.txt      # delete several
rm -r /tmp/apt-backup                    # -r walks into subfolders (needed for folders)
```

`-f` forces the delete without prompting, even for write-protected files. With `-r` it removes a whole folder without asking:

```bash
mkdir /tmp/scratch
rm -rf /tmp/scratch                      # recursive AND forced
```

### ⚠️ The `rm -rf` trap

> **⚠️ Warning:** `rm -rf` on a real server is the mistake that gets people fired.
> - **There's no trash.** Files are gone the second `rm` finishes.
> - **It's typo-friendly.** A stray space turns a targeted delete into deleting your whole OS.

These two commands are one keystroke apart:

```bash
rm -rf /root/oldproject      # deletes one folder
rm -rf / root/oldproject     # deletes "/" (everything!) and then tries root/oldproject
```

Habits to build before pressing Enter on any `rm`:

1. **Read the path twice**, especially the leading `/`.
2. If deleting a folder, run **`ls` on it first** to see what's inside.
3. **Prefer specific paths over wildcards:**

```bash
rm /var/log/oldapp/2023.log     # one exact file
rm /var/log/oldapp/*            # every file in the folder
```

### `-i`: a safety net

`-i` (interactive) asks for confirmation before each delete.

```bash
touch /tmp/scratch.txt
rm -i /tmp/scratch.txt
```

Annoying for batch jobs, but a sensible default when you're new or the delete matters.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Wildcards and globs

A **glob** names many files at once with a pattern. The shell expands the pattern into matching filenames **before the command runs**.

| Pattern | Matches |
|---------|---------|
| `*` | any number of characters (including zero) |
| `?` | exactly one character |

### Try it

```bash
mkdir -p /tmp/globs
cd /tmp/globs
touch a.log app.log notes.txt

ls *.log      # a.log app.log        (any name ending in .log)
ls ?.log      # a.log                (one character, then .log; app.log has three)
ls *          # a.log app.log notes.txt
```

### Common uses

```bash
touch server.conf db.conf
touch old_data.txt old_report.txt
touch access.log.gz error.log.gz
```

```bash
# copy every .conf file
mkdir /tmp/globs/backup
cp *.conf /tmp/globs/backup/
ls /tmp/globs/backup/            # db.conf server.conf

# move every file starting with old_
mkdir /tmp/globs/archive
mv old_* /tmp/globs/archive/
ls /tmp/globs/archive/           # old_data.txt old_report.txt

# list every gzipped file
ls -la *.gz                      # access.log.gz error.log.gz
```

### ⚠️ The `rm` + glob trap

The shell expands globs **before** the command sees them, so a slip widens the match silently:

```bash
rm *.tmp       # removes .tmp files. Good.
rm * .tmp      # removes EVERYTHING in the folder AND tries to remove ".tmp"
```

> **💡 Tip:** Always **preview** a glob before deleting with it:
> ```bash
> ls *.tmp     # what will get matched?
> rm *.tmp     # now delete
> ```

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Organize a project folder

Final layout under `/root/webapp`:

```
/root/webapp/
├── config.yaml
├── README.txt
└── backup/
    └── config.yaml
```

Steps:

1. Create `/root/webapp` and `/root/webapp/backup`.
2. Create an empty `/root/webapp/config.yaml`.
3. Copy it to `/root/webapp/backup/config.yaml`.
4. Create `/root/webapp/README.md`, then **rename** it to `README.txt` (the old `README.md` must no longer exist).
5. Create placeholder files matching `/root/webapp/log-*.old`, then remove every `.old` file in one command using a wildcard.

**Solution (from the site):**

```bash
mkdir -p /root/webapp/backup
touch /root/webapp/config.yaml
cp /root/webapp/config.yaml /root/webapp/backup/config.yaml
touch /root/webapp/README.md
mv /root/webapp/README.md /root/webapp/README.txt
touch /root/webapp/log-1.old /root/webapp/log-2.old
rm -f /root/webapp/*.old
```

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Rename and archive a backup folder

Move an old backup into an archive under a timestamped name.

1. Create `/root/backup` with `a.txt`, `b.txt`, `c.txt` (contents don't matter).
2. Create `/root/archive`.
3. Move `/root/backup` to `/root/archive/2026-07-17` in a **single step**. All three files must end up inside.
4. After the move, `/root/backup` must no longer exist.

`mv` works on folders directly (no `-r`). The files come along with the folder.

**Solution (from the site):**

```bash
mkdir -p /root/backup
touch /root/backup/a.txt /root/backup/b.txt /root/backup/c.txt
mkdir -p /root/archive
mv /root/backup /root/archive/2026-07-17
```

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: cleaning temp files from a folder

**Q:** `/root/webapp` holds hundreds of `*.tmp` files plus a few important `*.log` files you need to keep. You want to remove only the `.tmp` files. What's the safest way?

- [ ] Force-remove everything under `/root/webapp` with `rm -rf` and a wildcard
- [ ] `cd /root/webapp && rm -rf *`
- [ ] `rm /root/webapp/*.tmp` directly
- [x] **`ls /root/webapp/*.tmp` first to see the match, then `rm /root/webapp/*.tmp`**

**Why:** the glob is expanded before `rm` runs, so **preview it with `ls`** and only then delete. The first two destroy the `.log` files you wanted to keep; the third works but skips the free safety check.

*Confirmed against the site's solution: preview with `ls`, then `rm`. Broad `rm -rf` variants destroy the `.log` files you were told to keep, and there is no trash.*

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `touch file` | create empty file (or update its mtime) |
| `cp src dst` | copy a file |
| `cp -r src dst` | copy a folder |
| `mv src dst` | move **or** rename (files and folders, no `-r`) |
| `rm file` | delete a file |
| `rm -r dir` | delete a folder and its contents |
| `rm -rf dir` | delete forcibly, no prompts (**dangerous**) |
| `rm -i file` | ask before each delete |
| `*` | any number of characters |
| `?` | exactly one character |
| `ls pattern` then `rm pattern` | preview a glob before deleting |

**Safety habits:** read the path twice · `ls` a folder before deleting it · prefer exact paths over wildcards · never leave a stray space in `rm -rf`.

<br>

[↑ Back to top](#top)
