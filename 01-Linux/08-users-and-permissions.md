<a id="top"></a>

# 🔐 08 - Users and permissions

> **Source:** <https://www.escbash.com/topics/linux-users-and-permissions>
> **Contents:** 4 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [The ten characters](#lesson-1) | 📖 Lesson |
| 2 | [Users and groups](#lesson-2) | 📖 Lesson |
| 3 | [Reading permissions in ls -l](#lesson-3) | 📖 Lesson |
| 4 | [chmod and chown](#lesson-4) | 📖 Lesson |
| 5 | [Set permissions on a script and a secret](#task-1) | 🛠️ Task |
| 6 | [Set up an SSH key with correct permissions](#task-2) | 🛠️ Task |
| 7 | [Scenario: an SSH key that won't work](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: The ten characters

Every file answers `ls -l` with a string like `-rw-r--r--`, and that string decides **who is allowed to touch it**. Don't read it as ten characters: read it as **one, then three, then three, then three**.

```
-  rw-  r--  r--
↑   ↑    ↑    ↑
type owner group other
```

- First character = file type: `-` regular file, `d` directory, `l` symlink.
- The next nine = three identical triplets (owner, group, other), each in the order **read, write, execute**, with `-` where denied.

### The numbers are the same thing

**r = 4, w = 2, x = 1.** Add up one triplet to get one digit, so `chmod 755` and `-rwxr-xr-x` are two spellings of the same thing.

| Mode | Bits | Use case |
|------|------|----------|
| `755` | `rwxr-xr-x` | scripts and executables anyone can run |
| `644` | `rw-r--r--` | configs and text files anyone can read |
| `700` | `rwx------` | a directory only the owner should enter |
| `600` | `rw-------` | secrets: SSH keys, API tokens, `.env` files |

### Why it bites

SSH refuses to use a private key that anyone else can read. A key at `644` gives `Permissions 0644 for '~/.ssh/id_rsa' are too open` and a rejected login, not because the key is wrong, but because **four digits are**. Fix: `chmod 600` on the key and `chmod 700` on `~/.ssh`.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Users and groups

Every process runs as some user; every file belongs to a user. That's how the system decides who can read what.

### Who am I

```bash
whoami     # just the username
id         # username, user ID, primary group, secondary groups
```

Typical `id` output: `uid=0(root) gid=0(root) groups=0(root)`.

### Users live in `/etc/passwd`

One colon-separated line per user:

```
username:x:UID:GID:full name:home directory:login shell
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
```

The `x` is a leftover from when the password hash lived here. Hashes are now in `/etc/shadow`, readable only by root.

### Two kinds of user

| Kind | Notes |
|------|-------|
| Regular | owns a home folder under `/home/<name>`, runs everyday commands |
| `root` | superuser: UID 0, unrestricted, can undo anything |

On production servers you almost never log in as root. Log in as yourself and use `sudo` when root is actually needed:

```bash
sudo apt update     # run this one command as root
sudo -i             # interactive root shell
```

(In the lab you're already root, so `sudo` is a no-op, but the habit matters everywhere else.)

### Groups

Groups let several users share access. Each user has one **primary group** (the GID in `/etc/passwd`) and any number of **secondary groups** listed in `/etc/group`:

```
groupname:x:GID:comma,separated,members
sudo:x:27:ubuntu
```

`ubuntu` is a member of `sudo`, and that membership is what lets it run `sudo` commands.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Reading permissions in `ls -l`

```bash
ls -l /etc/hosts
```

```
-rw-r--r-- 1 root root 178 Feb 10 14:22 /etc/hosts
```

### First character: file type

| Char | Type |
|------|------|
| `-` | regular file |
| `d` | directory |
| `l` | symbolic link |
| `c`, `b`, `s`, `p` | special files (later) |

### Next nine: three triplets

`rw- r-- r--` → **owner**, **group**, **other** (everyone else).

| Letter | On a file | On a directory |
|--------|-----------|----------------|
| `r` | read the file | see the list of names inside |
| `w` | write to the file | create, rename, delete entries |
| `x` | execute the file | enter it (`cd`) and access files by name |
| `-` | denied | denied |

So `-rw-r--r--` = regular file; owner reads/writes; group reads only; others read only.

> **📝 Note:** A directory with `r` but no `x` is a half-locked state: you can see names but not reach the files. Almost every useful directory has both.

### A directory in a real listing

`-d` describes the directory itself instead of its contents:

```bash
mkdir -p /root/demo
ls -ld /root/demo
```

```
drwxr-xr-x 2 root root 4096 Feb 10 14:22 /root/demo
```

`d` = directory; `rwx` owner (list, add/delete, enter); `r-x` group; `r-x` other. This is a normal directory: the owner controls the contents, everyone else can look and enter but not change.

### Owner and group columns

```
-rw-r--r-- 1 root root 178 Feb 10 14:22 /etc/hosts
           ^^^^^^^^^
           owner group
```

Reading permissions is a **two-step check**: which triplet applies to you (owner, group, or neither?), then look for `r/w/x` in that triplet.

<br>

---

<br>

<a id="lesson-4"></a>

## 📖 Lesson 4: `chmod` and `chown`

`chmod` changes permissions; `chown` changes ownership.

### `chmod`, the symbolic way

```bash
touch /root/report.txt
ls -l /root/report.txt        # starts at 644: -rw-r--r--
```

Read it as a mini-sentence: **who, operator, permission**.

| Who | Operator | Permission |
|-----|----------|------------|
| `u` user/owner | `+` add | `r` |
| `g` group | `-` remove | `w` |
| `o` other | `=` set exactly | `x` |
| `a` all | | |

```bash
chmod u+x /root/report.txt     # owner: rw- → rwx, nothing else moves
chmod g-w /root/report.txt     # remove write from group
chmod o=r /root/report.txt     # set other to read only
chmod a+r /root/report.txt     # give everyone read
```

### `chmod`, the octal way

Each digit = sum of read (4), write (2), execute (1):

| Digit | Meaning |
|-------|---------|
| 7 | `rwx` |
| 6 | `rw-` |
| 5 | `r-x` |
| 4 | `r--` |
| 0 | `---` |

```bash
chmod 755 /root/report.txt      # owner rwx, group r-x, other r-x
```

Three digits, one call, unambiguous. Learn the four presets (**755, 644, 700, 600**) and you can set the right mode on ~90% of files.

### `chown` for ownership

```bash
chown root /root/report.txt          # set the owner
chown root:root /root/report.txt     # set owner and group together
chown :root /root/report.txt         # set only the group
```

- `-R` applies to a folder and everything inside (e.g. handing a web root from one service to another).
- On a real server name a real user/group, e.g. `chown alice:developers deploy.sh`, with `sudo` in front when moving files between services.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Set permissions on a script and a secret

Create `/root/mission` with two files:

| File | Contents | Mode |
|------|----------|------|
| `/root/mission/deploy.sh` | shell script with a proper shebang and at least one command that prints output; runnable by its path | `755` |
| `/root/mission/secrets.env` | a single line beginning `API_KEY=` followed by any value | `600` |

**Solution (from the site):**

```bash
mkdir -p /root/mission
cat > /root/mission/deploy.sh <<'EOF'
#!/bin/bash
echo "deploy: ok"
EOF
chmod 755 /root/mission/deploy.sh
echo "API_KEY=placeholder-value-123" > /root/mission/secrets.env
chmod 600 /root/mission/secrets.env
```

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Set up an SSH key with correct permissions

SSH refuses keys with loose permissions ("permissions are too open"). Set it up correctly from the start:

| Path | Type | Mode |
|------|------|------|
| `/root/.ssh` | folder | `700` |
| `/root/.ssh/id_rsa` | file (placeholder contents) | `600` |
| `/root/.ssh/id_rsa.pub` | file (placeholder contents) | `644` |

`/root/.ssh` already exists in the lab, so a plain `mkdir` fails with `File exists`. Use `mkdir -p`: it creates the folder if missing and stays quiet if present, which is why scripts use it.

**Solution (from the site):**

```bash
mkdir -p /root/.ssh
chmod 700 /root/.ssh
echo "placeholder private key material" > /root/.ssh/id_rsa
chmod 600 /root/.ssh/id_rsa
echo "ssh-rsa placeholder-public-key" > /root/.ssh/id_rsa.pub
chmod 644 /root/.ssh/id_rsa.pub
```

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: an SSH key that won't work

**Q:** A teammate's new SSH key isn't letting them log in. They copied the public key into `~/.ssh/authorized_keys` on the server. What do you check first?

- [x] **Check that `~/.ssh` is `700` and `~/.ssh/authorized_keys` is `600`, and fix them if not**
- [ ] Check that the public key was pasted correctly, with no line breaks in the middle
- [ ] Restart the SSH service on the server
- [ ] Run `chmod -R 777 ~/.ssh` to make sure everything is accessible

**Why:** `sshd` refuses to trust key files that other users could read or modify, so a looser folder/file makes it **silently ignore the key**, the most common cause of "my key isn't working". The check takes seconds and needs no restart, because `sshd` re-reads `authorized_keys` on every login attempt.

| Option | Verdict |
|--------|---------|
| mangled paste | real, but rarer |
| restart `sshd` | changes nothing about how keys are evaluated |
| `chmod 777` | guarantees the rejection **and** exposes keys to every user |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command / concept | Meaning |
|-------------------|---------|
| `whoami` / `id` | username / uid, gid, groups |
| `/etc/passwd` | one line per user |
| `/etc/shadow` | password hashes (root only) |
| `/etc/group` | groups and their members |
| `sudo cmd` / `sudo -i` | run as root / root shell |
| `ls -l` / `ls -ld dir` | show permissions / describe the directory itself |
| `r=4 w=2 x=1` | octal values |
| `chmod u+x file` | add execute for owner |
| `chmod 755 file` | `rwxr-xr-x` |
| `chown user:group file` | change owner and group (`-R` for folders) |

| Mode | Use |
|------|-----|
| `755` | scripts/executables |
| `644` | configs, text files |
| `700` | private directory |
| `600` | secrets, SSH keys |

**SSH rule:** `~/.ssh` = `700`, private key and `authorized_keys` = `600`, public key = `644`.

<br>

[↑ Back to top](#top)
