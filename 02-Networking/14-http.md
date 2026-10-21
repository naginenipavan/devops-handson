<a id="top"></a>

# 🌍 14 - HTTP

> **Source:** <https://www.escbash.com/topics/net-http>
> **Contents:** 4 lessons · 5 hands-on tasks

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [The request and the response](#lesson-1) | 📖 Lesson |
| 2 | [Send a raw HTTP request with nc](#task-1) | 🛠️ Task |
| 3 | [Methods, GET and POST](#lesson-2) | 📖 Lesson |
| 4 | [Send GET and POST with curl](#task-2) | 🛠️ Task |
| 5 | [Status codes](#lesson-3) | 📖 Lesson |
| 6 | [Trigger 200, 404 and 301](#task-3) | 🛠️ Task |
| 7 | [Reading curl -v output](#lesson-4) | 📖 Lesson |
| 8 | [Answer questions about a curl -v output](#task-4) | 🛠️ Task |
| 9 | [Practice with more curl requests](#task-5) | 🛠️ Task |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: The request and the response

**HTTP** is the language between web clients and web servers. Both the request and the response are **plain readable text**.

### The request: four parts

```
GET /index.html HTTP/1.1        ← request line
Host: localhost                 ← header
Connection: close               ← header
                                ← blank line: headers end here
(body)                          ← empty for GET
```

| Part | Meaning |
|------|---------|
| **request line** | what you want: a **method** (`GET`), a **path** (`/index.html`), and the protocol version |
| **headers** | extra details, one per line, as `Name: value` |
| **blank line** | marks the end of the headers |
| **body** | optional data (empty for `GET`) |

### The response: a mirror image

```
HTTP/1.1 200 OK                 ← status line
Content-Type: text/html         ← header
Content-Length: 615             ← header
                                ← blank line
<html>...</html>                ← body
```

Only the first line differs: instead of asking for something it reports a **status code** (`200 OK` = success).

### Speak HTTP yourself with `nc`

Because HTTP is text, you can send a request with `nc` and no browser at all. Lines end with **`\r\n`**, and **`Connection: close`** asks the server to hang up after answering so `nc` exits cleanly:

```bash
printf 'GET / HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n' | nc 127.0.0.1 80
```

```
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Content-Type: text/html
...
<!DOCTYPE html>
<html>
```

> **💡 Tip:** Every tool you'll ever use for the web (`curl`, browsers, load balancers) builds and reads exactly this text. When you can read it raw, **no HTTP problem can hide from you.**

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Send a raw HTTP request with nc

nginx serves its default page on port 80. Talk to it **without `curl`**, using only `nc`:

1. Send a valid `GET` request for `/` to `127.0.0.1` port 80, including a **`Host`** header and a header telling the server to **close the connection** after answering.
2. Capture the **complete raw response** (status line, headers, body) into `/root/http/raw-response.txt`.
3. Read the file and find the parts: status line, headers, blank line, body.

**Solution (from the site):**

```bash
mkdir -p /root/http
printf 'GET / HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n' | nc 127.0.0.1 80 > /root/http/raw-response.txt
cat /root/http/raw-response.txt
```

`printf` builds the request byte by byte, `nc` delivers it to nginx on port 80, and the redirect stores the raw response: status line, headers, blank line and HTML body, exactly as they crossed the network.

What the file looked like in my lab run:

```
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Fri, 16 Oct 2026 21:14:56 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Mon, 14 Sep 2026 14:31:25 GMT
Connection: close
ETag: "6aa8053d-267"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Methods, GET and POST

The **method** is the first word of the request line. It tells the server what kind of action you want.

| Method | Meaning | Body? |
|--------|---------|-------|
| **GET** | "send me this resource": loading a page, downloading a file, a read-only API call. **Never changes anything**, so it's safe to repeat | no |
| **POST** | "here is data, process it": logging in, submitting a form, creating a record via an API | **yes**, below the blank line |
| `PUT` | replace a resource with the body you send | yes |
| `DELETE` | remove a resource | |
| `PATCH` | change part of a resource | yes |
| `HEAD` | like GET but returns **only the headers** | no |

With `curl`, **`-d`** attaches a body and switches the method to POST in one move:

```bash
curl -d 'name=alice' http://127.0.0.1:9000/submit
```

```
method=POST path=/submit body=name=alice
```

That example talks to a small test server that **repeats back what it received**. Without `-d`, the same `curl` command would send a GET.

> **💡 Tip:** When you debug a failing request, **the method is the first thing to confirm.** An API that expects POST answers a GET with an error, and the fix is one flag, not a server change.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Send GET and POST with curl

A small test server runs on `127.0.0.1:9000` and answers every request with one line showing the method, path and body it received. nginx is still on port 80.

1. Fetch the nginx default page on port 80 with a **GET** and save the body to `/root/http/get-response.txt`.
2. Send a **POST** to `http://127.0.0.1:9000/submit` with body `name=alice` and save the reply to `/root/http/post-response.txt`.
3. Read the second file: it must show the server saw a POST carrying your data.

**Solution (from the site):**

```bash
mkdir -p /root/http
curl -s http://127.0.0.1:80/ > /root/http/get-response.txt
curl -s -d 'name=alice' http://127.0.0.1:9000/submit > /root/http/post-response.txt
cat /root/http/post-response.txt
```

The first is a GET (no body). The second uses `-d`, so `curl` switches to POST and puts `name=alice` in the body. The echo server writes back `method=POST path=/submit body=name=alice`: your proof.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Status codes

Every response starts with a **three-digit status code**. **The first digit names the family, and the family alone tells you who to blame.**

### The four families

| Family | Meaning | Who's at fault |
|--------|---------|----------------|
| **2xx** | success: the request worked | |
| **3xx** | redirect: what you want is at another URL | |
| **4xx** | **client error**: your request was wrong | **you** |
| **5xx** | **server error**: your request was fine, the server broke | **the server** |

### The dozen codes you'll actually meet

| Code | Name | Meaning |
|------|------|---------|
| 200 | OK | the normal success |
| 201 | Created | success that made a new resource |
| 204 | No Content | success with an empty body |
| 301 | Moved Permanently | the URL changed for good |
| 302 | Found | a temporary redirect |
| 304 | Not Modified | your cached copy is still current |
| 400 | Bad Request | the server could not parse your request |
| 401 | Unauthorized | you did not prove who you are |
| 403 | Forbidden | you are known but not allowed |
| 404 | Not Found | that path does not exist |
| 500 | Internal Server Error | the app crashed handling you |
| 502 / 503 | Bad Gateway / Service Unavailable | a proxy could not reach a healthy backend |

### Print just the code with `curl`

`-w` prints values after a transfer; `%{http_code}` is the status code. Send the body to `/dev/null` to see only the code:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/nope
```

```
404
```

> **💡 Tip:** This one-liner is a standard **health check**. Monitoring systems run exactly this, expecting `200`, and page someone when a `5xx` shows up.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Trigger 200, 404 and 301

nginx now serves its default page on port 80 and a site on **port 8080** where the path `/old-docs` has permanently moved to `/docs`. Produce each code on purpose, **one code per file** (code only, three digits):

| File | Request |
|------|---------|
| `/root/http/code-200.txt` | a page that exists |
| `/root/http/code-404.txt` | a path that doesn't exist on port 80 |
| `/root/http/code-301.txt` | `http://127.0.0.1:8080/old-docs` |

**Solution (from the site):**

```bash
mkdir -p /root/http
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/ > /root/http/code-200.txt
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/does-not-exist > /root/http/code-404.txt
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/old-docs > /root/http/code-301.txt
```

The existing page gives 200, the invented path gives 404, and the configured redirect at `/old-docs` answers **301** with a `Location` header pointing at `/docs`.

<br>

---

<br>

<a id="lesson-4"></a>

## 📖 Lesson 4: Reading curl -v output

Add **`-v`** to any `curl` and it prints the whole conversation, marked line by line:

| Prefix | Meaning |
|--------|---------|
| `>` | what **curl sent**: the request |
| `<` | what **the server sent**: the response headers |
| `*` | curl talking about its own work: resolving, connecting, closing |

### A full transcript, annotated

```bash
curl -v http://127.0.0.1/ -o /dev/null
```

```
* Host 127.0.0.1:80 was resolved.
* Trying 127.0.0.1:80...
* Connected to 127.0.0.1 (127.0.0.1) port 80
> GET / HTTP/1.1
> Host: 127.0.0.1
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Content-Type: text/html
< Content-Length: 615
<
{ [615 bytes data]
* Connection #0 to host 127.0.0.1 left intact
```

Read it top to bottom and you can answer every debugging question:

| Question | Where to look |
|----------|---------------|
| Did DNS resolve? | the `*` lines |
| Did TCP connect? | `Connected to` |
| What exactly did we ask for? | the `>` block (the request from lesson 1, live) |
| What came back? | the `<` block: status line and headers |
| The body? | not prefixed (here it went to `/dev/null`) |

> **💡 Tip:** When a request fails, the transcript shows which step died. **Stuck at `Trying`?** Network or firewall. **Connected but a 4xx came back?** Read your own `>` lines: the mistake is usually there.

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Answer questions about a curl -v output

A saved verbose transcript from a failed API call sits at `/root/http/transcript.txt`. Read it and write four answer files (each holds **only the answer**):

| File | Answer |
|------|--------|
| `/root/http/method.txt` | the request method |
| `/root/http/path.txt` | the path requested |
| `/root/http/status.txt` | the status code returned (digits only) |
| `/root/http/content-type.txt` | the value of the **response's** `Content-Type` header |

**Input file** `/root/http/transcript.txt` ([raw copy](lab-files/14-http/transcript.txt)):

```
* Host api.shop.internal:80 was resolved.
*   Trying 10.0.4.20:80...
* Connected to api.shop.internal (10.0.4.20) port 80
> POST /api/login HTTP/1.1
> Host: api.shop.internal
> User-Agent: curl/8.5.0
> Accept: */*
> Content-Type: application/x-www-form-urlencoded
> Content-Length: 29
>
* upload completely sent off: 29 bytes
< HTTP/1.1 401 Unauthorized
< Server: nginx/1.24.0 (Ubuntu)
< Date: Tue, 09 Sep 2026 10:14:03 GMT
< Content-Type: application/json
< Content-Length: 38
< Connection: keep-alive
<
{"error":"invalid username or password"}
* Connection #0 to host api.shop.internal left intact
```

**Solution (from the site):**

```bash
echo 'POST' > /root/http/method.txt
echo '/api/login' > /root/http/path.txt
echo '401' > /root/http/status.txt
echo 'application/json' > /root/http/content-type.txt
```

- The request line `> POST /api/login HTTP/1.1` gives the **method and path**.
- The status line `< HTTP/1.1 401 Unauthorized` gives the **code**.

> **⚠️ Warning:** The transcript has **two `Content-Type` headers**. The response's is the **`<`** one (`application/json`); the **`>`** one (`application/x-www-form-urlencoded`) belongs to the request body the client uploaded.

<br>

---

<br>

<a id="task-5"></a>

## 🛠️ Task 5: Practice with more curl requests

Same setup: nginx on port 80, the moved docs site on port 8080, the echo server on port 9000. Run five requests, record each status code, and build `/root/http/report.txt` with exactly one line per request, `METHOD path code`:

| Request | Expected |
|---------|----------|
| `GET /` on port 80 | `GET / 200` |
| `GET /missing` on port 80 | `GET /missing 404` |
| `GET /old-docs` on port 8080, **without following** the redirect | `GET /old-docs 301` |
| `GET /docs` on port 8080 | `GET /docs 200` |
| `POST` any body to `/submit` on port 9000 | `POST /submit 200` |

The codes must be the ones the servers **actually returned**.

**Solution (from the site):**

```bash
mkdir -p /root/http
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/missing
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/old-docs
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/docs
curl -s -o /dev/null -w '%{http_code}\n' -d 'x=1' http://127.0.0.1:9000/submit
cat > /root/http/report.txt <<'TXT'
GET / 200
GET /missing 404
GET /old-docs 301
GET /docs 200
POST /submit 200
TXT
```

A success, a missing path, a redirect, the redirect target, and a POST. Reading codes this fast is exactly what you do when a service acts strangely.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Command | Purpose |
|---------|---------|
| `printf 'GET / HTTP/1.1\r\nHost: h\r\nConnection: close\r\n\r\n' \| nc HOST 80` | raw HTTP by hand |
| `curl URL` | GET |
| `curl -d 'a=b' URL` | POST (adds a body) |
| `curl -I URL` | headers only (HEAD) |
| `curl -v URL` | full conversation (`>` sent, `<` received, `*` curl) |
| `curl -s -o /dev/null -w '%{http_code}\n' URL` | status code only (health check) |
| `curl -L URL` | follow redirects (**don't** use it when you want the 301) |

| Status family | Meaning |
|---------------|---------|
| 2xx | success |
| 3xx | redirect (301 permanent, 302 temporary, 304 cached) |
| 4xx | **you** (400 bad, 401 who are you?, 403 not allowed, 404 missing) |
| 5xx | **the server** (500 crashed, 502/503 proxy can't reach a healthy backend) |

| Concept | Remember |
|---------|----------|
| request | request line + headers + blank line + optional body |
| response | status line + headers + blank line + body |
| GET vs POST | read-only, no body vs sends data in the body |
| two Content-Types in `-v` | `>` = request, `<` = response |
| `-v` debugging | stuck at `Trying` = network/firewall; 4xx = re-read your `>` lines |

<br>

[↑ Back to top](#top)
