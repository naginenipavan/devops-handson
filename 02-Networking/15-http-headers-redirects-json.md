<a id="top"></a>

# 🧾 15 - HTTP headers, redirects, JSON

> **Source:** <https://www.escbash.com/topics/net-http-headers>
> **Contents:** 4 lessons · 5 hands-on tasks

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [Important headers](#lesson-1) | 📖 Lesson |
| 2 | [Send custom headers with curl](#task-1) | 🛠️ Task |
| 3 | [Redirects](#lesson-2) | 📖 Lesson |
| 4 | [Follow a redirect with curl](#task-2) | 🛠️ Task |
| 5 | [Cookies](#lesson-3) | 📖 Lesson |
| 6 | [Use cookies with curl](#task-3) | 🛠️ Task |
| 7 | [APIs and JSON](#lesson-4) | 📖 Lesson |
| 8 | [Fetch JSON and parse it with jq](#task-4) | 🛠️ Task |
| 9 | [Practice jq on API responses](#task-5) | 🛠️ Task |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: Important headers

Headers carry everything about a request that **isn't the path or the body**. Three do most of the work:

| Header | What it does |
|--------|--------------|
| **`Host`** | names the **site** you want. One server, one IP can host many websites; `Host` tells the server which one you mean. `curl` fills it from the URL; when two names point at the same IP, **`Host` is the only difference** between the two requests |
| **`Content-Type`** | describes the **body's format**: `text/html` for pages, `application/json` for API data, `application/x-www-form-urlencoded` for form posts. Servers reject bodies whose declared type they don't expect, a **very common API error** |
| **`User-Agent`** | identifies the client software (`curl/8.5.0`, or a long browser string). Used for statistics and sometimes to serve different content to different clients |

### Set any header with `-H`

`-H` adds or replaces one header, written exactly as it appears on the wire. Repeat it for several:

```bash
curl -H 'Host: app-a.lab' http://127.0.0.1:8080/
```

```
welcome to app a
```

`curl` connects to `127.0.0.1` but **claims to want the site named `app-a.lab`**, and the server believed the header.

> **💡 Tip:** Faking a `Host` header is a **daily DevOps trick**: you can **test a site on a new server before its DNS points there**.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Send custom headers with curl

nginx serves two different sites on `127.0.0.1` port **8080**: one answers to `app-a.lab`, the other to `app-b.lab`. Neither name exists in DNS. Reach both anyway:

1. Request `app-a.lab` → `/root/headers/app-a.txt`.
2. Request `app-b.lab` → `/root/headers/app-b.txt`.

Both requests go to `http://127.0.0.1:8080/`; **the header you send selects the site.**

**Solution (from the site):**

```bash
mkdir -p /root/headers
curl -s -H 'Host: app-a.lab' http://127.0.0.1:8080/ > /root/headers/app-a.txt
curl -s -H 'Host: app-b.lab' http://127.0.0.1:8080/ > /root/headers/app-b.txt
```

nginx holds two sites on port 8080 and picks one **by the `Host` header alone**. Everything else about the two requests is identical.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Redirects

A **redirect** is a response that says "**not here, go there**". The status is in the **3xx** family and a **`Location`** header carries the new URL.

```
 client ── GET /promo ──────────────► old URL
 client ◄── 301 + Location: /spring-sale
 client ── GET /spring-sale ────────► new URL
 client ◄── 200 + page
```

| Code | Meaning |
|------|---------|
| **301 Moved Permanently** | the resource lives at the new URL **forever**; browsers and search engines remember it and stop visiting the old one |
| **302 Found** | a **temporary** move; keep using the old URL next time (login flows, maintenance pages) |

> **📝 Note:** Rule of thumb: renamed a page or moved a domain → **301**. Sending users somewhere briefly → **302**.

### curl stops, `-L` follows

By default `curl` does **one** request and shows you the redirect itself, which is exactly what you want when debugging:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/promo
```

```
301
```

Add **`-L`** and `curl` reads the `Location` header, requests the new URL and repeats until it gets a non-redirect answer, so you see the final page instead of the hop.

> **⚠️ Warning:** Redirect chains cause real outages: a wrong `Location` can bounce clients in a **loop**, and every hop adds a round trip. See each hop with plain `curl`, then the final result with `-L`, and you can take a chain apart in seconds.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Follow a redirect with curl

On `127.0.0.1` port 8080, `/promo` has permanently moved to `/spring-sale`. Show both halves:

1. Request `http://127.0.0.1:8080/promo` so the **status and headers are visible**, and save it to `/root/headers/redirect-stop.txt` (must show the 301 and the `Location` header).
2. Request the same URL **following the redirect to the end** and save the final page to `/root/headers/redirect-followed.txt`.

**Solution (from the site):**

```bash
mkdir -p /root/headers
curl -s -i http://127.0.0.1:8080/promo > /root/headers/redirect-stop.txt
curl -s -L http://127.0.0.1:8080/promo > /root/headers/redirect-followed.txt
```

`-i` includes the **response headers** in the output, so the first file shows `HTTP/1.1 301` and the `Location` header naming `/spring-sale`. With `-L`, `curl` requests `/promo`, reads `Location`, requests `/spring-sale`, and saves the final landing page.

What the two files looked like in my lab run:

```
HTTP/1.1 301 Moved Permanently
Server: nginx/1.24.0 (Ubuntu)
Date: Fri, 16 Oct 2026 21:22:41 GMT
Content-Type: text/html
Content-Length: 178
Connection: keep-alive
Location: http://127.0.0.1:8080/spring-sale

<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.24.0 (Ubuntu)</center>
</body>
</html>
```

```
spring sale landing page
```

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Cookies

HTTP is **stateless**: the server forgets you the moment it sends a response, so your second request looks exactly like a stranger's first. **Cookies** are the fix, and they're just **two headers**.

### `Set-Cookie` goes down, `Cookie` comes back up

```
Set-Cookie: session=abc123      ← server → client (in the response)
Cookie: session=abc123          ← client → server (on every later request)
```

The server reads `Cookie`, looks up `abc123` in its records, and knows it's you again. Log in once, and the session cookie keeps you logged in on every page after.

### curl must be told to behave like a browser

Browsers store cookies automatically; `curl` does not, unless you ask:

| Flag | Meaning |
|------|---------|
| `-c <file>` | **save** cookies from the response into a file (a **cookie jar**) |
| `-b <file>` | **read** the jar and send its cookies with the request |

```bash
curl -c /root/headers/jar.txt http://127.0.0.1:9000/login
curl -b /root/headers/jar.txt http://127.0.0.1:9000/profile
```

```
logged in, cookie set
welcome back, session abc123
```

> **💡 Tip:** "The site keeps logging me out": this header pair is where you look. **Is `Set-Cookie` arriving, and is `Cookie` going back?** `curl` with a jar reproduces the whole flow from your terminal.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Use cookies with curl

A login service runs on `127.0.0.1:9000`. `/login` sets a session cookie; `/profile` answers differently depending on whether you send that cookie back.

1. Request `/login` and store the cookie in a jar at `/root/headers/jar.txt`.
2. Request `/profile` **sending the jar's cookie**, and save the response to `/root/headers/profile.txt`. It must show the server recognised your session (if it says no cookie was sent, the second request is missing something).

**Solution (from the site):**

```bash
mkdir -p /root/headers
curl -s -c /root/headers/jar.txt http://127.0.0.1:9000/login
curl -s -b /root/headers/jar.txt http://127.0.0.1:9000/profile > /root/headers/profile.txt
cat /root/headers/profile.txt
```

`-c` saved the `Set-Cookie` value `session=abc123` into the jar; `-b` attached it to the `/profile` request as a `Cookie` header, and the server's welcome-back reply proves the round trip worked.

<br>

---

<br>

<a id="lesson-4"></a>

## 📖 Lesson 4: APIs and JSON

Most traffic between systems is **APIs**: one program asking another for data, almost always as **JSON** with `Content-Type: application/json`. JSON is text: curly braces around name/value pairs.

```json
{"service": "payments-api", "version": "2.4.1",
 "owner": {"team": "platform"},
 "servers": [{"name": "pay-1", "status": "up"}]}
```

Values can be **text, numbers, another object** in braces, or a **list** in square brackets.

### `jq` slices JSON

`jq` takes JSON on stdin and a **filter** naming what you want:

| Filter | Picks |
|--------|-------|
| `.version` | a top-level field |
| `.owner.team` | walks into a nested object |
| `.servers[].name` | a field from **every item of a list** |
| `-r` flag | print **raw text** without the quotes |

### curl and jq together

```bash
curl -s http://127.0.0.1:9001/service.json | jq -r '.version'
```

```
2.4.1
```

`-s` keeps `curl` quiet so only the JSON reaches `jq`, which prints exactly the one value you asked for.

> **💡 Tip:** This pipeline is **everywhere in DevOps**: cloud tools, Kubernetes and monitoring systems all answer in JSON, and `curl | jq` is how you ask them questions from a shell and use the answer in a script.

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Fetch JSON and parse it with jq

A JSON endpoint at `http://127.0.0.1:9001/service.json` describes a service. Look at the whole response first, then extract the top-level **`service`** field into `/root/headers/service.txt` as **bare text**: no quotes, no braces (there's a `jq` flag for exactly that).

**Input** (the endpoint's response, [raw copy](lab-files/15-http-headers/service.json)):

```json
{
  "service": "payments-api",
  "version": "2.4.1",
  "owner": {
    "team": "platform",
    "email": "platform@corp.example"
  },
  "servers": [
    {"name": "pay-1", "status": "up"},
    {"name": "pay-2", "status": "down"},
    {"name": "pay-3", "status": "up"}
  ]
}
```

**Solution (from the site):**

```bash
mkdir -p /root/headers
curl -s http://127.0.0.1:9001/service.json | jq -r '.service' > /root/headers/service.txt
cat /root/headers/service.txt
```

`curl` fetches the JSON, `jq`'s `.service` filter picks the field, and `-r` drops the quotes so the file holds plain text (`payments-api`).

<br>

---

<br>

<a id="task-5"></a>

## 🛠️ Task 5: Practice jq on API responses

The same endpoint is still at `http://127.0.0.1:9001/service.json`. Answer three questions with `curl` and `jq`, each as **bare unquoted text in its own file**:

| File | Answer |
|------|--------|
| `/root/headers/version.txt` | the top-level `version` |
| `/root/headers/owner-email.txt` | the `email` inside the `owner` object |
| `/root/headers/down-server.txt` | the `name` of the one server whose `status` is `down` |

The third needs a filter that **selects list items by a condition** (jq's `select`).

**Solution (from the site):**

```bash
curl -s http://127.0.0.1:9001/service.json | jq -r '.version' > /root/headers/version.txt
curl -s http://127.0.0.1:9001/service.json | jq -r '.owner.email' > /root/headers/owner-email.txt
curl -s http://127.0.0.1:9001/service.json | jq -r '.servers[] | select(.status == "down") | .name' > /root/headers/down-server.txt
```

| Filter | Explanation | Result |
|--------|-------------|--------|
| `.version` | a plain field | `2.4.1` |
| `.owner.email` | a second dot walks into the nested object | `platform@corp.example` |
| `.servers[] \| select(.status == "down") \| .name` | iterate the list, keep entries whose status is `down`, print their names | `pay-2` |

> **💡 Tip:** **Filtering a list by a condition** like this is the single most useful `jq` pattern in real operations work.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `curl -H 'Host: name' URL` | pick a site on a shared IP / test before DNS points there |
| `curl -H 'Header: value' URL` | set any header (repeat `-H` for several) |
| `curl -i URL` | include response headers in the output |
| `curl -L URL` | follow redirects (`Location`) |
| `curl -s -o /dev/null -w '%{http_code}\n' URL` | status code only (see the hop, don't follow) |
| `curl -c jar URL` / `curl -b jar URL` | save / send cookies |
| `curl -s URL \| jq -r '.field'` | extract a JSON field as raw text |
| `jq -r '.a.b'` | nested field |
| `jq -r '.list[].name'` | field from every list item |
| `jq -r '.list[] \| select(.status == "down") \| .name'` | filter a list by a condition |

| Concept | Remember |
|---------|----------|
| `Host` | selects the site on a shared IP |
| `Content-Type` | declares the body format (wrong type = common API error) |
| 301 vs 302 | permanent vs temporary |
| `Location` | where a 3xx points |
| cookies | `Set-Cookie` down, `Cookie` back up; stateless HTTP needs them |
| `-c` / `-b` | write / read the cookie jar |

<br>

[↑ Back to top](#top)
