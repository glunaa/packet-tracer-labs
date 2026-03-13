# Lab 07 — OSPF (Single-Area OSPFv2)

## 🎯 Objectives

- Understand OSPF as a link-state routing protocol
- Configure single-area OSPF (Area 0) on multiple routers
- Use wildcard masks to advertise networks
- Set router IDs manually for predictability
- Configure passive interfaces to suppress OSPF hellos on LAN-facing ports
- Observe DR/BDR election on multi-access segments
- Verify OSPF neighbor adjacencies and routing table

---

## 🖧 Topology

```
  PC1                  PC2                  PC3
  192.168.1.10         192.168.2.10         192.168.3.10
  GW: 192.168.1.1      GW: 192.168.2.1      GW: 192.168.3.1
  │                    │                    │
  Fa0/0                Fa0/0                Fa0/0
  R1 ─────────────────── R2 ─────────────────── R3
  1.1.1.1            2.2.2.2              3.3.3.3  ← Router IDs
  Fa0/1              Fa0/1  Fa0/2         Fa0/1
  10.0.12.1/30 ─── 10.0.12.2/30   10.0.23.1/30 ─── 10.0.23.2/30
```

| Device | Interface | IP Address         |
|--------|-----------|--------------------|
| R1     | Fa0/0     | 192.168.1.1/24     |
| R1     | Fa0/1     | 10.0.12.1/30       |
| R2     | Fa0/0     | 192.168.2.1/24     |
| R2     | Fa0/1     | 10.0.12.2/30       |
| R2     | Fa0/2     | 10.0.23.1/30       |
| R3     | Fa0/0     | 192.168.3.1/24     |
| R3     | Fa0/1     | 10.0.23.2/30       |

---

## ⚙️ Step-by-Step Configuration

### R1 — Interfaces

```ios
enable
configure terminal
hostname R1

interface fastEthernet 0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit

interface fastEthernet 0/1
 ip address 10.0.12.1 255.255.255.252
 no shutdown
 exit
```

### R1 — OSPF Configuration

```ios
router ospf 1
 router-id 1.1.1.1

 ! Wildcard mask = inverse of subnet mask
 ! /24 → wildcard 0.0.0.255
 ! /30 → wildcard 0.0.0.3
 network 192.168.1.0 0.0.0.255 area 0
 network 10.0.12.0 0.0.0.3 area 0

 ! No OSPF hellos toward PC1 — no neighbors there
 passive-interface fastEthernet 0/0

 exit

end
wr
```

### R2 — Interfaces

```ios
enable
configure terminal
hostname R2

interface fastEthernet 0/0
 ip address 192.168.2.1 255.255.255.0
 no shutdown
 exit

interface fastEthernet 0/1
 ip address 10.0.12.2 255.255.255.252
 no shutdown
 exit

interface fastEthernet 0/2
 ip address 10.0.23.1 255.255.255.252
 no shutdown
 exit
```

### R2 — OSPF Configuration

```ios
router ospf 1
 router-id 2.2.2.2
 network 192.168.2.0 0.0.0.255 area 0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
 passive-interface fastEthernet 0/0
 exit

end
wr
```

### R3 — Interfaces

```ios
enable
configure terminal
hostname R3

interface fastEthernet 0/0
 ip address 192.168.3.1 255.255.255.0
 no shutdown
 exit

interface fastEthernet 0/1
 ip address 10.0.23.2 255.255.255.252
 no shutdown
 exit
```

### R3 — OSPF Configuration

```ios
router ospf 1
 router-id 3.3.3.3
 network 192.168.3.0 0.0.0.255 area 0
 network 10.0.23.0 0.0.0.3 area 0
 passive-interface fastEthernet 0/0
 exit

end
wr
```

---

## ✅ Verification Commands

```ios
! Neighbor adjacency — must show FULL state
show ip ospf neighbor

! Routing table — O = OSPF learned routes
show ip route ospf
show ip route

! OSPF process details
show ip ospf
show ip ospf interface fastEthernet 0/1   ! Hello/dead timers, DR/BDR info

! Link State Database
show ip ospf database
```

**Expected `show ip route` on R1:**
```
O    192.168.2.0/24 [110/2] via 10.0.12.2, Fa0/1
O    192.168.3.0/24 [110/3] via 10.0.12.2, Fa0/1
C    192.168.1.0/24 is directly connected, Fa0/0
C    10.0.12.0/30 is directly connected, Fa0/1
```

**Expected pings from PC1:**
```
ping 192.168.2.10    ✅  (PC1 → PC2)
ping 192.168.3.10    ✅  (PC1 → PC3)
```

---

## 💡 OSPF Key Concepts Reference

| Concept | Detail |
|---------|--------|
| Protocol type | Link-state (builds full topology map) |
| Algorithm | Dijkstra's SPF |
| AD | 110 |
| Metric | Cost = 100 Mbps / interface bandwidth |
| Hello interval | 10s (point-to-point) |
| Dead interval | 40s (4x hello) |
| Area 0 | Backbone — all areas must connect to it |
| Passive interface | Advertises the network but sends no hellos |
| Wildcard mask | Inverse of subnet mask |

### OSPF Neighbor States (in order)
```
Down → Init → 2-Way → ExStart → Exchange → Loading → Full
```
Neighbors must reach **Full** state to exchange routes.

### Cost Calculation
```
Cost = Reference Bandwidth / Interface Bandwidth
Default reference = 100 Mbps

FastEthernet (100 Mbps) → Cost 1
Serial (1.544 Mbps)     → Cost 64
```

To adjust for GigE networks:
```ios
router ospf 1
 auto-cost reference-bandwidth 1000
```
