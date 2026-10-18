# From Frame to Forward: Building a Small Office Network

GoMyCode checkpoint. I designed a small office LAN with internet access and followed the data
from Ethernet frames up to IP routing and NAT.

**Deliverable:** [From-Frame-to-Forward.pptx](From-Frame-to-Forward.pptx) (18 slides, with diagrams)

---

## The network

| Equipment          | Qty |
|--------------------|-----|
| Router (internet)  | 1   |
| Switches           | 2   |
| Employee PCs       | 4   |
| Printer            | 1   |
| Wi-Fi access point | 1   |
| Wireless devices   | 2 (laptop + phone) |

```
                 Internet
                    |
                 Router (WAN 203.0.113.10)
              G0/1 /      \ G0/2
        Switch 1          Access point ~~~ Laptop, Phone
       /  |   \   \
    PC1  PC2 Printer Switch 2
                     /    \
                   PC3    PC4
```

---

## Part 1: MAC addresses and frames

**MAC addresses** (first half = maker, second half = device):

| Device | MAC |
|---|---|
| Router G0/0 / G0/1 / G0/2 | 00:1A:2B:10:00:01 / :02 / :03 |
| Switch 1 / Switch 2 | 00:1A:2B:20:00:01 / :02 |
| PC1 to PC4 | 00:1A:2B:30:00:01 to :04 |
| Printer | 00:1A:2B:40:00:01 |
| Access point | 00:1A:2B:50:00:01 |
| Laptop / Phone | 00:1A:2B:60:00:01 / :02 |

**Frame from PC1 to the printer:**

`Preamble (7) | SFD (1) | Dest MAC (6) | Source MAC (6) | Type (2) | Payload (46-1500) | FCS (4)`

- PC1 uses **ARP** to learn the printer's MAC.
- The switch reads the destination MAC and sends the frame only to the printer's port.
- The printer checks the **FCS**. A bad frame is dropped.

**Collisions:**
- **Ethernet:** CSMA/CD. Switches in full-duplex give each device its own collision domain, so in practice there are no collisions.
- **Wi-Fi:** CSMA/CA. Listen, wait a random time, send, wait for an ACK. RTS/CTS is optional.

---

## Part 2: IP addressing and subnetting

Private range **192.168.10.0/24**, split into /26 blocks (mask 255.255.255.192):

| Subnet | Network | Usable | Gateway |
|---|---|---|---|
| Wired | 192.168.10.0/26 | .1 to .62 | 192.168.10.1 |
| Wireless | 192.168.10.64/26 | .65 to .126 | 192.168.10.65 |
| Spare | .128/26 and .192/26 | for later | - |

| Device | IP | Assigned by |
|---|---|---|
| Router G0/1 / G0/2 / WAN | .1 / .65 / 203.0.113.10 | Static / from the ISP |
| Switch 1 / Switch 2 | .2 / .3 | Static |
| Printer | .5 | Static |
| PC1 to PC4 | .10 to .13 | DHCP |
| Access point | .66 | Static |
| Laptop / Phone | .70 / .71 | DHCP |

**DHCP** runs on the router and has two pools: wired .10 to .50 and wireless .70 to .120.
It hands out the IP, mask, gateway, DNS and lease time using four messages:
Discover, Offer, Request and Acknowledge. The infrastructure devices and the printer keep static IPs.

---

## Part 3: Routing and NAT

The laptop opens `www.example.com` (93.184.216.34):

1. **DNS** turns the name into an IP.
2. The mask shows the IP is **not local**, so the laptop sends the packet to the gateway 192.168.10.65.
3. **ARP** gets the gateway's MAC.
4. A **Wi-Fi frame** goes to the AP, which passes it to the router over Ethernet.
5. The **router** reads the IP header, lowers the TTL, recalculates the checksum, finds the next hop and applies NAT.
6. Internet routers repeat this until the packet reaches the server. The reply comes back the same way.

**Routing table:**

| Destination | Next hop | Port |
|---|---|---|
| 192.168.10.0/26 | connected | G0/1 |
| 192.168.10.64/26 | connected | G0/2 |
| 0.0.0.0/0 | 203.0.113.1 (ISP) | G0/0 |

The longest matching prefix wins. The default route is used when nothing else matches.

**NAT (PAT):** `192.168.10.70:51000` becomes `203.0.113.10:40001`. The router stores this in its
NAT table so it can translate the reply back. This lets the whole office share one public IP.

---

*Note: 203.0.113.0/24 is a documentation-only IP range used here as an example public address.*
