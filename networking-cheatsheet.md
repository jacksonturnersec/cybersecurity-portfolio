# Networking Fundamentals
*SOC Analyst Study Notes — Jackson Turner*

---

## Core Concepts

**Cybersecurity** is the ongoing effort to protect individuals, organisations and governments from digital attacks by protecting networked systems and data from unauthorised use or harm.

**IP Address** — A unique numerical label assigned to every device on a network. Written as four numbers separated by dots (called octets) e.g. 192.168.1.1. Used to identify where to send and receive data. Can be temporary (dynamic) or fixed (static). There are two types: **IPv4** (the four octet format) and **IPv6** (newer, longer format used because we're running out of IPv4 addresses).

**Ping** — A network tool that sends **ICMP (Internet Control Message Protocol)** echo request packets to a target device and waits for a reply. Used to test whether a device is reachable on the network and how long the response takes (measured in milliseconds). A high response time or no response can indicate connectivity issues or a firewall blocking traffic. As an SOC analyst ping will be used in two ways — legitimate network troubleshooting, and by attackers doing reconnaissance to check if a host is alive before attacking it. Know the difference.

---

## DNS (Domain Name System)

Acts like a phone book for the internet. When you type a website address like google.com, DNS translates it into an IP address so your device knows where to send the request.

| Record | What it does |
|--------|-------------|
| A Record | Maps a domain to an IPv4 address (e.g. 192.168.1.1) |
| AAAA Record | Maps a domain to an IPv6 address (longer, newer format) |
| CNAME | Points one domain to another domain |
| MX Record | Tells email where to go (mail servers) |
| TXT Record | Stores text info, often used for security verification |

**Why DNS matters in SOC:** Attackers abuse DNS constantly. Things like DNS tunnelling (hiding data in DNS queries), typosquatting (fake domains like g00gle.com), and phishing domains all show up in DNS logs. As an analyst you'll be querying DNS records regularly to investigate suspicious traffic.

---

## HTTP / HTTPS

**HTTP — HyperText Transfer Protocol** — The set of rules that governs how data is requested and sent between your browser and a web server. Transfers HTML, images, video and other web content. Created by Tim Berners-Lee. **Runs on port 80.**

**HTTPS — HTTP Secure** — The encrypted version of HTTP using TLS (Transport Layer Security). It protects data in transit so it can't be intercepted or read by a third party. Also verifies you're talking to the legitimate server, not an imposter. **Runs on port 443.**

**SOC Relevance:**
- Unencrypted HTTP traffic can be captured and read in tools like Wireshark
- Attackers use HTTP to deliver malware and phishing pages
- Seeing traffic on port 80 when it should be on 443 is a red flag
- HTTPS doesn't mean a site is safe — attackers use HTTPS too. It just means the connection is encrypted

---

## Ports

A numerical value that tells a device which application or service a piece of network traffic is intended for. Think of it like an apartment building — the IP address gets you to the building, but the port number tells you which apartment to knock on. **Every network connection has both an IP address and a port number.**

### Ports SOC Analysts Must Know

| Port | Protocol | What It Is |
|------|----------|------------|
| 80 | HTTP | Unencrypted web traffic |
| 443 | HTTPS | Encrypted web traffic |
| 22 | SSH | Secure remote login |
| 21 | FTP | File transfers |
| 25 | SMTP | Sending email |
| 53 | DNS | Domain Name Lookups |
| 3389 | RDP | Remote desktop (Windows) |

**Why Ports matter in SOC:**
- Attackers use unusual ports to hide malicious traffic
- Seeing RDP (3389) open to the internet is a major red flag
- Port scanning is one of the first things attackers do during reconnaissance
- You'll filter by port constantly in Wireshark and SIEM tools
