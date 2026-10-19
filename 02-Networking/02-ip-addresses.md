<a id="top"></a>

# 🔢 02 - IP addresses

> **Source:** <https://www.escbash.com/topics/net-ip-addresses>
> **Contents:** 4 lessons · 5 hands-on tasks

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [What an IPv4 address looks like](#lesson-1) | 📖 Lesson |
| 2 | [Find your IP address](#task-1) | 🛠️ Task |
| 3 | [Private and public addresses](#lesson-2) | 📖 Lesson |
| 4 | [Classify private and public addresses](#task-2) | 🛠️ Task |
| 5 | [Classify more addresses](#task-3) | 🛠️ Task |
| 6 | [The loopback address 127.0.0.1](#lesson-3) | 📖 Lesson |
| 7 | [Serve and fetch on loopback](#task-4) | 🛠️ Task |
| 8 | [IPv6 in one lesson](#lesson-4) | 📖 Lesson |
| 9 | [Ping the IPv6 loopback](#task-5) | 🛠️ Task |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: What an IPv4 address looks like

Every machine on a network needs an **address** so packets know where to go. The most common kind is **IPv4**: four numbers separated by dots, like `192.168.1.42`, each between **0 and 255**.

| Address | Valid? | Why |
|---------|--------|-----|
| `192.168.1.42` | ✅ | four numbers, each 0–255 |
| `192.168.1.300` | ❌ | 300 is over 255 |
| `10.1.2` | ❌ | only three numbers |

### Why 0 to 255

Underneath, an IPv4 address is **32 bits**, split into four groups of **8 bits**. Eight bits hold 0–255. The dotted form exists only so humans can read it.

### See your machine's addresses

```bash
ip addr
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
    inet 127.0.0.1/8 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 172.16.0.2/24 scope global eth0
```

Look for lines starting with **`inet`**: each is an IPv4 address the machine answers to. This machine has two (one under `lo`, one under `eth0`). The number after the slash describes the network (covered in a later topic; ignore for now).

> **💡 Tip:** Which address does this server have? Is it the one the load balancer expects? Almost every network question starts with reading `ip addr`.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Find your IP address

Find the IPv4 address of the **`eth0`** interface and write it to `/root/answers/my-ip.txt`: a single line, only the address (**no** `/24` part).

**Solution (from the site):**

```bash
ip addr                                   # read the inet line under eth0
echo 172.16.0.2 > /root/answers/my-ip.txt
```

Use the address your own machine shows; it may differ from this example.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Private and public addresses

There aren't enough IPv4 addresses for every device on Earth to own a unique one, so some ranges are marked **private**: free for anyone to use inside their own network and **never routed on the public internet**. **Public** addresses are the rest: globally unique and reachable from anywhere.

### The three private ranges (memorise these)

| Range | Rule of thumb |
|-------|---------------|
| `10.0.0.0` – `10.255.255.255` | anything starting with `10.` |
| `172.16.0.0` – `172.31.255.255` | starts `172.`, second number **16 through 31 only** |
| `192.168.0.0` – `192.168.255.255` | anything starting with `192.168.` |

> **⚠️ Warning:** The middle range is the tricky one. `172.20.1.1` is private but `172.32.1.1` is **public** (32 is outside 16–31). Likewise `192.169.5.5` is public: only `192.168.` is private, not all of `192.`.

### Many private devices, one public address

```
                         internet
                            │
                    public 84.20.5.7
                            │
                         ┌──┴───┐
                         │router│
                         └──┬───┘
         ┌──────────────────┼──────────────────┐
   laptop 192.168.1.10  phone 192.168.1.11  tv 192.168.1.12
            home network (private)
```

Millions of homes reuse `192.168.1.10` at the same time without conflict, because those addresses never appear on the internet.

```bash
ip addr show eth0
```

Your sandbox's `eth0` address falls inside one of the three private ranges, like nearly every server a DevOps engineer touches. Cloud servers almost always have private addresses, with public ones attached only where needed.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Classify private and public addresses

`/root/addresses/list1.txt` holds 8 IPv4 addresses. Decide private or public for each and write `/root/answers/classify1.txt`: one line per address, in list order, format `ADDRESS private|public`.

**Solution (from the site):**

```bash
cat > /root/answers/classify1.txt <<'DONE'
10.4.200.7 private
8.8.8.8 public
192.168.50.1 private
54.180.2.99 public
172.20.14.3 private
1.1.1.1 public
10.255.0.1 private
142.250.72.14 public
DONE
```

Everything starting with `10.` or `192.168.` is private, and `172.20.` sits inside 172.16–172.31, so it's private too.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Classify more addresses

A trickier list at `/root/addresses/list2.txt`: every address sits close to the edge of a private range. Same format; write `/root/answers/classify2.txt`.

**Solution (from the site):**

```bash
cat > /root/answers/classify2.txt <<'DONE'
172.32.0.1 public
172.16.0.1 private
192.169.4.20 public
192.168.255.254 private
172.31.255.1 private
172.15.9.9 public
11.0.0.5 public
10.0.0.5 private
DONE
```

The 172 private range is 172.16 through 172.31, so **32 and 15 are outside** it. Only `192.168.` and `10.` are private, so `192.169.` and `11.` are public.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: The loopback address 127.0.0.1

Every machine has a special address that always means "**this machine itself**": `127.0.0.1`, the **loopback** address. `localhost` is a friendly alias for it. A packet sent to `127.0.0.1` is turned straight around inside the machine; it never touches a cable, a wifi radio, or any other computer.

### What loopback is for

- Lets programs on one machine talk using normal networking, **with no network required** (e.g. a database and app on the same server).
- A **safety tool**: a server bound to `127.0.0.1` accepts connections only from its own machine, so nothing outside can reach it, ever.

### Bind a server to loopback only

```bash
python3 -m http.server 8000 --bind 127.0.0.1 &
curl http://127.0.0.1:8000
```

```
<!DOCTYPE HTML>
<html lang="en">
<head>
<title>Directory listing for /</title>
...
```

The `curl` works because it comes from the same machine. A request from any other machine would fail: the server isn't listening on any address the outside world can reach.

> **💡 Tip:** DevOps engineers do this deliberately: an internal admin panel, a metrics endpoint, a database bound to `127.0.0.1` can't be attacked from the network, no firewall needed. And when a service works with `curl` on the machine but **not from outside**, a loopback-only bind is one of the first things to suspect.

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Serve and fetch on loopback

1. Start the Python web server on port **8001**, bound to **`127.0.0.1` only**, in the background.
2. Fetch a page over loopback and save it to `/root/answers/loopback.txt`.

**Solution (from the site):**

```bash
python3 -m http.server 8001 --bind 127.0.0.1 &
sleep 1
curl http://127.0.0.1:8001 > /root/answers/loopback.txt
```

The server accepts connections only from this machine; from any other machine port 8001 looks closed.

<br>

---

<br>

<a id="lesson-4"></a>

## 📖 Lesson 4: IPv6 in one lesson

IPv4 has 32 bits, about **4.3 billion** addresses: far fewer than the world's devices. Private ranges stretched the supply, but the real fix is **IPv6**: **128 bits**, enough to give every grain of sand on Earth billions of addresses. The goal here is **recognition**; the rest of the course uses IPv4.

### What an IPv6 address looks like

Hexadecimal digits (0–9, a–f) in **eight groups separated by colons**:

```
2001:0db8:0000:0000:0000:0000:0370:7334
```

Two shortening rules:

1. Leading zeros in a group can be dropped.
2. **One** run of all-zero groups can be replaced with `::`.

```
2001:db8::370:7334        ← the same address, shortened
```

> **💡 Tip:** Dots mean IPv4, colons mean IPv6.

### The IPv6 loopback

The twin of `127.0.0.1`: one bit set, everything else zero, shortened to **`::1`**.

```bash
ip -6 addr show lo
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN ...
    inet6 ::1/128 scope host
```

`-6` shows only IPv6 addresses. When `::1` or a colon-filled address appears in a config file, log line or listening-ports list, you'll know you're looking at IPv6.

<br>

---

<br>

<a id="task-5"></a>

## 🛠️ Task 5: Ping the IPv6 loopback

Send exactly **4** ping packets to `::1` and save the full output (including the statistics summary) to `/root/answers/ipv6-ping.txt`. Every packet should be received.

**Solution (from the site):**

```bash
ping -c 4 ::1 > /root/answers/ipv6-ping.txt
```

Also works as `ping -6 ::1`. Replies are instant because, like `127.0.0.1`, the packets never leave the machine.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Concept / command | Remember |
|-------------------|----------|
| IPv4 | 4 numbers, each 0–255 (32 bits) |
| `ip addr` / `ip addr show eth0` | list addresses; read the `inet` lines |
| `ip -6 addr show lo` | IPv6 only |
| Private ranges | `10.x.x.x` · `172.16–31.x.x` · `192.168.x.x` |
| Public | everything else (`172.32.x.x`, `192.169.x.x`, `11.x.x.x` are public!) |
| `127.0.0.1` / `localhost` | loopback: this machine, never leaves it |
| `--bind 127.0.0.1` | make a server reachable only from the same machine |
| IPv6 | 128 bits, hex, colons, `::` shortens one run of zeros |
| `::1` | IPv6 loopback |

| Private? | Address |
|----------|---------|
| ✅ | `10.255.0.1`, `172.16.0.1`, `172.31.255.1`, `192.168.255.254` |
| ❌ | `172.15.9.9`, `172.32.0.1`, `192.169.4.20`, `11.0.0.5` |

<br>

[↑ Back to top](#top)
