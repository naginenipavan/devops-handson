<a id="top"></a>

# 🥞 06 - The layers model

> **Source:** <https://www.escbash.com/topics/net-layers>
> **Contents:** 3 lessons · 3 hands-on tasks

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Why layers exist](#lesson-1) | 📖 Lesson |
| 2 | [Match tools and protocols to layers](#task-1) | 🛠️ Task |
| 3 | [One request through the stack](#lesson-2) | 📖 Lesson |
| 4 | [Order the steps of a request](#task-2) | 🛠️ Task |
| 5 | [Which tool works at which layer](#lesson-3) | 📖 Lesson |
| 6 | [Choose the tool for each symptom](#task-3) | 🛠️ Task |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: Why layers exist

Sending a web page across the world involves many separate jobs: putting bits on a wire, choosing a path across networks, delivering data to the right program, and speaking that program's language. No single piece of software does all of it, so the work is split into **layers**, each doing **exactly one job**.

**The rule that makes it work:** each layer trusts the layer below it and doesn't care how it works. The web server doesn't know whether packets travel over wifi or fibre; the routing code doesn't know it's carrying a web page. Like the post: the letter writer doesn't plan truck routes, and the driver doesn't read the letters.

### The four layers of TCP/IP

| # | Layer | Job | Where you've met it |
|---|-------|-----|---------------------|
| 4 | **Application** | the actual conversation between programs | HTTP, DNS, SSH |
| 3 | **Transport** | deliver data to the **right program** on the machine, optionally reliably | TCP, **ports** (next two topics) |
| 2 | **Internet** | move packets **between networks**, hop by hop, to the right machine | IP addresses, subnets, routing |
| 1 | **Link** | move frames between cards on **one local network** | MAC addresses, ARP |

The model wasn't invented first: ethernet, IP, TCP and applications already existed, and the four-layer model is just named after them, which is why it maps cleanly onto real tools.

> **💡 Tip:** "The site is down" can mean a failure at **any** of the four layers, each with its own tools. Debugging layer by layer, **bottom to top**, turns a vague outage into a short checklist.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Match tools and protocols to layers

Sort six things into the four layers. Create `/root/answers/layers.txt` with exactly six lines, `<item> = <layer>`, using the exact item spellings `mac`, `arp`, `ip-address`, `tcp`, `port`, `http` and layer names `link`, `internet`, `transport`, `application`.

**Solution (from the site):**

```bash
cat > /root/answers/layers.txt <<'DONE'
mac = link
arp = link
ip-address = internet
tcp = transport
port = transport
http = application
DONE
```

- **MAC and ARP** never leave the local network → **link**.
- **IP addresses** route between networks → **internet**.
- **TCP and ports** deliver data to the right program → **transport**.
- **HTTP** is a conversation between programs → **application**.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: One request through the stack

When you run `curl` against a website, it writes a few lines of HTTP text, something like `GET / please`. That's the application layer's whole contribution; **everything after that is wrapping.**

### Going down: encapsulation

```
┌─ frame ───────────────────────────────────────────────┐
│ MAC addresses  ┌─ IP packet ──────────────────────────┐│
│                │ IP addresses  ┌─ TCP segment ───────┐││
│                │               │ ports  ┌─ HTTP ───┐ │││
│                │               │        │ GET /    │ │││
│                │               │        └──────────┘ │││
│                │               └─────────────────────┘││
│                └──────────────────────────────────────┘│
└────────────────────────────────────────────────────────┘
```

| Layer | What it does with what it was handed |
|-------|--------------------------------------|
| **Transport** | puts the HTTP text in a **TCP segment**, stamped with a **port** so the far side knows which program it's for |
| **Internet** | puts the segment in an **IP packet**, stamped with source and destination **IP addresses** |
| **Link** | puts the packet in an **ethernet frame**, stamped with the **MAC** of the next machine on the local network |

Then the frame goes on the wire. This wrapping is **encapsulation**: each layer adds its own envelope, writes its own addressing on it, and hands it down. **No layer looks inside the data it wraps.**

### The far side unwraps in reverse

1. The network card receives the frame and **strips the ethernet envelope**.
2. The IP code checks the destination address and **strips the IP envelope**.
3. The TCP code reads the **port**, strips its envelope and hands the bytes to the program listening on that port.
4. The web server finally sees exactly the HTTP text `curl` wrote.

Four wraps down, four unwraps up, and neither program ever saw an envelope.

> **💡 Tip:** This picture is the debugging map. A failing request lives in exactly **one** envelope: wrong MAC (ARP problem), wrong route (IP problem), nothing listening on the port (transport problem), or a bad answer (application problem).

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Order the steps of a request

`/root/quiz/steps-shuffled.txt` lists six steps of one web request, labelled A to F, in scrambled order. Write `/root/answers/step-order.txt` with the six letters **in the order the steps really happen**, one per line.

**Solution (from the site):**

```bash
cat > /root/answers/step-order.txt <<'DONE'
A
B
C
D
E
F
DONE
```

The order is down the stack, across, and up again:

| Letter | Step |
|--------|------|
| A | `curl` writes the HTTP text |
| B | transport wraps it with a port |
| C | after IP addressing, the link layer wraps it in a frame |
| D | the server unwraps the envelopes |
| E | TCP delivers the bytes to the right program |
| F | the web server reads the request |

*(The scrambled step texts live in the lab file; only this explanation is shown on the site. The letters above are the site's answer file, not a re-ordering of the task text.)*

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Which tool works at which layer

Every diagnostic tool answers a question about **one layer**. Knowing the mapping turns the layers model into a debugging method.

### The mapping

| Tool | Layer | The question it answers |
|------|-------|-------------------------|
| `ip link` | **Link** | is the interface itself up, and what's its MAC? If the card is down, nothing above can work |
| `ip addr` | link / internet boundary | which IP addresses are assigned to each interface? |
| `ping` | **Internet** | can IP packets travel to that machine and back? Says nothing about programs or ports |
| `ss` | **Transport** | which programs listen on which ports, and which connections exist (main tool of the next topic) |
| `curl` | **Application** | speaks real HTTP to a real server and shows the actual answer: the **only** way to know the application is healthy |

### Debug from the bottom up

Each layer only works if the one below it works, so start at the bottom:

```bash
ip link show eth0
```

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
```

Interface up → link layer is fine. Next `ping` the target (internet), then check the port (transport), then `curl` the service (application).

> **💡 Tip:** **The first check that fails names the broken layer**, and every layer above it is innocent until that one is fixed. That's how experienced engineers look fast during an outage: they're not guessing, they're walking up the stack.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Choose the tool for each symptom

`/root/quiz/symptoms.txt` describes five outage symptoms, numbered 1 to 5. Each is best diagnosed with exactly one of `ip link`, `ip addr`, `ping`, `ss`, `curl` (**use each tool exactly once**). Write `/root/answers/symptom-tools.txt` as `<number> = <tool>`.

**Solution (from the site):**

```bash
cat > /root/answers/symptom-tools.txt <<'DONE'
1 = ip link
2 = ip addr
3 = ping
4 = ss
5 = curl
DONE
```

Each symptom lives one layer higher than the last:

| # | Symptom type | Tool | Layer |
|---|--------------|------|-------|
| 1 | interface down | `ip link` | link |
| 2 | no address | `ip addr` | link/internet boundary |
| 3 | unreachable machine | `ping` | internet |
| 4 | nothing listening | `ss` | transport |
| 5 | wrong page from a healthy port | `curl` | application |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Layer | Job | Addressing | Examples | Tool |
|-------|-----|------------|----------|------|
| **Application** | programs talking | — | HTTP, DNS, SSH | `curl` |
| **Transport** | right program, reliability | **port** | TCP | `ss` |
| **Internet** | between networks | **IP address** | IP, routing, subnets | `ping`, `ip addr` |
| **Link** | one local network | **MAC address** | ethernet, ARP | `ip link`, `ip neigh` |

| Concept | Remember |
|---------|----------|
| encapsulation | going down: HTTP → TCP segment (ports) → IP packet (IPs) → frame (MACs) |
| unwrapping | the server reverses it: frame → packet → segment → HTTP |
| layers trust the one below | no layer looks inside what it wraps |
| debug order | **bottom → top**: `ip link` → `ping` → `ss` → `curl`; the first failure names the broken layer |
| failure ↔ envelope | wrong MAC = ARP, wrong route = IP, nothing on the port = transport, bad answer = application |

<br>

[↑ Back to top](#top)
