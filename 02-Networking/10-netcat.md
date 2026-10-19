<a id="top"></a>

# 🐱 10 - Netcat

> **Source:** <https://www.escbash.com/topics/net-netcat>
> **Contents:** 3 lessons · 4 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Netcat as client and listener](#lesson-1) | 📖 Lesson |
| 2 | [Send messages between two nc processes](#task-1) | 🛠️ Task |
| 3 | [Sending files over a raw connection](#lesson-2) | 📖 Lesson |
| 4 | [Send a file with nc](#task-2) | 🛠️ Task |
| 5 | [Send a directory with nc and tar](#task-3) | 🛠️ Task |
| 6 | [Checking ports with nc -z](#lesson-3) | 📖 Lesson |
| 7 | [Check which ports are open with nc](#task-4) | 🛠️ Task |
| 8 | [Scenario: find the service on an unknown port](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: Netcat as client and listener

Every tool you've used so far has a fixed job: `curl` is always a web client, `nginx` always a web server. **Netcat (`nc`) has no fixed job.** It opens a **raw TCP or UDP connection** and moves bytes through it in either direction, with **no protocol on top**. What the bytes mean is up to you.

### The two modes

| Mode | Command | Behaviour |
|------|---------|-----------|
| **Client** | `nc 127.0.0.1 9000` | connects to a host and port; what you type goes to the other side, and what comes back is printed |
| **Listener** | `nc -l 9000` | waits on a port for a client; once one connects, typed input goes out and received bytes are printed |

```
        one raw TCP connection
  nc -l 9000  ◄════ bytes ════►  nc 127.0.0.1 9000
   (listener)                        (client)
```

### Why a raw endpoint is useful

With `nc` you control **both ends** completely, which makes it the perfect test instrument:

- *Is a port reachable?* Connect to it with `nc`.
- *Need a server on port 9000 right now, with zero setup?* Start a listener.
- *Want to see exactly what bytes a client sends?* Listen and read them.

Without `-u`, `nc` speaks **TCP**: a connect does the full handshake and a refused port fails immediately with an error. (`-u` was UDP mode in the last topic.)

> **💡 Tip:** A DevOps engineer reaches for `nc` many times a week. It answers "can this machine reach that port?" in one command, and stands in for any missing client or server during a debug session.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Send messages between two nc processes

Run both ends of a TCP conversation yourself. One terminal means the listener runs in the background.

1. Start an `nc` listener on port **9000** in the background, with everything it receives **appended** to `/root/chat.log`.
2. Connect a second `nc` to port 9000 on `127.0.0.1` as the client.
3. Send these two lines, each on its own line: `hello from the client` and `raw tcp is simple`.
4. Close the client with `Ctrl+C`, then confirm both lines landed in `/root/chat.log`.

**Solution (from the site):**

Start the listener in the background, appending everything it receives to the log:

```bash
nc -l 9000 >> /root/chat.log &
```

Connect a client and send the two lines. Typing them into `nc 127.0.0.1 9000` and closing with `Ctrl+C` works, and so does piping them in one shot:

```bash
printf 'hello from the client\nraw tcp is simple\n' | nc -w1 127.0.0.1 9000
```

`-w1` makes `nc` stop waiting and exit one second after the input ends. Either way the connection closes, the background listener exits, and the log holds both lines:

```bash
cat /root/chat.log
```

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Sending files over a raw connection

`nc` moves bytes and doesn't care where they come from. A **redirect** swaps the keyboard for a file and the screen for another file: a complete file transfer tool in two commands.

### The receiving and sending side

```bash
nc -l 9000 > /root/incoming.txt          # receiver: write what arrives into a file
nc -N 127.0.0.1 9000 < /root/outgoing.txt   # sender: push a file through
```

**`-N`** closes the connection when the file ends, so both sides exit cleanly: the listener sees the end of the stream and the received file is complete. **Without `-N`, both ends hang** waiting for more bytes that never come.

### Whole directories travel as a tar stream

`nc` carries **one stream of bytes**, and a directory is many files. `tar` packs a directory into one stream on the way in and unpacks it on the way out. Pipe `tar` into `nc` on one side and `nc` into `tar` on the other:

```bash
nc -l 9000 | tar -x -C /root/copy           # receiver: extract the stream under /root/copy
tar -c -C /root site | nc -N 127.0.0.1 9000  # sender: pack /root/site, push it
```

| Flag | Meaning |
|------|---------|
| `tar -c` / `tar -x` | create / extract the stream |
| `-C DIR` | sender: directory to start from; receiver: where files land |

> **📝 Note:** Real transfers use `scp` or `rsync` (encryption, resume). But on a machine with nothing installed, or when you need to see exactly what crosses the wire, `nc` plus redirects is the fallback every DevOps engineer keeps in their pocket.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Send a file with nc

A report sits at `/root/data/report.txt`. Transfer it across a raw TCP connection on this machine and prove the copy is exact.

1. Run an `nc` listener on port **9000** in the background that writes what it receives to `/root/received/report.txt`.
2. Send `/root/data/report.txt` through a client `nc` to port 9000 on `127.0.0.1`, using the flag that **closes the connection when the file ends**.
3. Compare the two copies with `md5sum`. The hashes must match.
4. Record the result in `/root/answers/file-transfer.txt`: the hash of the original on line 1 and the hash of the received copy on line 2, **hash values only**.

**Input file** `/root/data/report.txt` ([raw copy](lab-files/10-netcat/data/report.txt)):

```
weekly capacity report
web tier: 4 nodes, 61 percent average cpu
db tier: 2 nodes, 44 percent average cpu
action: add one web node before the sale event
```

**Solution (from the site):**

Start the receiving side in the background:

```bash
nc -l 9000 > /root/received/report.txt &
```

Send the file; `-N` closes the connection when the file ends so both `nc` processes exit:

```bash
nc -N 127.0.0.1 9000 < /root/data/report.txt
```

Give the listener a second to finish writing, then prove the copy is intact and record both checksums:

```bash
sleep 1
md5sum /root/data/report.txt /root/received/report.txt
md5sum /root/data/report.txt | awk '{print $1}' > /root/answers/file-transfer.txt
md5sum /root/received/report.txt | awk '{print $1}' >> /root/answers/file-transfer.txt
```

The two hashes are identical, which means every byte crossed the connection unchanged.

> **📝 Note:** `/root/received` must exist before the listener starts (`mkdir -p /root/received`).

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Send a directory with nc and tar

A small website lives at `/root/site`: two HTML pages and one CSS file in a subfolder. Move the **whole directory** across a raw TCP connection so an exact copy appears at `/root/copy/site`.

1. Receiving side on port **9000**, in the background: an `nc` listener piped into `tar`, extracting under `/root/copy`.
2. Sending side: pack the `site` directory (starting from `/root`) with `tar` and pipe the stream into an `nc` client aimed at port 9000 on `127.0.0.1`, with the flag that closes the connection at the end of the stream.
3. Verify with `diff -r`, which compares two directories file by file and prints **nothing** when they match.

**Input files** (raw copies in [lab-files/10-netcat/site](lab-files/10-netcat/site/)):

| File | Contents |
|------|----------|
| `/root/site/index.html` | `<html><body><h1>team status page</h1></body></html>` |
| `/root/site/about.html` | `<html><body><p>maintained by the platform team</p></body></html>` |
| `/root/site/css/main.css` | `body { font-family: sans-serif; }` |

**Solution (from the site):**

Start the receiving side in the background. It listens and feeds the incoming stream to `tar` for unpacking under `/root/copy`:

```bash
nc -l 9000 | tar -x -C /root/copy &
```

Pack the directory into a stream and push it through the connection:

```bash
tar -c -C /root site | nc -N 127.0.0.1 9000
```

Give the receiving pipeline a second to finish unpacking, then verify that every file arrived unchanged:

```bash
sleep 1
diff -r /root/site /root/copy/site
```

No output from `diff` means the trees are identical.

> **📝 Note:** Create the destination first: `mkdir -p /root/copy`.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Checking ports with nc -z

Often the question isn't "talk to this service" but "**is anything there?**". The **`-z`** flag answers it: `nc` does the TCP handshake, and the moment the connection is open it **closes it again without sending a byte**. Add **`-v`** so it prints the result:

```bash
nc -zv 127.0.0.1 80
```

```
Connection to 127.0.0.1 80 port [tcp/http] succeeded!
```

### Open versus refused

| Outcome | What happens | `nc` reports |
|---------|--------------|--------------|
| **open** | a program is listening, handshake completes | success, exit code **0** |
| **refused** | nothing listening; the machine answers with a reset | `Connection refused`, exit code **1** |
| **timeout** | no answer at all (real networks) | waits until a timeout |

```bash
nc -zv 127.0.0.1 81
# nc: connect to 127.0.0.1 port 81 (tcp) failed: Connection refused
```

> **⚠️ Warning:** **Refused** = the machine is there and the port is closed. **Timeout** = the machine is gone, or a firewall is silently dropping your packets.

### Checking a range

`nc` accepts a **port range** and, with `-z`, walks the whole range (one quick handshake per port):

```bash
nc -zv 127.0.0.1 79-82
nc -zv 127.0.0.1 79-82 2>&1 | grep succeeded     # keep only the open ports
```

`nc` prints its results on **stderr**, so the filter needs **`2>&1`** before the pipe.

### `ss` vs `nc`

| Tool | View |
|------|------|
| `ss` | lists listeners **from the inside** of a machine |
| `nc -z` | proves reachability **from the outside** |

A DevOps engineer uses both, because **a port can be listening yet unreachable**, and only an outside probe shows it.

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Check which ports are open with nc

Some services are running on ports somewhere between **7000 and 7100**. Find them from the outside.

1. Probe every port from 7000 to 7100 on `127.0.0.1` with the `nc` port-check flag (one command for the range).
2. Filter the output down to the ports that answered with an open connection.
3. Write the open port numbers into `/root/answers/open-ports.txt`, one per line, in ascending order.

**Solution (from the site):**

Sweep the range and keep only the successful connects:

```bash
nc -zv 127.0.0.1 7000-7100 2>&1 | grep succeeded
```

```
Connection to 127.0.0.1 7003 port [tcp/*] succeeded!
Connection to 127.0.0.1 7040 port [tcp/*] succeeded!
Connection to 127.0.0.1 7077 port [tcp/*] succeeded!
```

Record the three port numbers, one per line:

```bash
mkdir -p /root/answers
printf '7003\n7040\n7077\n' > /root/answers/open-ports.txt
```

Each success line was one completed TCP handshake, immediately closed again. The other 98 ports answered *refused*.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: find the service on an unknown port

**Q:** A teammate left a note: "the config service is running on this box on some high port, the deploy token is on its front page, I forgot which port I gave it". You're logged in as root on the machine. What's your first move to find the service and read the token?

- [ ] Reboot the machine so the service comes up on its usual default port
- [ ] Sweep the high ports from outside with `nc -zv 127.0.0.1 1024-65535`, then `curl` whatever answers
- [ ] Run `nmap localhost` and `curl` the port it reports
- [x] **`ss -tlnp`, then `curl` the high port it shows**

**Why:** **fastest and most precise.** You're on the machine, so `ss` lists every listening socket along with the process that owns it. One unusual high port with a service process next to it is your answer, and `curl` on that port shows the front page with the token. Two commands, no guessing.

When you have a shell on the machine, the listener list isn't a secret to probe for: it's **kernel state you can read directly**.

| Option | Verdict |
|--------|---------|
| outside probes with `nc` or `nmap` | reach the same answer more slowly; **the right approach only when you can't log in to the target** |
| reboot | risks losing the service entirely and helps with nothing |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `nc HOST PORT` | raw TCP client |
| `nc -l PORT` | raw TCP listener |
| `nc -u ...` | UDP mode |
| `printf 'a\nb\n' \| nc -w1 HOST PORT` | send lines, exit 1 s after input ends |
| `nc -l PORT > file` | receive a file |
| `nc -N HOST PORT < file` | send a file, close at end |
| `nc -l PORT \| tar -x -C DIR` | receive a directory |
| `tar -c -C PARENT DIR \| nc -N HOST PORT` | send a directory |
| `md5sum file` | checksum to prove a copy is exact |
| `diff -r dirA dirB` | compare two trees (no output = identical) |
| `nc -zv HOST PORT` | is this port open? (connect then close) |
| `nc -zv HOST 7000-7100 2>&1 \| grep succeeded` | sweep a range, keep the open ports |

| Concept | Remember |
|---------|----------|
| raw connection | bytes only, no protocol on top |
| `-N` | close at end of input (otherwise both ends hang) |
| `-z` | probe only: handshake then close |
| results on stderr | add `2>&1` before the pipe |
| listener visibility | `ss` = inside view, `nc -z` = outside view |
| root on the box | `ss -tlnp` first; scan from outside only if you can't log in |

<br>

[↑ Back to top](#top)
