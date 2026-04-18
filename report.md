# Detailed Findings Report
## Project: ARP Spoofing + MITM Attack
**Date:** April 2026  
**Author:** [Your Name]  
**Environment:** Controlled Home Lab  
**Classification:** Educational / Ethical Hacking  

---

## 1. Executive Summary

This report documents the findings of a Man-in-the-Middle (MITM) attack conducted using ARP Spoofing in a controlled home lab environment. The attack was successfully executed between an attacker machine (Kali Linux) and a vulnerable victim machine (Metasploitable 2). All network traffic from the victim was successfully intercepted, analyzed, and documented using Wireshark.

The findings confirm that unencrypted protocols such as HTTP and FTP expose sensitive data — including credentials and full web traffic — to any attacker positioned on the same network segment.

---

## 2. Lab Setup

| Item | Details |
|---|---|
| Date of Test | April 2026 |
| Attacker Machine | Kali Linux (192.168.1.119) |
| Victim Machine | Metasploitable 2 (192.168.1.3) |
| Gateway | 192.168.1.1 |
| Network | Host-Only Adapter (VirtualBox) |
| Tools Used | arpspoof, Wireshark, curl, ftp |

---

## 3. Attack Methodology

### 3.1 Phase 1 — Reconnaissance
Identified all IP addresses on the network:
- Attacker (Kali): `192.168.1.119`
- Victim (Metasploitable 2): `192.168.1.3`
- Gateway: `192.168.1.1`

Confirmed connectivity between all machines using `ping`.

### 3.2 Phase 2 — Pre-Attack Setup
Enabled IP forwarding on Kali Linux to ensure victim traffic is silently forwarded and not dropped:
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```
Started Wireshark capture on `eth0` interface before launching the attack.

### 3.3 Phase 3 — ARP Spoofing Execution
Launched two simultaneous arpspoof commands:

```bash
# Poisoning the victim — telling it Kali is the gateway
sudo arpspoof -i eth0 -t 192.168.1.3 192.168.1.1

# Poisoning the gateway — telling it Kali is the victim
sudo arpspoof -i eth0 -t 192.168.1.1 192.168.1.3
```

Both commands ran continuously, sending fake ARP replies every ~1 second.

### 3.4 Phase 4 — Traffic Generation
Generated the following traffic from the victim machine (Metasploitable 2):

| Command | Traffic Type |
|---|---|
| `curl http://neverssl.com` | HTTP web request |
| `ftp 192.168.1.119` | FTP with plaintext login |
| `ping 192.168.1.1` | ICMP traffic |

### 3.5 Phase 5 — Traffic Analysis
Applied the following Wireshark filters to analyze intercepted traffic:

| Filter | Purpose |
|---|---|
| `arp` | View fake ARP packets |
| `ip.src == 192.168.1.3` | All traffic from victim |
| `http` | Intercepted HTTP requests |
| `ftp` | Plaintext FTP credentials |
| `dns` | DNS queries from victim |
| `arp.duplicate-address-detected` | Detect spoofing |

---

## 4. Findings

### Finding 1 — ARP Cache Poisoning Successful
**Severity: High**

The ARP cache of both the victim and gateway were successfully poisoned. Wireshark confirmed duplicate IP addresses for both `192.168.1.1` and `192.168.1.3` — each appearing with the attacker's MAC address `08:00:27:fc:e7:ab`.

**Evidence:** Screenshot 01, Screenshot 06  
**Wireshark Warning:** `Duplicate IP address detected for 192.168.1.1` and `192.168.1.3`

---

### Finding 2 — HTTP Traffic Fully Intercepted
**Severity: High**

The victim's HTTP request (`GET / HTTP/1.1`) to external server `34.223.124.45` was fully intercepted. The destination MAC address in the Ethernet frame was the attacker's MAC — confirming Kali was in the middle of the communication.

**Evidence:** Screenshot 04  
**Wireshark Filter Used:** `http`

---

### Finding 3 — Full TCP Stream Reconstructed
**Severity: High**

Using Wireshark's **Follow TCP Stream** feature, the complete HTTP conversation was reconstructed — including full request headers, response headers, and the entire HTML body of the webpage. An attacker could read every word of unencrypted web content.

**Evidence:** TCP Stream screenshot  
**Action:** Right-click packet → Follow → TCP Stream

---

### Finding 4 — DNS Queries Intercepted
**Severity: Medium**

Every DNS query made by the victim passed through the attacker machine. This means the attacker could see every domain the victim was trying to visit — even before the connection was established.

**Evidence:** DNS filter in Wireshark  
**Wireshark Filter Used:** `dns and ip.src == 192.168.1.3`

---

### Finding 5 — FTP Plaintext Credentials Visible
**Severity: Critical**

FTP login credentials (username and password) were transmitted in plaintext and were fully visible in the Wireshark capture. This demonstrates the critical risk of using unencrypted protocols on any shared network.

**Evidence:** FTP filter in Wireshark  
**Wireshark Filter Used:** `ftp`

---

### Finding 6 — Attack Detectable by Blue Team
**Severity:** Informational

Wireshark's Expert Info system flagged the attack automatically with the warning:  
`Duplicate IP address configured (192.168.1.1)` — Severity: Warning  

This confirms that a trained SOC analyst monitoring network traffic with Wireshark or an IDS/IPS system with ARP anomaly detection rules would be able to detect this attack in real time.

**Evidence:** Screenshot 06  
**Wireshark Filter Used:** `arp.duplicate-address-detected`

---

## 5. Risk Summary

| Finding | Severity | Impact |
|---|---|---|
| ARP Cache Poisoning | High | Full traffic redirection |
| HTTP Traffic Intercepted | High | Web data exposed |
| TCP Stream Reconstructed | High | Full content readable |
| DNS Queries Intercepted | Medium | Browsing activity exposed |
| FTP Plaintext Credentials | Critical | Login credentials stolen |
| Attack Detection Possible | Informational | Defender can catch it |

---

## 6. Recommendations (Blue Team / Defence)

| Recommendation | How it Helps |
|---|---|
| Use HTTPS everywhere | Encrypts web traffic even if intercepted |
| Use SFTP instead of FTP | Encrypts file transfer credentials |
| Enable Dynamic ARP Inspection (DAI) | Blocks fake ARP replies on managed switches |
| Use a VPN | Encrypts all traffic end to end |
| Deploy IDS/IPS (Snort/Suricata) | Detects ARP anomalies automatically |
| Enable Static ARP entries | Prevents ARP cache from being poisoned |
| Use VLAN segmentation | Limits ARP broadcast domain |
| Monitor with Wireshark/Zeek | Real time ARP anomaly detection |

---

## 7. Conclusion

The ARP Spoofing + MITM attack was successfully executed in a controlled home lab environment. The attack confirms that on any unencrypted local network, an attacker with basic tools can silently intercept all traffic between a victim and the gateway — reading HTTP content, DNS queries, and plaintext credentials without the victim knowing.

This project provided hands-on experience with both offensive techniques (Red Team) and defensive detection methods (Blue Team) — reinforcing the importance of encryption, network monitoring, and proper security controls.

---

## 8. References

- Wireshark Documentation — wireshark.org/docs
- arpspoof man page — dsniff toolkit
- MITRE ATT&CK — T1557.002 ARP Cache Poisoning
- OWASP — Man-in-the-Middle Attack

---

*This report was created for educational purposes as part of a personal cybersecurity portfolio project. All testing was performed ethically on machines owned by the author in a controlled lab environment.*
