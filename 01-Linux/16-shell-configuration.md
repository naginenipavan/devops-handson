<a id="top"></a>

# 🐚 16 - Shell configuration

> **Source:** <https://www.escbash.com/topics/linux-shell-configuration>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Environment variables](#lesson-1) | 📖 Lesson |
| 2 | [PATH and command lookup](#lesson-2) | 📖 Lesson |
| 3 | [Startup files and aliases](#lesson-3) | 📖 Lesson |
| 4 | [Configure your shell](#task-1) | 🛠️ Task |
| 5 | [Set persistent environment variables](#task-2) | 🛠️ Task |
| 6 | [Scenario: installed a tool, command not found](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: Environment variables

Every shell session carries **environment variables**: name-value pairs the shell (and any program it runs) can read. They carry per-user, per-machine or per-session configuration without touching source code.

### Seeing them

```bash
env               # every exported variable in this session
printenv          # same, slightly different format
echo $HOME        # one specific variable
echo $USER
echo $PATH
```

`$VAR` is how the shell expands a variable's value.

### Ones you'll meet daily

| Variable | Meaning |
|----------|---------|
| `HOME` | your home folder path |
| `USER` | your username |
| `PWD` | the folder you're in |
| `SHELL` | which shell is running |
| `PATH` | where the shell looks for command binaries |
| `LANG` | locale: sorting and message language |

### Setting a variable

```bash
name=alice         # shell variable, NOT exported
echo $name         # alice
```

A plain variable exists only in the current shell; child processes don't see it. Prove it with a fresh child shell:

```bash
name=alice
bash -c 'echo $name'       # prints a blank line

export name=alice
bash -c 'echo $name'       # alice
```

`export` copies the variable into the environment so every program you start can read it.

### The inline form

Set a variable for **one command** only:

```bash
LOG_LEVEL=debug bash -c 'echo $LOG_LEVEL'      # debug
```

Exported to that command, then discarded. Same pattern as `LOG_LEVEL=debug ./deploy.sh`.

### Removing a variable

```bash
unset name
```

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: PATH and command lookup

`PATH` is the most important variable you'll configure: it tells the shell where to find command binaries.

```bash
echo $PATH
```

```
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

A colon-separated list of folders. When you type `ls`, the shell walks the list **left to right** and runs the first executable named `ls`.

### Which file runs?

```bash
type ls          # /usr/bin/ls
which ls         # older command, same idea
command -v ls    # works inside scripts too
```

> **💡 Tip:** Use `command -v` in scripts: it's built into every POSIX shell and doesn't depend on `which` being installed.

### Order matters

**First match wins.** If `/usr/local/bin/python` and `/usr/bin/python` both exist, whichever is earlier in `PATH` runs. So custom tool folders are *prepended*:

```bash
export PATH="$HOME/bin:$PATH"
```

`~/bin` is now at the front and overrides system defaults with the same name.

### When "command not found" hits

```
$ myapp
bash: myapp: command not found
```

Almost always one of two causes:

1. The tool was installed somewhere **not in `PATH`**.
2. It's in a folder that needs a **new shell** to be picked up.

Find where it landed:

```bash
find / -name myapp 2>/dev/null
```

If it's `/opt/myapp/bin/myapp`, that folder isn't in `PATH`; add it.

### Absolute paths always work

```bash
/opt/myapp/bin/myapp
```

They never depend on `PATH`, a handy troubleshooting fallback.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Startup files and aliases

Each login or new shell makes bash read startup files. Editing them makes settings stick.

### The files that matter

| File | When it runs |
|------|--------------|
| `~/.bashrc` | interactive non-login shells (every new terminal tab on a logged-in machine) |
| `~/.profile` or `~/.bash_profile` | at login (SSH into a fresh session) |
| `/etc/bash.bashrc` | system-wide, every user's interactive shells |
| `/etc/profile` | system-wide, at login for every user |

> **💡 Tip:** Modern Ubuntu rule of thumb: put customisations in **`~/.bashrc`**. `~/.profile` sources `.bashrc` for you, so any session picks them up.

### Making changes take effect

1. Open a new terminal / SSH session, or
2. Reload in the current session:

```bash
source ~/.bashrc
. ~/.bashrc        # one-character shorthand
```

`source` runs the file **in the current shell**. `bash ~/.bashrc` would start a child shell, apply the changes there, exit, and leave your session unchanged.

### Aliases

```bash
alias ll='ls -lah'
alias ..='cd ..'
alias grep='grep --color=auto'
alias gs='git status'
```

`alias` alone lists them; `unalias ll` removes one.

### Common `.bashrc` additions

```bash
# Prepend your own bin folder to PATH
export PATH="$HOME/bin:$PATH"

# Handy aliases
alias ll='ls -lah'
alias df='df -h'
alias free='free -h'

# Default editor for tools that read $EDITOR
export EDITOR=nano
```

`ll` gives a full listing, `df`/`free` are human-readable, and anything reading `$EDITOR` (git commits, `crontab -e`, `visudo`) opens nano.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Configure your shell

Add a personal bin folder to `PATH`, drop an executable script into it, and define an alias, **persistently**.

| Item | Requirement |
|------|-------------|
| `/root/bin/hello` | executable script that prints any message |
| `/root/.bashrc` | **append** an export prepending `/root/bin` to `PATH`, plus an `ll` alias |
| result | in a fresh shell, typing `hello` alone runs the script |

**Solution (from the site):**

```bash
mkdir -p /root/bin
cat > /root/bin/hello <<'EOF'
#!/bin/bash
echo "hello from /root/bin"
EOF
chmod +x /root/bin/hello
```

```bash
grep -qxF 'export PATH="$HOME/bin:$PATH"' /root/.bashrc || sed -i '1i export PATH="$HOME/bin:$PATH"' /root/.bashrc
grep -q '^alias ll=' /root/.bashrc || echo "alias ll='ls -lah'" >> /root/.bashrc
```

> **📝 Note:** The default `.bashrc` starts with an "If not running interactively, don't do anything" **guard**, so a `PATH` export at the bottom is never reached when sourced non-interactively. Insert the export at the **top** (`sed -i '1i ...'`). The `grep ... ||` guard keeps a re-run from adding duplicates.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Set persistent environment variables

| Variable | Value |
|----------|-------|
| `EDITOR` | `nano` (so `crontab -e` / `git commit` never drop you into vim) |
| `LANG` | `en_US.UTF-8` (or any valid locale) |

Add both `export` lines to `/root/.bashrc`. After a fresh login shell, `echo $EDITOR` must print `nano`.

**Solution (from the site):**

```bash
grep -qxF 'export EDITOR=nano' /root/.bashrc || sed -i '1i export EDITOR=nano' /root/.bashrc
grep -qxF 'export LANG=en_US.UTF-8' /root/.bashrc || sed -i '1i export LANG=en_US.UTF-8' /root/.bashrc
```

A fresh login shell sources `.bashrc` non-interactively through `.profile`, so lines below the guard never take effect; insert at the top.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: installed a tool, command not found

**Q:** You extracted a release tarball into `/opt/tool/`; the binary is `/opt/tool/bin/tool`. Running `tool` says "command not found". What's the right fix?

- [x] **Edit `~/.bashrc` to add `export PATH="/opt/tool/bin:$PATH"`, then `source ~/.bashrc`**
- [ ] Create a symlink from `/usr/local/bin/tool` to `/opt/tool/bin/tool`
- [ ] Copy the binary to `/usr/bin/`
- [ ] Add `alias tool='/opt/tool/bin/tool'` to `.bashrc`

**Why:** prepending to `PATH` in `~/.bashrc` fixes it **permanently**, covers every binary the tool ships, and `source` applies it now. It's the pattern for every tarball install.

| Option | Verdict |
|--------|---------|
| symlink in `/usr/local/bin` | legitimate per-binary alternative |
| copy to `/usr/bin` | package manager may silently overwrite it on upgrade |
| alias | exists only in interactive shells: cron, services and scripts still fail |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `env` / `printenv` | list exported variables |
| `echo $VAR` | read a variable |
| `VAR=x` / `export VAR=x` | shell-only / exported to child processes |
| `VAR=x cmd` | set for one command only |
| `unset VAR` | remove |
| `echo $PATH` | where commands are searched (left → right, first match wins) |
| `type` / `which` / `command -v` | which file runs |
| `export PATH="/new/bin:$PATH"` | prepend a folder |
| `source ~/.bashrc` (or `. ~/.bashrc`) | reload in current shell |
| `alias ll='ls -lah'` / `unalias ll` | create / remove alias |
| `export EDITOR=nano` | default editor |

| File | Use |
|------|-----|
| `~/.bashrc` | your customisations (put them here) |
| `~/.profile` | login; sources `.bashrc` |
| `/etc/profile`, `/etc/bash.bashrc` | system-wide |

<br>

[↑ Back to top](#top)
