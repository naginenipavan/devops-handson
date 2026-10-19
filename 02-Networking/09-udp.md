<a id="top"></a>

# 📨 09 - UDP

> **Source:** <https://www.escbash.com/topics/net-udp>
> **Contents:** 2 lessons · 2 hands-on tasks

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [UDP, no connection and no promise](#lesson-1) | 📖 Lesson |
| 2 | [Send messages over UDP with nc](#task-1) | 🛠️ Task |
| 3 | [When UDP is the right choice](#lesson-2) | 📖 Lesson |
| 4 | [Confirm DNS uses UDP port 53](#task-2) | 🛠️ Task |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: UDP, no connection and no promise

TCP does a lot of work: a three-step handshake to open, an acknowledgement of every piece of data, and re-ordering on the receiving side.

**UDP does none of that.** A program hands UDP a small package called a **datagram**, UDP stamps the **source and destination port** on it, and sends it. That's the whole protocol: **no connection to open, no acknowledgement to wait for, no ordering.** If a datagram is lost, nobody notices and nobody resends it.

```
udp: just send                         tcp: open a connection first
  client ── datagram ──► server          client ── SYN ───────► server
                                         client ◄── SYN-ACK ──── server
                                         client ── ACK ───────► server
                                         (then data)
```

### What you give up, and what you get

| | TCP | UDP |
|---|-----|-----|
| delivery | complete and in order, or an error | may arrive, may arrive **twice**, may arrive **out of order**, may **never arrive**; the sender is never told |
| speed | handshake and acks add waiting | the **first byte leaves in the first packet**; no acks means no waiting between sends |

For one small question and one small answer, UDP is the cheapest possible way to ask.

### Both live on ports

UDP uses port numbers exactly like TCP. A program can listen on **UDP port 9000** while another listens on **TCP port 9000**: they never conflict, because they're separate protocols. `ss` shows UDP sockets with **`-u`** instead of `-t`.

> **⚠️ Warning:** You'll meet UDP constantly (DNS lookups, metrics agents, video streams). When you debug them, remember: **no answer does not mean refused.** UDP has no refusal handshake, so a lost or blocked datagram just stays **silent**.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Send messages over UDP with nc

Prove UDP delivers data with no connection by sending a message between two `nc` processes on this machine.

1. Start an `nc` listener in **UDP mode** on port **9000**, in the background, with everything it receives redirected into `/root/udp-received.txt`.
2. With a second `nc` in UDP mode, send a line to port 9000 on `127.0.0.1`.
3. The UDP client doesn't exit by itself (there's no connection to close). Stop it with `Ctrl+C`, or use **`-w1`**, which makes `nc` stop waiting one second after the send.
4. Confirm `/root/udp-received.txt` contains the message.

> **📝 Note:** The task text says "the exact line `hello`", but the check looks for the line **`hello over udp`**. Send that.

**Solution (from the site):**

Start a UDP listener in the background, writing everything it receives into the file:

```bash
nc -u -l 9000 > /root/udp-received.txt &
```

`-u` switches `nc` to UDP, `-l` makes it listen on port 9000, and `&` puts it in the background.

Send a message to it with a second `nc`, also in UDP mode:

```bash
echo "hello over udp" | nc -u -w1 127.0.0.1 9000
```

UDP has no connection to close, so without help the client never exits. **`-w1`** tells `nc` to give up waiting after one second. Then confirm:

```bash
cat /root/udp-received.txt
```

The datagram left the client, arrived at the listener and was written to the file. **No handshake happened at any point.**

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: When UDP is the right choice

UDP is **not a worse TCP**. It's the right tool whenever **the cost of TCP's promises is higher than the cost of a lost packet.**

### Case one: DNS, where retry is cheap

A DNS lookup is one tiny question and one tiny answer, usually well under a hundred bytes each. A TCP handshake first would take three packets before the question even leaves; over UDP **the question is the first and only packet**.

If the datagram is lost, the client simply waits a moment and asks again, which costs almost nothing. So **DNS uses UDP by default**, and the retry logic lives in the client, not the protocol.

### Case two: live video, where old data is useless

Suppose frame 412 of a live video call is lost. TCP would stop everything, resend frame 412 and hold back all later frames, so by the time it arrives the call has moved on: the video freezes, then jumps. Over UDP the lost frame is **skipped**: a glitch for a fraction of a second, then it continues. **Fresh and slightly broken wins over complete and old.**

### The rule of thumb

Ask two questions about the traffic:

1. Is each message **small enough that resending is cheap**?
2. Does the data **go stale so fast** that resending is pointless?

A yes to either makes UDP a good fit. If the data must arrive **complete and in order** (a file download, a web page), use **TCP**.

> **💡 Tip:** You'll meet UDP in **DNS**, metrics/logging agents like **statsd**, **NTP** time sync, and streaming media. Knowing why they chose UDP tells you how they fail: **silently, with missing data instead of error messages.**

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Confirm DNS uses UDP port 53

Verify on this machine that DNS runs over UDP on port 53.

1. Find which DNS server the machine uses: the **`nameserver`** line in `/etc/resolv.conf`.
2. Catch a lookup in the act: start a name lookup in the background (`getent hosts example.com`) and **at the same moment** list UDP sockets with `ss`. The socket's remote address shows where the query went and to which port. A lookup finishes fast, so you may need a few tries.
3. Record the result in `/root/answers/dns-transport.txt`:
   - **line 1:** the resolver address from `/etc/resolv.conf`
   - **line 2:** the port the DNS query was sent to

**Solution (from the site):**

The machine's DNS server is listed in `/etc/resolv.conf`:

```bash
cat /etc/resolv.conf
```

In the lab it contains (comments removed):

```
nameserver 1.1.1.1
nameserver 8.8.8.8
nameserver 9.9.9.9
options timeout:2 attempts:3 rotate edns0
```

Run a lookup in the background and catch the socket with `ss` in the same moment:

```bash
for i in {1..20}; do getent hosts example.com > /dev/null & done; ss -unp
```

`-u` shows UDP sockets. While the lookup is in flight, a socket appears with the resolver's address and **`:53`** as the remote end. Run it a few times if the lookup finishes too fast to catch.

Record what you saw:

```bash
then fill the answers file
```

> **📝 Note:** `awk '/^nameserver/ {print $2; exit}'` prints the address of the **first** `nameserver` line only (here `1.1.1.1`).

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Concept | TCP | UDP |
|---------|-----|-----|
| connection | handshake first | none |
| delivery | reliable, ordered | best effort: may be lost, duplicated, reordered |
| cost | handshake + acks | cheapest possible |
| failure looks like | error / slowness | **silence** (no refusal handshake) |
| typical use | web, ssh, databases, downloads | DNS, NTP, statsd, live audio/video |

| Command | Purpose |
|---------|---------|
| `nc -u -l PORT` | listen in UDP mode |
| `echo msg \| nc -u -w1 HOST PORT` | send one datagram and exit after 1 s |
| `ss -u` / `ss -unp` | UDP sockets (with owning process) |
| `cat /etc/resolv.conf` | which DNS servers this machine uses |
| `getent hosts name` | do a name lookup |

**Rule of thumb:** use UDP when messages are small and retry is cheap, or when old data is useless. Use TCP when data must arrive complete and in order. UDP and TCP can share a port number without conflict.

<br>

[↑ Back to top](#top)
