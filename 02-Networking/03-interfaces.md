<a id="top"></a>

# 🚪 03 - Interfaces

> **Source:** <https://www.escbash.com/topics/net-interfaces>
> **Contents:** 3 lessons · 4 hands-on tasks

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [What an interface is](#lesson-1) | 📖 Lesson |
| 2 | [List interfaces with ip link](#task-1) | 🛠️ Task |
| 3 | [Reading ip addr output](#lesson-2) | 📖 Lesson |
| 4 | [Match addresses to interfaces](#task-2) | 🛠️ Task |
| 5 | [Creating interfaces, the dummy type](#lesson-3) | 📖 Lesson |
| 6 | [Create a dummy interface](#task-3) | 🛠️ Task |
| 7 | [Add a second address to an interface](#task-4) | 🛠️ Task |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: What an interface is

An IP address doesn't float in the air: it's attached to a **network interface**, the connection point the OS uses to send and receive packets. Think of a building: the building is the machine, each **door** is an interface, and each door has its own address posted on it.

| Interface | Meaning |
|-----------|---------|
| `lo` | the **loopback** interface: an internal door leading back inside. Packets to `127.0.0.1` and `::1` go through it and never leave the machine |
| `eth0` | the machine's connection to the **outside network**. Every packet you sent to the internet went through it |

```
              one machine
   ┌─────────────────────────────┐
   │          programs           │
   │        │          │         │
   │   lo 127.0.0.1  eth0 172.16.0.2 ──► outside network
   └─────────────────────────────┘
```

### Listing the doors

```bash
ip link
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN ...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP ...
```

| State | Meaning |
|-------|---------|
| `UP` | enabled and passing traffic |
| `DOWN` | a closed door: its addresses stop answering |
| `UNKNOWN` | normal for `lo`; nothing to worry about |

> **💡 Tip:** Real servers often have more doors: a second NIC, VPN interfaces, bridges for containers. A door showing `DOWN` when it should be `UP` explains many outages in one line.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: List interfaces with ip link

Write **one line per interface** to `/root/answers/interfaces.txt`: the interface name, a space, then the state word exactly as printed (e.g. `eth0 UP`). Include every interface.

**Solution (from the site):**

```bash
ip link

cat > /root/answers/interfaces.txt <<'DONE'
lo UNKNOWN
eth0 UP
DONE
```

`lo` normally reports `UNKNOWN`; `eth0` reports `UP` because it's carrying traffic.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Reading `ip addr` output

```bash
ip addr show eth0
```

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 06:a2:1f:44:9c:d1 brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.2/24 brd 172.16.0.255 scope global eth0
    inet6 fe80::4a2:1fff:fe44:9cd1/64 scope link
```

| Line | What it tells you |
|------|-------------------|
| `2: eth0: <...flags> ... state UP` | interface number, name, flags. **`state UP`** is the first thing to check |
| `inet 172.16.0.2/24` | the **IPv4 address**; `/24` is the **prefix length** (how large the local network is, next topic) |
| `scope global` | meant for talking to other machines (on `lo` you'd see `scope host`: valid only inside this machine) |
| `link/ether 06:a2:...` | the **MAC address** of the card (a later topic) |
| `inet6 fe80::...` | IPv6 address; `fe80` = **link-local**, which every IPv6 interface gives itself automatically |

One interface can carry **several `inet` lines**, one per address; machines hosting many services often answer on several addresses through one interface.

> **💡 Tip:** Reading this fast is a core DevOps skill: **state, address, prefix, scope**: four things, one glance.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Match addresses to interfaces

Map **every IPv4 address** to its interface in `/root/answers/addr-map.txt`: `ADDRESS INTERFACE` per line, **without** the `/prefix`. Cover both IPv4 addresses the machine has.

**Solution (from the site):**

```bash
ip addr

cat > /root/answers/addr-map.txt <<'DONE'
127.0.0.1 lo
172.16.0.2 eth0
DONE
```

Use the `eth0` address your machine actually shows; loopback is always on `lo`.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Creating interfaces, the dummy type

Interfaces aren't only hardware. Linux lets you **create virtual interfaces** with a command, and much of modern infrastructure (container networking, VPNs, bridges) is built from them. The simplest is the **dummy** interface: it connects to nothing (a door that leads nowhere) but the machine treats it like any real one: it holds addresses, goes up and down, and answers pings from its own machine.

### The three commands

```bash
ip link add name test0 type dummy        # 1. create it
ip addr add 10.5.5.1/24 dev test0        # 2. give it an address (with prefix length)
ip link set test0 up                     # 3. enable it (new interfaces start DOWN)

ping -c 2 10.5.5.1
```

```
64 bytes from 10.5.5.1: icmp_seq=1 ttl=64 time=0.028 ms
64 bytes from 10.5.5.1: icmp_seq=2 ttl=64 time=0.025 ms
```

Remove it with `ip link del test0`, which also removes its addresses.

> **💡 Tip:** Dummy interfaces hold a **stable service address** independent of any physical card, and are a safe practice ground: create, address, break and delete them without touching the machine's real connectivity.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Create a dummy interface

1. Create a dummy interface named **`dummy0`**.
2. Give it **`10.9.9.1/24`**.
3. Bring it **up**.
4. Ping the address to confirm it answers.

**Solution (from the site):**

```bash
ip link add name dummy0 type dummy
ip addr add 10.9.9.1/24 dev dummy0
ip link set dummy0 up
ping -c 2 10.9.9.1
```

Once up, the machine answers ping on its address even though the interface connects to nothing.

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Add a second address to an interface

One interface can carry several addresses. On `dummy0` (still holding `10.9.9.1/24`), add **`10.9.9.2/24`** (keeping the first) and ping both.

**Solution (from the site):**

```bash
ip addr add 10.9.9.2/24 dev dummy0
ping -c 2 10.9.9.1
ping -c 2 10.9.9.2
```

The interface now shows two `inet` lines and answers on both addresses at once.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `ip link` | list interfaces and their **state** |
| `ip addr` / `ip addr show eth0` | addresses per interface |
| `ip link add name X type dummy` | create a dummy interface |
| `ip addr add A.B.C.D/N dev X` | add an address (repeat for more) |
| `ip link set X up` / `down` | enable / disable |
| `ip link del X` | delete (addresses go with it) |

| Output field | Meaning |
|--------------|---------|
| `state UP / DOWN / UNKNOWN` | enabled / closed door / normal for `lo` |
| `inet a.b.c.d/N` | IPv4 address / prefix length |
| `scope global` / `scope host` | for other machines / only inside this machine |
| `link/ether` | MAC address |
| `inet6 fe80::` | IPv6 link-local |

<br>

[↑ Back to top](#top)
