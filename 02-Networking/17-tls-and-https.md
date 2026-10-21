<a id="top"></a>

# 🔒 17 - TLS and HTTPS

> **Source:** <https://www.escbash.com/topics/net-tls>
> **Contents:** 3 lessons · 4 hands-on tasks

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Why plain HTTP is unsafe](#lesson-1) | 📖 Lesson |
| 2 | [Compare HTTP and HTTPS with curl](#task-1) | 🛠️ Task |
| 3 | [Certificates and trust](#lesson-2) | 📖 Lesson |
| 4 | [Inspect a certificate with openssl](#task-2) | 🛠️ Task |
| 5 | [Inspect certificates of more sites](#task-3) | 🛠️ Task |
| 6 | [The TLS handshake](#lesson-3) | 📖 Lesson |
| 7 | [Find the TLS details in curl -v](#task-4) | 🛠️ Task |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: Why plain HTTP is unsafe

Everything you've sent with `curl` so far crossed the network as **readable text**: request line, headers, body, response. **A password sent over plain HTTP crosses the internet as plain text.**

### Your traffic doesn't go straight to the server

A request passes through many machines: your wifi router, your ISP's routers, and more routers between networks (you'll count them in the routing topic). **With plain HTTP every one of them can read the full content**, and so can anyone on the same wifi running a packet capture tool.

> Plain HTTP is a **postcard**: the mail arrives, but everyone who handles it can read it. **HTTPS is the same message in a sealed envelope.**

### What HTTPS changes

HTTPS is the same HTTP wrapped in **TLS** (Transport Layer Security). TLS does **two jobs**:

| Job | Effect |
|-----|--------|
| **Encrypts** the traffic | machines on the path see only scrambled bytes; request and response are unreadable to everyone except you and the server |
| **Proves the server's identity** | you know you're talking to the real server, not an attacker pretending to be it |

| | HTTP | HTTPS |
|---|------|-------|
| port | **80** | **443** |

```bash
curl https://example.com
```

The output looks the same, but on the wire the request and response crossed the internet **encrypted**, and `curl` checked the server's identity first.

> **💡 Tip:** A DevOps engineer treats **plain HTTP on the public internet as a bug.** Login pages, APIs, anything with user data must run over HTTPS, and you're usually the person who sets that up.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Compare HTTP and HTTPS with curl

Make the same request twice with verbose output and find where TLS begins.

1. Request `http://example.com` in verbose mode. Note `curl` connects and sends the HTTP request **straight away**.
2. Request `https://example.com` the same way. **Extra lines** appear between the connect and the HTTP request: those are TLS.
3. Create `/root/answers/http-vs-https.txt` with the `curl` verbose lines from the **https** run that show the TLS setup: the handshake lines, the line naming the TLS version and cipher, and at least one server certificate line.

**Solution (from the site):**

```bash
curl -v http://example.com -o /dev/null
curl -v https://example.com -o /dev/null
```

The http run connects and sends the request immediately. The https run shows extra lines **before any HTTP appears**:

```
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* Server certificate:
* subject: CN=*.example.com
```

Save those lines into the answers file:

```bash
curl -v https://example.com -o /dev/null 2>&1 | grep -i 'TLS\|certificate\|subject' > /root/answers/http-vs-https.txt
```

What the saved file looked like in my lab run:

```
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/certs/ca-certificates.crt
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519 / id-ecPublicKey
* Server certificate:
*  subject: CN=example.com
*  subjectAltName: host "example.com" matched cert's "example.com"
*  issuer: C=US; O=SSL Corporation; CN=Cloudflare TLS Issuing ECC CA 3
*  SSL certificate verify ok.
```

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Certificates and trust

**Encryption alone isn't enough.** An attacker between you and the server could encrypt traffic with you while pretending to be the server. You need **proof that the machine on the other end really is `example.com`**. That proof is the **certificate**.

### What a certificate contains

A small file the server sends you during the TLS setup:

| Field | Meaning |
|-------|---------|
| **subject** | the name of the site it's for (like `example.com`) |
| **public key** | used to start the encryption |
| **expiry date** | certificates are only valid for a limited time |
| **signature** | from a **certificate authority** |

### The certificate authority (CA)

Anyone can create a certificate file that says `example.com`. What makes it **trustworthy is who signed it**. A **CA** is a company whose job is to check that whoever requests a certificate for `example.com` really controls `example.com`, then **sign** it to vouch for it.

Your machine ships with a list of CAs it trusts (on the lab machine, the `ca-certificates` package). When a server presents its certificate, your machine checks the signature against that list:

| Result | Outcome |
|--------|---------|
| signed by a known CA | **trusted** |
| signed by nobody it knows | **rejected with an error** |

```
 certificate authority ──signs──► server certificate ──checked against──► trust store on your machine
```

> A certificate works like a **passport**: it names you, but what makes it believable is the **government stamp**. Border control trusts the stamp, not your word. **Your machine trusts the CA signature, not the name written in the certificate.**

> **⚠️ Warning:** A DevOps engineer deals with certificates constantly: requesting them, installing them, and fixing outages when one **expires**. **The expiry date in that file has taken down more production sites than most bugs.**

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Inspect a certificate with openssl

Pull the certificate from a real server and record its identity fields.

1. Use `openssl s_client` to connect to `example.com` on port 443 (`-connect host:port`). Feed it **empty input from `/dev/null`** so it exits instead of waiting.
2. Pipe the output into `openssl x509` with `-noout -issuer -subject -dates` to print only the fields that matter.
3. Save the result as `/root/answers/example-cert.txt`: it must contain the **issuer** line, the **subject** line and **both validity dates**.

Then read it: *which CA vouches for example.com? when does the certificate expire?*

**Solution (from the site):**

```bash
openssl s_client -connect example.com:443 </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject -dates > /root/answers/example-cert.txt
cat /root/answers/example-cert.txt
```

```
issuer=C=US, O=DigiCert Inc, CN=DigiCert Global G3 ...
subject=C=US, ST=California, ... CN=*.example.com
notBefore=Jan 15 00:00:00 2026 GMT
notAfter=Jan 15 23:59:59 2027 GMT
```

`s_client` opens a TLS connection and prints the certificate the server sent; `x509` reads it and prints only the fields you asked for; `</dev/null` closes the connection instead of leaving it open.

> **📝 Note:** The values depend on the live certificate. In my lab run they were:
> ```
> issuer=C = US, O = SSL Corporation, CN = Cloudflare TLS Issuing ECC CA 3
> subject=CN = example.com
> notBefore=Sep 26 22:49:11 2026 GMT
> notAfter=Dec 25 22:56:35 2026 GMT
> ```

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Inspect certificates of more sites

One certificate is a data point; three make a pattern.

1. Pick **three real HTTPS sites** (for example `wikipedia.org`, `github.com`, `cloudflare.com`).
2. For each, fetch the certificate and print the **issuer, subject and validity dates** the same way as the previous task.
3. Collect all three into `/root/answers/cert-report.txt` with the **site name above each block**.

Then compare: *do the three use the same CA? which expires first?* Sites using the free **Let's Encrypt** CA renew every **90 days**, so their expiry dates sit much closer than the others.

**Solution (from the site):**

```bash
for site in wikipedia.org github.com cloudflare.com; do
  echo "== $site ==" >> /root/answers/cert-report.txt
  openssl s_client -connect $site:443 </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject -dates >> /root/answers/cert-report.txt
done
cat /root/answers/cert-report.txt
```

A loop is convenient, but three separate commands work just as well. Different companies sign for different sites, and Let's Encrypt certificates expire in about 90 days while paid CAs often issue for a year.

What the report looked like in my lab run:

```
== wikipedia.org ==
issuer=C = US, O = Let's Encrypt, CN = YE2
subject=CN = *.wikipedia.org
notBefore=Oct  4 18:34:51 2026 GMT
notAfter=Jan  2 18:34:50 2027 GMT
== github.com ==
issuer=C = GB, O = Sectigo Limited, CN = Sectigo Public Server Authentication CA DV E36
subject=CN = github.com
notBefore=Sep  1 00:00:00 2026 GMT
notAfter=Nov 29 23:59:59 2026 GMT
== cloudflare.com ==
issuer=C = US, O = Google Trust Services, CN = WE1
subject=CN = cloudflare.com
notBefore=Sep  5 22:29:39 2026 GMT
notAfter=Dec  4 23:29:33 2026 GMT
```

> **📝 Note:** Three different CAs (Let's Encrypt, Sectigo, Google Trust Services), and all three certificates are valid for about **90 days**: the common pattern now.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: The TLS handshake

When `curl` opens an https connection it can't just start sending encrypted data. Both sides must **prove identity and agree on how to encrypt**. That agreement is the **handshake**, and it happens at the start of **every new TLS connection**.

### The four steps (simplified on purpose)

| # | Step | What happens |
|---|------|--------------|
| 1 | **Hello** | the client says which TLS versions and ciphers it supports; the server picks one and answers |
| 2 | **Certificate** | the server sends its certificate; the client checks the **name** and the **CA signature** against its trust store |
| 3 | **Key agreement** | both sides use math on exchanged values to arrive at the **same secret key**; anyone watching can't compute it |
| 4 | **Encrypted traffic** | from here on, **everything** (the HTTP request, the response) is encrypted with the agreed key |

### Where HTTP fits

**HTTP appears only in step four.** The `GET` line, headers, the `Host` header and the body don't cross the wire until the handshake is done. That's why **a TLS problem always shows up before any HTTP output**: if the certificate check in step two fails, `curl` stops right there and you never see a status code at all.

> **💡 Tip:** Use this order when debugging:
> - **Error before any HTTP appears** → the problem is **TLS**, usually the certificate.
> - **Error with a status code** → TLS worked and the problem is the **application**.
>
> That one distinction sends you to the right layer immediately.

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Find the TLS details in curl -v

Map the four handshake steps onto real `curl` output.

1. Make a verbose `curl` request to any real https site and discard the body (you want only the connection lines).
2. Find: the **client and server hello** lines, the line naming the **negotiated TLS version and cipher**, and the **server certificate** block with its **subject and issuer**.
3. Save those lines into `/root/answers/tls-details.txt` (TLS version, cipher, and both subject and issuer lines).

As you copy each line, say which handshake step it belongs to.

**Solution (from the site):**

```bash
curl -v https://wikipedia.org -o /dev/null 2>&1 | grep -i 'TLS\|SSL\|subject\|issuer\|expire' > /root/answers/tls-details.txt
cat /root/answers/tls-details.txt
```

```
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* subject: CN=wikipedia.org
* issuer: C=US, O=Let's Encrypt, CN=E5
```

| Lines | Handshake step |
|-------|----------------|
| `Client hello` / `Server hello` | **1. Hello** |
| `Server certificate`, `subject`, `issuer`, `expire date` | **2. Certificate** |
| `SSL connection using TLSv1.3 / <cipher>` | **3. Key agreement**: what it settled on |
| everything after that line | **4. Encrypted traffic** |

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `curl -v https://host -o /dev/null` | show the TLS handshake lines |
| `curl -v URL 2>&1 \| grep -i 'TLS\|SSL\|subject\|issuer\|expire'` | just the TLS facts |
| `openssl s_client -connect host:443 </dev/null` | open a TLS connection, show the cert chain |
| `... \| openssl x509 -noout -issuer -subject -dates` | print who issued it, who it's for, when it's valid |

| Concept | Remember |
|---------|----------|
| HTTP vs HTTPS | port 80 plaintext vs port 443 encrypted (TLS) |
| TLS does two jobs | **encrypts** the traffic + **proves the server's identity** |
| certificate | subject name, public key, expiry date, **CA signature** |
| CA | vouches for the name; your machine trusts the **CA's signature**, not the name written in the cert |
| trust store | list of CAs your machine trusts (`ca-certificates`) |
| handshake | **hello → certificate → key agreement → encrypted HTTP** |
| debugging | error **before** any HTTP = TLS/certificate problem; error **with** a status code = application problem |
| expiry | an expired certificate takes down sites; Let's Encrypt renews every ~90 days |

<br>

[↑ Back to top](#top)
