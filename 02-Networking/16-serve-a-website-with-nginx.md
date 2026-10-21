<a id="top"></a>

# 🌐 16 - Serve a website with nginx

> **Source:** <https://www.escbash.com/topics/net-serve-web>
> **Contents:** 3 lessons · 4 hands-on tasks · 1 quiz

## 📑 Table of Contents

| # | Section | Type |
|---|---------|------|
| 1 | [How nginx serves files](#lesson-1) | 📖 Lesson |
| 2 | [Serve your own page](#task-1) | 🛠️ Task |
| 3 | [Change the port nginx listens on](#task-2) | 🛠️ Task |
| 4 | [Server blocks, many sites on one nginx](#lesson-2) | 📖 Lesson |
| 5 | [Set up two server blocks](#task-3) | 🛠️ Task |
| 6 | [Reading nginx logs](#lesson-3) | 📖 Lesson |
| 7 | [Find a failing request in the logs](#task-4) | 🛠️ Task |
| 8 | [Scenario: nginx is down after a config change](#quiz-1) | ❓ Quiz |
| — | [Cheat sheet](#cheat-sheet) | ⚡ Summary |

<br>

---

<br>

<a id="lesson-1"></a>

## 📖 Lesson 1: How nginx serves files

nginx's job is simple: take the **path from an HTTP request**, find the **matching file on disk**, and send it back.

```
 client asks for /about.html ──► nginx ──► /var/www/html/about.html on disk
```

### Where the config lives

nginx reads **`/etc/nginx/nginx.conf`**, which pulls in two folders: **`/etc/nginx/conf.d/`** and **`/etc/nginx/sites-enabled/`**. Each file there can hold a **server block**, one website each:

```nginx
server {
    listen 8080;
    root /var/www/myapp;
    index index.html;
}
```

| Setting | Meaning |
|---------|---------|
| `listen` | the port |
| `root` | the folder paths are looked up under: a request for `/about.html` becomes `/var/www/myapp/about.html` |
| `index` | the file served when the path is just `/` |

### Test, then reload

Config changes take effect only when you tell nginx. **Always in two steps:**

```bash
nginx -t
systemctl reload nginx
```

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

| Command | What it does |
|---------|--------------|
| `nginx -t` | checks the syntax **without touching the running server** |
| `systemctl reload nginx` | tells the running nginx to **reread config without dropping connections** |
| `systemctl restart nginx` | stops and starts the process; needed rarely, mostly when reload can't apply a change |

> **⚠️ Warning:** **Test first, every time.** A reload with a broken config fails, and a **restart with one leaves nginx down.**

> **💡 Tip:** nginx is the most common web server in DevOps work. Every reverse proxy, ingress and load balancer chapter ahead builds on these blocks.

<br>

---

<br>

<a id="task-1"></a>

## 🛠️ Task 1: Serve your own page

Put a site of your own next to the nginx default site:

1. Create `/var/www/myapp` and inside it an `index.html` whose content includes the exact text **`my first nginx site`**.
2. Write a server block in a **new file under `/etc/nginx/conf.d/`** that listens on **8080** and serves that folder, with `index.html` as the index.
3. Apply the config the safe way, then confirm with `curl` that `http://127.0.0.1:8080/` returns your page.
4. Leave the default site on port 80 untouched.

**Solution (from the site):**

```bash
mkdir -p /var/www/myapp
echo '<h1>my first nginx site</h1>' > /var/www/myapp/index.html
cat > /etc/nginx/conf.d/myapp.conf <<'CONF'
server {
    listen 8080;
    root /var/www/myapp;
    index index.html;
}
CONF
nginx -t
systemctl reload nginx
curl http://127.0.0.1:8080/
```

The block maps port 8080 to the `/var/www/myapp` folder, and `index.html` is what a request for `/` receives. `nginx -t` before reload catches typos while the old config is still serving.

<br>

---

<br>

<a id="task-2"></a>

## 🛠️ Task 2: Change the port nginx listens on

The `myapp` site is on 8080. The port is needed for something else, so **move the site to 8081**:

1. Change the server block in `/etc/nginx/conf.d/myapp.conf` to listen on **8081**.
2. Apply it safely and verify two things: the **listening socket moved to 8081**, and `http://127.0.0.1:8081/` returns the `my first nginx site` page.
3. **Nothing may listen on 8080 afterwards.**

**Solution (from the site):**

```bash
sed -i 's/listen 8080;/listen 8081;/' /etc/nginx/conf.d/myapp.conf
nginx -t
systemctl reload nginx
ss -tlnp | grep nginx
curl http://127.0.0.1:8081/
```

One number in the config is the whole change. After the reload, `ss` shows nginx bound to **8081 and not 8080**, and `curl` proves the same site answers at its new address.

> **📝 Note:** Moving a port means the old `listen` line is **gone, not duplicated**.

<br>

---

<br>

<a id="lesson-2"></a>

## 📖 Lesson 2: Server blocks, many sites on one nginx

You met this from the client side in the headers topic: the **`Host` header picks the site**. Now build the server side. Each site is its own **server block**, and **`server_name`** is what nginx matches the `Host` header against:

```nginx
server {
    listen 8080;
    server_name site-a.test;
    root /var/www/site-a;
    index index.html;
}
server {
    listen 8080;
    server_name site-b.test;
    root /var/www/site-b;
    index index.html;
}
```

Both blocks listen on the **same port**: that's not a conflict. There's one listening socket, and for each request nginx reads the `Host` header, finds the block whose `server_name` matches, and serves from that block's `root`. These are **virtual hosts**: many sites sharing one server and one IP.

### When no name matches

A request whose `Host` matches nothing gets the **default block**: the **first one for that port**, or the one marked `listen 8080 default_server;`.

> **⚠️ Warning:** **Set the default deliberately.** Sites appearing under the wrong name almost always mean a request fell through to an unintended default.

### Reaching test names

Names like `site-a.test` exist in no DNS. For local testing you already know both tricks: an **`/etc/hosts` entry** pointing the name at `127.0.0.1`, or a **dnsmasq `address=`** record. Then:

```bash
curl http://site-a.test:8080/
```

resolves the name locally, connects, and sends `Host: site-a.test`, which selects the right block.

> **💡 Tip:** Nearly every real web server is set up this way. Hosting companies pack hundreds of sites on one machine, and a Kubernetes ingress routing by hostname is the same idea.

<br>

---

<br>

<a id="task-3"></a>

## 🛠️ Task 3: Set up two server blocks

Run two named sites on one nginx, both on port **8080**:

1. Create `/var/www/site-a/index.html` containing the exact text **`this is site a`**, and `/var/www/site-b/index.html` containing **`this is site b`**.
2. Two server blocks: `site-a.test` serving `/var/www/site-a`, and `site-b.test` serving `/var/www/site-b`, both listening on 8080.
3. Make both names resolve to `127.0.0.1` on this machine.
4. Verify each by name: `http://site-a.test:8080/` and `http://site-b.test:8080/` must return **different pages**.

**Solution (from the site):**

```bash
mkdir -p /var/www/site-a /var/www/site-b
echo '<h1>this is site a</h1>' > /var/www/site-a/index.html
echo '<h1>this is site b</h1>' > /var/www/site-b/index.html
cat > /etc/nginx/conf.d/sites.conf <<'CONF'
server {
    listen 8080;
    server_name site-a.test;
    root /var/www/site-a;
    index index.html;
}
server {
    listen 8080;
    server_name site-b.test;
    root /var/www/site-b;
    index index.html;
}
CONF
echo '127.0.0.1 site-a.test' >> /etc/hosts
echo '127.0.0.1 site-b.test' >> /etc/hosts
nginx -t
systemctl reload nginx
curl http://site-a.test:8080/
curl http://site-b.test:8080/
```

Both blocks share port 8080. The hosts entries make the names resolve locally, `curl` sends each name as the `Host` header, and nginx's `server_name` matching picks the right root folder.

<br>

---

<br>

<a id="lesson-3"></a>

## 📖 Lesson 3: Reading nginx logs

nginx keeps two logs under **`/var/log/nginx/`**:

| Log | Contains |
|-----|----------|
| **`access.log`** | **one line per request**, successful or not |
| **`error.log`** | details when something goes wrong **inside nginx itself**: config problems, missing files, backends not answering |

### Anatomy of an access log line

```
203.0.113.9 - - [09/Sep/2026:10:14:03 +0000] "GET /admin/config HTTP/1.1" 404 162 "-" "curl/8.5.0"
```

| Field | Meaning |
|-------|---------|
| `203.0.113.9` | **who**: the client IP |
| `[09/Sep/2026:10:14:03 +0000]` | timestamp |
| `"GET /admin/config HTTP/1.1"` | **what**: the request line (method, path, version) |
| `404` | **how it went**: the status code |
| `162` | response size in bytes |
| `"-"` | referrer |
| `"curl/8.5.0"` | the client's User-Agent |

### Asking the log questions

The log is plain text, so `grep` works:

```bash
grep ' 404 ' /var/log/nginx/access.log | tail -3
```

The same client asking for the same missing path repeatedly might be a **broken script, an old bookmark, or someone probing for an admin page**. The log is how you find out which.

> **💡 Tip:** When a user says "the site is broken", the **access log** tells you whether their requests arrived and what code they got. When nginx itself misbehaves, **`error.log`** holds the reason. **Check both before touching any config.**

<br>

---

<br>

<a id="task-4"></a>

## 🛠️ Task 4: Find a failing request in the logs

Overnight traffic landed in `/var/log/nginx/access.log` and something kept failing. Record what you find, **one bare answer per file**:

| File | Answer |
|------|--------|
| `/root/logs/path.txt` | the path the failing requests asked for |
| `/root/logs/code.txt` | the status code (digits only) |
| `/root/logs/client.txt` | the client IP the failures came from |

All the failures share one path, one code, and one client.

**Input:** the lab's `access.log` has 22 lines; the failing ones (non-200) are ([raw copy](lab-files/16-serve-web/access-log-failures.txt)):

```
198.51.100.23 - - [13/Sep/2026:08:11:44 +0000] "GET /reports/latest.pdf HTTP/1.1" 404 162 "-" "python-requests/2.31"
198.51.100.23 - - [13/Sep/2026:08:12:44 +0000] "GET /reports/latest.pdf HTTP/1.1" 404 162 "-" "python-requests/2.31"
198.51.100.23 - - [13/Sep/2026:08:13:44 +0000] "GET /reports/latest.pdf HTTP/1.1" 404 162 "-" "python-requests/2.31"
198.51.100.23 - - [13/Sep/2026:08:14:44 +0000] "GET /reports/latest.pdf HTTP/1.1" 404 162 "-" "python-requests/2.31"
198.51.100.23 - - [13/Sep/2026:08:15:44 +0000] "GET /reports/latest.pdf HTTP/1.1" 404 162 "-" "python-requests/2.31"
```

> **📝 Note:** The other 17 lines are successful (`200`) requests and weren't copied. The minutes in the timestamps above are as I read them off the terminal.

**Solution (from the site):**

```bash
grep -v ' 200 ' /var/log/nginx/access.log
mkdir -p /root/logs
echo '/reports/latest.pdf' > /root/logs/path.txt
echo '404' > /root/logs/code.txt
echo '198.51.100.23' > /root/logs/client.txt
```

Every failing line looks the same: client `198.51.100.23` requesting `/reports/latest.pdf` and receiving `404`, **five times in a row**. That pattern, **one client retrying one missing path**, usually means an automated job pointing at a file that was moved or never published.

<br>

---

<br>

<a id="quiz-1"></a>

## ❓ Quiz 1: Scenario: nginx is down after a config change

**Q:** A teammate edited an nginx server block, ran `systemctl restart nginx`, and now every site on the machine is down. `systemctl status nginx` shows the service failed to start. What do you do first?

- [x] **Run `nginx -t` and read the error message it prints**
- [ ] Look at the end of `/var/log/nginx/error.log` for the startup failure
- [ ] Reboot the machine, and if nginx still fails, reinstall the nginx package
- [ ] Ask the teammate what they changed and revert their edit from memory

**Why:** **this is the move.** `nginx -t` validates the config and prints the **exact file and line** that breaks it (like `unexpected end of file` or `unknown directive`). Fix that line, run `nginx -t` again until it passes, then start the service: about a minute.

| Option | Verdict |
|--------|---------|
| `error.log` | holds the same message, but `nginx -t` is the tool built for this |
| revert from memory | sound only when you can revert **exactly** |
| reboot / reinstall | **superstition**: the broken file survives both, so the failure does too |

> **💡 Tip:** This is the argument for the habit: **`nginx -t` before every reload or restart**, so a bad edit never takes the server down at all.

<br>

---

<br>

<a id="cheat-sheet"></a>

## ⚡ Cheat sheet

| Item | Meaning |
|------|---------|
| `/etc/nginx/nginx.conf` | main config; includes `conf.d/` and `sites-enabled/` |
| `/etc/nginx/conf.d/*.conf` | one file per site (server blocks) |
| `listen PORT;` | the port |
| `server_name NAME;` | matched against the `Host` header |
| `root DIR;` / `index FILE;` | where files live / what `/` serves |
| `listen PORT default_server;` | the fallback block |

| Command | Purpose |
|---------|---------|
| `nginx -t` | test config (names the file and line on errors) |
| `systemctl reload nginx` | apply config without dropping connections |
| `systemctl restart nginx` | full restart (only when needed; **test first**) |
| `ss -tlnp \| grep nginx` | which ports nginx listens on |
| `curl http://name:PORT/` | test a site (name must resolve: `/etc/hosts` or dnsmasq) |
| `/var/log/nginx/access.log` | one line per request |
| `/var/log/nginx/error.log` | nginx's own problems |
| `grep ' 404 ' access.log \| tail` | recent missing paths |
| `grep -v ' 200 ' access.log` | everything that failed |

| Concept | Remember |
|---------|----------|
| virtual hosts | many `server` blocks on one port, chosen by `Host` = `server_name` |
| no match | falls to the **default** block (set it deliberately) |
| access-log line | who (IP) · what (request line) · how it went (status) |
| nginx won't start | `nginx -t` first |

<br>

[↑ Back to top](#top)
