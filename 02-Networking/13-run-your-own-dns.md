<a id="top"></a>

# 🖧 13 - Run your own DNS

> **Source:** <https://www.escbash.com/topics/net-run-dns>
> **Contents:** 3 lessons · 3 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [dnsmasq, a local DNS server](#lesson-1) | 📖 Lesson |
| 2 | [Start dnsmasq and query it](#task-1) | 🛠️ Task |
| 3 | [Adding your own names in dnsmasq](#lesson-2) | 📖 Lesson |
| 4 | [Add a custom name in dnsmasq](#task-2) | 🛠️ Task |
| 5 | [Making your server the system resolver](#lesson-3) | 📖 Lesson |
| 6 | [Make dnsmasq the default resolver](#task-3) | 🛠️ Task |
| 7 | [Scenario: a DNS entry points to a dead address](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: dnsmasq, a local DNS server

So far you've always asked **someone else's** DNS server. Now you run one yourself. **`dnsmasq`** is a small DNS server that does two jobs:

| Job | Behaviour |
|-----|-----------|
| names it **knows** (configured by hand) | **answers directly** |
| every other name | **forwards** the question to a bigger DNS server **upstream**, waits, and passes the answer back |

```
 dig or curl ──► dnsmasq on 127.0.0.1 ── unknown names ──► upstream resolver 1.1.1.1
              ◄──────────── answer ◄─────────────────────────────────┘
```

### How you configure it

`dnsmasq` reads config files from **`/etc/dnsmasq.d/`**. A minimal server needs only a few lines:

```
listen-address=127.0.0.1
bind-interfaces
no-resolv
server=1.1.1.1
```

| Line | Meaning |
|------|---------|
| `listen-address=127.0.0.1` | listen only on this address |
| `bind-interfaces` | bind exactly to that address |
| `no-resolv` | **ignore `/etc/resolv.conf`** when choosing an upstream |
| `server=1.1.1.1` | forward unknown names here |

> **⚠️ Warning:** The `no-resolv` line matters. Without it `dnsmasq` picks its upstream from `/etc/resolv.conf`, and later in this topic that file will point **at dnsmasq itself**, sending questions in a circle.

### Start it and ask it a question

```bash
systemctl start dnsmasq
dig @127.0.0.1 example.com +short
```

```
23.192.228.80
```

The `@127.0.0.1` sends the question to **your own server** instead of the system resolver. `dnsmasq` didn't know `example.com`, so it asked `1.1.1.1` and relayed the answer.

> **💡 Tip:** DevOps teams run servers like this everywhere: a resolver inside a Kubernetes cluster, on an office router, on a VPN box. Once you've run one yourself, those setups stop being mysterious.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Start dnsmasq and query it

Port 53 is free on this machine. Bring up `dnsmasq` as a forwarding resolver:

1. Create **`/etc/dnsmasq.d/lab.conf`** with exactly the minimal setup: listen on `127.0.0.1`, bind to that interface, ignore `/etc/resolv.conf`, forward to `1.1.1.1`.
2. Start the `dnsmasq` service and confirm it owns port 53.
3. Prove forwarding works by resolving `example.com` through your server at `127.0.0.1`.

**Solution (from the site):**

```bash
cat > /etc/dnsmasq.d/lab.conf <<'CONF'
listen-address=127.0.0.1
bind-interfaces
no-resolv
server=1.1.1.1
CONF
systemctl restart dnsmasq
# dnsmasq needs a moment to open port 53 after the restart
for i in 1 2 3 4 5; do dig @127.0.0.1 example.com +short && break; sleep 1; done
```

The config makes `dnsmasq` listen only on `127.0.0.1` and forward every unknown name to `1.1.1.1`. The `dig` answer proves the **whole chain**: your question went to your server, your server asked `1.1.1.1`, and the answer came back.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Adding your own names in dnsmasq

Right now `dnsmasq` only forwards; it has no opinions of its own. One config line changes that:

```
address=/app.internal/10.0.0.5
```

"When anyone asks for `app.internal`, **answer `10.0.0.5` yourself. Do not forward.**" `dnsmasq` is now **authoritative** for that name: the source of truth, not a messenger.

### The pattern is `address=/name/ip`

```
address=/app.internal/10.0.0.5
address=/db.internal/10.0.0.6
address=/test.corp.example.com/192.168.1.40
```

> **📝 Note:** The name also matches **every subdomain**: the line for `app.internal` also answers `api.app.internal` and `anything.app.internal` with the same IP.

### Changes need a restart

`dnsmasq` reads its config files **once, at startup**. After adding or editing an `address=` line:

```bash
systemctl restart dnsmasq
dig @127.0.0.1 app.internal +short
```

```
10.0.0.5
```

No server on the internet knows `app.internal`; the answer came entirely from your config file. This is how companies run **internal DNS**: names like `gitlab.corp` or `db.internal` resolve only inside the network because an internal server holds `address=` style records for them and forwards everything else out.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Add a custom name in dnsmasq

Your `dnsmasq` is running (restored automatically if you reset the machine). Teach it a name that exists nowhere on the internet:

1. Add a record to `/etc/dnsmasq.d/lab.conf` that answers **`app.internal`** with **`127.0.0.1`**.
2. Make `dnsmasq` pick up the change.
3. Query your server at `127.0.0.1` for `app.internal`: the answer must be `127.0.0.1`.
4. Also verify `example.com` still resolves through your server (forwarding intact).

**Solution (from the site):**

```bash
echo 'address=/app.internal/127.0.0.1' >> /etc/dnsmasq.d/lab.conf
systemctl restart dnsmasq
dig @127.0.0.1 app.internal +short
dig @127.0.0.1 example.com +short
```

The first `dig` returns `127.0.0.1` straight from your config; the second proves forwarding still works for real names.

> **📝 Note:** Plain `curl http://app.internal` **still fails** at this point: `curl` asks the **system resolver**, and that isn't your `dnsmasq` yet. The next lesson fixes that.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Making your server the system resolver

You've been steering `dig` at `dnsmasq` by hand with `@127.0.0.1`. Every other program (`curl`, `ping`, `ssh`, `apt`) still asks whatever server **`/etc/resolv.conf`** names. To make `dnsmasq` the resolver for everything, change that file:

```
nameserver 127.0.0.1
```

### The effect is system wide

```bash
dig app.internal +short       # no @ needed anymore
```

```
127.0.0.1
```

`curl http://app.internal` now works too, because `curl` asks the system resolver, the system resolver is `dnsmasq`, and `dnsmasq` knows the name. Public names keep working because `dnsmasq` forwards them to `1.1.1.1`.

### How to restore it

Before editing `/etc/resolv.conf`, **save a copy**:

```bash
cp /etc/resolv.conf /root/resolv.conf.backup
```

If DNS ever breaks after your change (for example `dnsmasq` stops), copy the backup over `/etc/resolv.conf` and the system goes back to the old resolver.

> **⚠️ Warning:** A broken resolver makes a machine feel **completely dead**, so always know your way back.

> **📝 Note:** Many Ubuntu systems run **`systemd-resolved`**, which manages this file for you. This lab machine runs it, but its `/etc/resolv.conf` is a plain file pointing at public DNS servers, so the file is yours to edit.

This exact setup, **all lookups through a local server you control**, is how VPN clients, Kubernetes nodes and container runtimes handle DNS.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Make dnsmasq the default resolver

`dnsmasq` is running with the `app.internal` record from the last task. Make it the resolver the **whole system** uses:

1. **Save a copy** of the current `/etc/resolv.conf` to `/root/resolv.conf.backup` first.
2. Change `/etc/resolv.conf` so the system resolver is `127.0.0.1`.
3. Verify `app.internal` now resolves **without** pointing `dig` at a specific server.
4. Verify `example.com` still resolves too: the change must not break normal internet lookups.

**Solution (from the site):**

```bash
cp /etc/resolv.conf /root/resolv.conf.backup
echo 'nameserver 127.0.0.1' > /etc/resolv.conf
dig app.internal +short
dig example.com +short
curl http://app.internal
```

Every tool now asks `dnsmasq` first. `dig` needs no `@`, and `curl` reaches `app.internal` because the system resolver finally knows that name. The request lands on the **local nginx**, since `app.internal` points at `127.0.0.1`.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: a DNS entry points to a dead address

**Q:** Your team moved the internal app from `10.0.0.5` to `10.0.0.8` last night. This morning `curl http://app.internal` times out for everyone. You run `dig app.internal +short` on the office `dnsmasq` server and it answers `10.0.0.5`, the old address. What do you do?

- [ ] Run `dig @127.0.0.1 app.internal` on the dnsmasq server first to confirm the stale answer really comes from this server and not from upstream
- [ ] Add `10.0.0.8 app.internal` to `/etc/hosts` on your own machine so `curl` works again for you
- [x] **Open the dnsmasq config in `/etc/dnsmasq.d/`, change the `address=` line for `app.internal` to `10.0.0.8`, and restart `dnsmasq`**
- [ ] Ask everyone to wait a few hours in case the old record is cached and expires on its own

**Why:** this **fixes the actual source of truth.** `dnsmasq` answers whatever its `address=` lines say, so a stale answer means the config still holds the old IP. Update it, restart so it rereads the file, and every client gets the right address on their next lookup.

| Option | Verdict |
|--------|---------|
| `dig @127.0.0.1 ...` first | a fine first step, but doesn't fix anything |
| wait for caches | harmless but useless: the authoritative record itself is wrong |
| `/etc/hosts` on your machine | **the trap:** fixes exactly one machine, leaves the team broken, and plants a hidden override that will contradict DNS again later |

> **💡 Tip:** When DNS returns a stale address, **fix the record where it is authoritative.**

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Item | Meaning |
|------|---------|
| `/etc/dnsmasq.d/*.conf` | `dnsmasq` config files (read **once at startup**) |
| `listen-address=127.0.0.1` / `bind-interfaces` | where it listens |
| `no-resolv` | don't take the upstream from `/etc/resolv.conf` (avoids a loop) |
| `server=1.1.1.1` | forward unknown names upstream |
| `address=/name/ip` | answer this name (and its subdomains) yourself |
| `systemctl restart dnsmasq` | needed after every config change |
| `dig @127.0.0.1 name +short` | ask your own server directly |
| `echo 'nameserver 127.0.0.1' > /etc/resolv.conf` | make it the system resolver |
| `cp /etc/resolv.conf /root/resolv.conf.backup` | **always back up first** |

| Concept | Remember |
|---------|----------|
| forwarder vs authoritative | forwards unknown names; answers `address=` names itself |
| `dig @server` vs system | `@` bypasses `resolv.conf`; `curl` uses the system resolver |
| stale internal answer | fix the `address=` line, restart; don't patch `/etc/hosts` |

<br>

[↑ Back to top](#top)
