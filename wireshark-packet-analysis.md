# Wireshark Packet Analysis — Home Lab Notes

**Author:** Jackson Turner  
**Environment:** Kali Linux 2025.2 — VirtualBox (eth0)  
**Sprint Week:** 4 — Packet Analysis  
**Date:** June 2026

---

## What is Wireshark

Wireshark is a free, open-source network protocol analyser used to capture and inspect traffic passing through a network interface in real time. It decodes packets at every layer of the OSI model and presents the raw data in a human-readable format.

It is one of the most widely used tools in network forensics, threat hunting, and SOC investigations. Every packet captured contains the source, destination, protocol, and payload — giving analysts a ground-level view of what is actually travelling across the wire.

---

## Why SOC Analysts Use It

Wireshark allows analysts to:

- **Detect malicious traffic patterns** — C2 beaconing, data exfiltration, DNS tunnelling, and protocol abuse all leave signatures in captured traffic
- **Investigate suspicious hosts** — confirm whether a flagged IP is actively communicating and what it is sending
- **Verify alerts** — SIEM alerts often need packet-level confirmation before escalation
- **Analyse protocols in depth** — understand exactly what a process is doing at the network layer, not just that it connected somewhere
- **Support incident response** — reconstruct what happened on the wire during a security event

Wireshark works alongside SIEM and endpoint tools like Sysmon. Sysmon logs which process made a connection (Event ID 3). Wireshark shows you exactly what was in it.

---

## Interface Overview — Key Columns

When a capture is running, Wireshark displays packets in a table with the following columns:

| Column | What It Shows |
|--------|--------------|
| **No.** | Packet number in capture sequence |
| **Time** | Timestamp since capture started |
| **Source** | IP or MAC address the packet came from |
| **Destination** | IP or MAC address the packet is going to |
| **Protocol** | Protocol used (DNS, TCP, ICMP, ARP, etc.) |
| **Length** | Packet size in bytes |
| **Info** | Plain-English summary of what the packet is doing |

The **Info** column is the fastest triage field — it gives you the human-readable summary before you drill into the layers.

---

## Protocols Captured — Live Lab (ping google.com)

A single `ping google.com` command on Kali Linux generated three distinct protocol types: **ARP**, **DNS**, and **ICMP**. Each serves a different purpose in getting that ping off the machine and back.

---

### ARP — Address Resolution Protocol

**What it does:**  
Translates IP addresses into MAC addresses on a local network. Before any traffic leaves the machine, it needs to know the MAC address of the next hop (usually the default gateway). ARP broadcasts "who has this IP?" across the local network and the owner replies with their MAC address. It operates at **Layer 2 (Data Link)** and never leaves the local network segment.

**SOC Relevance:**  
ARP spoofing is a common man-in-the-middle (MitM) attack. An attacker sends fake ARP replies to poison the ARP cache of other hosts, redirecting traffic through their machine before forwarding it on. The victim doesn't know their traffic is being intercepted. Signs in Wireshark: duplicate ARP replies for the same IP, or a MAC address associated with multiple IPs.

---

### DNS — Domain Name System

**What it does:**  
Translates human-readable domain names into IP addresses. When `ping google.com` runs, DNS is queried first to convert `google.com` into a routable IP address (e.g. 142.250.124.101). DNS runs on **UDP port 53**.

**Why UDP:**  
DNS uses UDP instead of TCP because speed matters more than guaranteed delivery for a quick lookup. The overhead of a TCP handshake is unnecessary for a small, fast query.

**SOC Relevance:**  
DNS is heavily abused by attackers because it is rarely blocked at the firewall. Common abuse cases include:
- **DNS tunnelling** — encoding data inside domain names to exfiltrate information or establish C2 channels
- **C2 callbacks** — malware checking in with its command server via DNS queries
- **Domain generation algorithms (DGA)** — malware generating random-looking domain names to avoid blocklists
- **Typosquatting** — fake domains like `g00gle.com` used in phishing

---

### ICMP — Internet Control Message Protocol

**What it does:**  
A diagnostic and error-reporting protocol. `ping` uses ICMP — it sends an **Echo Request** packet and waits for an **Echo Reply** to confirm the host is reachable. ICMP carries network health information, not application data.

**Sequence Numbers:**  
ICMP Echo Request/Reply pairs are numbered with sequence numbers (e.g. seq=1/256, seq=2/512). Analysts use these to detect anomalies such as out-of-order packets, missing replies, or unexpectedly large payloads.

**SOC Relevance:**  
- **ICMP tunnelling** — attackers embed data inside ICMP packets to create a covert communication channel
- **Network reconnaissance** — ping sweeps across IP ranges are used to map live hosts before an attack; a single host generating hundreds of ICMP requests to different destinations is a red flag
- **Blocked replies** — a host that doesn't respond to ICMP may have a firewall in place, which is itself useful intelligence during reconnaissance

---

## Packet Layer Breakdown — DNS Query (Live Capture)

After clicking on a DNS packet in the capture, Wireshark expands it layer by layer following the OSI model. The following breakdown is from a live DNS query for `google.com` captured on `eth0`.

**Packet:** Frame 4 — 70 bytes, captured on eth0

---

### Layer 2 — Ethernet II (Data Link)
MAC addresses. Identifies who is talking to who on the local network. This is where ARP resolution feeds in — the destination MAC is the gateway, resolved by ARP before the packet was sent.

---

### Layer 3 — Internet Protocol Version 4 (Network)
IP addresses. Routes the packet across networks.  
- **Source:** 10.0.2.15 (the Kali VM)  
- **Destination:** 209.22.190.18 (the DNS server)

This is how the packet gets from one network to another. Layer 2 handles the local segment; Layer 3 handles routing beyond it.

---

### Layer 4 — User Datagram Protocol (Transport)
Transport layer. Protocol and port.  
- **Destination Port: 53** — DNS port, confirming this is a DNS query  
- **Protocol: UDP** — no handshake, no guaranteed delivery, fast

Port 53 is the identifier that tells the DNS server this packet is for it, not another service running on the same IP.

---

### Layer 7 — Domain Name System / Query (Application)
The actual content of the DNS query. The VM is asking: *"What is the IP address for google.com?"*

**Fields expanded:**

| Field | Value | What It Means |
|-------|-------|---------------|
| **Transaction ID** | 0x6dc4 | Unique ID matching this query to its response. The DNS server echoes this ID back so the machine knows which reply belongs to which question. |
| **Flags** | Standard query | Confirms this is a request, not a response. |
| **Questions** | 1 | One domain being looked up. |
| **Answer RRs** | 0 | No answers yet — this is the outgoing question, not the reply. |
| **Query Type** | AAAA | The VM is requesting Google's **IPv6** address. A second DNS query appeared requesting type **A** (IPv4). Both are normal — the system checks for IPv6 first, then falls back to IPv4 if needed. |

---

## SOC Analyst View — What to Look For in DNS Traffic

This layer breakdown is exactly what an analyst examines when investigating suspicious DNS activity.

**Red flags to look for:**

| Indicator | What It Suggests |
|-----------|-----------------|
| Domain names that are long, random-looking strings | DNS tunnelling — data encoded in the subdomain |
| Unusually high query volume from one host | Beaconing or DGA malware checking in with C2 |
| Queries to domains never seen before (first-seen domains) | Newly registered malicious domain |
| Large DNS response sizes | Data being exfiltrated in DNS replies |
| DNS queries on non-standard ports | Attacker trying to evade DNS-specific monitoring |
| Mismatched Transaction IDs | DNS spoofing/cache poisoning attempt |

**Normal traffic** looks exactly like the capture above — a short query, predictable destination port, small payload, matching Transaction ID in the reply.

---

## Display Filter Reference

Wireshark display filters allow analysts to isolate specific traffic types during triage.

| Filter | What It Shows |
|--------|--------------|
| `dns` | All DNS traffic |
| `icmp` | All ICMP traffic (ping, error messages) |
| `arp` | All ARP traffic |
| `tcp` | All TCP connections |
| `udp` | All UDP traffic |
| `ip.addr == 10.0.2.15` | All traffic to or from a specific IP |
| `tcp.port == 443` | HTTPS traffic only |
| `tcp.port == 3389` | RDP traffic — flag if exposed to internet |
| `dns.qry.name contains "google"` | DNS queries containing a specific string |
| `frame.len > 1000` | Large packets — worth reviewing in DNS context |

---

## Key Takeaways

- A single `ping` generates three protocol types (ARP → DNS → ICMP) before a single byte of ping data leaves the machine — this shows how layered network communication actually is
- Wireshark exposes every OSI layer in a single click — analysts use this to move from "there's traffic" to "here's exactly what it is and where it's going"
- DNS is the most analyst-relevant protocol in this capture — it's the protocol most commonly abused for covert communication, and Wireshark gives full visibility into query content
- Transaction IDs, query types, and domain names are the specific fields that distinguish legitimate DNS from malicious DNS
- Wireshark and Sysmon are complementary: Sysmon (Event ID 3) tells you *which process* made the connection; Wireshark tells you *what was in it*
