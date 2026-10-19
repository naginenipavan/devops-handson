<a id="top"></a>

# 🚪 07 - Ports and sockets

> **Source:** <https://www.escbash.com/topics/net-ports-sockets>
> **Contents:** 3 lessons · 4 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [What a port is](#lesson-1) | 📖 Lesson |
| 2 | [Run two servers on two ports](#task-1) | 🛠️ Task |
| 3 | [Well-known ports](#lesson-2) | 📖 Lesson |
| 4 | [Match services to ports](#task-2) | 🛠️ Task |
| 5 | [Listening and connected sockets](#lesson-3) | 📖 Lesson |
| 6 | [Find which process listens on a port](#task-3) | 🛠️ Task |
| 7 | [List all listening ports and their processes](#task-4) | 🛠️ Task |
| 8 | [Scenario: port already in use](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: What a port is

An **IP address** gets a packet to the right *machine*. But a machine runs many programs at once (a web server, an ssh server, a database), so when a packet arrives the kernel needs to know **which program** it belongs to. That's the job of the **port number**.

A **port** is a number from **1 to 65535** attached to every piece of transport-layer traffic. One machine = one building with one street address (the IP) and thousands of numbered **doors** (ports). A server program picks a door and waits behind it; a client knocks on a specific door and the kernel delivers the request to whatever waits there.

```
            one machine, one IP, many doors
   ┌──────────────────────────────────────────┐
   │ port 22   ── ssh server                  │
   │ port 80   ── nginx                       │ ◄── client
   │ port 8000 ── python web server           │
   └──────────────────────────────────────────┘
```

### Address plus port names one program

A full destination is the pair, with a colon: **`192.168.1.20:80`** = port 80 on that machine. Same IP, different port = completely different program. That's why your practice server on port 8000 could run next to nginx on port 80: two doors, no conflict.

### Only one program per door

A port can be claimed by **only one listening program at a time**. A second program trying to listen on a taken port fails immediately with **"address already in use"**.

> **💡 Tip:** Half of all "service is down" tickets come down to one question: **is the right program listening on the right port?** This topic gives you the tools to answer it in seconds.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Run two servers on two ports

Two small websites exist at `/root/site-a` and `/root/site-b`, each with its own `index.html`.

1. Start a web server on port **8000** serving `/root/site-a`, in the background.
2. Start a second on port **9000** serving `/root/site-b`, in the background.
3. Request both ports and save both responses into `/root/answers/two-ports.txt`.

Both servers must still be running when you submit.

**Solution (from the site):**

```bash
python3 -m http.server 8000 -d /root/site-a &
python3 -m http.server 9000 -d /root/site-b &
```

```bash
sleep 1
curl -s http://localhost:8000/ > /root/answers/two-ports.txt
curl -s http://localhost:9000/ >> /root/answers/two-ports.txt
```

Same machine, same IP, two ports, two different answers: **the port number alone decided which program received each request.**

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Well-known ports

A client can't guess which of 65535 ports a web server picked, so the common services have **fixed, agreed port numbers**, the same on every machine in the world: **well-known ports**.

### The numbers to memorise

| Port | Service | Used for |
|------|---------|----------|
| **22** | ssh | remote shell access |
| **25** | smtp | sending mail between servers |
| **53** | dns | name lookups |
| **80** | http | web traffic |
| **443** | https | encrypted web traffic |
| **3306** | mysql | a common database |

When you type a web address without a port, the browser quietly uses 80 for http and 443 for https. The convention typed it for you.

### Below 1024 needs root

Ports **1–1023 are reserved**: on Linux only **root** can start a program that listens on them. A safety rule: a normal user on a shared machine can't start a fake ssh server on port 22 and collect passwords. It's also why practice servers use high ports like 8000.

### Servers keep the convention, clients don't

Only the **server** side is fixed. The client side uses a **random high port** chosen by the kernel, an **ephemeral port**, for that one conversation.

```
curl → your machine :51844  ──────────►  server :443
        (ephemeral, random)                (well-known)
```

> **💡 Tip:** Firewall rules, load balancer configs and error messages all speak in **port numbers**, not service names. Seeing `3306` and thinking "database" without looking it up saves real time.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Match services to ports

Create `/root/answers/ports.txt` with six lines `<service> = <port>` for: `ssh`, `smtp`, `dns`, `http`, `https`, `mysql`.

**Solution (from the site):**

```bash
cat > /root/answers/ports.txt <<'DONE'
ssh = 22
smtp = 25
dns = 53
http = 80
https = 443
mysql = 3306
DONE
```

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Listening and connected sockets

A **socket** is one endpoint of network communication: an **IP address plus a port**, held open by a program. Two kinds:

| Kind | Meaning |
|------|---------|
| **Listening socket** | a server waiting at a door for anyone to knock |
| **Connected socket** | one live conversation between two specific machines |

### `ss -tlnp`: the listening sockets

```bash
ss -tlnp
```

```
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port  Process
LISTEN 0      511    0.0.0.0:80          0.0.0.0:*          users:(("nginx",pid=612,fd=6))
LISTEN 0      128    0.0.0.0:22          0.0.0.0:*          users:(("sshd",pid=301,fd=3))
```

| Flag | Meaning |
|------|---------|
| `-t` | TCP sockets |
| `-l` | listening only |
| `-n` | numbers instead of service names |
| `-p` | show the owning process |

| Column | What it tells you |
|--------|-------------------|
| **Local Address:Port** | where the program listens. `0.0.0.0:80` = port 80 on **every** address; `127.0.0.1:80` = **local connections only** (a very common misconfiguration) |
| **Process** | the program name and PID: "what is actually running on that port" with certainty |

A listening socket's peer is `*` because it isn't talking to anyone yet: an open door, not a conversation.

### `ss -tnp`: the live connections

Drop the `-l` to see established conversations:

```bash
ss -tnp
```

```
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port
ESTAB  0      0      10.0.5.4:22         10.0.5.9:50022
```

Both sides are filled in: someone at `10.0.5.9` on ephemeral port `50022` is connected to ssh on this machine. `ESTAB` = the connection is up and carrying data.

> **💡 Tip:** These two commands are the **transport layer's eyes**. Almost every "is the service up?" question starts with one of them.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Find which process listens on a port

nginx is running as a system service and nobody told you its port. Find it from **live socket state**, not a config file.

1. Write the port nginx listens on to `/root/answers/nginx-port.txt`.
2. Write the **PID** of the nginx process owning the listening socket to `/root/answers/nginx-pid.txt`.

**Solution (from the site):**

```bash
ss -tlnp
```

The nginx line shows `0.0.0.0:80` and something like `users:(("nginx",pid=612,fd=6))`. Record both:

```bash
echo 80 > /root/answers/nginx-port.txt
echo 612 > /root/answers/nginx-pid.txt
```

> **📝 Note:** Use the PID from **your own** `ss` output, not the one in this example.

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: List all listening ports and their processes

Take a full inventory of this machine's open doors, the snapshot an engineer saves before and after a change. Save every listening TCP socket, with numeric ports and the owning process, to `/root/answers/listening-report.txt`. It must include at least nginx and sshd.

**Solution (from the site):**

```bash
ss -tlnp > /root/answers/listening-report.txt
cat /root/answers/listening-report.txt
```

Every line is one open door: nginx on 80, sshd on 22, and anything else running. This is the first command many engineers run on any unfamiliar machine.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: port already in use

**Q:** You deploy a new API service. It refuses to start, and its log ends with `bind: address already in use` for port `8080`. It worked on staging. What do you do first?

- [x] **Run `ss -tlnp | grep 8080` to see which process holds the port**
- [ ] Run `lsof -i :8080` to see which process has the port open
- [ ] Change the new service's config to listen on port `8081` and start it
- [ ] Reboot the server so the port gets freed, then start the service

**Why:** "address already in use" means one thing at the transport layer: **some process already has a listening socket on that port, and the kernel allows only one.** Identify the holder **before changing anything**. `ss -tlnp` with a grep does that in seconds and names the program and PID. Common causes: an old instance of the same service that never exited, or two apps configured with the same port. Once you know who holds it, stop it cleanly or move one service.

| Option | Verdict |
|--------|---------|
| `lsof -i :8080` | answers the same question (the site marks `ss` as best) |
| switch to `8081` | hides a process you haven't identified |
| reboot | the most expensive way to free a port, and often doesn't work: boot-time services reclaim it first |

> **💡 Tip:** **Find the holder, then decide.**

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Concept / command | Remember |
|-------------------|----------|
| port | number 1–65535 that picks the **program** on a machine |
| `IP:port` | the full destination, e.g. `192.168.1.20:80` |
| one listener per port | a second one gets "address already in use" |
| ports 1–1023 | reserved: only **root** may listen |
| ephemeral port | random high client port, one per conversation |
| `ss -tlnp` | **listening** TCP sockets + owning process |
| `ss -tnp` | **established** connections (`ESTAB`) |
| `0.0.0.0:80` vs `127.0.0.1:80` | every address vs local only |
| `ss -tlnp \| grep 8080` | who holds port 8080? |
| `lsof -i :8080` | same question |
| `python3 -m http.server PORT -d DIR` | serve a folder on a port |

| Port | Service |
|------|---------|
| 22 | ssh |
| 25 | smtp |
| 53 | dns |
| 80 | http |
| 443 | https |
| 3306 | mysql |

**Port-conflict rule:** find the holder with `ss -tlnp | grep PORT`, then decide: stop it cleanly or move one service. Never reboot first.

<br>

[↑ Back to top](#top)
