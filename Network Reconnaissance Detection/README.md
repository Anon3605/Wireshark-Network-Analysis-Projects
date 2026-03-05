# Wireshark Network Reconnaissance Detection Analysis

> **Classification:** Network Reconnaissance Detection 
> **Capture Duration:** 00:09:10 | **Total Packets:** 4,819  
> **Capture Window:** 2026-03-05 18:51:02 — 2026-03-05 19:00:13

---

## Table of Contents

- [Capture File Properties](#step-01-capture-file-properties)
- [Protocol Hierarchy](#step-02-protocol-hierarchy)
- [Conversation Analysis](#step-03-conversation-analysis)
- [Time Configuration](#step-04-time-settings)
- [Display Filter Analysis](#step-05-display-filter-analysis)
- [Conclusion](#conclusion)

---

## Step 01: Capture File Properties

`Statistics` → `Capture File Properties`

| Property | Value |
|----------|-------|
| First Packet | 2026-03-05 18:51:02 |
| Last Packet | 2026-03-05 19:00:13 |
| Elapsed Time | 00:09:10 |
| Total Packets | 4,819 captured (100.0% displayed) |

---

## Step 02: Protocol Hierarchy

`Statistics` → `Protocol Hierarchy`

Primary transport layer: **IPv4** — 4,283 of 4,819 packets

| Protocol | Packet Count | % of Total |
|----------|-------------|------------|
| TCP | 4,059 | 84.2% |
| ARP | 597 | 12.4% |

---

## Step 03: Conversation Analysis

`Statistics` → `Conversations`

### IPv4 Hosts Identified

| IP Address | Classification |
|------------|----------------|
| 192.168.1.103 | Internal Host |
| 192.168.1.109 | Internal Host |
| 192.168.1.113 | Internal Host — **Victim** |
| 192.168.1.116 | Internal Host — **Attacker** |
| 192.168.1.254 | Internal Host (Gateway) |
| *(2 external addresses)* | External |

**Notable observation:** The highest packet volume is recorded between `192.168.1.113` and `192.168.1.216`, with symmetric bidirectional packet counts.

---

### TCP Conversation Patterns

A high volume of short-lived TCP connections originates from `192.168.1.116` targeting `192.168.1.113`:

**Initial probe connections (single-packet):**

| Source Port (Attacker) | Destination Port (Victim) | Packet Count |
|------------------------|--------------------------|--------------|
| 39640 | 1 | 1 |
| 39641 | 1 | 1 |
| 39642 | 1 | 1 |
| 39896 | 1 | 1 |
| 39897 | 1 | 1 |
| 39898 | 1 | 1 |
| 57214 | 1 | 1 |

**Sustained scan connections:**

| Source Port (Attacker) | Behavior |
|------------------------|----------|
| 41761 | Multiple packets — 60B sent / 54B reply (constant) |
| 41858 | Multiple packets — 60B sent / 54B reply (constant) |

Destination port range swept: **1 through 65,389**

---

### Host Role Identification

Based on TCP conversation patterns:

```
Attacker:  192.168.1.116
Victim:    192.168.1.113
```

---

## Step 04: Time Settings

`View` → `Time Display Format` → `UTC Date and Time of Day`

Enable delta time column for per-packet interval analysis. UTC format is recommended for accurate cross-timezone log correlation and incident timeline reconstruction.

---

## Step 05: Display Filter Analysis

### Filter Results Summary

| # | Filter Expression | Result |
|---|-------------------|--------|
| F01 | `ip.src==192.168.1.116 && tcp.flags.syn==1` | **2,008 packets** |
| F02 | `(ip.src==192.168.1.113) && (ip.dst==192.168.1.116) && (tcp.flags.ack==1 && tcp.flags.syn==1)` | **0 packets** |
| F03 | `(ip.src==192.168.1.113) && (ip.dst==192.168.1.116) && (tcp.flags.ack==1 && tcp.flags.reset==1)` | **2,012 packets** |
| F04 | `ip.src==192.168.1.116 && tcp.flags.ack==1` | **4 packets** |
| F06 | `eth.addr == 08:00:27:63:b0:05 && arp` | **531 packets** |

---

### TCP Handshake Analysis

A standard TCP three-way handshake proceeds as follows:

```
Client (Attacker)          Server (Victim)
      |                         |
      |-------- SYN ----------->|
      |<------- SYN-ACK --------|
      |-------- ACK ----------->|
      |     [Connection Est.]   |
```

**Observed behavior:**

```
Attacker (192.168.1.116)   Victim (192.168.1.113)
      |                         |
      |------ SYN (x2008) ----->|   F01: 2,008 SYN packets sent
      |<----- RST-ACK (x2012) --|   F03: 2,012 RST-ACK responses (ports closed)
      |                         |
      |  [No ACK from attacker] |   F02: 0 SYN-ACK responses received
      |  [No handshake complete]|   F04: Only 4 ACK packets total
```

**Interpretation:**

- The attacker sends 2,008 SYN packets across ports 1–65,389, indicating a systematic port scan.
- The victim responds with RST-ACK on all probed ports, confirming all target ports are closed.
- Zero SYN-ACK responses (F02) indicate no open ports were discovered on the victim.
- The attacker sends only 4 ACK packets total (F04), meaning the TCP handshake is never completed — consistent with a half-open (stealth) SYN scan, commonly associated with tools such as Nmap `-sS`.

---

### ARP Reconnaissance Analysis

**Attacker MAC Address:** `08:00:27:63:b0:05`

| Timeline | Protocol | Packet Count |
|----------|----------|-------------|
| 12:52:25 — 12:53:38 | ARP | 531 packets |
| 12:52:33 — 12:55:08 | TCP | *(scan activity)* |

ARP requests were broadcast to `192.168.1.254` (gateway) to resolve the MAC address of `192.168.1.113`. This ARP activity precedes the TCP scan by approximately 8 seconds, consistent with standard network host discovery prior to port enumeration.

---

## Conclusion

### Attack Profile

| Attribute | Detail |
|-----------|--------|
| Attacker IP | 192.168.1.116 |
| Attacker MAC | 08:00:27:63:b0:05 |
| Victim IP | 192.168.1.113 |
| Attack Type | Half-open SYN Port Scan (Stealth Scan) |
| Scan Range | TCP ports 1–65,389 |
| ARP Phase | 531 broadcast requests (host discovery) |
| Open Ports Found | None (all ports responded RST-ACK) |

---

### Attack Sequence

```
Phase 1 — ARP Discovery       [12:52:25 - 12:53:38]
  192.168.1.116 broadcasts ARP requests to identify 192.168.1.113

Phase 2 — TCP SYN Port Scan   [12:52:33 - 12:55:08]
  192.168.1.116 sends SYN packets to ports 1-65389 on 192.168.1.113
  192.168.1.113 responds with RST-ACK (all ports closed)
  192.168.1.116 does not complete the three-way handshake (half-open scan)
```

---

### Indicators of Compromise (IOC)

- Sequential SYN packets targeting a wide port range from a single source IP within a compressed time window.
- Absence of corresponding ACK packets following SYN responses (incomplete handshake).
- High-volume ARP broadcasts immediately preceding TCP scan activity.
- Constant packet size pattern: 60 bytes outbound / 54 bytes reply — characteristic of automated scanning tools.

---

### Detection Signatures

The following behaviors are indicative of a **Nmap SYN Scan (`-sS`)** or equivalent half-open reconnaissance tool:

- `tcp.flags.syn == 1 && !tcp.flags.ack` from a single source exceeding threshold volume
- Source port increments (39640, 39641, 39642...) indicating sequential socket allocation
- Victim RST-ACK volume proportional to attacker SYN volume with no established sessions