# Lab 02 — VLANs & Trunking

## 🎯 Objectives

- Create VLANs and assign meaningful names
- Configure switch ports as access ports in specific VLANs
- Configure an 802.1Q trunk link between two switches
- Verify VLAN isolation (PCs in different VLANs cannot ping each other without a router)

---

## 🖧 Topology

```
  PC1 (Sales)      PC2 (Sales)      PC3 (IT)         PC4 (IT)
  VLAN 10          VLAN 10          VLAN 20           VLAN 20
  192.168.10.10    192.168.10.11    192.168.20.10     192.168.20.11
  │                │                │                 │
  Fa0/1            Fa0/2            Fa0/3             Fa0/4
  │                │                │                 │
  └────────────────┴──── SW1 ───────┴─────────────────┘
                         │
                      Gi0/1 (Trunk 802.1Q)
                         │
                      Gi0/1
                    SW2 (2960)
                    │         │
                  Fa0/1     Fa0/2
                    │         │
                  PC5         PC6
                VLAN 10     VLAN 20
              192.168.10.12  192.168.20.12
```

| Device | VLAN | IP Address      |
|--------|------|-----------------|
| PC1    | 10   | 192.168.10.10   |
| PC2    | 10   | 192.168.10.11   |
| PC3    | 20   | 192.168.20.10   |
| PC4    | 20   | 192.168.20.11   |
| PC5    | 10   | 192.168.10.12   |
| PC6    | 20   | 192.168.20.12   |

---

## ⚙️ Step-by-Step Configuration

### SW1 — Create VLANs

```ios
enable
configure terminal
hostname SW1

! Create VLANs in VLAN database
vlan 10
 name Sales
 exit

vlan 20
 name IT
 exit
```

### SW1 — Assign Access Ports

```ios
! PC1 — Sales
interface fastEthernet 0/1
 switchport mode access
 switchport access vlan 10
 exit

! PC2 — Sales
interface fastEthernet 0/2
 switchport mode access
 switchport access vlan 10
 exit

! PC3 — IT
interface fastEthernet 0/3
 switchport mode access
 switchport access vlan 20
 exit

! PC4 — IT
interface fastEthernet 0/4
 switchport mode access
 switchport access vlan 20
 exit
```

### SW1 — Configure Trunk to SW2

```ios
interface gigabitEthernet 0/1
 switchport mode trunk
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 10,20
 exit
```

> **Note:** On some 2960 models in Packet Tracer, `switchport trunk encapsulation dot1q` is not needed — the port defaults to 802.1Q.

### SW2 — Mirror VLANs + Access Ports + Trunk

```ios
enable
configure terminal
hostname SW2

vlan 10
 name Sales
 exit

vlan 20
 name IT
 exit

! PC5 — Sales
interface fastEthernet 0/1
 switchport mode access
 switchport access vlan 10
 exit

! PC6 — IT
interface fastEthernet 0/2
 switchport mode access
 switchport access vlan 20
 exit

! Trunk back to SW1
interface gigabitEthernet 0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 exit

end
wr
```

---

## ✅ Verification Commands

```ios
show vlan brief                        ! Confirm VLANs exist and ports assigned
show interfaces trunk                  ! Confirm trunk is up and VLANs allowed
show interfaces fa0/1 switchport       ! Check individual port mode
```

**Expected ping results:**
- PC1 → PC2 ✅ (same VLAN 10, same switch)
- PC1 → PC5 ✅ (same VLAN 10, across trunk)
- PC1 → PC3 ❌ (different VLANs — no router yet)
- PC1 → PC6 ❌ (different VLANs — no router yet)

> Inter-VLAN communication requires a router or Layer 3 switch — covered in Lab 03.
