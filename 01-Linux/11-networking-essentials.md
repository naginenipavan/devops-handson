<a id="top"></a>

# 🌐 11 - Networking essentials

> **Source:** <https://www.escbash.com/topics/linux-networking-essentials>
> **Contents:** 3 lessons · 2 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [IPs and interfaces](#lesson-1) | 📖 Lesson |
| 2 | [Ports and listening](#lesson-2) | 📖 Lesson |
| 3 | [Reaching other machines](#lesson-3) | 📖 Lesson |
| 4 | [Add a local hostname to /etc/hosts](#task-1) | 🛠️ Task |
| 5 | [Configure local service discovery](#task-2) | 🛠️ Task |
| 6 | [Scenario: an unreachable service](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: IPs and interfaces

A **network interface** is a lane the machine uses to send and receive traffic. Most Linux servers have at least two:

| Interface | Meaning |
|-----------|---------|
| `lo` | **loopback**, address `127.0.0.1`. Traffic that never leaves the machine |
| `eth0` (or `ens3`, `enp0s3`) | the main outbound interface with the machine's real network address |

### Listing interfaces

```bash
ip addr show     # long form
ip a             # short form, same result
```

One block per interface. The key line is `inet <address>/<mask>`:

```
inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
```

`10.0.2.15` = the IP on that interface; `/24` = the size of the network.

### Just the IP: `hostname -I`

```bash
hostname -I
```

Prints only IP addresses, space-separated. Perfect for scripts.

> **📝 Note:** `ifconfig` is the older command, still in many tutorials but not installed by default on modern Ubuntu. Use `ip a`.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Ports and listening

An IP gets you to a machine; a **port** gets you to a specific service on it. Ports are 16-bit numbers (0–65535).

| Port | Service |
|------|---------|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 5432 | PostgreSQL |
| 6379 | Redis |

### TCP vs UDP

| Protocol | Behaviour | Used by |
|----------|-----------|---------|
| **TCP** | connection-based, ordered, retries on packet loss | HTTP, SSH, most databases |
| **UDP** | fire-and-forget | DNS, VoIP, some games |

As a DevOps engineer you'll deal with TCP almost every time.

### Seeing what's listening: `ss`

```bash
ss -tlnp
```

| Flag | Meaning |
|------|---------|
| `-t` | TCP only |
| `-l` | listening sockets only |
| `-n` | don't resolve port numbers to names (faster, clearer) |
| `-p` | show which process owns the socket |

```
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port  Process
LISTEN 0      511    0.0.0.0:80          0.0.0.0:*          users:(("nginx",pid=1234,fd=6))
LISTEN 0      128    0.0.0.0:22          0.0.0.0:*          users:(("sshd",pid=1500,fd=3))
```

The Process column ties each open port back to its program, ideal for "what is using this port?".

### `0.0.0.0` vs `127.0.0.1`

| Local address | Meaning |
|---------------|---------|
| `0.0.0.0:80` | listen on port 80 on **every interface**: anyone who can reach the machine can connect |
| `127.0.0.1:5432` | **loopback only**: only processes on the same machine can connect |

> **💡 Tip:** When a service is "unreachable from another machine", a common cause is that it bound to `127.0.0.1` when you needed `0.0.0.0`. `ss -tlnp` shows that in one line.

`netstat -tlnp` is the older equivalent; use `ss` unless it's missing.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Reaching other machines

Three tools cover almost every "can I reach it?" check: **`ping`** (raw reachability), **`curl`** (HTTP), **`dig`/`getent`** (name resolution).

### `ping`

```bash
ping -c 4 127.0.0.1      # loopback always answers, no internet needed
ping -c 4 example.com    # needs internet + DNS
```

`-c 4` = send four packets and stop; look for `0% packet loss`. A successful remote ping proves the name resolved, the path works, and the host answers. Some networks block ping on purpose, so a failed ping alone doesn't prove a machine is down.

### `curl`

The lab runs nginx on port 80, so no internet is needed:

```bash
curl http://localhost        # HTML of the nginx welcome page
curl -I http://localhost     # HEAD request: headers only (first line HTTP/1.1 200 OK)
curl -v http://localhost     # verbose: connection, request and response headers
```

If `curl` is missing, use `wget`:

```bash
wget -qO- http://localhost
```

### Resolving names

```bash
dig example.com
dig +short example.com        # just the IPs
getent hosts localhost        # → 127.0.0.1 localhost
```

| Tool | Behaviour |
|------|-----------|
| `dig` | queries DNS servers over the network (times out offline); **never sees `/etc/hosts`** |
| `getent hosts` | uses the system resolver: checks `/etc/hosts` first, then DNS; works offline |

> **💡 Tip:** To confirm an `/etc/hosts` entry, use `getent`, not `dig`.

### `/etc/hosts`

Before DNS, hostnames were mapped to IPs in this plain text file. Linux checks it **before** asking any DNS server, which is handy for testing.

```bash
echo "127.0.0.1 api.local" >> /etc/hosts     # needs root
getent hosts api.local                       # → 127.0.0.1 api.local
curl http://api.local                        # reaches nginx on this machine
```

Local overrides like this are a standard development trick.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Add a local hostname to /etc/hosts

Make `api.local` resolve to `127.0.0.1`. Keep the existing `localhost` line; **add** a new line, don't replace one. Confirm with `getent hosts api.local`.

**Solution (from the site):**

```bash
grep -qE '^127\.0\.0\.1[[:space:]]+api\.local\b' /etc/hosts || echo '127.0.0.1 api.local' >> /etc/hosts
getent hosts api.local
```

The `grep ... ||` guard stops a re-run from adding a duplicate line.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Configure local service discovery

Add three entries to `/etc/hosts` (keep `localhost` intact):

| Name | IP | Note |
|------|----|------|
| `web.local` | `127.0.0.1` | local web service |
| `db.local` | `127.0.0.1` | local database |
| `backup.internal` | `10.42.0.7` | placeholder remote IP |

All three must resolve via `getent hosts`.

**Solution (from the site):**

```bash
grep -qE '^127\.0\.0\.1[[:space:]]+web\.local\b' /etc/hosts || echo '127.0.0.1 web.local' >> /etc/hosts
grep -qE '^127\.0\.0\.1[[:space:]]+db\.local\b' /etc/hosts || echo '127.0.0.1 db.local' >> /etc/hosts
grep -qE '^10\.42\.0\.7[[:space:]]+backup\.internal\b' /etc/hosts || echo '10.42.0.7 backup.internal' >> /etc/hosts
getent hosts web.local db.local backup.internal
```

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: an unreachable service

**Q:** A colleague says the API on `server-b` is broken: `curl http://server-b:8080` from their laptop gives **connection refused**. You log in to `server-b`. What's your first check?

- [x] **`ss -tlnp | grep 8080` on server-b**
- [ ] Check `api.service` with `systemctl status` on server-b
- [ ] `curl http://localhost:8080` on server-b
- [ ] Restart `api.service` with `systemctl restart`

**Why:** "connection refused" means the handshake reached the machine and **nothing was accepting on that port**. `ss -tlnp | grep 8080` answers both *is anything listening?* and *on which address?*:

- no line → the service is down or bound elsewhere
- `127.0.0.1:8080` instead of `0.0.0.0:8080` → the classic bind bug (works locally, refuses remote)

| Option | Verdict |
|--------|---------|
| `systemctl status`, local `curl` | each answers only half the question; good follow-ups |
| restart first | guessing; fails the same way or hides the cause |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `ip a` | interfaces and IPs |
| `hostname -I` | IPs only |
| `ss -tlnp` | listening TCP sockets + owning process |
| `ping -c 4 host` | reachability test |
| `curl URL` / `curl -I` / `curl -v` | fetch / headers only / verbose |
| `wget -qO- URL` | curl alternative |
| `dig +short name` | DNS lookup (DNS only) |
| `getent hosts name` | resolver lookup (`/etc/hosts` then DNS) |
| `/etc/hosts` | local name → IP overrides |

| Port | Service |
|------|---------|
| 22 / 80 / 443 | SSH / HTTP / HTTPS |
| 5432 / 6379 | PostgreSQL / Redis |

**Debug rule:** *connection refused* → check `ss -tlnp` for the port and whether it's bound to `127.0.0.1` vs `0.0.0.0`.

<br>

[↑ Back to top](#top)
