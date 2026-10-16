<a id="top"></a>

# ✏️ 07 - Editing files

> **Source:** <https://www.escbash.com/topics/linux-editing-files>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [vim basics](#lesson-1) | 📖 Lesson |
| 2 | [nano](#lesson-2) | 📖 Lesson |
| 3 | [Edits without opening an editor](#lesson-3) | 📖 Lesson |
| 4 | [Edit a small config file](#task-1) | 🛠️ Task |
| 5 | [Bulk-substitute in a config file](#task-2) | 🛠️ Task |
| 6 | [Scenario: changing a single value in nginx.conf](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: vim basics

`vim` is installed on almost every Linux system, so sooner or later you'll have to use it. Beginners get stuck because of **one idea nobody explains up front: modes**.

### Why vim feels weird: modes

In Notepad the keyboard always types. In vim, **the same key does different things depending on the mode**.

| Mode | What it does |
|------|--------------|
| **Normal** (vim starts here) | keys are **commands**, not text. `i`, `d`, `x` insert/delete/move; they don't type letters |
| **Insert** | the keyboard behaves like Notepad, whatever you type goes in the file |

The whole trick: know which mode you're in and how to switch.

### Step by step: edit a file

```bash
vim /tmp/hello.txt
```

1. You start in **normal mode**. Typing now gives strange behaviour, don't panic.
2. Press **`i`**. The bottom-left shows `-- INSERT --`.
3. Type your text:
   ```
   Hello from vim
   This is my first edit
   ```
4. Press **`Esc`** to return to normal mode (`-- INSERT --` disappears).
5. Type **`:w`** + Enter to save (`:` opens the command line, `w` = write).
6. Type **`:q`** + Enter to quit.

Full edit: **open → `i` → type → `Esc` → `:w` → `:q`**.

### Commands worth memorising (typed in normal mode)

| Command | Action |
|---------|--------|
| `i` | enter insert mode |
| `Esc` | leave insert mode, back to normal |
| `:w` | save (Enter after) |
| `:q` | quit (fails with unsaved changes) |
| `:wq` | save and quit |
| `:q!` | quit and **discard** changes (escape hatch) |
| `dd` | delete the current line |
| `u` | undo the last change |

### Stuck? How to get out of vim

1. Press **`Esc`** two or three times (guarantees normal mode).
2. Type **`:q!`** and press Enter.

`:q!` always works, even with unsaved edits. To keep your changes use `:wq` instead.

### Mental model

- Can't type text? You're in normal mode → press **`i`**.
- Typed a command and it appeared as letters? You were in insert mode → press **`Esc`** first.
- Want out no matter what? **`Esc`, then `:q!`**.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: nano

`nano` behaves the way most people expect: arrow keys move, typing types, and every non-obvious keystroke is listed at the bottom of the screen.

```bash
nano /root/notes.txt
```

If the file doesn't exist, nano opens an empty buffer and creates it on save.

### Keystrokes (`^` means the `Ctrl` key)

| Keys | Action |
|------|--------|
| `Ctrl+O` | write (save); press Enter to confirm the filename |
| `Ctrl+X` | exit (offers to save if there are unsaved changes) |
| `Ctrl+K` | cut the current line |
| `Ctrl+W` | search; Enter jumps to the next match |

### When nano isn't installed

Minimal container images and older servers may lack it. If you get "command not found", fall back to vim, or install it on Ubuntu:

```bash
apt install nano
```

(`apt` is covered in a later topic.)

### Writing an executable script

Two things make a text file a runnable script:

1. The **first line is a shebang**: `#!` followed by the interpreter path. `#!/bin/bash` = "run this with bash". It must be line 1, no blank line above, no space after `#!`.
2. The file must be **marked executable** (`chmod`, next topic).

```bash
#!/bin/bash
echo "hello from a script"
```

Any other `#` on any other line is just a comment.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Edits without opening an editor

Three tools cover in-place changes without nano or vim: **`>>`**, **heredocs**, and **`sed -i`**.

### Append with `>>`

`>` overwrites; `>>` **appends**.

```bash
echo "127.0.0.1 example.local" >> /etc/hosts
```

Adds a line at the end without touching existing contents. Great for adding entries, useless for changing something already there.

### Multiple lines with a heredoc

```bash
cat << EOF > /root/hello.txt
first line
second line
third line
EOF
```

Everything between `<< EOF` and the closing `EOF` becomes the file's contents. Handy for generating small config files from scripts.

### `sed -i`: in-place substitution

```bash
echo "mode: old" > /root/config.yaml
sed -i 's/old/new/' /root/config.yaml
```

Substitutes the **first** occurrence of `old` with `new` on every line. Add `g` (`s/old/new/g`) to replace **every** occurrence on each line.

### Always write a backup

`sed -i` has **no undo**. Use `-i.bak`:

```bash
sed -i.bak 's/listen 80/listen 8080/' /etc/nginx/sites-available/default
```

This writes the change and saves the original as `default.bak` in the same folder. To roll back:

```bash
mv /etc/nginx/sites-available/default.bak /etc/nginx/sites-available/default
```

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Edit a small config file

Create `/root/app/config.yaml` with exactly:

```yaml
app: hello
port: 80
```

Then:

1. **Append** `env: production` to the end (existing two lines intact).
2. Change `port` from `80` to `8080`. Exactly **one** `port:` line in the final file.
3. Keep a backup of the original as `/root/app/config.yaml.bak`, still containing `port: 80`.

**Solution (from the site):**

```bash
mkdir -p /root/app
cat << 'EOF' > /root/app/config.yaml
app: hello
port: 80
EOF
echo "env: production" >> /root/app/config.yaml
sed -i.bak 's/^port: 80$/port: 8080/' /root/app/config.yaml
```

> **📝 Note:** The order matters: the `.bak` is made by the `sed` step, so it also contains `env: production`, but still `port: 80`, which is what the check needs.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Bulk-substitute in a config file

An nginx server listens only on `localhost` and must accept connections from any address (`0.0.0.0`). Every occurrence must change, with a backup.

Create `/root/nginx.conf`:

```
server {
    listen localhost:80;
    listen localhost:443;
    server_name localhost;
}
```

Replace **every** `localhost` with `0.0.0.0` in place, keeping the original as `/root/nginx.conf.bak`.

**Solution (from the site):**

```bash
cat << 'EOF' > /root/nginx.conf
server {
listen localhost:80;
listen localhost:443;
server_name localhost;
}
EOF
sed -i.bak 's/localhost/0.0.0.0/g' /root/nginx.conf
```

The `g` flag replaces all occurrences on each line, and `-i.bak` keeps the original.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: changing a single value in nginx.conf

**Q:** You need to change `listen 80;` to `listen 8080;` in `/etc/nginx/nginx.conf` on a production server. What's the cleanest move?

- [ ] `sed -i.bak 's/listen 80;/listen 8080;/' /etc/nginx/nginx.conf`
- [x] **Open the file in nano, change the line, save with `Ctrl+O`, exit with `Ctrl+X`**
- [ ] `echo "listen 8080;" >> /etc/nginx/nginx.conf`
- [ ] `rm /etc/nginx/nginx.conf && nano /etc/nginx/nginx.conf`

**Why:** for a **single edit on a live config**, an interactive editor lets you see the surrounding lines and confirm exactly what changed before saving; one wrong keystroke doesn't destroy anything.

| Option | Verdict |
|--------|---------|
| `sed -i.bak` | the right tool once the edit is **scripted or repeated** across servers |
| `>>` append | creates a conflicting duplicate directive |
| `rm` then re-create | risks nginx failing to reload at all |

> **💡 Tip:** Note the subtlety: `sed -i.bak` is tempting, but the site's answer for a one-off edit on production is the visible, interactive editor.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

### vim

| Keys | Action |
|------|--------|
| `i` | insert mode |
| `Esc` | back to normal mode |
| `:w` / `:q` / `:wq` | save / quit / save+quit |
| `:q!` | quit without saving |
| `dd` / `u` | delete line / undo |

### nano

| Keys | Action |
|------|--------|
| `Ctrl+O` | save |
| `Ctrl+X` | exit |
| `Ctrl+K` | cut line |
| `Ctrl+W` | search |

### No-editor edits

| Command | Purpose |
|---------|---------|
| `cmd >> file` | append a line |
| `cat << 'EOF' > file` … `EOF` | write a multi-line file |
| `sed -i 's/old/new/' file` | replace first match per line, in place |
| `sed -i 's/old/new/g' file` | replace all matches per line |
| `sed -i.bak '...' file` | same, and keep `file.bak` |
| `#!/bin/bash` (line 1) | shebang for a runnable script |

<br>

[↑ Back to top](#top)
