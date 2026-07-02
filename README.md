# End of the Experiment — Enterprise Network Design
### CSE421 — Computer Networks | Group 1812 | BRAC University

A fully functional enterprise network designed and simulated in **Cisco Packet Tracer**, modeling a multi-site organization (inspired by The Big Bang Theory's Caltech campus) with six distinct subnets, three interconnected routers, dynamic and static routing, DHCP, DNS, and email services.

The network was designed from scratch: starting from a single base address `172.16.0.0/16` and using **Variable Length Subnet Masking (VLSM)** to efficiently allocate IP ranges to each site based on its host requirements — no wasted addresses.

---

## Group Members

| Student ID | Name |
|---|---|
| 22101311 | Md. Tanvirul Islam Rifat |
| 24341270 | Sartiz Alam Ayon |
| 22101472 | Syed Mahbubun Nabi |

---

## Network Overview

| Site | Description | Hosts | Subnet Assigned |
|---|---|---|---|
| Sheldon's Lab | Theoretical Physics Lab | 812 | 172.16.0.0/22 |
| Leonard's Lab | Experimental Physics Lab | 360 | 172.16.4.0/23 |
| Raj's Lab | Particle Astrophysics Center | 275 | 172.16.6.0/23 |
| Howard's Lab | JPL Engineering Bay | 190 | 172.16.8.0/24 |
| Penny's Network | Cheesecake Factory | 58 | 172.16.9.0/26 |
| Pasadena Apartment | Apartment Committee | 60 | 172.16.9.64/26 |
| WAN Link RB↔RC | Router serial link | 2 | 172.16.9.128/30 |
| WAN Link RB↔RA | Router serial link | 2 | 172.16.9.132/30 |
| WAN Link RA↔RC | Router serial link | 2 | 172.16.9.136/30 |

---

## Network Topology

The three routers form a **triangular mesh topology** — every router is directly connected to both others via serial WAN links. This provides redundancy: if any single link fails, traffic automatically reroutes through the remaining router.

```
        Router-B (RB)
        /            \
      RA  ─────────  RC
   Router-A        Router-C
```

- **Router-B** serves: Sheldon's Lab (812 hosts), Penny's Network (58 hosts)
- **Router-A** serves: Raj's Lab (275 hosts), Pasadena Apartment (60 hosts)
- **Router-C** serves: Leonard's Lab (360 hosts), Howard's Lab (190 hosts)

See `diagrams/network-topology.pdf` for the full labeled Packet Tracer topology diagram.

---

## VLSM Tree

The base network `172.16.0.0/16` was subdivided using VLSM — assigning the largest subnets first to minimize waste:

```
172.16.0.0/16  (root)
├── 172.16.0.0/22   → Sheldon's Lab     (1022 usable, need 812)
└── 172.16.4.0/22
    ├── 172.16.4.0/23  → Leonard's Lab  (510 usable, need 360)
    ├── 172.16.6.0/23  → Raj's Lab      (510 usable, need 275)
    └── 172.16.8.0/23
        ├── 172.16.8.0/24  → Howard's Lab          (254 usable, need 190)
        └── 172.16.9.0/24
            ├── 172.16.9.0/26    → Penny's Network  (62 usable, need 58)
            ├── 172.16.9.64/26   → Pasadena Apt     (62 usable, need 60)
            └── 172.16.9.128/26  → WAN /30 links (3 × serial connections)
```

---

## Router Address Table

### Router-A (Raj's Lab & Pasadena Apartment)

| Interface | IP Address | Subnet Mask | Description |
|---|---|---|---|
| G0/0/0 | 172.16.6.1 | 255.255.254.0 | Raj's Lab gateway |
| G0/0/1 | 172.16.9.65 | 255.255.255.192 | Apartment gateway |
| S0/1/0 | 172.16.9.134 | 255.255.255.252 | WAN link to Router-B |
| S0/1/1 | 172.16.9.137 | 255.255.255.252 | WAN link to Router-C |

### Router-B (Sheldon's Lab & Penny's Network)

| Interface | IP Address | Subnet Mask | Description |
|---|---|---|---|
| G0/0/0 | 172.16.0.1 | 255.255.252.0 | Sheldon's Lab gateway |
| G0/0/1 | 172.16.9.1 | 255.255.255.192 | Penny's gateway |
| S0/1/0 | 172.16.9.129 | 255.255.255.252 | WAN link to Router-C |
| S0/1/1 | 172.16.9.133 | 255.255.255.252 | WAN link to Router-A |

### Router-C (Leonard's Lab & Howard's Lab)

| Interface | IP Address | Subnet Mask | Description |
|---|---|---|---|
| G0/0/0 | 172.16.4.1 | 255.255.254.0 | Leonard's Lab gateway |
| G0/0/1 | 172.16.8.1 | 255.255.255.0 | Howard's Lab gateway |
| S0/1/0 | 172.16.9.138 | 255.255.255.252 | WAN link to Router-A |
| S0/1/1 | 172.16.9.130 | 255.255.255.252 | WAN link to Router-B |

---

## Routing Strategy

A hybrid routing approach was used — dynamic routing for most subnets, static routing for specific cases where manual control was needed.

### Dynamic Routing — RIPv2
All three routers run **RIPv2** with `no auto-summary` to propagate routes automatically across the mesh. RIPv2 was chosen over RIPv1 because it supports VLSM (classless routing), which is required for the /22, /23, /24, /26, and /30 subnets in this design.

```
router rip
 version 2
 no auto-summary
 network 172.16.0.0
```

### Static Routing
Penny's and Leonard's subnets are reachable via static routes on specific routers where RIPv2 alone was insufficient:

```
# On Router-A: reach Leonard's Lab via Router-C
ip route 172.16.4.0 255.255.254.0 172.16.9.138

# On Router-A: reach Penny's Lab via Router-B
ip route 172.16.9.0 255.255.255.192 172.16.9.133

# On Router-B: reach Leonard's Lab via Router-C
ip route 172.16.4.0 255.255.254.0 172.16.9.130

# On Router-C: reach Penny's Lab via Router-B
ip route 172.16.9.0 255.255.255.192 172.16.9.129
```

### Floating Static Route (Backup Route)
A floating static route with **administrative distance 25** (higher than RIPv2's default of 120, lower priority but always present as backup) provides failover for Raj's Lab if the primary RIPv2 route goes down:

```
# On Router-A: backup route to Raj's Lab via Router-C
ip route 172.16.6.0 255.255.254.0 172.16.9.138 25
```

---

## Services Configured

### DHCP
Each router acts as a DHCP server for its directly connected subnets. All DHCP pools point to the central DNS server at `172.16.9.66` (Apartment Server):

| Pool Name | Network | Default Gateway | DNS Server |
|---|---|---|---|
| SHELDON | 172.16.0.0/22 | 172.16.0.1 | 172.16.9.66 |
| PENNY | 172.16.9.0/26 | 172.16.9.1 | 172.16.9.66 |
| RAJ | 172.16.6.0/23 | 172.16.6.1 | 172.16.9.66 |
| LEONARD | 172.16.4.0/23 | 172.16.4.1 | 172.16.9.66 |
| HOWARD | 172.16.8.0/24 | 172.16.8.1 | 172.16.9.66 |

### DNS Server — Apartment Server (172.16.9.66)
A single DNS server handles name resolution for the entire network:

| Domain | Resolves To | Service |
|---|---|---|
| sheldon.com | 172.16.0.2 | Sheldon's Email Server |
| raj.com | 172.16.6.2 | Raj's Email Server |
| www.caltechsupport.com | 172.16.9.66 | Web Server (self) |

### Web Server — Apartment Server (172.16.9.66)
HTTP enabled. Displays: *"Bazinga! We've got your back!"*

### Email Servers
| Server | Domain | IP | Credentials |
|---|---|---|---|
| Sheldon's Email Server | sheldon.com | 172.16.0.2 | user: sheldon / pass: 74328 |
| Raj's Email Server | raj.com | 172.16.6.2 | user: raj / pass: 74328 |

---

## Files in This Repository

```
cse421-enterprise-network/
├── network-simulation.pkt        # Cisco Packet Tracer simulation file
│                                  (open with Packet Tracer 8.x or later)
├── project-report.pdf            # Full project report with VLSM tree,
│                                  address table, and router configurations
├── diagrams/
│   └── network-topology.pdf      # Labeled topology diagram showing all
│                                  devices, subnets, and connections
└── README.md
```

> **To open the simulation:** Download `network-simulation.pkt` and open it in Cisco Packet Tracer (free download from Cisco NetAcad). All router configurations, DHCP pools, DNS records, and server settings are pre-configured.

---

## Key Design Decisions

**Why /30 for WAN links?**
Router-to-router serial links only need 2 usable host addresses (one per router interface). A /30 subnet provides exactly 2 usable IPs — using anything larger would waste addresses in a VLSM-optimized design.

**Why a triangular mesh?**
A star topology (all routers connecting to a central hub) creates a single point of failure. The triangular mesh means any single link failure still leaves two working paths between all sites, and RIPv2 + floating static routes handle the automatic failover.

**Why RIPv2 over static routing everywhere?**
Static routes on 3 routers for 6+ subnets means 18+ route entries, all maintained manually. RIPv2 propagates route updates automatically — adding a new subnet only requires one `network` statement per router. The specific static overrides (Leonard's and Penny's) were kept manual because those subnets had specific routing behavior requirements.

---

## Technical Architecture

- **Simulation Tool:** Cisco Packet Tracer 8.x
- **Routing Protocols:** RIPv2 (dynamic) + Static routes + Floating static (failover)
- **IP Addressing:** IPv4 with VLSM (Variable Length Subnet Masking)
- **Services:** DHCP, DNS, HTTP, SMTP (Email)
- **Topology:** Triangular mesh (3 routers) with star sub-topologies per site

## Core Engineering Practices Demonstrated

- **VLSM-based IP Planning:** The entire `172.16.0.0/16` address space was subdivided top-down, largest subnet first, with no overlapping ranges and minimal waste — each subnet sized to exactly the next power-of-2 above the host requirement
- **Hybrid Routing Architecture:** RIPv2 handles dynamic convergence across the mesh while manual static overrides ensure specific paths for subnets that need deterministic routing behavior
- **Fault-Tolerant Topology:** The triangular mesh with floating static routes ensures the network survives any single link failure — traffic automatically reroutes without manual intervention
- **Centralized Service Architecture:** A single DNS server at `172.16.9.66` serves all six subnets across three routers, demonstrating how shared infrastructure is reachable across a routed multi-subnet network
- **WAN Link Conservation:** /30 subnets on all serial links conserve the address space while providing the minimum required addressing for point-to-point connections

## Author

**Md. Tanvirul Islam Rifat**

* **GitHub:** [@tanvirul-islam-rifat](https://github.com/tanvirul-islam-rifat)
* **LinkedIn:** [Tanvirul Islam Rifat](https://www.linkedin.com/in/tanvirul-islam-rifat)
