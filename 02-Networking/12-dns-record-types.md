<a id="top"></a>

# 🗂️ 12 - DNS record types

> **Source:** <https://www.escbash.com/topics/net-dns-records>
> **Contents:** 4 lessons · 5 hands-on tasks

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [A and AAAA records](#lesson-1) | 📖 Lesson |
| 2 | [Query A and AAAA records](#task-1) | 🛠️ Task |
| 3 | [CNAME records](#lesson-2) | 📖 Lesson |
| 4 | [Follow a CNAME chain](#task-2) | 🛠️ Task |
| 5 | [MX and TXT records](#lesson-3) | 📖 Lesson |
| 6 | [Query MX and TXT records](#task-3) | 🛠️ Task |
| 7 | [TTL and caching](#lesson-4) | 📖 Lesson |
| 8 | [Watch a TTL count down](#task-4) | 🛠️ Task |
| 9 | [Query records for more domains](#task-5) | 🛠️ Task |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: A and AAAA records

DNS stores more than one kind of fact. Every entry is a **record**, and every record has a **type** saying what kind of fact it holds. The two most common types both answer "what is the address of this name?":

| Type | Maps a name to | Name comes from |
|------|----------------|-----------------|
| **A** | an **IPv4** address (four numbers) | "address record" |
| **AAAA** | an **IPv6** address (longer, colon-separated) | IPv6 is 4× as long as IPv4: four A's |

### Reading the five fields

Every answer line `dig` prints has the **same five columns, for every record type**:

```
example.com.    3600    IN    A    93.184.215.14
```

| Column | Meaning |
|--------|---------|
| 1. name | the name the record belongs to |
| 2. TTL | seconds a cache may keep this answer |
| 3. class | always `IN` (internet) |
| 4. type | the record type |
| 5. value | the data; only type and value change between record kinds |

### Picking the type in a query

`dig` asks for **A** unless you say otherwise. Put the type after the name:

```bash
dig example.com AAAA
```

```
;; ANSWER SECTION:
example.com.    3600    IN    AAAA    2606:2800:21f:cb07:6820:80da:af6b:8b2c
```

> **💡 Tip:** A name can hold **both** types at once, and most large sites do. Clients with IPv6 use the AAAA answer; IPv4-only clients use A. When a service works for some users and not others, **comparing its A and AAAA records is a real diagnostic step.**

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Query A and AAAA records

1. Query the **A** record of `example.com` with `dig`.
2. Query the **AAAA** record of the same name.
3. Record both in `/root/answers/a-aaaa.txt`: **line 1** the IPv4 address, **line 2** the IPv6 address (values only).

**Solution (from the site):**

```bash
dig example.com
dig example.com AAAA
```

The first ANSWER SECTION holds an A record (IPv4), the second an AAAA record (IPv6). Record the two values:

```bash
mkdir -p /root/answers
echo "93.184.215.14" > /root/answers/a-aaaa.txt
echo "2606:2800:21f:cb07:6820:80da:af6b:8b2c" >> /root/answers/a-aaaa.txt
```

> **📝 Note:** Use the values from **your own output**; they can differ from these. A one-liner version that passed in my lab run:
> ```bash
> dig +short example.com A | head -1 > /root/answers/a-aaaa.txt
> dig +short example.com AAAA | head -1 >> /root/answers/a-aaaa.txt
> ```

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: CNAME records

Not every name maps straight to an address. A **CNAME** (canonical name) says "**this name is an alias; the real name is over there**". The value of a CNAME is **another name**, and the lookup continues with that name until it reaches a record with an address (A or AAAA).

```
 www.shop.example ──CNAME──► shop.hosting.example ──A──► 203.0.113.9
```

### Why aliases exist

A CNAME lets many names follow **one** address without repeating it. A company hosting its shop on a platform points `www.shop.example` at the platform's name. When the platform moves its servers and updates **its own** A record, the shop's name follows automatically, with no change on the shop's side.

### Reading a chain in `dig`

When the queried name is an alias, `dig` shows the **whole chain**, one line per link:

```bash
dig www.github.com
```

```
;; ANSWER SECTION:
www.github.com.  3600  IN  CNAME  github.com.
github.com.        60  IN  A      140.82.121.4
```

Read it top to bottom: line 1 says `www.github.com` is an alias for `github.com`; line 2 resolves `github.com` to an address. Same five columns, and **only the type column tells you whether the value is a name or an address.**

> **📝 Note:** A chain **ends in an address record**. A CNAME pointing at another CNAME is allowed, but eventually some name in the chain must carry an A or AAAA, or the lookup fails.

> **💡 Tip:** CNAME chains are everyday reading: load balancers, CDNs and managed platforms all hand you a name to CNAME to. When resolution breaks, **follow the chain link by link to find where it stops.**

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Follow a CNAME chain

`www.github.com` is an alias. Follow its chain to the final address and write down every link.

1. Query `www.github.com` with `dig` and read the ANSWER SECTION top to bottom.
2. Record each link in `/root/answers/cname-chain.txt`, one line per link, in chain order, **dropping the trailing dots**:
   - **line 1:** `<queried name> CNAME <target name>`
   - **line 2:** `<final name> A <IPv4 address>`

**Solution (from the site):**

```bash
dig www.github.com
```

```
;; ANSWER SECTION:
www.github.com.  3600  IN  CNAME  github.com.
github.com.        60  IN  A      140.82.121.4
```

Two links: an alias, then the address that ends the chain.

```bash
mkdir -p /root/answers
echo "www.github.com CNAME github.com" > /root/answers/cname-chain.txt
echo "github.com A 140.82.121.4" >> /root/answers/cname-chain.txt
```

> **📝 Note:** Use the target and address from **your own output** (in my lab run the A record was `140.82.121.3`).

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: MX and TXT records

Two record types carry something other than an address.

### MX: where a domain's mail goes

When a mail server has a message for `someone@example.com`, it doesn't look up the A record. It asks for the **MX records** (mail exchange), which name the servers that accept mail for the domain:

```bash
dig gmail.com MX
```

```
;; ANSWER SECTION:
gmail.com.  3600  IN  MX  5   gmail-smtp-in.l.google.com.
gmail.com.  3600  IN  MX  10  alt1.gmail-smtp-in.l.google.com.
```

The value has **two parts: a priority number and a server name**. **Lower priority is tried first**; the others are backups. That's why a website and its mail can live on completely different machines: the A record and the MX records point wherever each service actually runs.

### TXT: free text with real jobs

A **TXT** record holds a plain text string. Two jobs dominate:

| Job | How it works |
|-----|--------------|
| **Proving ownership** | a service asks you to publish a random token as a TXT record. Only someone who controls the domain's DNS can do that, so a successful lookup **proves you own the domain** (every certificate authority and cloud provider uses this) |
| **Mail policy** | a TXT record starting **`v=spf1`** is an **SPF** rule: which servers may send mail claiming to come from the domain. Receiving servers check it to reject forged senders |

```bash
dig google.com TXT
```

```
;; ANSWER SECTION:
google.com.  3600  IN  TXT  "v=spf1 include:_spf.google.com ~all"
```

> **💡 Tip:** You'll touch **MX** when a domain's mail moves to a new provider, and **TXT** every time a domain must be verified or mail lands in spam because SPF is wrong.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Query MX and TXT records

Find where `gmail.com` receives mail and what SPF policy `google.com` publishes.

1. Query the **MX** records of `gmail.com` and pick the **lowest priority number** (the primary mail server).
2. Query the **TXT** records of `google.com` and find the record whose text begins with `v=spf1`.
3. Record in `/root/answers/mail-and-txt.txt`:
   - **line 1:** the primary MX as `priority servername` (single space, **no trailing dot**)
   - **line 2:** the full SPF text, **without the surrounding quotes**

**Solution (from the site):**

```bash
dig gmail.com MX
dig google.com TXT
```

In the MX answers, the record with the **lowest priority number** is the primary mail server:

```
gmail.com.  3600  IN  MX  5  gmail-smtp-in.l.google.com.
```

Among the TXT answers, one starts with `v=spf1`. Record both, dropping the trailing dot and the TXT quotes:

```bash
mkdir -p /root/answers
echo "5 gmail-smtp-in.l.google.com" > /root/answers/mail-and-txt.txt
echo "v=spf1 include:_spf.google.com ~all" >> /root/answers/mail-and-txt.txt
```

<br>

---

<br>

<a id="lesson-4"></a>

## 📖 Lesson 4: TTL and caching

The **TTL** (second column of every answer) is an **expiry time**. When a resolver fetches an answer it may keep a copy and hand it to anyone asking for the same name, but **only for TTL seconds**. After that the copy must be thrown away and the next query fetched fresh from the authoritative server.

### The countdown is visible

A cached answer doesn't keep its original TTL: the resolver reports **how many seconds are left**. Ask the same resolver twice and the number shrinks:

```
example.com.  300  IN  A  93.184.215.14        ← fresh from the authoritative server

example.com.  280  IN  A  93.184.215.14        ← 20 seconds later, from the resolver's cache
```

| What you see | Means |
|--------------|-------|
| TTL **falling** | the answer came from a **cache** |
| TTL **jumps back up** to a round number | the cache expired and the resolver fetched a **fresh copy** |

### Why DNS changes are slow to arrive

If a record has TTL `3600` (one hour) and you change its address on the authoritative server, resolvers that cached the old answer **keep serving it until their copy expires**, up to a full hour, each on its own schedule. Some users see the new address immediately, others the old one for a while. **There is no command to push the change out; the caches simply age.**

> **💡 Tip:** The standard playbook for a **planned DNS change**: **lower the TTL to something small a day in advance**, make the change, watch it spread quickly, then **raise the TTL again**. Low TTLs spread changes fast but cause more queries; high TTLs are cheap but slow to update. Every value is a trade between the two.

### A practical note for the next task

`dig` can send its query to a different port than 53 with **`-p`**, before the server:

```bash
dig -p 5353 @127.0.0.1 example.com
```

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Watch a TTL count down

A **caching DNS resolver** runs on this machine at `127.0.0.1` port **5353**. Use it to watch an answer age.

1. Query `example.com` through it with `dig` (using `-p`).
2. Wait several seconds and run the **exact same query** again (once or twice more if you like).
3. Read the TTL column of each ANSWER SECTION: the first is fresh, the rest come from the cache and the TTL keeps falling.
4. Record the TTLs in `/root/answers/ttl.txt`, one number per line, in the order seen. **At least two values, with the later one lower.**

**Solution (from the site):**

```bash
dig -p 5353 @127.0.0.1 example.com
```

```
example.com.  300  IN  A  93.184.215.14
```

Wait a few seconds and ask again:

```bash
dig -p 5353 @127.0.0.1 example.com
```

```
example.com.  287  IN  A  93.184.215.14
```

The second answer came from the resolver's cache, and its TTL shows the **seconds remaining**, not the original value. Record the values you saw:

```bash
mkdir -p /root/answers
printf '300\n287\n' > /root/answers/ttl.txt
```

> **📝 Note:** Use your own numbers: any pair where the later value is lower proves you watched a cache age. A scripted version that passed in my lab run:
> ```bash
> dig -p 5353 @127.0.0.1 +noall +answer example.com | awk '{print $2}' | head -1 > /root/answers/ttl.txt
> sleep 7
> dig -p 5353 @127.0.0.1 +noall +answer example.com | awk '{print $2}' | head -1 >> /root/answers/ttl.txt
> ```

<br>

---

<br>

<a id="task-5"></a>

## 🛠️ Task 5: Query records for more domains

Build one small DNS report covering **all four record types**, against real domains. Create `/root/answers/record-report.txt` with these four lines, each as **name, type, value** (single spaces, trailing dots removed):

1. the **A** record of `wikipedia.org`
2. the **CNAME** record of `www.wikipedia.org` and its target
3. **one MX** record of `github.com`, with both the priority number and the server name as the value
4. the **TXT** record of `github.com` carrying its SPF policy (the one starting `v=spf1`), **without the quotes**

**Solution (from the site):**

```bash
dig wikipedia.org
dig www.wikipedia.org
dig github.com MX
dig github.com TXT
```

From each ANSWER SECTION take the requested record and write it as name, type, value:

```bash
mkdir -p /root/answers
cat > /root/answers/record-report.txt <<'RPT'
wikipedia.org A 185.15.59.224
www.wikipedia.org CNAME dyna.wikimedia.org
github.com MX 1 aspmx.l.google.com
github.com TXT v=spf1 ip4:192.30.252.0/22 include:_spf.google.com ~all
RPT
```

> **📝 Note:** Use the values from **your own answers**. Addresses, targets and the exact SPF text change over time; **the shape of each line is what matters**. In my lab run the live values were `github.com MX 0 github-com.mail.protection.outlook.com` and a longer SPF string, and the check passed.

A scripted version that built the lines from live queries (passed in my lab run):

```bash
A=$(dig +short wikipedia.org | head -1)
C=$(dig +short www.wikipedia.org CNAME | head -1 | sed 's/\.$//')
M=$(dig +short github.com MX | sort -n | head -1 | sed 's/\.$//')
T=$(dig +short github.com TXT | grep spf1 | tr -d '"')
printf "wikipedia.org A $A\nwww.wikipedia.org CNAME $C\ngithub.com MX $M\ngithub.com TXT $T\n" > /root/answers/record-report.txt
```

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Record | Value | Used for |
|--------|-------|----------|
| **A** | IPv4 address | name → IPv4 |
| **AAAA** | IPv6 address | name → IPv6 |
| **CNAME** | another **name** | alias; chain ends at an A/AAAA |
| **MX** | `priority servername` | where a domain's mail goes (**lowest number first**) |
| **TXT** | free text | ownership proof, SPF (`v=spf1 ...`) |

| Answer line | `name  TTL  class  type  value` |
|-------------|--------------------------------|

| Command | Purpose |
|---------|---------|
| `dig NAME` | A record (default) |
| `dig NAME AAAA` / `CNAME` / `MX` / `TXT` | a specific type |
| `dig +short NAME TYPE` | value only |
| `dig +noall +answer NAME` | just the answer lines |
| `dig -p PORT @SERVER NAME` | ask a server on a non-default port |

| Concept | Remember |
|---------|----------|
| TTL | seconds a cache may keep an answer |
| falling TTL | answer is coming from a cache |
| DNS change delay | caches age out on their own; **no push** |
| planned change | lower TTL a day ahead, change, raise TTL again |
| A + AAAA | compare both when a service works for some users only |

<br>

[↑ Back to top](#top)
