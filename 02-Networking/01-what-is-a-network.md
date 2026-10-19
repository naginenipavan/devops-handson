<a id="top"></a>

# 🌐 01 - What is a network

> **Source:** <https://www.escbash.com/topics/net-what-is-a-network>
> **Contents:** 3 lessons · 4 hands-on tasks

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [What a network is](#lesson-1) | 📖 Lesson |
| 2 | [Run a web server and fetch a page](#task-1) | 🛠️ Task |
| 3 | [Run the server on a different port](#task-2) | 🛠️ Task |
| 4 | [What a packet is](#lesson-2) | 📖 Lesson |
| 5 | [Ping localhost and read the output](#task-3) | 🛠️ Task |
| 6 | [Clients and servers](#lesson-3) | 📖 Lesson |
| 7 | [Send a request to a real website](#task-4) | 🛠️ Task |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: What a network is

A **network** is two or more machines connected so they can send data to each other. Your laptop and phone on the same wifi form one; a company's thousand servers form one; the internet is a *network of networks*.

Every useful thing a network does follows one pattern: **one machine asks, another answers.**

```
  ┌────────┐    request     ┌────────┐
  │ client │ ─────────────► │ server │
  │        │ ◄───────────── │        │
  └────────┘    response    └────────┘
```

| Role | Meaning |
|------|---------|
| **Client** | the machine/program that **asks** |
| **Server** | the machine/program that waits for requests and **answers** |

### Client and server are roles, not machines

"Server" doesn't mean special hardware. Your laptop is a client when you open a website, and becomes a server the moment a program on it starts answering requests. One machine can play **both roles at once**.

### See one conversation happen

```bash
curl http://example.com
```

```
<!doctype html>
<html>
<head>
<title>Example Domain</title>
...
```

That output is the **response**: your machine (client) asked a server for a page and it sent the page back. One request + one response = one complete network conversation.

> **💡 Tip:** A DevOps engineer lives inside this pattern all day. Website down? A client is asking and some server isn't answering. This course teaches every step between the ask and the answer so you can find the exact step that broke.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Run a web server and fetch a page

A web server never exits on its own, so if you start it normally it takes over your terminal. Put **`&`** at the end of a command to run it in the background:

```bash
sleep 300 &
```

Python ships a tiny web server that serves the current folder on a port you name:

```bash
python3 -m http.server 8000
```

**Do:**

1. Start that server on port **8000**, in the background.
2. Act as the client: fetch a page from port 8000 on this machine and save the full response to `/root/answers/page.txt`.

**Solution (from the site):**

```bash
python3 -m http.server 8000 &
sleep 1
curl http://localhost:8000 > /root/answers/page.txt
```

`&` keeps the terminal free; `curl` asks the server for a page and `>` writes the response into the file.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Run the server on a different port

A machine can run many servers at once; each needs **its own port number** so requests for one don't land on another.

1. Start the Python web server on port **9090**, in the background.
2. Fetch a page and save it to `/root/answers/page-9090.txt`.

The 8000 server can keep running; different ports don't conflict.

**Solution (from the site):**

```bash
python3 -m http.server 9090 &
sleep 1
curl http://localhost:9090 > /root/answers/page-9090.txt
```

The port is the last argument, so one machine can run many servers as long as each uses its own port.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: What a packet is

Data never crosses a network in one piece. It's cut into small pieces called **packets** (typically ~1500 bytes or less), sent one by one, and **reassembled in order** by the receiver.

> Think of mailing a book one chapter at a time: each envelope holds one chapter and carries the full delivery address. They may arrive out of order, but the numbering lets the reader rebuild the book.

### Every packet carries its own addresses

| Part | Holds |
|------|-------|
| **Header** | source address, destination address, bookkeeping such as position in the sequence |
| **Payload** | the actual slice of data |

Since every packet has the full source and destination, each can find its own way, even if packets of one message take **different paths**.

### Watch packets go out and come back

```bash
ping -c 4 127.0.0.1
```

```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.031 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.042 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.040 ms
64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.039 ms

--- 127.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3065ms
```

- `-c 4` = send 4 packets and stop.
- Each line = one packet: sequence number, size, round-trip time.
- The summary counts sent, received and **packet loss**.

> **⚠️ Warning:** Packet loss is one of the first things to check when a connection is slow or unstable. A healthy link loses nothing; even a few percent makes every application on it feel broken.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Ping localhost and read the output

`127.0.0.1` always means *this machine itself*, so pinging it tests your own network stack without touching any cable or wifi.

1. Send exactly **4** ping packets to `127.0.0.1` and save the full output to `/root/answers/ping.txt`.
2. Read the statistics summary and create `/root/answers/ping-counts.txt` in exactly this format (use the numbers your output reports):

```
sent: 4
received: 4
```

**Solution (from the site):**

```bash
ping -c 4 127.0.0.1 > /root/answers/ping.txt

cat > /root/answers/ping-counts.txt <<'DONE'
sent: 4
received: 4
DONE
```

On a healthy machine the loopback answers every packet: sent and received are both 4, loss 0%.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Clients and servers

A **client** is any program that sends a request; a **server** is any program that waits and answers. These are roles that *programs* play, not types of machines.

### Clients you already use

| Client | What it asks |
|--------|--------------|
| `curl` | sends one web request, prints the answer |
| web browser | web pages |
| `ssh` | a login session on a remote machine |
| `ping` | an address to echo test packets back |

### Servers that answer them

| Server | What it waits for |
|--------|-------------------|
| `nginx` | web requests → returns pages |
| `sshd` | login requests from `ssh` clients |
| the Python server from Task 1 | a small web server |

### One machine, both roles

The lab machine runs servers right now (nginx on port 80, sshd for logins) and acts as a client whenever you run `curl`. Both roles in one command:

```bash
curl http://localhost
```

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

The `curl` client asked port 80, the nginx server answered, and the whole conversation stayed inside one machine.

> **💡 Tip:** When something is unreachable, the first question is always **which client is asking, and which server should be answering?** Naming both ends turns a vague problem into a specific one.

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Send a request to a real website

This time the server is on the real internet. Fetch `http://example.com` and save the full response (it should contain the page's HTML and title) to `/root/answers/website.txt`.

**Solution (from the site):**

```bash
curl http://example.com > /root/answers/website.txt
```

Your machine is the client; a server run by the example.com operators, far away, received the request and sent the page back.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Concept / command | Remember |
|-------------------|----------|
| network | two or more machines that can send data to each other |
| client / server | roles: the one that asks / the one that answers; one machine can be both |
| packet | small chunk of data (≈1500 bytes max) with a header (addresses) and a payload |
| `curl URL` | act as a web client |
| `python3 -m http.server PORT` | tiny web server for the current folder |
| `cmd &` | run in the background |
| `ping -c 4 host` | send 4 test packets; read sent / received / **% loss** |
| `127.0.0.1` / `localhost` | this machine itself |
| port | which server on a machine; two servers can't share one port |

<br>

[↑ Back to top](#top)
