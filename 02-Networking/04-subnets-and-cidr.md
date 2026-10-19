<a id="top"></a>

# 🧮 04 - Subnets and CIDR

> **Source:** <https://www.escbash.com/topics/net-subnets-cidr>
> **Contents:** 4 lessons · 6 hands-on tasks

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [The netmask](#lesson-1) | 📖 Lesson |
| 2 | [Find the network part of an address](#task-1) | 🛠️ Task |
| 3 | [CIDR notation](#lesson-2) | 📖 Lesson |
| 4 | [Count hosts in a subnet](#task-2) | 🛠️ Task |
| 5 | [Same network or not](#lesson-3) | 📖 Lesson |
| 6 | [Decide local or remote](#task-3) | 🛠️ Task |
| 7 | [Network and broadcast addresses](#lesson-4) | 📖 Lesson |
| 8 | [Find network and broadcast addresses](#task-4) | 🛠️ Task |
| 9 | [More subnet calculations](#task-5) | 🛠️ Task |
| 10 | [Plan subnets for a small network](#task-6) | 🛠️ Task |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: The netmask

An IP address is two pieces glued together: the **front** names the **network**, the **back** names the **machine inside it**. Like a street address: the street name is shared by every house, the house number picks one.

The **netmask** says where the split is. `255.255.255.0` = the first three numbers are the network part, the last is the host part. In binary the mask is a run of **1s followed by 0s**; address bits under the 1s are network, under the 0s are host.

```
address  192.168.1.42    mask 255.255.255.0

bits     11000000.10101000.00000001 . 00101010
         └──────── network ─────────┘   └ host ┘
```

### One network, many hosts

With mask `255.255.255.0`, all addresses starting `192.168.1.` form one network:

```
network 192.168.1.0
   ├── host .10
   ├── host .11
   └── host .42
```

### Finding the network part

Keep the address numbers where the mask is **255**; set them to **0** where the mask is **0**.

| Address | Mask | Network |
|---------|------|---------|
| `192.168.1.42` | `255.255.255.0` | `192.168.1.0` |
| `10.44.8.200` | `255.255.0.0` | `10.44.0.0` |

> **💡 Tip:** This split is the base fact under every routing and firewall decision: *which machines can reach each other directly? which addresses does this rule cover?* Both start with the mask.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Find the network part of an address

`/root/subnet-work/pairs.txt` holds 5 lines, each an IP address and its netmask. Compute the network address for each and write `/root/answers/network-part.txt`: one network address per line, in order, nothing else.

**Solution (from the site):**

```bash
cat > /root/answers/network-part.txt <<'DONE'
192.168.10.0
10.44.0.0
172.0.0.0
192.168.77.0
10.0.99.0
DONE
```

> **⚠️ Warning:** The third one is the trap: mask `255.0.0.0` keeps only the first number, so `172.20.5.9` belongs to network `172.0.0.0`.

*(The pairs themselves live in the lab file; only the answers are shown on the site.)*

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: CIDR notation

Writing `255.255.255.0` every time is clumsy. **CIDR notation** replaces the mask with one number: **how many bits, counted from the left, belong to the network**. `255.255.255.0` = 24 one-bits = **`/24`**. That's the number after every address in `ip addr`: `172.16.0.2/24` = address `172.16.0.2` with a 24-bit network part.

### The three sizes to know by heart

| Prefix | Mask | Network part | Addresses |
|--------|------|--------------|-----------|
| `/8` | `255.0.0.0` | first number | over 16 million |
| `/16` | `255.255.0.0` | first two numbers | 65,536 |
| `/24` | `255.255.255.0` | first three numbers | 256 |

The prefix needn't land on a dot: a `/26` splits **inside** the last number, which is how one `/24` gets carved into smaller pieces. The math is the same.

### Counting usable hosts

A **bigger prefix = a smaller network**. With prefix `/p` there are `32 − p` host bits, so the network holds `2^(32−p)` addresses. Two are reserved (next lessons), so:

```
usable hosts = 2^(32 − p) − 2
```

```bash
awk 'BEGIN { p = 24; print 2^(32-p) - 2 }'      # 254
```

A `/24` holds **254** usable hosts; a `/16` holds **65,534**.

> **💡 Tip:** Every VPC, subnet and firewall rule is written in CIDR, and choosing `/24` vs `/26` decides how many servers fit. A DevOps engineer reads `/26` and thinks "62 hosts".

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Count hosts in a subnet

Usable hosts for a **/24**, **/26** and **/16**, written to `/root/answers/host-counts.txt` as `PREFIX COUNT`, in that order.

**Solution (from the site):**

```bash
awk 'BEGIN { print 2^(32-24)-2, 2^(32-26)-2, 2^(32-16)-2 }'

cat > /root/answers/host-counts.txt <<'DONE'
/24 254
/26 62
/16 65534
DONE
```

Each prefix leaves `32 − p` host bits, and two addresses per network are reserved.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Same network or not

Before sending any packet, a machine asks **"is the destination on my network or not?"** It applies its own netmask to its own address and to the destination:

| Result | Action |
|--------|--------|
| **same** network part → **local** | deliver directly |
| **different** network part → **remote** | hand it to the **gateway** (the router that knows the way out) |

The postal rule: a letter for your own street you can walk over yourself; anything else goes to the post office.

### Working an example

Machine `192.168.1.42/24` → network part `192.168.1`.

| Destination | Network part | Decision |
|-------------|--------------|----------|
| `192.168.1.77` | `192.168.1` | **local**: deliver directly |
| `192.168.2.77` | `192.168.2` | **remote**: send to gateway |
| `8.8.8.8` | `8.8.8` | **remote**: send to gateway |

The machine ignores distance, ping time, and whether the destination even exists. **Only the address math counts.**

### Where the machine keeps this rule

```bash
ip -4 addr show eth0
```

```
inet 172.30.0.2/30 brd 172.30.0.3 scope global eth0
```

The `/30` means this machine treats only the four addresses `172.30.0.0`–`172.30.0.3` (itself and its gateway) as local and everything else as remote.

> **⚠️ Warning:** Whole categories of outage are this decision going wrong. Machines with different masks can each believe the other is remote, or a wrong mask makes a machine try to deliver directly to an address no local network holds.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Decide local or remote

`/root/subnet-work/routing-question.txt` shows a machine's interface config and 6 destinations. For each, decide **local** or **remote** and write `/root/answers/local-remote.txt` as `ADDRESS local|remote`, in file order.

**Solution (from the site):**

The machine is a `/24` with network part `10.50.20`, so only destinations starting with those three numbers are local:

```bash
cat > /root/answers/local-remote.txt <<'DONE'
10.50.20.200 local
10.50.21.5 remote
8.8.8.8 remote
10.50.20.9 local
10.99.20.14 remote
10.50.20.254 local
DONE
```

`10.50.21.5` and `10.99.20.14` look close, but **any** difference in the network part means the packet goes to the gateway.

<br>

---

<br>

<a id="lesson-4"></a>

## 📖 Lesson 4: Network and broadcast addresses

A `/24` has 256 addresses but only 254 usable hosts. In every subnet, the **first** and **last** addresses are reserved.

| Address | Host bits | Role |
|---------|-----------|------|
| **Network address** (first) | all **0** | the name of the subnet itself, not a machine. Shows up in routing tables and firewall rules to mean "this whole subnet" |
| **Broadcast address** (last) | all **1** | a packet sent here is delivered to **every machine on the subnet at once**. Used for questions like "who on this network owns this address?" (later topic) |
| everything in between | | **usable hosts** |

For `192.168.1.0/24`:

```
network    192.168.1.0
first host 192.168.1.1
last host  192.168.1.254
broadcast  192.168.1.255
```

The pattern holds for every prefix. For `10.7.0.0/16`: network `10.7.0.0`, broadcast `10.7.255.255`, usable `10.7.0.1` → `10.7.255.254`.

> **📝 Note:** In a `/16`, `10.7.1.255` and `10.7.4.0` are perfectly usable hosts: **only the very first and very last address of the whole subnet are reserved.**

> **⚠️ Warning:** Assigning the network or broadcast address to a server is a classic configuration mistake; cloud consoles reject it with errors that only make sense once you know these two addresses exist.

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Find network and broadcast addresses

`/root/subnet-work/boundaries.txt` lists two CIDR addresses. For each write one line to `/root/answers/boundaries.txt` with five fields: **CIDR as given, network, broadcast, first usable, last usable.**

**Solution (from the site):**

```bash
cat > /root/answers/boundaries.txt <<'DONE'
192.168.4.77/24 192.168.4.0 192.168.4.255 192.168.4.1 192.168.4.254
10.7.200.9/16 10.7.0.0 10.7.255.255 10.7.0.1 10.7.255.254
DONE
```

Zero the host part for the network address, max it for the broadcast, step one inward for the usable range. For the `/16` the host part is the last two numbers.

<br>

---

<br>

<a id="task-5"></a>

## 🛠️ Task 5: More subnet calculations

`/root/subnet-work/drill.txt` has 8 mixed questions (network addresses, broadcast addresses, host counts, the same-network decision). Write `/root/answers/drill.txt` as `NUMBER ANSWER` per line; question 8 is `yes`/`no`.

**Solution (from the site):**

```bash
cat > /root/answers/drill.txt <<'DONE'
1 192.168.9.0
2 192.168.9.255
3 62
4 510
5 10.20.0.0
6 172.16.5.1
7 172.16.5.254
8 no
DONE
```

- Q4: a `/23` leaves 9 host bits → 2⁹ − 2 = **510**.
- Q8: the two `/24` network parts are `192.168.1` and `192.168.2`, which differ → **no**.

*(The question text lives in the lab file; only the answers are shown on the site.)*

<br>

---

<br>

<a id="task-6"></a>

## 🛠️ Task 6: Plan subnets for a small network

Three teams need address space; don't waste a `/24` on ten machines. `/root/subnet-work/teams.txt` lists each team's size and the allowed prefixes. For each team pick the **smallest prefix whose usable host count still covers the team**. Write `/root/answers/subnet-plan.txt` as `TEAM PREFIX`, in the order **web, db, monitoring**.

**Solution (from the site):**

```bash
awk 'BEGIN { for (p=24; p<=28; p++) print "/" p, 2^(32-p)-2 }'
```

```
/24 254
/25 126
/26 62
/27 30
/28 14
```

```bash
cat > /root/answers/subnet-plan.txt <<'DONE'
web /25
db /26
monitoring /28
DONE
```

100 machines need ≥ 100 usable hosts, so `/26` (62) is too small and `/25` (126) fits; 40 fits in `/26`; 10 fits in `/28`.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Prefix | Mask | Addresses | Usable hosts |
|--------|------|-----------|--------------|
| `/8` | `255.0.0.0` | 16,777,216 | 16,777,214 |
| `/16` | `255.255.0.0` | 65,536 | 65,534 |
| `/24` | `255.255.255.0` | 256 | 254 |
| `/25` | `255.255.255.128` | 128 | 126 |
| `/26` | `255.255.255.192` | 64 | 62 |
| `/27` | `255.255.255.224` | 32 | 30 |
| `/28` | `255.255.255.240` | 16 | 14 |
| `/30` | `255.255.255.252` | 4 | 2 |

| Rule | Remember |
|------|----------|
| network address | keep the mask-255 numbers, zero the rest (host bits all 0) |
| broadcast address | host bits all 1 |
| usable hosts | `2^(32−p) − 2`; first = network + 1, last = broadcast − 1 |
| local vs remote | same network part → deliver directly; different → send to the **gateway** |
| compute fast | `awk 'BEGIN { p=26; print 2^(32-p)-2 }'` |

**Planning rule:** pick the **smallest** prefix whose usable hosts still cover the team.

<br>

[↑ Back to top](#top)
