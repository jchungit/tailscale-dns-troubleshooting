# Tailscale MagicDNS Troubleshooting — LinkedIn & Twitter Inaccessible on Public WiFi

**Author:** Jinho (Kai) Chung  
**Date:** May 11, 2026  
**Environment:** MacBook Air M1 · Tailscale · Ubiquiti Cloud Gateway Fiber · AT&T Fiber

---

## Problem Statement

While connected to Tailscale (VPN tunnel back to home network) on public WiFi (coffee shop), LinkedIn and Twitter/X would intermittently fail to load — despite both sites working perfectly at home without Tailscale active.

---

## Network Environment

| Component | Details |
|---|---|
| Device | MacBook Air M1 (8GB RAM, 256GB) |
| VPN | Tailscale (MagicDNS enabled) |
| Home Router | Ubiquiti Cloud Gateway Fiber (UCG Fiber) |
| Access Point | UniFi U7 Pro (WiFi 7 Tri-Band) |
| ISP | AT&T Fiber |
| Location during issue | Public WiFi (coffee shop) |

### Home Network Layout (VLANs)
| Network | Subnet | Purpose |
|---|---|---|
| Primary LAN | 192.168.0.0/24 | Main devices |
| Homelab | 192.168.20.0/24 | VirtualBox VMs |
| IoT | 192.168.30.0/24 | IoT devices |

---

## Diagnostic Steps

### Step 1 — Verify Actual Exit IP
```bash
# Visited whatismyip.com while Tailscale was active
```
**Result:** [REDACTED — AT&T relay IP]

**Finding:** Traffic was routing through the Tailscale tunnel correctly but exiting via an AT&T lightspeed relay node, not directly through the home residential IP.

---

### Step 2 — MTU (Maximum Transmission Unit) Test
```bash
ping -D -s 1400 www.linkedin.com
```
**Result:**
```
PING www.linkedin.com (198.51.100.91): 1400 data bytes
ping: sendto: Message too long
Request timeout for icmp_seq 0
ping: sendto: Message too long
Request timeout for icmp_seq 1
```
**Finding:** MTU mismatch confirmed. 1400 byte packets are too large to traverse the Tailscale tunnel. VPN tunnels add overhead reducing effective MTU from the standard 1500 bytes.

---

### Step 3 — DNS (Domain Name System) Lookup
```bash
nslookup linkedin.com
```
**Result:**
```
Server:     100.100.100.100
Address:    100.100.100.100#53

Non-authoritative answer:
Name:   linkedin.com
Address: 198.51.100.91
```
**Finding:** Critical issue found. Tailscale MagicDNS (100.100.100.100) returned 198.51.100.91 for linkedin.com. This IP falls in the RFC 5737 TEST-NET-2 reserved documentation range and is non-routable on the public internet.

---

### Step 4 — Traceroute
```bash
traceroute linkedin.com
```
**Result:**
```
traceroute to linkedin.com (198.51.100.91), 64 hops max, 40 byte packets
 1  [REDACTED].lightspeed.[region].sbcglobal.net ([REDACTED])  ~19 ms
 2  192.168.x.x (private gateway)  ~26 ms
 3  [REDACTED].lightspeed.[region].sbcglobal.net ([REDACTED])  ~29 ms
 4  * * *
 5  * *
```
**Finding:** Traffic correctly routes through the home ISP network (hops 1-3 confirm AT&T infrastructure) but dies at hop 4-5 trying to reach the non-routable documentation IP returned by MagicDNS.

---

## Root Cause Analysis

### Issue 1 — Tailscale MagicDNS Serving Bad Cached IP (Primary Cause)
When Tailscale is active, it installs its own DNS resolver at 100.100.100.100 and overrides all other DNS settings on the device, including DHCP from the home router. MagicDNS had a stale or corrupted cache entry returning a non-routable documentation IP for LinkedIn.

```
With Tailscale ON (broken state):
MacBook → Tailscale intercepts DNS → 100.100.100.100 → bad IP 198.51.100.91 FAIL

Without Tailscale:
MacBook → Router DHCP → 1.1.1.1 Cloudflare → real LinkedIn IP SUCCESS
```

### Issue 2 — MTU Mismatch (Secondary Cause)
VPN tunnels add encryption overhead reducing the effective packet size. Standard Ethernet MTU is 1500 bytes but Tailscale tunnel requires approximately 1280 bytes. Packets at 1400 bytes were dropped intermittently causing instability on heavy sites like LinkedIn.

---

## Solution

### Fix 1 — Configure Tailscale Global Nameservers (Permanent DNS Fix)

1. Navigate to https://login.tailscale.com/admin/dns
2. Under Global Nameservers, add 8.8.8.8 (Google Public DNS) and 1.1.1.1 (Cloudflare Public DNS)
3. Enable Override DNS servers toggle
4. Reconnect Tailscale on all devices

Result: MagicDNS (100.100.100.100) now forwards upstream to Google/Cloudflare instead of serving its own cached entries, ensuring accurate DNS resolution on any network.

### Fix 2 — Lower MTU on Tailscale Interface
```bash
sudo ifconfig utun0 mtu 1280
```
Run ifconfig first to confirm the correct Tailscale interface name.

### Fix 3 — UniFi Router DNS (Supporting Fix)
UniFi Network App > Networks > [LAN Name] > IPv4 > DHCP > DHCP Name Server > Manual
- DNS Server 1: 1.1.1.1
- DNS Server 2: 8.8.8.8

---

## Verification

After applying Fix 1, nslookup linkedin.com returned a real Microsoft/LinkedIn IP. LinkedIn and Twitter/X both loaded successfully through Tailscale on public WiFi.

```bash
nslookup linkedin.com
```
```
Server:     100.100.100.100
Address:    100.100.100.100#53

Non-authoritative answer:
Name:   linkedin.com
Address: 150.171.22.12    # Real Microsoft/LinkedIn IP
```

---

Add full Tailscale MagicDNS troubleshooting documentation (redacted)
- VPN tools override device DNS — router-level DNS settings do not apply while a VPN is active
- MagicDNS caching can serve stale or corrupted entries — always configure reliable upstream nameservers
- RFC 5737 documentation IPs (192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24) are non-routable — seeing one in nslookup is an immediate red flag
- MTU mismatch is a common VPN issue — tunnel overhead reduces effective packet size from 1500 to ~1280 bytes
- Systematic diagnosis (whatismyip → ping MTU → nslookup → traceroute) isolates network issues layer by layer

---

## Tools Used

| Tool | Purpose |
|---|---|
| whatismyip.com | Verify actual exit IP through tunnel |
| ping -D -s 1400 | Test MTU packet size limits |
| nslookup | Verify DNS resolution |
| traceroute | Trace packet path and find where traffic dies |
| Tailscale Admin Console | Configure global DNS nameservers |
| UniFi Network App | Configure router-level DHCP DNS |

---

## References
- Tailscale MagicDNS Documentation: https://tailscale.com/kb/1081/magicdns
- Tailscale DNS Configuration: https://tailscale.com/kb/1054/dns
- RFC 5737 IPv4 Address Blocks Reserved for Documentation: https://datatracker.ietf.org/doc/html/rfc5737
- Ubiquiti UniFi Documentation: https://help.ui.com
