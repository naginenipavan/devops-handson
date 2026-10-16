<a id="top"></a>

# 🔗 18 - Links

> **Source:** <https://www.escbash.com/topics/linux-links>
> **Contents:** 2 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [What are links](#lesson-1) | 📖 Lesson |
| 2 | [Using symlinks](#lesson-2) | 📖 Lesson |
| 3 | [Set up a release symlink](#task-1) | 🛠️ Task |
| 4 | [Share one config with a symlink](#task-2) | 🛠️ Task |
| 5 | [Scenario: switching a current-release symlink](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: What are links

A **link** is an alternate name for a file. Linux has two flavours that behave very differently.

### Hard links

Every file is stored at an **inode** (a numeric ID). A file's "name" is a directory entry pointing at that inode. A **hard link** is just another directory entry pointing at the **same inode**.

```bash
echo hi > original.txt      # a file to link to
ln original.txt copy.txt    # hard link
ls -li                      # -i shows the inode number
```

Both names show the same inode: two names for one file. Delete one and the file remains under the other; edit one and both "see" it.

Limitations:

- can't cross filesystems (an inode is only meaningful within one)
- can't point at directories (with rare exceptions)

Useful for deduplication and backup tools; rarely created on purpose in daily DevOps work.

### Symbolic links (symlinks)

A **symlink** is a small file holding a **path** to another file or folder, like a shortcut.

```bash
mkdir -p /root/releases/v2
echo "v2" > /root/releases/v2/version.txt
ln -s /root/releases/v2 /root/current

cat /root/current/version.txt        # v2
```

Key difference: a symlink stores the **target path**, not the inode. It's a signpost, not a second name.

### Telling them apart in a listing

```bash
ls -l /root/current
```

```
lrwxrwxrwx 1 root root 17 Jul 28 10:30 /root/current -> /root/releases/v2
```

| First char | Meaning |
|------------|---------|
| `l` + `->` target | symlink |
| `-` | regular file (**hard links look exactly like regular files**, because they are) |
| `d` | directory |

### When the target is deleted or moved

```bash
rm -r /root/releases/v2
cat /root/current/version.txt
# cat: /root/current/version.txt: No such file or directory
```

The link still exists but points at nothing. A hard link would have survived (it names the inode; data stays until the last name is gone). Moving the target breaks a symlink the same way.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Using symlinks

```
ln -s  target  link_name
```

Read it as "make `link_name` a symbolic link to `target`": the first path already exists, the second is the new name.

> **⚠️ Warning:** The argument order trips up almost everyone. `ln -s` writes the first path into the link exactly as typed, so swapping the arguments makes a link with the wrong name pointing at the wrong place, and it usually **doesn't error**. Always say it: *target first, then the link's name.*

```bash
mkdir -p /root/releases/v2
echo "v2" > /root/releases/v2/version.txt
ln -s /root/releases/v2 /root/current
readlink /root/current        # /root/releases/v2
```

### The "current release" pattern

The most common DevOps use of symlinks: versioned releases in their own folders, with a symlink at whichever is live.

```
/var/www/releases/2026-07-01/
/var/www/releases/2026-07-14/
/var/www/current -> /var/www/releases/2026-07-14
```

- **Deploy:** extract the new release, update the symlink, reload nginx.
- **Rollback:** point the symlink at the previous release.
- No file copies, no downtime.

Same trick for tool versions: Ubuntu's `python3` is a symlink to `python3.12`.

```bash
ls -l /usr/bin/python3
# lrwxrwxrwx 1 root root 10 ... /usr/bin/python3 -> python3.12
```

### Replacing an existing symlink (the classic gotcha)

```bash
mkdir -p /root/releases/v3
echo "v3" > /root/releases/v3/version.txt
ln -s /root/releases/v3 /root/current
readlink /root/current        # /root/releases/v2   ← still v2!
```

Plain `ln -s` didn't overwrite. Because `/root/current` already points at a directory, `ln` followed it and quietly created a stray `v3` link **inside** the v2 folder.

The fix:

```bash
ln -sfn /root/releases/v3 /root/current
readlink /root/current        # /root/releases/v3
```

| Flag | Meaning |
|------|---------|
| `-s` | symbolic |
| `-f` | force: overwrite the link if it exists |
| `-n` | treat the existing link as a plain file: don't follow it into the directory it points at |

> **💡 Tip:** `ln -sfn` is the exact one-liner every zero-downtime deploy script uses to swap the live release.

### Finding what a symlink points at

```bash
readlink /root/current         # the target path it holds
readlink -f /root/current      # follow the whole chain to the real file
```

### Broken symlinks

If the target no longer exists the symlink is **broken**: listings still show it but reading through it fails.

```bash
rm -r /root/releases/v3
cat /root/current/version.txt
# cat: /root/current/version.txt: No such file or directory

find /root -xtype l            # list symlinks whose target doesn't exist
```

`-xtype l` is very useful for bad-deploy cleanup.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Set up a release symlink

Practise the "current release" pattern.

- `/root/releases/v1/version.txt` contains `v1`
- `/root/releases/v2/version.txt` contains `v2`
- `/root/current` is a **symbolic link** ending up at `/root/releases/v2` (so `cat /root/current/version.txt` prints `v2`)
- Practise switching between v1 and v2 along the way; end on v2

**Solution (from the site):**

```bash
mkdir -p /root/releases/v1 /root/releases/v2
echo v1 > /root/releases/v1/version.txt
echo v2 > /root/releases/v2/version.txt
ln -sfn /root/releases/v1 /root/current
ln -sfn /root/releases/v2 /root/current
```

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Share one config with a symlink

Instead of copying a file into each app (and remembering to update both), keep **one real file** and symlink it.

```
/root/shared/config.yaml            (real file, contains "shared")
/root/app1/config.yaml  ->  /root/shared/config.yaml
/root/app2/config.yaml  ->  /root/shared/config.yaml
```

Editing the shared file is then visible to both apps immediately.

**Solution (from the site):**

```bash
mkdir -p /root/shared /root/app1 /root/app2
echo shared > /root/shared/config.yaml
ln -sfn /root/shared/config.yaml /root/app1/config.yaml
ln -sfn /root/shared/config.yaml /root/app2/config.yaml
```

Target first, link name second.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: switching a current-release symlink

**Q:** `/var/www/current` is a symlink to `/var/www/releases/2025-11-04`, a folder cleaned up months ago. You need to point it at `/var/www/releases/2026-07-14`, which exists. What's the right command?

- [x] **`ln -sfn /var/www/releases/2026-07-14 /var/www/current`**
- [ ] `rm /var/www/current && ln -s /var/www/releases/2026-07-14 /var/www/current`
- [ ] `mv /var/www/releases/2026-07-14 /var/www/current`
- [ ] `cp -r /var/www/releases/2026-07-14 /var/www/current`

**Why:** `-f` overwrites the old link and `-n` stops `ln` following it and dropping the new link inside the target directory. Replaces a symlink in one step.

| Option | Problem |
|--------|---------|
| `rm` then `ln -s` | leaves a window where the path is broken |
| `mv` / `cp -r` | destroy the one-folder-per-release pattern that makes rollbacks a pointer update |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `ln target name` | hard link (same inode, same filesystem, files only) |
| `ln -s target name` | symlink (**target first**, then link name) |
| `ln -sfn target name` | create **or replace** a symlink safely |
| `ls -li` | show inode numbers |
| `ls -l` | `l` + `->` marks symlinks |
| `readlink name` / `readlink -f name` | target / full resolved path |
| `find /path -xtype l` | list broken symlinks |

| | Hard link | Symlink |
|--|-----------|---------|
| Stores | inode | a path |
| Survives target deletion | yes | no (breaks) |
| Crosses filesystems | no | yes |
| Can point at a directory | no (rarely) | yes |

**Deploy pattern:** `releases/<version>/` folders + `current -> releases/<live>`; deploy and rollback = `ln -sfn`.

<br>

[↑ Back to top](#top)
