<a id="top"></a>

# 🤝 08 - TCP

> **Source:** <https://www.escbash.com/topics/net-tcp>
> **Contents:** 4 lessons · 5 hands-on tasks

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [TCP connections](#lesson-1) | 📖 Lesson |
| 2 | [Create a TCP connection and view it in ss](#task-1) | 🛠️ Task |
| 3 | [The three-way handshake](#lesson-2) | 📖 Lesson |
| 4 | [Watch connection states change](#task-2) | 🛠️ Task |
| 5 | [Reliability, acks and retransmits](#lesson-3) | 📖 Lesson |
| 6 | [Compare connection refused and timeout](#task-3) | 🛠️ Task |
| 7 | [Closing a connection, FIN and TIME_WAIT](#lesson-4) | 📖 Lesson |
| 8 | [Find TIME_WAIT sockets](#task-4) | 🛠️ Task |
| 9 | [Practice reading ss output](#task-5) | 🛠️ Task |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: TCP connections

A web page is thousands of packets, and they must **all arrive, in the right order, with none missing**. **TCP** is the transport protocol that makes this happen.

TCP works like a **phone call, not like sending letters**. Before any data moves, the two machines set up a **connection**: a two-way channel between one port on each side. Then either side can send a **stream of bytes**, and TCP promises two things: **everything arrives**, and **it arrives in order**. If a packet is lost, TCP notices and resends it without either program knowing.

### A connection is a pair of endpoints

One TCP connection is fully named by **four numbers**:

```
source IP, source port, destination IP, destination port
```

That's why one server port can serve thousands of clients at once: each client comes from a different IP/port pair, so every connection has its own unique name.

### A tool to make raw connections: `nc` (netcat)

`nc` just opens a connection and does nothing else (topic 10 covers it fully). Two forms for now:

```bash
nc <host> <port>      # connect to a port, like a client
nc -l <port>          # listen on a port, like a server
```

Whatever you type on one side comes out on the other: one raw TCP connection with no protocol on top.

> **💡 Tip:** TCP is the floor everything stands on: ssh, http, databases. When TCP misbehaves, every service on top of it does, so knowing how connections form and fail is core debugging knowledge.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Create a TCP connection and view it in ss

Build a TCP connection with your own hands, no web server involved, and observe it from outside.

1. Start an `nc` **listener on port 9000**, in the background.
2. Connect an `nc` **client** to it on `localhost`, also in the background, so the connection stays open.
3. While both run, save the `ss` view of established TCP connections into `/root/answers/tcp-estab.txt`. It must show the `ESTAB` state for port 9000.

Both processes must still be running when you submit.

**Solution (from the site):**

```bash
nc -l 9000 &
nc localhost 9000 &
```

```bash
ss -tn > /root/answers/tcp-estab.txt
cat /root/answers/tcp-estab.txt
```

You see **two `ESTAB` lines for the same connection**, one from each end's point of view: the listener side and the client side, both on port 9000.

> **📝 Note:** In the lab terminal, backgrounded `nc` jobs can show as `Stopped` because they try to read the terminal. If a check fails for that reason, feed them from `sleep infinity` instead: `sleep infinity | nc -l 9000 &`.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: The three-way handshake

A connection can't start with data. Before either side sends a byte, both must agree a connection exists and that **both directions work**. TCP does it with three small packets: the **three-way handshake**.

```
 client                          server
   │ ── 1. SYN ───────────────►  │   "I want to open a connection"
   │ ◄── 2. SYN-ACK ──────────── │   "I accept, and I confirm I received your request"
   │ ── 3. ACK ───────────────►  │   "confirmed, we are connected"
```

The server answers SYN-ACK **only if a program is listening** on that port.

### Why three and not two

| Packet | Proves |
|--------|--------|
| SYN | the client can reach the server |
| SYN-ACK | the server can reach the client |
| ACK | the client received that answer |

Only after all three does either side trust the channel enough to send data.

### What each side sees

During the handshake the sockets move through states you can watch with `ss`: the server's listening socket sits in **`LISTEN`**; when the handshake completes **both ends show `ESTAB`**.

### Where connections fail

| Situation | What happens |
|-----------|--------------|
| nothing listens on the port | the server answers the SYN with a **reset**; the client reports **"connection refused"** instantly |
| the SYN gets no answer at all | the client waits and retries until it gives up (**timeout**) |

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Watch connection states change

Capture a socket in both of its lives: first as an open door, then as a live conversation.

1. Start an `nc` listener on port **8001** in the background.
2. **Before any client connects**, save the `ss` view showing that port in `LISTEN` state to `/root/answers/state-listen.txt`.
3. Connect an `nc` client to `localhost` port 8001 in the background.
4. Save the `ss` view showing the port 8001 connection in `ESTAB` state to `/root/answers/state-estab.txt`.

**Solution (from the site):**

First the listener alone, and a snapshot of its state:

```bash
nc -l 8001 &
sleep 1
ss -tln > /root/answers/state-listen.txt
```

Then a client, and a snapshot of the new state:

```bash
nc localhost 8001 &
sleep 1
ss -tn > /root/answers/state-estab.txt
```

The first snapshot shows the **open door** (`LISTEN`, peer is `*`). The second shows a **real conversation** (`ESTAB`, both addresses filled in). The handshake happened between your two commands.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Reliability, acks and retransmits

Networks lose packets all the time (an overloaded router, a wifi glitch, a cable hiccup). **IP does nothing about it: a lost packet is simply gone.** TCP builds reliability on top with two mechanisms: **acknowledgements** and **retransmits**.

### Every piece is acknowledged

TCP numbers every byte it sends. When data arrives, the receiver sends back an **ack**: "I have everything up to byte N". The sender keeps a copy of everything it sent **until the matching ack arrives**, like a delivery receipt.

### Lost pieces are resent

If an ack doesn't arrive within a short time, the sender assumes the piece was lost and **sends it again**, waiting a little longer each retry. The receiver uses the byte numbers to **reorder** everything and **throw away duplicates**, so the program above sees a perfect stream even when the wire was messy.

All of this is invisible to programs: `curl` never resends anything, `nginx` never sorts packets. The **kernel's TCP code** does the numbering, receipts and retries on both sides.

### What it looks like from outside

A healthy connection acks quickly and rarely retransmits. A connection over a bad link still works, it just gets **slow**. That's a useful diagnostic fact: **TCP hides packet loss so well that loss shows up as slowness, not as errors.** `ss` can show retransmit counters:

```bash
ss -ti
```

```
ESTAB 0 0 10.0.5.4:22 10.0.5.9:50022
     cubic rto:204 rtt:1.5/0.8 ... retrans:0/3 ...
```

The `retrans` counter says this connection has resent 3 segments in its lifetime. A **large and growing number** means the path is losing packets.

> **💡 Tip:** When users say "the site works but is very slow", think of exactly this: the connection is alive, but retransmits are eating the time.

> **📝 Note (for the next task):** `curl` prints error messages to the **error stream**, not normal output. To capture an error into a file, add `2>&1` after the redirection, which sends the error stream to the same place as normal output.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Compare connection refused and timeout

A failing connection fails in one of two very different ways. Produce both on purpose.

1. Try to connect to port **8002 on localhost**, where nothing is listening. The failure is **instant**. Save the error message to `/root/answers/refused.txt`.
2. Try to connect to **`http://10.255.255.1/`**, an address that exists on paper but where no machine answers. Give it a **time limit** of a few seconds. Save the error message to `/root/answers/timeout.txt`.

**Solution (from the site):**

A closed port on a reachable machine answers instantly:

```bash
curl http://localhost:8002/ 2>&1 > /dev/null | tee /root/answers/refused.txt
```

```
curl: (7) Failed to connect to localhost port 8002 ... Couldn't connect to server
```

Depending on the curl version the last words are either "Couldn't connect to server" or "Connection refused". Both mean the same refusal.

An address that never answers just hangs, so give curl a time limit:

```bash
curl --max-time 5 http://10.255.255.1/ 2>&1 > /dev/null | tee /root/answers/timeout.txt
```

```
curl: (28) Connection timed out after 5002 milliseconds
```

| Failure | Meaning |
|---------|---------|
| **Refused** | a machine answered "nothing here", and it answered **fast** |
| **Timeout** | **nothing answered at all** |

<br>

---

<br>

<a id="lesson-4"></a>

## 📖 Lesson 4: Closing a connection, FIN and TIME_WAIT

Connections are opened with a handshake and **closed with one too**. Hanging up without warning would risk losing data still on the way, so each side announces it's finished.

```
 client                          server
   │ ── 1. FIN ───────────────►  │   "I have no more data to send"
   │ ◄── 2. ACK, then own FIN ── │   server acks, sends its own FIN when done
   │ ── 3. ACK ───────────────►  │   client acks, and waits in TIME_WAIT
```

Each direction of the channel is closed **separately**, like two people each saying goodbye before the call really ends.

### Why TIME_WAIT lingers

The side that **closed first** keeps the socket in **`TIME_WAIT` for about a minute**:

- The very last ack it sent might have been lost. If so, the other side resends its FIN and someone must still be there to answer: the socket "stays by the phone briefly in case the goodbye needs repeating".
- It protects the **next connection**: the four numbers naming a connection could be reused immediately, and old packets still wandering the network could arrive into a new conversation. `TIME_WAIT` lets those stragglers die out first.

### What you see in `ss`

```bash
ss -tan
```

```
State      Recv-Q Send-Q Local Address:Port  Peer Address:Port
TIME-WAIT  0      0      127.0.0.1:51844     127.0.0.1:80
```

Every quick `curl` leaves a `TIME-WAIT` socket behind for about a minute.

> **💡 Tip:** On a busy server making many short connections, **thousands of `TIME-WAIT` entries are completely normal**: not leftover garbage and not a leak. Knowing that saves you from "fixing" a healthy machine.

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Find TIME_WAIT sockets

Prove finished connections leave a trace by creating some and catching them before they expire.

1. Make several quick requests to the **nginx** server on port 80 of this machine.
2. List the TCP sockets, find the ones in `TIME-WAIT`, and save those lines to `/root/answers/time-wait.txt`. The file must show `TIME-WAIT` sockets connected to **port 80**.

Work quickly: the entries disappear after about a minute.

**Solution (from the site):**

```bash
curl -s http://localhost/ > /dev/null
curl -s http://localhost/ > /dev/null
curl -s http://localhost/ > /dev/null
```

Each curl opened a connection, got the page and closed. Your side closed first, so each sits in `TIME-WAIT`. Capture them:

```bash
ss -tan | grep TIME-WAIT > /root/answers/time-wait.txt
cat /root/answers/time-wait.txt
```

One `TIME-WAIT` line per curl, each with peer port 80. Wait a minute and run `ss` again: they're gone on their own.

<br>

---

<br>

<a id="task-5"></a>

## 🛠️ Task 5: Practice reading ss output

`/root/quiz/ss-snapshot.txt` holds an `ss` capture from another server. Read it the way you would during an incident and answer four questions.

Create `/root/answers/ss-quiz.txt` with four lines, `<number> = <answer>`:

1. which port does the web server listen on?
2. how many connections are fully established?
3. one socket is in a state that is neither `LISTEN` nor `ESTAB`. Which state?
4. which process listens on port 22?

**Input file** `/root/quiz/ss-snapshot.txt` ([raw copy](lab-files/08-tcp/ss-snapshot.txt)):

```
State      Recv-Q Send-Q Local Address:Port   Peer Address:Port   Process
LISTEN     0      511    0.0.0.0:8080         0.0.0.0:*           users:(("nginx",pid=612,fd=6))
LISTEN     0      128    0.0.0.0:22           0.0.0.0:*           users:(("sshd",pid=301,fd=3))
ESTAB      0      0      10.0.5.4:8080        10.0.5.9:52144      users:(("nginx",pid=612,fd=12))
ESTAB      0      0      10.0.5.4:22          10.0.5.9:50022      users:(("sshd",pid=890,fd=4))
ESTAB      0      0      10.0.5.4:8080        10.0.5.11:41780     users:(("nginx",pid=612,fd=13))
TIME-WAIT  0      0      10.0.5.4:8080        10.0.5.9:52020
```

**Solution (from the site):**

```bash
cat > /root/answers/ss-quiz.txt <<'DONE'
1 = 8080
2 = 3
3 = TIME-WAIT
4 = sshd
DONE
```

- nginx listens on **8080** (its `LISTEN` line).
- **Three** lines show `ESTAB`: two nginx clients and one ssh session.
- The one remaining state is **`TIME-WAIT`**, a recently closed connection.
- Port 22's `LISTEN` line belongs to **sshd**.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Concept | Remember |
|---------|----------|
| TCP | connection-based, reliable, ordered byte stream (a phone call, not letters) |
| connection name | `src IP, src port, dst IP, dst port` |
| handshake | `SYN` → `SYN-ACK` → `ACK` |
| close | `FIN` / `ACK` each way; the side that closed first sits in `TIME_WAIT` (~1 min) |
| ack / retransmit | receiver acks byte N; sender resends un-acked data; loss shows up as **slowness** |
| refused | instant: a machine answered "nothing here" (reset) |
| timeout | slow: nothing answered at all |

| Command | Purpose |
|---------|---------|
| `nc -l PORT` / `nc HOST PORT` | listen / connect (raw TCP) |
| `ss -tln` | listening sockets (`LISTEN`, peer `*`) |
| `ss -tn` | established connections (`ESTAB`) |
| `ss -tan` | all states, including `TIME-WAIT` |
| `ss -ti` | per-connection details incl. `retrans:` |
| `curl --max-time 5 URL` | time-limited request (to see a timeout) |
| `cmd 2>&1 > /dev/null \| tee file` | capture only the error output |

| `ss` state | Meaning |
|------------|---------|
| `LISTEN` | an open door waiting for clients |
| `ESTAB` | live conversation, both ends filled in |
| `TIME-WAIT` | recently closed; normal, expires in ~1 min |

**Debug rule:** *refused* = fast answer, nothing listening; *timeout* = no answer at all; *slow but working* = look at retransmits; thousands of `TIME-WAIT` = normal.

<br>

[↑ Back to top](#top)
