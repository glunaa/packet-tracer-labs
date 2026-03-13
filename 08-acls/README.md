# Lab 08 — Access Control Lists (ACLs)

## 🎯 Objectives

- Understand the difference between standard and extended ACLs
- Configure a standard ACL to filter traffic based on source IP
- Configure an extended ACL to filter traffic based on source, destination, and protocol
- Apply ACLs to router interfaces in the correct direction (inbound vs outbound)
- Verify permit/deny behavior with ping tests

---

## 🖧 Topology

```
  PC1 (Sales)          PC2 (IT)             Server
  192.168.10.10        192.168.20.10        192.168.30.10
  GW: 192.168.10.1     GW: 192.168.20.1     GW: 192.168.30.1
  │                    │                    │
  Fa0/0                Fa0/1                Fa0/2
  └────────────────────┴──── R1 (2911) ─────┘
                             │
                        Manages all
                        three subnets
                        via subinterfaces
                        or separate interfaces
```

| Device | Interface | IP Address       |
|--------|-----------|------------------|
| R1     | Fa0/0     | 192.168.10.1/24  |
| R1     | Fa0/1     | 192.168.20.1/24  |
| R1     | Fa0/2     | 192.168.30.1/24  |
| PC1    | NIC       | 192.168.10.10/24 |
| PC2    | NIC       | 192.168.20.10/24 |
| Server | NIC       | 192.168.30.10/24 |

---

## ⚙️ Step-by-Step Configuration

### R1 — Interface Setup

```ios
enable
configure terminal
hostname R1

interface fastEthernet 0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
 exit

interface fastEthernet 0/1
 ip address 192.168.20.1 255.255.255.0
 no shutdown
 exit

interface fastEthernet 0/2
 ip address 192.168.30.1 255.255.255.0
 no shutdown
 exit
```

---

### Part A — Standard ACL

**Goal:** Block PC1 (Sales - 192.168.10.0/24) from reaching the Server network. Allow IT (192.168.20.0/24) through.

> **Standard ACLs** filter by **source IP only**.
> **Placement rule:** Place standard ACLs as **close to the destination** as possible.

```ios
! ACL 10 — deny Sales, permit IT
ip access-list standard BLOCK_SALES
 deny   192.168.10.0 0.0.0.255
 permit 192.168.20.0 0.0.0.255
 exit

! Apply OUTBOUND on Fa0/2 (toward Server)
interface fastEthernet 0/2
 ip access-group BLOCK_SALES out
 exit
```

> **Note:** Every ACL has an **implicit deny any** at the end. Always add an explicit `permit` for traffic you want to allow, or everything else gets blocked.

**Test results:**
```
PC1 ping 192.168.30.10   ❌  (Sales blocked)
PC2 ping 192.168.30.10   ✅  (IT allowed)
```

---

### Part B — Extended ACL

**Goal:** Allow IT (192.168.20.0/24) to reach the Server via HTTP (port 80) only. Block all other traffic from IT to the Server. Sales remains fully blocked.

> **Extended ACLs** filter by **source IP, destination IP, protocol, and port**.
> **Placement rule:** Place extended ACLs as **close to the source** as possible.

First, remove the standard ACL:
```ios
interface fastEthernet 0/2
 no ip access-group BLOCK_SALES out
 exit

no ip access-list standard BLOCK_SALES
```

Now configure the extended ACL:
```ios
ip access-list extended SERVER_POLICY
 ! Deny Sales completely
 deny   ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255

 ! Allow IT HTTP to server only (TCP port 80)
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.30.10 eq 80

 ! Allow IT to ping server (ICMP)
 permit icmp 192.168.20.0 0.0.0.255 host 192.168.30.10

 ! Implicit deny all — everything else dropped
 exit

! Apply INBOUND on Fa0/1 (close to IT source)
interface fastEthernet 0/1
 ip access-group SERVER_POLICY in
 exit
```

**Test results:**
```
PC1 ping 192.168.30.10          ❌  (Sales denied)
PC2 ping 192.168.30.10          ✅  (ICMP permitted)
PC2 browser http://192.168.30.10 ✅  (HTTP port 80 permitted)
PC2 SSH to 192.168.30.10        ❌  (port 22 not permitted)
```

---

## ✅ Verification Commands

```ios
show access-lists                        ! All ACLs and hit counters
show access-lists SERVER_POLICY          ! Specific ACL
show ip interface fastEthernet 0/1       ! Shows which ACL is applied + direction
show running-config | include access     ! Quick config check
```

> Hit counters increment each time a rule matches. Run a ping, then `show access-lists` to confirm which rule is being hit.

---

## 💡 ACL Quick Reference

| Type | Filters On | Number Range | Placement |
|------|-----------|--------------|-----------|
| Standard | Source IP only | 1–99, 1300–1999 | Close to destination |
| Extended | Src, Dst, Protocol, Port | 100–199, 2000–2699 | Close to source |

### Common Protocol Keywords
```ios
permit tcp ...   ! TCP traffic
permit udp ...   ! UDP traffic
permit icmp ...  ! Ping
permit ip ...    ! All IP traffic
```

### Common Port Keywords
```ios
eq 80     ! HTTP
eq 443    ! HTTPS
eq 22     ! SSH
eq 23     ! Telnet
eq 53     ! DNS
eq 21     ! FTP
```

### Direction
```ios
ip access-group ACL_NAME in    ! Filter traffic entering the interface
ip access-group ACL_NAME out   ! Filter traffic leaving the interface
```

### Wildcard Mask Cheat Sheet
```
/24 → 0.0.0.255
/25 → 0.0.0.127
/26 → 0.0.0.63
/30 → 0.0.0.3
host 192.168.1.10  =  192.168.1.10 0.0.0.0
any                =  0.0.0.0 255.255.255.255
```
