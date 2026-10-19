<a id="top"></a>

# 🌐 Networking: Study Notes

> **Source course:** <https://www.escbash.com/skills/networking>
> **Done so far:** modules 1–22 of 32

## 📑 Index

| # | Module | Lessons | Tasks | Quizzes | Status |
|---|--------|:------:|:-----:|:------:|--------|
| 01 | [What is a network](01-what-is-a-network.md) | 3 | 4 | 0 | ✅ |
| 02 | [IP addresses](02-ip-addresses.md) | 4 | 5 | 0 | ✅ |
| 03 | [Interfaces](03-interfaces.md) | 3 | 4 | 0 | ✅ |
| 04 | [Subnets and CIDR](04-subnets-and-cidr.md) | 4 | 6 | 0 | ✅ |
| 05 | [MAC addresses and ARP](05-mac-addresses-and-arp.md) | 2 | 3 | 1 | ✅ |
| 06 | [The layers model](06-the-layers-model.md) | 3 | 3 | 0 | ✅ |
| 07 | [Ports and sockets](07-ports-and-sockets.md) | 3 | 4 | 1 | ✅ |
| 08 | [TCP](08-tcp.md) | 4 | 5 | 0 | ✅ |
| 09 | [UDP](09-udp.md) | 2 | 2 | 0 | ✅ |
| 10 | [Netcat](10-netcat.md) | 3 | 4 | 1 | ✅ |
| 11 | [DNS basics](11-dns-basics.md) | 4 | 5 | 1 | ✅ |
| 12 | [DNS record types](12-dns-record-types.md) | 4 | 5 | 0 | ✅ |
| 13 | [Run your own DNS](13-run-your-own-dns.md) | 3 | 3 | 1 | ✅ |
| 14 | [HTTP](14-http.md) | 4 | 5 | 0 | ✅ |
| 15 | [HTTP headers, redirects, JSON](15-http-headers-redirects-json.md) | 4 | 5 | 0 | ✅ |
| 16 | [Serve a website with nginx](16-serve-a-website-with-nginx.md) | 3 | 4 | 1 | ✅ |
| 17 | [TLS and HTTPS](17-tls-and-https.md) | 3 | 4 | 0 | ✅ |
| 18 | [Make your own certificates](18-make-your-own-certificates.md) | 3 | 3 | 1 | ✅ |
| 19 | [Routing and gateways](19-routing-and-gateways.md) | 4 | 6 | 0 | ✅ |
| 20 | [NAT](20-nat.md) | 3 | 4 | 0 | ✅ |
| 21 | [DHCP](21-dhcp.md) | 2 | 2 | 0 | ✅ |
| 22 | [Firewall basics](22-firewall-basics.md) | 3 | 4 | 0 | ✅ |

<br>

## 📝 How accurate are the answers?

**Modules 1–7** were completed on the account first, so every task solution and quiz answer is the site's own.

**Modules 8–22** were completed in the lab as part of writing the notes. For each task I used the site's own flow (submit, reveal the hint, submit again, then show the solution) or solved it myself, and the site's solution is what's quoted. Where I used a different command that also passed the site's checks, the note says so ("my lab run" / "what I ran"). In modules 19–22 most tasks were solved directly from the lesson commands (all passed 100%); module 20's conntrack task is the one place the site's `conntrack -L` failed in my lab, so the note shows both the site's solution and the `/proc/net/nf_conntrack` route that passed. Module 22's prediction tasks include the rulesets I read from the lab and my reasoning for each packet.

Lab input files (for example `pairs.txt`, `drill.txt`, `ss-snapshot.txt`, `dig-output.txt`) are shown under the task that uses them, with raw copies in [lab-files/](lab-files/). Modules with input files: 2, 4, 6, 7, 8, 10, 11, 14, 15 and 16 (module 16's `access.log` has 22 lines; only the 5 failing ones were copied). They were read with `cat` in the lab terminal and copied from what the terminal showed, so trailing newlines may differ slightly from the originals.

Several tasks query live DNS (for example `dig example.com`), so the addresses in your own run can differ from the examples in the notes. The shape of the answer is what matters.

Explanations I added myself are marked with a note (for example the three-networks worked example in module 5).

<br>

[↑ Back to top](#top)
