# Lab 04 — DHCP + DNS

## 🎯 Objectives

- Configure a Cisco router as a DHCP server
- Create DHCP pools for multiple subnets
- Exclude static IP addresses from DHCP pools
- Configure a DHCP relay agent (ip helper-address) for remote subnets
- Set up a DNS server and verify name resolution

---

## 🖧 Topology

```
  PC1              PC2                PC3              PC4
  (DHCP)           (DHCP)             (DHCP)           (DHCP)
  VLAN 10          VLAN 10            VLAN 20           VLAN 20
  │                │                  │                 │
  Fa0/1            Fa0/2              Fa0/3             Fa0/4
  └────────────────┘                  └─────────────────┘
        SW1                                  SW1
        Fa0/24 ──────────────────────── Fa0/23
                         │
                      R1 (2911) ← DHCP Server
                      Fa0/0.10 → 192.168.10.1/24
                      Fa0/0.20 → 192.168.20.1/24
                         │
                      Fa0/1 → 10.0.0.1/30
                         │
                    DNS Server
                    10.0.0.2/30
```

| Device     | Role        | IP              |
|------------|-------------|-----------------|
| R1         | DHCP Server | 192.168.10.1 / 192.168.20.1 |
| DNS Server | DNS         | 10.0.0.2        |
| PC1–2      | DHCP Client | 192.168.10.x    |
| PC3–4      | DHCP Client | 192.168.20.x    |

---

## ⚙️ Step-by-Step Configuration

### R1 — Exclude Static/Reserved IPs

```ios
enable
configure terminal

! Exclude first 10 addresses in each pool (routers, servers, printers, etc.)
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10
```

### R1 — Create DHCP Pool for VLAN 10 (Sales)

```ios
ip dhcp pool SALES_POOL
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 10.0.0.2
 domain-name sales.lab.local
 lease 7
 exit
```

### R1 — Create DHCP Pool for VLAN 20 (IT)

```ios
ip dhcp pool IT_POOL
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 10.0.0.2
 domain-name it.lab.local
 lease 7
 exit
```

> **`lease` syntax:** `lease <days> [hours] [minutes]` — default is 1 day. Use `lease infinite` for static-like behavior in labs.

### R1 — Subinterfaces (from Lab 03, repeated for completeness)

```ios
interface fastEthernet 0/0
 no shutdown
 exit

interface fastEthernet 0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
 exit

interface fastEthernet 0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
 exit
```

### R1 — Interface to DNS Server

```ios
interface fastEthernet 0/1
 ip address 10.0.0.1 255.255.255.252
 no shutdown
 exit

end
wr
```

### DNS Server — Packet Tracer GUI Config

1. Click the **DNS Server** device → **Desktop** → **IP Configuration**
   - IP: `10.0.0.2`, Mask: `255.255.255.252`, GW: `10.0.0.1`

2. Click **Services** → **DNS**
   - Toggle DNS: **On**
   - Add A records:

| Name              | Type | Address       |
|-------------------|------|---------------|
| router.lab.local  | A    | 192.168.10.1  |
| dns.lab.local     | A    | 10.0.0.2      |

### PC1–PC4 — Enable DHCP

In Packet Tracer: **Desktop** → **IP Configuration** → select **DHCP**

Wait a few seconds — the PC will show an assigned IP from the correct pool.

---

## ✅ Verification Commands

```ios
! On R1:
show ip dhcp pool                   ! Pool names, utilization
show ip dhcp binding                ! List of assigned IPs + MAC addresses
show ip dhcp conflict               ! Any address conflicts detected
show ip dhcp server statistics      ! Discover/Offer/Request/Ack counters

! Confirm excluded addresses aren't handed out:
show running-config | include excluded
```

**Expected results:**
- PC1 and PC2 receive `192.168.10.11+` addresses
- PC3 and PC4 receive `192.168.20.11+` addresses
- All PCs show DNS server `10.0.0.2`

**DNS test from PC (Command Prompt):**
```
nslookup router.lab.local
ping router.lab.local
```

---

## 💡 DHCP Relay (ip helper-address)

If the DHCP server is on a **different subnet** than the clients (common in real networks), routers don't forward broadcasts by default. Fix it with:

```ios
! On the router interface facing the client subnet:
interface fastEthernet 0/0.10
 ip helper-address 10.0.0.5    ! IP of the remote DHCP server
 exit
```

This converts DHCP broadcast → unicast, forwarded to the DHCP server. Critical concept for Network+ and CCNA exams.
