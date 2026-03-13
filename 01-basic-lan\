# Lab 01 — Basic LAN + Switch Configuration

## 🎯 Objectives

- Set hostnames and secure console/VTY/enable passwords
- Configure SSH for remote management
- Assign IP addresses to PCs and test connectivity with `ping`
- Enable port security to restrict MAC addresses per port
- Save configuration to NVRAM

---

## 🖧 Topology

```
  PC1              PC2              PC3
  │                │                │
  Fa0/1            Fa0/2            Fa0/3
  │                │                │
  └────────────────┴────────────────┘
               SW1 (2960)
               Vlan 1: 192.168.1.1/24
               │
               Fa0/24 (uplink / management)
```

| Device | Interface   | IP Address      | Subnet Mask     |
|--------|-------------|-----------------|-----------------|
| SW1    | VLAN 1 SVI  | 192.168.1.1     | 255.255.255.0   |
| PC1    | NIC         | 192.168.1.10    | 255.255.255.0   |
| PC2    | NIC         | 192.168.1.11    | 255.255.255.0   |
| PC3    | NIC         | 192.168.1.12    | 255.255.255.0   |

Default gateway for all PCs: `192.168.1.1`

---

## ⚙️ Step-by-Step Configuration

### 1. Initial Switch Setup

```ios
enable
configure terminal

! Set hostname
hostname SW1

! Disable DNS lookup (prevents typo delays)
no ip domain-lookup

! Set privileged EXEC password (encrypted)
enable secret cisco123

! Secure console port
line console 0
 password console123
 login
 logging synchronous
 exit

! Secure VTY lines (Telnet/SSH)
line vty 0 15
 password vty123
 login
 transport input ssh
 exit

! Encrypt all plaintext passwords
service password-encryption

! Set MOTD banner
banner motd # Authorized Access Only — SW1 #
```

### 2. Configure Management IP (SVI)

```ios
interface vlan 1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit
```

### 3. Configure SSH

```ios
! SSH requires a domain name and RSA key
ip domain-name lab.local
crypto key generate rsa
! When prompted: enter 1024

username admin privilege 15 secret admin123

line vty 0 15
 login local
 transport input ssh
 exit

ip ssh version 2
```

### 4. Configure Port Security on Fa0/1

```ios
interface fastEthernet 0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 exit
```

> **Violation modes:**
> - `protect` — drops frames, no log
> - `restrict` — drops frames, logs/increments counter
> - `shutdown` — shuts port down (default)

### 5. Save Configuration

```ios
end
copy running-config startup-config
! or shorthand:
wr
```

---

## ✅ Verification Commands

```ios
show running-config
show interfaces vlan 1
show ip interface brief
show port-security interface fa0/1
show port-security address
show ssh
show version
```

From PC1:
```
ping 192.168.1.11     ! PC1 → PC2
ping 192.168.1.1      ! PC1 → Switch SVI
```
