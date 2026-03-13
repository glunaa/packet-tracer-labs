# Lab 06 — Static Routing & Default Routes

## 🎯 Objectives

- Understand when static routes are used vs dynamic routing protocols
- Configure static routes between routers using `ip route`
- Configure a default route (gateway of last resort)
- Configure a floating static route for backup path redundancy
- Verify routing tables and end-to-end connectivity

---

## 🖧 Topology

```
  PC1                                               PC2
  10.1.1.10/24                                 10.3.3.10/24
  GW: 10.1.1.1                                GW: 10.3.3.1
  │                                                 │
  Fa0/0                                          Fa0/0
  R1                  R2                           R3
  10.1.1.1/24    10.2.2.1/30──10.2.2.2/30    10.3.3.1/24
  Fa0/1 ─────────────────────────────────── Fa0/1
  10.2.2.1/30                               10.2.2.2/30

                  Backup Link (Serial)
  R1 S0/0/0 ──────────────────────────── S0/0/0 R3
  10.4.4.1/30                            10.4.4.2/30
```

| Device | Interface | IP Address      |
|--------|-----------|-----------------|
| R1     | Fa0/0     | 10.1.1.1/24     |
| R1     | Fa0/1     | 10.2.2.1/30     |
| R1     | S0/0/0    | 10.4.4.1/30     |
| R2     | Fa0/0     | 10.2.2.2/30     |
| R2     | Fa0/1     | 10.2.2.5/30     |
| R3     | Fa0/0     | 10.3.3.1/24     |
| R3     | Fa0/1     | 10.2.2.6/30     |
| R3     | S0/0/0    | 10.4.4.2/30     |
| PC1    | NIC       | 10.1.1.10/24    |
| PC2    | NIC       | 10.3.3.10/24    |

---

## ⚙️ Step-by-Step Configuration

### R1 — Interface Setup

```ios
enable
configure terminal
hostname R1

interface fastEthernet 0/0
 ip address 10.1.1.1 255.255.255.0
 no shutdown
 exit

interface fastEthernet 0/1
 ip address 10.2.2.1 255.255.255.252
 no shutdown
 exit

interface serial 0/0/0
 ip address 10.4.4.1 255.255.255.252
 clock rate 64000
 no shutdown
 exit
```

### R1 — Static Routes

```ios
! Route to PC2's network via R2 (primary)
ip route 10.3.3.0 255.255.255.0 10.2.2.2

! Default route — send all unknown traffic to R2
ip route 0.0.0.0 0.0.0.0 10.2.2.2

! Floating static route (backup via serial, AD = 5)
! Only kicks in if primary route goes down
ip route 10.3.3.0 255.255.255.0 10.4.4.2 5

end
wr
```

> **Administrative Distance (AD):** Lower = more preferred. Static default = 1. Setting AD to 5 on the backup means it's only used when the primary (AD 1) is gone. This is a **floating static route**.

### R2 — Interface Setup + Static Routes

```ios
enable
configure terminal
hostname R2

interface fastEthernet 0/0
 ip address 10.2.2.2 255.255.255.252
 no shutdown
 exit

interface fastEthernet 0/1
 ip address 10.2.2.5 255.255.255.252
 no shutdown
 exit

! Route to PC1's network
ip route 10.1.1.0 255.255.255.0 10.2.2.1

! Route to PC2's network
ip route 10.3.3.0 255.255.255.0 10.2.2.6

end
wr
```

### R3 — Interface Setup + Static Routes

```ios
enable
configure terminal
hostname R3

interface fastEthernet 0/0
 ip address 10.3.3.1 255.255.255.0
 no shutdown
 exit

interface fastEthernet 0/1
 ip address 10.2.2.6 255.255.255.252
 no shutdown
 exit

interface serial 0/0/0
 ip address 10.4.4.2 255.255.255.252
 no shutdown
 exit

! Route to PC1's network via R2 (primary)
ip route 10.1.1.0 255.255.255.0 10.2.2.5

! Default route
ip route 0.0.0.0 0.0.0.0 10.2.2.5

! Floating static backup via serial
ip route 10.1.1.0 255.255.255.0 10.4.4.1 5

end
wr
```

---

## ✅ Verification Commands

```ios
show ip route                        ! Full routing table
show ip route static                 ! Only static routes
show ip route 10.3.3.0               ! Details for a specific network
show ip interface brief              ! Interface status
traceroute 10.3.3.10                 ! Shows hop path from R1
```

**From PC1:**
```
ping 10.3.3.10          ✅  (PC1 → PC2 via R1 → R2 → R3)
tracert 10.3.3.10           (shows each hop)
```

**Failover test:**
1. On R1: `interface fa0/1` → `shutdown`
2. Run `show ip route` — primary route disappears
3. Floating static (via serial) takes over automatically
4. `ping 10.3.3.10` should still succeed

---

## 💡 Static Route Quick Reference

| Command | Purpose |
|---------|---------|
| `ip route <network> <mask> <next-hop>` | Standard static route |
| `ip route 0.0.0.0 0.0.0.0 <next-hop>` | Default route |
| `ip route <network> <mask> <next-hop> <AD>` | Floating static (backup) |
| `no ip route <network> <mask> <next-hop>` | Remove a static route |

**When to use static routes:**
- Small networks with few routers
- Stub networks (one path in/out)
- Backup/floating routes
- Default route to ISP
