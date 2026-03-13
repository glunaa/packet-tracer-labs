# Lab 03 — Inter-VLAN Routing (Router-on-a-Stick)

## 🎯 Objectives

- Understand why inter-VLAN communication requires a Layer 3 device
- Configure a router with subinterfaces for each VLAN (router-on-a-stick)
- Configure 802.1Q encapsulation on each subinterface
- Set the router as the default gateway for each VLAN
- Verify that PCs across VLANs can now communicate

---

## 🖧 Topology

```
  PC1 (Sales)          PC2 (IT)
  VLAN 10              VLAN 20
  192.168.10.10        192.168.20.10
  GW: 192.168.10.1     GW: 192.168.20.1
  │                    │
  Fa0/1                Fa0/2
  │                    │
  └────────SW1 (2960)──┘
              │
           Fa0/24 (Trunk)
              │
           Fa0/0 (physical — no IP)
           R1 (2911)
           ├── Fa0/0.10 → 192.168.10.1/24 (VLAN 10 gateway)
           └── Fa0/0.20 → 192.168.20.1/24 (VLAN 20 gateway)
```

| Device | Interface      | IP Address      | VLAN |
|--------|----------------|-----------------|------|
| R1     | Fa0/0.10 (SIF) | 192.168.10.1    | 10   |
| R1     | Fa0/0.20 (SIF) | 192.168.20.1    | 20   |
| PC1    | NIC            | 192.168.10.10   | 10   |
| PC2    | NIC            | 192.168.20.10   | 20   |

---

## ⚙️ Step-by-Step Configuration

### SW1 — VLANs + Access Ports + Trunk to Router

```ios
enable
configure terminal
hostname SW1

! Create VLANs
vlan 10
 name Sales
 exit
vlan 20
 name IT
 exit

! PC1 access port
interface fastEthernet 0/1
 switchport mode access
 switchport access vlan 10
 exit

! PC2 access port
interface fastEthernet 0/2
 switchport mode access
 switchport access vlan 20
 exit

! Trunk to R1 — this carries tagged VLAN traffic to the router
interface fastEthernet 0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 exit

end
wr
```

### R1 — Subinterface Configuration (Router-on-a-Stick)

```ios
enable
configure terminal
hostname R1

! Bring up the physical interface (no IP assigned here)
interface fastEthernet 0/0
 no shutdown
 exit

! Subinterface for VLAN 10
interface fastEthernet 0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
 exit

! Subinterface for VLAN 20
interface fastEthernet 0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
 exit

end
wr
```

> **Key concept:** `encapsulation dot1Q <vlan-id>` tells the router which VLAN tag to strip/add on incoming/outgoing frames. The subinterface number (`.10`, `.20`) doesn't have to match the VLAN ID — but it's best practice to keep them the same.

### PC Configuration

Set these in Packet Tracer's Desktop → IP Configuration:

**PC1:**
- IP: `192.168.10.10`
- Mask: `255.255.255.0`
- Gateway: `192.168.10.1`

**PC2:**
- IP: `192.168.20.10`
- Mask: `255.255.255.0`
- Gateway: `192.168.20.1`

---

## ✅ Verification Commands

```ios
! On R1:
show ip interface brief           ! Confirm subinterfaces are up/up
show ip route                     ! Should show connected routes for .10 and .20 networks
show interfaces fa0/0.10          ! Subinterface details

! On SW1:
show interfaces trunk             ! Confirm Fa0/24 is trunking with VLANs 10, 20
show vlan brief
```

**Expected ping results from PC1:**
```
ping 192.168.10.1    ✅  (VLAN 10 gateway — R1 subinterface)
ping 192.168.20.1    ✅  (VLAN 20 gateway — R1 subinterface)
ping 192.168.20.10   ✅  (PC2 — inter-VLAN routing working!)
```

---

## 💡 Router-on-a-Stick vs Layer 3 Switch

| Method | Pros | Cons |
|--------|------|------|
| Router-on-a-Stick | Simple, uses existing router | Single link = bottleneck |
| L3 Switch (SVI) | Wire-speed routing, no bottleneck | More expensive hardware |

For production environments with high inter-VLAN traffic, a Layer 3 switch with SVIs is preferred.
