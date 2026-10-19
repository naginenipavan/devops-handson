<a id="top"></a>

# 🪪 05 - MAC addresses and ARP

> **Source:** <https://www.escbash.com/topics/net-mac-arp>
> **Contents:** 2 lessons · 3 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [MAC addresses](#lesson-1) | 📖 Lesson |
| 2 | [Find the MAC address of each interface](#task-1) | 🛠️ Task |
| 3 | [ARP, finding the MAC for an IP](#lesson-2) | 📖 Lesson |
| 4 | [View the ARP table](#task-2) | 🛠️ Task |
| 5 | [Clear and rebuild the ARP table](#task-3) | 🛠️ Task |
| 6 | [Scenario: two machines with the same IP](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: MAC addresses

A **MAC address** is the **hardware address of a network card**, burned in by the manufacturer: six pairs of hex digits, e.g. `52:54:00:ab:12:34`. Every network card in the world gets its own value.

### Why a second address?

Think of sending a package:

| Address | Analogy | Job |
|---------|---------|-----|
| **IP** | street address | gets the package to the right **building**, across any number of roads |
| **MAC** | name on the apartment door | on the final street, says exactly **which door** to knock on |

```
one packet, two addresses
  IP address  → picks the machine, works across networks
  MAC address → picks the card on the local network
```

### Where each address is used

- The **IP** travels with the packet from the first machine to the last and **stays the same** the whole way.
- The **MAC** only matters on **one local network segment**. Every time a packet crosses a router into the next network, the router puts a **new MAC** on it.

### Worked example: a packet crossing three networks

> **📝 Note:** This section is an extra explanation I added, not text from the site.

Think of a parcel sent across the country by relay drivers, each covering one stretch of road. The **IP** is the final destination on the label, written once and never changed. The **MAC** is "which driver hands it to the next one" on the current stretch, and it changes at every stretch.

```
 Laptop ──────► Router A ──────► Router B ──────► Server
         net 1            net 2            net 3

 IP  (src → dst)   laptop → server   laptop → server   laptop → server   ← never changes
 MAC (src → dst)   laptop → RouterA  RouterA → RouterB RouterB → server   ← new at every hop
```

1. **Laptop → Router A.** The laptop can't reach the server directly, so it hands the packet to its **gateway**, Router A. MAC: laptop's card → Router A's card. IP: still "to the server".
2. **Router A → Router B.** Router A reads the **IP** destination, picks the next hop (Router B) and **rewrites the MAC** to Router A's card → Router B's card. The IP is untouched.
3. **Router B → Server.** Same again, ending with the server's MAC.

So "a MAC only matters on one local network segment" means it is only used to deliver a packet to the **next device on the same local network** (one hop). It means nothing beyond that hop, so each router swaps it for the next hop's MAC.

This is why ARP (Lesson 2) only asks "who has this IP?" about devices on **its own segment**. For a remote server a machine asks for the **gateway's** MAC instead, which ties back to the "local or remote" decision in [module 4](04-subnets-and-cidr.md).

### Look at your own MAC address

```bash
ip link show eth0
```

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    link/ether 52:54:00:ab:12:34 brd ff:ff:ff:ff:ff:ff
```

- The value after **`link/ether`** is this card's MAC.
- **`brd`** is the **broadcast MAC** (`ff:ff:ff:ff:ff:ff`), used to reach every card on the local network at once (next lesson).

> **💡 Tip:** You meet MAC addresses when a cloud firewall rule, a DHCP server or a VM config identifies a machine by its card instead of its IP.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Find the MAC address of each interface

Inspect every network interface and find the hardware address of each. Notice that the **loopback interface is different** from the others.

Record the MAC of **`eth0`** in `/root/answers/eth0-mac.txt`: the six hex pairs **exactly as the system reports them**.

**Solution (from the site):**

```bash
ip link                                  # list interfaces, read the link/ether line
ip link show eth0 | awk '/link\/ether/ {print $2}' > /root/answers/eth0-mac.txt
```

Copying the address by hand with `echo` works just as well. The loopback interface `lo` has no real MAC because it never touches a physical network.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: ARP, finding the MAC for an IP

To send a packet to `192.168.1.1` your machine knows the IP, but to put the packet on the wire it needs the **MAC** of that machine's card. **ARP** (Address Resolution Protocol) finds it.

ARP is like **shouting in a room**:

```
your machine ──► (broadcast to everyone)  "who has 192.168.1.1?"
owner of 192.168.1.1 ──► (directly back)  "that's me, my MAC is 52:54:00:aa:bb:cc"
```

**Broadcast question, unicast answer.** Every machine hears the question; only the owner answers.

### The neighbor table caches the answers

Asking before every packet would be slow, so the kernel caches answers in the **neighbor table**:

```bash
ip neigh
```

```
192.168.1.1 dev eth0 lladdr 52:54:00:aa:bb:cc REACHABLE
```

Each line maps an **IP → MAC** (`lladdr` = link layer address). The state at the end says how fresh it is:

| State | Meaning |
|-------|---------|
| `REACHABLE` | recently confirmed |
| `STALE` | old but still usable |
| `FAILED` | nobody answered |

### Ask on purpose with `arping`

```bash
arping -c 2 192.168.1.1
```

```
ARPING 192.168.1.1
60 bytes from 52:54:00:aa:bb:cc (192.168.1.1): index=0 time=0.512 msec
```

> **⚠️ Warning:** When a machine can ping the internet but **not its neighbor**, check the neighbor table first. A `FAILED` entry means the problem is on the **local network**, before any router is involved.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: View the ARP table

Every packet to the internet goes through the **default gateway**, so it's the neighbor the machine talks to most. Prove the machine knows the gateway's MAC.

1. Find the **default gateway IP** (printed in the routing table).
2. Make sure the neighbor table has a **resolved** entry for it (a MAC, not `FAILED`).
3. Save the gateway's neighbor-table line, including the MAC, to `/root/answers/gateway-neighbor.txt`.

**Solution (from the site):**

```bash
ip route                                   # the IP after "default via" is the gateway
```

```bash
ping -c 2 <gateway-ip>
ip neigh
ip neigh show <gateway-ip> > /root/answers/gateway-neighbor.txt
```

`arping -c 2 <gateway-ip>` works instead of `ping` too. Either way the table now holds the gateway MAC.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Clear and rebuild the ARP table

The neighbor table is only a **cache**. Prove it: empty it, then watch it refill when traffic flows.

1. **Flush** the whole table and, **on the same command line** (chain with `&&`), save its state to `/root/answers/neigh-before.txt`. The gateway must **not** appear resolved in this snapshot.
2. Send traffic to the default gateway so the kernel must resolve it again.
3. Save the table to `/root/answers/neigh-after.txt`. It must show the gateway entry **with its MAC** again.

**Solution (from the site):**

```bash
ip neigh flush all && ip neigh > /root/answers/neigh-before.txt
ping -c 2 <gateway-ip>
ip neigh > /root/answers/neigh-after.txt
```

> **⚠️ Warning:** The flush and the first snapshot must be **one command line**. Your terminal session itself talks through the gateway, so the moment the shell prints anything (even a new prompt) the kernel has already asked for the gateway's MAC again. Chained with `&&`, the snapshot runs before any output goes out.

The flush empties the cache; when the ping needs the gateway, the kernel sends a fresh ARP question and the entry reappears. Nothing breaks when you flush this table: it's rebuilt on demand.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: two machines with the same IP

**Q:** Users report an internal app at `10.0.2.50` works for a minute, fails, then works again. You `ping 10.0.2.50` from the same network and every ping answers, but `ssh` connections keep landing on what looks like a completely different machine. A colleague suspects someone plugged in a test box with the same IP. What do you do first?

- [x] **Check the neighbor table with `ip neigh`, then send `arping 10.0.2.50` and see how many different MAC addresses answer**
- [ ] Run `ping 10.0.2.50` for a few minutes and watch whether replies become unstable
- [ ] Compare the MAC in `ip neigh` now with the MAC recorded for the real server in your inventory
- [ ] Reboot the app server at `10.0.2.50` and see if the problem clears

**Why:** this is the **direct test for a duplicate IP**. `arping` shows every card that claims the address. **Two replies with two different MACs = two machines own the IP**, and traffic lands on whichever answered ARP most recently, which explains the on-off failures perfectly. Fix: find the second machine and change or disconnect it.

| Option | Problem |
|--------|---------|
| watching `ping` | proves little: *some* machine always answers |
| compare with inventory | only checks one moment, not who else is answering |
| reboot the real server | just hands the address to the intruder for a while |

> **💡 Tip:** An app that flips between working and broken while `ping` stays green is the **classic signature of a duplicate IP**. When symptoms come and go on a local network, check the MAC layer before touching any service.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Concept / command | Remember |
|-------------------|----------|
| MAC address | hardware address of a NIC: 6 hex pairs, e.g. `52:54:00:ab:12:34` |
| IP vs MAC | IP picks the **machine** across networks; MAC picks the **card** on the local segment |
| IP across hops | stays the same end to end |
| MAC across hops | replaced at every router |
| `ip link show eth0` | MAC is on the `link/ether` line (`lo` has none) |
| `brd ff:ff:ff:ff:ff:ff` | broadcast MAC: every card on the local network |
| ARP | "who has IP X?" (broadcast) → "me, MAC Y" (unicast) |
| `ip neigh` | the neighbor (ARP) table: `IP dev eth0 lladdr MAC STATE` |
| `REACHABLE / STALE / FAILED` | fresh / old but usable / nobody answered |
| `arping -c 2 IP` | send an ARP question on purpose; count distinct MACs answering |
| `ip route` | find the default gateway (`default via ...`) |
| `ip neigh flush all` | empty the cache (it rebuilds on demand) |

**Debug rules:** can ping the internet but not a neighbor → `ip neigh`, `FAILED` = local-network problem. Works-then-fails while ping is green → `arping` and count MACs (duplicate IP).

<br>

[↑ Back to top](#top)
