<a id="top"></a>

# 🧵 17 - Text processing

> **Source:** <https://www.escbash.com/topics/linux-text-processing>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [cut, sort, and uniq](#lesson-1) | 📖 Lesson |
| 2 | [tr, tee, and xargs](#lesson-2) | 📖 Lesson |
| 3 | [awk and sed basics](#lesson-3) | 📖 Lesson |
| 4 | [Parse a small access log](#task-1) | 🛠️ Task |
| 5 | [Extract sorted usernames from /etc/passwd](#task-2) | 🛠️ Task |
| 6 | [Scenario: counting accounts by login shell](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: `cut`, `sort`, and `uniq`

Three commands in almost every shell pipeline. Build a tiny file (fields: name, count, colour):

```bash
echo "apple:3:red" > /tmp/fruit.txt
echo "banana:2:yellow" >> /tmp/fruit.txt
echo "cherry:1:red" >> /tmp/fruit.txt
```

### `cut`: pull a column out of each line

| Flag | Meaning |
|------|---------|
| `-d` | delimiter (character between fields) |
| `-f` | which field number(s) |
| `-c` | character positions instead of fields |

```bash
cut -d ':' -f 1 /tmp/fruit.txt        # apple / banana / cherry
cut -d ':' -f 1,3 /tmp/fruit.txt      # apple:red / banana:yellow / cherry:red
cut -c 1-3 /tmp/fruit.txt             # app / ban / che
```

Classic real target: `cut -d ':' -f 1 /etc/passwd` prints every username on the machine.

### `sort`: put lines in order

```bash
cut -d ':' -f 1 /tmp/fruit.txt | sort        # alphabetical
```

`sort` is **alphabetical by default**, comparing character by character. With `10, 2, 1` you'd get `1, 10, 2`.

| Flag | Effect |
|------|--------|
| `-n` | numeric sort |
| `-r` | reverse |
| `-k N` | sort on column N |
| `-t X` | delimiter for `-k` |

```bash
cut -d ':' -f 2 /tmp/fruit.txt | sort -nr     # 3 / 2 / 1
sort -t ':' -k 2 -n /tmp/fruit.txt            # whole file ordered by field 2 numerically
```

### `uniq`: collapse repeated lines

`uniq` removes duplicates only when they sit **next to each other**, so it almost always runs after `sort`.

```bash
echo "red" > /tmp/colours.txt
echo "red" >> /tmp/colours.txt
echo "blue" >> /tmp/colours.txt
echo "red" >> /tmp/colours.txt

sort /tmp/colours.txt | uniq               # blue / red
sort /tmp/colours.txt | uniq -c            # 1 blue / 3 red
sort /tmp/colours.txt | uniq -c | sort -rn # 3 red / 1 blue
```

> **💡 Tip:** **Count-then-rank**: `sort | uniq -c | sort -rn`. Swap the input for IPs in an access log, error codes in a service log, or shells in `/etc/passwd` and it answers "which value shows up most?" every time.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: `tr`, `tee`, and `xargs`

### `tr`: translate or delete characters

Works character by character; maps each character in the first set to the same position in the second.

```bash
echo "hello world" | tr 'a-z' 'A-Z'       # HELLO WORLD
echo "my report file" | tr ' ' '_'        # my_report_file
echo "abc123def" | tr -d '0-9'            # abcdef   (-d deletes)
```

`tr` only handles **single characters**; for words or patterns use `sed`.

### `tee`: output to the screen **and** a file

`> file` writes the file and shows nothing; `tee` does both.

```bash
echo "deploy finished" | tee /tmp/run.log       # on screen + in file
echo "second run" | tee -a /tmp/run.log         # -a appends
```

Handy when you watch a long command but also want a saved copy.

### `xargs`: turn input into command arguments

Commands like `rm` and `mkdir` take **arguments**, not stdin. `xargs` converts piped text into arguments.

```bash
echo "apple banana cherry" | xargs -n1 echo
# apple
# banana
# cherry
```

`-n1` = one argument per run (otherwise all three go to a single `echo`).

A real example, find files and delete them:

```bash
find /var/log -name '*.gz' | xargs -r rm
```

`-r` = do nothing if `find` printed nothing (avoids running `rm` with no arguments).

### Two habits that save you

```bash
# Filenames with spaces: split on null instead of spaces
find /var/log -name '*.gz' -print0 | xargs -0 -r rm

# Preview a destructive command by putting echo in front
find /var/log -name '*.gz' | xargs echo rm
```

Once the preview looks right, drop the `echo`.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: `awk` and `sed` basics

Two small languages built for text. Here's the slice that covers day-to-day work.

### `awk`: work with fields

`awk` splits each line into fields (default: whitespace).

| Symbol | Meaning |
|--------|---------|
| `$1`, `$2`, … | field 1, 2, … |
| `$0` | the whole line |
| `$NF` | the **last** field |

```bash
echo "alice 90 bash" > /tmp/scores.txt
echo "bob 75 zsh" >> /tmp/scores.txt
echo "carol 82 bash" >> /tmp/scores.txt

awk '{print $1}' /tmp/scores.txt          # alice / bob / carol
awk '{print $NF}' /tmp/scores.txt         # bash / zsh / bash
awk '{print $1, $3}' /tmp/scores.txt      # alice bash / bob zsh / carol bash
awk -F: '{print $1}' /etc/passwd          # -F sets the separator
```

Put a **condition** in front of the action:

```bash
awk '$2 > 80 {print $1}' /tmp/scores.txt      # alice / carol
```

One `awk` call filtered on field 2 and printed field 1, replacing a chain of pipes.

### `sed`: substitute text

Everyday job: find-and-replace, `s/old/new/`.

```bash
echo "hello world" | sed 's/world/DevOps/'      # hello DevOps
echo "a-b-c" | sed 's/-/_/'                     # a_b-c   (first match only)
echo "a-b-c" | sed 's/-/_/g'                    # a_b_c   (g = every match)
```

On files:

```bash
echo "http://a.com" > /tmp/urls.txt
echo "http://b.com" >> /tmp/urls.txt
sed 's/http:/https:/g' /tmp/urls.txt
# https://a.com
# https://b.com
```

That prints the result without touching the file. Use `> newfile` to save a copy, or `sed -i` to edit in place.

### Which to reach for

| Job | Tool |
|-----|------|
| replace / rewrite text | `sed` |
| pick columns / filter by a condition | `awk` |
| both | `... \| awk '{print $2}' \| sed 's/old/new/'` |

You don't have to master either: recognise what they do, copy a pattern from the docs, adapt it, and read it in other people's scripts.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Parse a small access log

Create `/root/mission/access.log` with exactly these 10 lines (IP, method, path):

```
192.168.1.10 GET /api/users
192.168.1.11 GET /api/orders
192.168.1.10 POST /api/orders
10.0.0.5 GET /api/users
192.168.1.10 GET /health
10.0.0.5 GET /health
10.0.0.5 POST /api/users
192.168.1.10 GET /api/users
192.168.1.11 GET /api/users
192.168.1.10 GET /api/orders
```

Then in `/root/answers/` create:

| File | Contents |
|------|----------|
| `top-ips.txt` | unique client IPs prefixed with their count, busiest first |
| `unique-paths.txt` | the distinct request paths, one per line (3 in total) |
| `fetch-log.txt` | a copy of the log with every `GET` replaced by `FETCH` |

**Solution (from the site):**

```bash
mkdir -p /root/mission /root/answers
cat > /root/mission/access.log <<'EOF'
192.168.1.10 GET /api/users
192.168.1.11 GET /api/orders
192.168.1.10 POST /api/orders
10.0.0.5 GET /api/users
192.168.1.10 GET /health
10.0.0.5 GET /health
10.0.0.5 POST /api/users
192.168.1.10 GET /api/users
192.168.1.11 GET /api/users
192.168.1.10 GET /api/orders
EOF
```

```bash
cut -d ' ' -f 1 /root/mission/access.log | sort | uniq -c | sort -rn > /root/answers/top-ips.txt
cut -d ' ' -f 3 /root/mission/access.log | sort -u > /root/answers/unique-paths.txt
sed 's/GET/FETCH/g' /root/mission/access.log > /root/answers/fetch-log.txt
```

- count-then-rank for the IPs (field 1)
- `sort -u` for distinct paths (field 3)
- `sed` with the `g` flag for the FETCH copy

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Extract sorted usernames from /etc/passwd

Quick user audit: `/root/users-sorted.txt` with **one username per line**, just the first colon-separated field, **alphabetically sorted**.

**Solution (from the site):**

```bash
cut -d ':' -f 1 /etc/passwd | sort > /root/users-sorted.txt
```

`cut` pulls field 1, `sort` orders it, the redirect writes the file.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: counting accounts by login shell

**Q:** You want a report of how many accounts use each login shell (bash, sh, nologin, ...). Which one-liner?

- [x] **`awk -F: '{print $7}' /etc/passwd | sort | uniq -c | sort -rn`**
- [ ] `cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn`
- [ ] Write a bash loop that reads `/etc/passwd` line by line and increments a counter per shell
- [ ] `grep bash /etc/passwd | wc -l`, then repeat for each shell you can think of

**Why:** the classic **count-per-unique-value pipeline**: `awk` pulls field 7 (login shell), `sort` groups identical values so `uniq` sees them adjacent, `uniq -c` counts, `sort -rn` puts the biggest on top.

| Option | Verdict |
|--------|---------|
| `cut -d: -f7 ...` | **works just as well** for this fixed-column job (the site marks `awk` as best) |
| bash loop | reinvents the pipeline |
| `grep` per shell | silently misses shells you didn't think of |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `cut -d ':' -f 1,3` | pick fields by delimiter |
| `cut -c 1-3` | pick characters |
| `sort` / `-n` / `-r` / `-u` | alphabetical / numeric / reverse / unique |
| `sort -t ':' -k 2 -n` | sort by column 2 numerically |
| `uniq` / `uniq -c` | collapse adjacent duplicates / with counts |
| `sort \| uniq -c \| sort -rn` | **count-then-rank** |
| `tr 'a-z' 'A-Z'` / `tr -d '0-9'` | translate / delete characters |
| `cmd \| tee [-a] file` | screen + file (append with `-a`) |
| `xargs -n1 -r cmd` | stdin → arguments (one per run, skip if empty) |
| `find ... -print0 \| xargs -0 -r rm` | safe with spaces in names |
| `xargs echo rm` | dry-run preview |
| `awk '{print $1, $NF}'` | print first and last field |
| `awk -F: '{print $1}'` | custom separator |
| `awk '$2 > 80 {print $1}'` | filter then print |
| `sed 's/old/new/g'` | substitute all matches per line |
| `sed -i 's/old/new/' file` | edit in place |

<br>

[↑ Back to top](#top)
