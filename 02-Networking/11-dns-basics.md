<a id="top"></a>

# 📖 11 - DNS basics

> **Source:** <https://www.escbash.com/topics/net-dns-basics>
> **Contents:** 4 lessons · 5 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Names to addresses](#lesson-1) | 📖 Lesson |
| 2 | [Resolve names with dig](#task-1) | 🛠️ Task |
| 3 | [Practice reading dig output](#task-2) | 🛠️ Task |
| 4 | [/etc/hosts comes first](#lesson-2) | 📖 Lesson |
| 5 | [Add an entry to /etc/hosts](#task-3) | 🛠️ Task |
| 6 | [resolv.conf, which server answers you](#lesson-3) | 📖 Lesson |
| 7 | [Query a different DNS server](#task-4) | 🛠️ Task |
| 8 | [The full chain, root to authoritative](#lesson-4) | 📖 Lesson |
| 9 | [Trace a lookup with dig +trace](#task-5) | 🛠️ Task |
| 10 | [Scenario: a name resolves to the wrong address](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: Names to addresses

Every packet you've sent went to an **IP address**; machines only understand numbers. But nobody types `93.184.215.14`, people type `example.com`. Something translates between the two on **every single request**: **DNS**, the **Domain Name System**.

DNS is a worldwide lookup service, like a phone book split across thousands of servers. You hand it a name, it hands back the address registered for that name.

```
   name  example.com  ──►  dns servers  ──►  address 93.184.215.14
```

### Every connection starts with a lookup

When you ran `curl example.com` in topic 1, DNS had already done its job before the first TCP packet left: `curl` asked DNS for the address, got a number back, and only then opened the connection to that number. **The name never travels in the TCP or IP headers. Only the address does.**

DNS is therefore a dependency of almost everything:

- DNS slow → every connection starts slow.
- DNS gives a wrong answer → `curl` connects to the **wrong machine**, and no amount of network debugging below it will explain why.

### The tool for asking directly: `dig`

```bash
dig example.com
```

`dig` sends the same kind of query `curl` would but shows the **full answer**: which address came back, how long it may be cached, and which server answered.

> **💡 Tip:** A large share of "the network is broken" reports are really DNS problems. Check name resolution early in almost every debug session.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Resolve names with dig

Look up the addresses behind two real names.

1. Query `example.com` with `dig` and find the IPv4 address in the **ANSWER SECTION**.
2. Do the same for `github.com`.
3. Record both in `/root/answers/dig-addresses.txt`, one per line: `example.com <address>` then `github.com <address>`.

Use the addresses `dig` actually gave you: large sites return different addresses at different times, and any address from your ANSWER SECTION is valid.

**Solution (from the site):**

```bash
dig example.com
dig github.com
```

In each output the ANSWER SECTION holds the result:

```
;; ANSWER SECTION:
example.com.    3600    IN    A    93.184.215.14
```

The last column is the IPv4 address. Record both:

```bash
mkdir -p /root/answers
echo "example.com 93.184.215.14" > /root/answers/dig-addresses.txt
echo "github.com 140.82.121.4" >> /root/answers/dig-addresses.txt
```

> **📝 Note:** The addresses above are the site's example. **Use whatever your own `dig` shows.** A one-line way to pull the address: `dig +short example.com | head -1`. (In my lab run it returned `104.20.23.154` for example.com and `140.82.121.3` for github.com, and the check passed.)

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Practice reading dig output

A saved `dig` output sits at `/root/dns/dig-output.txt`. Pull out four facts, each in a different section, and write them to `/root/answers/dig-reading.txt`, one per line:

1. the **name that was queried** (without the trailing dot)
2. the **TTL** of the answer (seconds it may be cached)
3. the **IPv4 address** it resolved to
4. the **address of the DNS server that answered** (address only, no port)

In an ANSWER SECTION line the columns are always **name, TTL, class, type, value**. The server that answered is reported near the bottom of every `dig` output.

**Input file** `/root/dns/dig-output.txt` ([raw copy](lab-files/11-dns-basics/dig-output.txt)):

```
; <<>> DiG 9.18.28 <<>> shop.internal.example
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41785
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;shop.internal.example.		IN	A

;; ANSWER SECTION:
shop.internal.example.	300	IN	A	203.0.113.77

;; Query time: 2 msec
;; SERVER: 192.0.2.53#53(192.0.2.53) (UDP)
;; WHEN: Mon Sep 14 09:12:44 UTC 2026
;; MSG SIZE  rcvd: 66
```

**Solution (from the site):**

```bash
cat /root/dns/dig-output.txt
printf 'shop.internal.example\n300\n203.0.113.77\n192.0.2.53\n' > /root/answers/dig-reading.txt
```

Where each fact hides:

| Fact | Where | Value |
|------|-------|-------|
| name asked | **QUESTION SECTION** | `shop.internal.example` |
| TTL | **ANSWER SECTION**, column 2 | `300` |
| address | **ANSWER SECTION**, last column | `203.0.113.77` |
| who answered | **SERVER** line at the bottom | `192.0.2.53` (port 53, over UDP) |

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: /etc/hosts comes first

When a program on Linux resolves a name, it **doesn't go straight to DNS**. It first checks a plain text file on the machine: **`/etc/hosts`**. Each line maps an address to one or more names:

```bash
cat /etc/hosts
```

```
127.0.0.1 localhost
```

```
 program asks for a name
          │
          ▼
    /etc/hosts file ── found ──► use that address (no DNS query is ever sent)
          │
      not found
          ▼
    ask the DNS server
```

That's why `localhost` always works with no DNS involved: it's a hosts-file entry, present on every Linux system.

### A local override for any name

Because the file wins over DNS, **one line redirects any name on this machine only**:

- Point a production hostname at a **test server** to try a change safely.
- Give a name to a machine that has **no DNS entry** at all.

The rest of the world isn't affected, only lookups made on this machine.

### The tool that shows the real answer

`dig` speaks **only to DNS servers and never reads `/etc/hosts`**. To see what programs actually get, use **`getent`**, which follows the full resolution order, file first:

```bash
getent hosts localhost        # → 127.0.0.1 localhost
```

> **⚠️ Warning:** When `dig` shows one address but an application connects to another, a hosts-file entry is the usual suspect. It's also a classic leftover: a test override forgotten in the file keeps hijacking a name months later.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Add an entry to /etc/hosts

Give this machine's nginx a name of its own, without touching any DNS server.

1. Add an entry to `/etc/hosts` that points **`myapp.test`** at `127.0.0.1`, in the format of the lines already there; don't remove any.
2. Confirm it resolves locally with the lookup tool that reads the hosts file.
3. Request `http://myapp.test/` with `curl`. The local nginx welcome page must come back.

**Solution (from the site):**

```bash
echo "127.0.0.1 myapp.test" >> /etc/hosts
getent hosts myapp.test          # → 127.0.0.1 myapp.test
curl http://myapp.test/
```

nginx answers with its welcome page. **No DNS server anywhere has heard of `myapp.test`**, yet the name works on this machine because the hosts file answered before DNS was asked.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: resolv.conf, which server answers you

When a name isn't in `/etc/hosts`, the lookup goes to a DNS server. Which one? The machine reads that from one file:

```bash
cat /etc/resolv.conf
```

```
nameserver 10.0.2.3
```

Each **`nameserver`** line is the address of a DNS server. Every program on the machine, and `dig` when you give it no server, sends its queries there. (You met this file in the UDP topic.)

### The server you use is a *resolver*

The address in `resolv.conf` usually isn't the server that **owns** the answers. It's a **resolver**: a server that takes your question, finds the answer somewhere in the world, and returns it. Your machine asks one server one question and gets one answer; **all the chasing happens on the resolver's side**. Home routers, company networks and public services like `1.1.1.1` or `8.8.8.8` all offer resolvers.

### Asking a different server with `dig`

```bash
dig @1.1.1.1 example.com
```

The `@` form **ignores `resolv.conf`** and asks the public resolver at `1.1.1.1` directly. The **SERVER** line at the bottom confirms who answered.

### Why ask two servers the same question?

Their answers can differ:

- A **company resolver** may hold internal names public ones have never heard of.
- A **misbehaving resolver** may return stale or wrong answers.

> **💡 Tip:** Compare your configured resolver against a known public one when resolution looks wrong. **Same answer = the name itself is the problem. Different answers = your resolver is.**

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Query a different DNS server

Run the standard resolver comparison.

1. Find the resolver this machine uses in `/etc/resolv.conf`.
2. Query `example.com` twice with `dig`: once normally, once directed at **`1.1.1.1`**.
3. Check the **SERVER** line of each output (a different server answers each time) and compare the ANSWER sections.
4. Record in `/root/answers/resolvers.txt`:
   - **line 1:** the configured resolver's address from `/etc/resolv.conf`
   - **line 2:** the IPv4 address that `1.1.1.1` returned for `example.com`

**Solution (from the site):**

```bash
cat /etc/resolv.conf
dig example.com
dig @1.1.1.1 example.com
```

For a public name like `example.com` the two ANSWER sections agree; the SERVER lines name a different server each time.

```bash
mkdir -p /root/answers
awk '/^nameserver/ {print $2; exit}' /etc/resolv.conf > /root/answers/resolvers.txt
echo "93.184.215.14" >> /root/answers/resolvers.txt
```

> **📝 Note:** Use the address `1.1.1.1` **actually returned to you**; it can differ from the site's example. A one-liner: `dig @1.1.1.1 +short example.com | head -1 >> /root/answers/resolvers.txt` (it passed in my lab run).

<br>

---

<br>

<a id="lesson-4"></a>

## 📖 Lesson 4: The full chain, root to authoritative

No single server knows every name on the internet. DNS is split by the **parts of the name, read right to left**. For `www.example.com`: `.com` is managed at one level, `example.com` at the next, and the full name at the last. A resolver walks that chain, **one question per level**.

### The three levels

| Level | Knows | Answers with |
|-------|-------|--------------|
| **Root servers** (top) | no addresses, but for any name, **which servers handle its ending** | "ask these `.com` servers" |
| **Top-Level-Domain (TLD) servers** (`.com`, `.org`, ...) | no addresses either, but which servers **hold the records for `example.com`** | "ask these servers for `example.com`" |
| **Authoritative servers** | the **actual records** for a domain (the server the domain's owner controls, where DNS changes are really made) | the **final answer: the address** |

```
                resolving www.example.com
 resolver ── "who handles .com?" ───────────► root server
 resolver ── "who holds example.com?" ──────► .com TLD server
 resolver ── "address of www.example.com?" ─► authoritative server
 resolver ◄── the address
```

### The resolver does the legwork

Your machine **never talks to root, TLD or authoritative servers**. It asks its resolver one question; the resolver walks root → TLD → authoritative and hands the final answer back. The resolver also **caches every step**, so the next lookup for anything under `.com` skips the root entirely.

> **💡 Tip:** This chain explains where DNS changes happen and where they can go stale: you change a record on the **authoritative** server, but the world reads it through **resolvers and their caches**.

<br>

---

<br>

<a id="task-5"></a>

## 🛠️ Task 5: Trace a lookup with dig +trace

`dig +trace` skips the resolver and **walks the chain itself**, printing every step.

1. Trace `example.com` with `dig +trace`.
2. Read it block by block; each block ends with a `Received ... from <server>` line. Match each step to the three server types.
3. Record in `/root/answers/trace.txt`:
   - **line 1:** type that answered the first step
   - **line 2:** type that answered the second step
   - **line 3:** type that gave the final answer (use exactly `root`, `tld`, `authoritative`)
   - **line 4:** the IPv4 address from the final answer

**Solution (from the site):**

```bash
dig +trace example.com
```

| Block | Came from | Lists |
|-------|-----------|-------|
| 1 | the root hints | the **root** servers |
| 2 | a **root** server | the `.com` **TLD** servers |
| 3 | a **TLD** server | the **authoritative** servers for `example.com` |
| 4 (final) | an **authoritative** server | the **A record**, e.g. `example.com. 3600 IN A 93.184.215.14` |

```bash
mkdir -p /root/answers
printf 'root\ntld\nauthoritative\n93.184.215.14\n' > /root/answers/trace.txt
```

> **📝 Note:** Use the address **your own trace ended with** (it can differ from the site's example). In my lab run I pulled it from the trace output with `dig +trace example.com | grep -E '^example.com.*IN.*A.*[0-9]$' | head -1 | awk '{print $NF}'` and the check passed.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: a name resolves to the wrong address

**Q:** A developer reports that `api.staging.example.com` behaves strangely from their workstation: `curl` reaches a server, but it serves an **old version** of the API. From every other machine the same name serves the current version. The DNS record was updated last month and its TTL has long expired. What do you check first on the workstation?

- [ ] `dig api.staging.example.com` on the workstation and compare it with `dig` on a healthy machine
- [x] **`getent hosts api.staging.example.com` on the workstation, and look in `/etc/hosts`**
- [ ] Flush every cache on the workstation and restart its network service, then test again
- [ ] Ask a public resolver directly with `dig @1.1.1.1 api.staging.example.com` to confirm the record is correct

**Why:** it checks **the answer the workstation's programs actually use, hosts file included.** One machine resolving differently from all others, long after caches expired, is the **signature of a local override**: a forgotten `/etc/hosts` line pins the name to the old server forever, and `getent` shows it in one command.

| Option | Verdict |
|--------|---------|
| `dig` comparisons / `@1.1.1.1` | `dig` **bypasses the hosts file** and only shows what DNS servers say; it can't see the local override |
| flush everything | changes state without inspecting it, so it ranks last |

> **💡 Tip:** Comparing `getent` (what programs get) with `dig` (what DNS says) is exactly how this class of bug is caught.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command / file | Purpose |
|----------------|---------|
| `dig NAME` | query DNS via the configured resolver (full output) |
| `dig +short NAME` | just the answer |
| `dig @SERVER NAME` | ask a specific server (ignores `resolv.conf`) |
| `dig +trace NAME` | walk root → TLD → authoritative yourself |
| `getent hosts NAME` | what **programs** get (reads `/etc/hosts` first, then DNS) |
| `/etc/hosts` | local overrides: `ADDRESS NAME`; wins over DNS |
| `/etc/resolv.conf` | `nameserver ADDRESS` lines = your resolver(s) |

| Concept | Remember |
|---------|----------|
| resolution order | `/etc/hosts` → DNS resolver |
| dig vs getent | `dig` never reads `/etc/hosts`; `getent` follows the full order |
| ANSWER SECTION | `name  TTL  class  type  value` |
| resolver vs authoritative | resolver chases and caches; authoritative holds the records |
| the chain | root → TLD (`.com`) → authoritative |

**Debug rules:** wrong address on **one machine only** → `getent hosts` and look in `/etc/hosts`. Resolution looks wrong → ask your resolver **and** `@1.1.1.1`: same answer = the name's the problem, different = your resolver is.

<br>

[↑ Back to top](#top)
