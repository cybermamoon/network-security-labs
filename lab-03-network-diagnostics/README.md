# Lab 03 — Network Diagnostics & Incident Analysis

**Scenario:** Students in a university computer lab report they can't access
the Learning Management System (LMS). It worked in the previous session, and
another department can still reach it. As a junior analyst: inspect one
affected workstation, test connectivity, trace the route, and submit an
evidence-based incident report.

**Approved configuration:**

| Component | Expected |
|---|---|
| Workstation | 192.168.10.20 |
| Subnet mask | 255.255.255.0 |
| Default gateway | 192.168.10.1 |
| DNS server | 192.168.20.5 |
| LMS server | 192.168.20.10 |

## Task 1 — Inspect the workstation (`ipconfig`)

| Component | Observed | Approved |
|---|---|---|
| IPv4 address | 192.168.100.7 | 192.168.10.20 |
| Subnet mask | 255.255.255.0 | 255.255.255.0 |
| Default gateway | 192.168.100.1 | 192.168.10.1 |
| DNS server | 192.168.100.1 | 192.168.20.5 |

Three of four values are wrong — the workstation sits on the wrong subnet
with the wrong gateway and DNS.

**Why does this workstation need a gateway to reach the LMS?**
The workstation (192.168.10.20) and the LMS server (192.168.20.10) are on
different subnets. Hosts on separate subnets can't communicate via Layer 2
(MAC addressing), so traffic to another network must go through a Layer 3
device — the default gateway.

**Does an unexpected gateway prove an attack?**
No. It's frequently manual misconfiguration, a misconfigured DHCP scope, or
a VLAN assignment error. Next checks: local adapter settings, DHCP lease
logs, switch port configs, and administrative change logs.

## Task 2 — Test connectivity (`ping`)

| Test destination | Replies | Packet loss | Avg time |
|---|---|---|---|
| Loopback (127.0.0.1) | 4 | 0% | 0ms |
| Default gateway | 0 | 100% | Request timed out |
| LMS server | 0 | 100% | Request timed out |

**What a successful loopback proves:** the local TCP/IP stack, protocol
drivers, and NIC internals work — the problem is off the host.

**Gateway replies but LMS doesn't — what remains?** Routing failures between
subnets, firewall rules blocking ICMP at the gateway or server, the LMS
server being down, or bad addressing/name resolution on the path.

**"Request timed out" vs "Destination host unreachable":** timed out means
packets left but no echo reply came back (firewall drops, routing black
holes). Unreachable means no valid route exists at all (ARP failure, missing
route, downed link).

**Does a successful ping prove the website works?** No — ping only checks
ICMP reachability at the network layer. It says nothing about the web
service on port 80/443.

## Task 3 — Trace the route (`tracert`)

`tracert -d -h 12 -w 1000 192.168.20.10`

- **First responding hop:** 192.168.100.1 (matches the workstation's configured gateway)
- **Destination reached:** No — timed out after hop 2
- **Asterisks:** intermediate hops not returning ICMP Time Exceeded, usually
  rate-limiting or firewall filtering — not proof forwarding stopped

**Can a trace alone prove interception?** No. Traceroute shows the path, not
what happens to the payload. Proving interception needs packet inspection,
certificate verification, or ARP/DHCP spoofing detection.

## Task 4 — Incident analysis

**Observed symptoms:** lab workstations can't reach the LMS while other
departments can. Loopback and gateway pings succeed; LMS pings show 100%
loss; traceroute dies before the destination subnet.

**Most likely explanation:** incorrect default gateway / subnet routing
misconfiguration on the affected workstation — inter-subnet traffic isn't
being directed toward the LMS network.

**Other possibilities:** temporary switch port shutdown or VLAN
misconfiguration on the lab's switch interface.

**Corrective actions:**
1. Reconfigure the adapter to the approved gateway (192.168.10.1) and DNS
   (192.168.20.5).
2. Flush the DNS cache (`ipconfig /flushdns`).

**Verify recovery:** `ipconfig /all` matches the approved schema →
`ping 192.168.20.10` gets replies → LMS login page loads in a browser.

**Core lesson:** a timeout alone proves nothing — not an attack, not a dead
server. ICMP filtering can hide a healthy host. Conclusions need converging
evidence from multiple tools.
