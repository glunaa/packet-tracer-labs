# Lab 05 — DNS & HTTP

## 🎯 Objectives

- Configure a DNS server to resolve hostnames to IP addresses
- Configure an HTTP server to host a simple web page
- Set DNS server address on client PCs
- Test name resolution using `nslookup` and a web browser
- Understand how DNS and HTTP work together in a client-server model

---

## 🖧 Topology

```
  PC1                   PC2
  192.168.1.10          192.168.1.11
  DNS: 192.168.1.2      DNS: 192.168.1.2
  │                     │
  Fa0/1                 Fa0/2
  │                     │
  └──────── SW1 ─────────┘
                │
              Fa0/3          Fa0/4
                │              │
           DNS Server      Web Server
           192.168.1.2     192.168.1.3
           (Server-PT)     (Server-PT)
```

| Device     | IP Address    | Role        |
|------------|---------------|-------------|
| PC1        | 192.168.1.10  | Client      |
| PC2        | 192.168.1.11  | Client      |
| DNS Server | 192.168.1.2   | DNS         |
| Web Server | 192.168.1.3   | HTTP        |
| SW1        | 192.168.1.1   | Switch SVI  |

---

## ⚙️ Step-by-Step Configuration

### SW1 — Basic Setup

```ios
enable
configure terminal
hostname SW1

interface vlan 1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit

end
wr
```

### DNS Server — Packet Tracer GUI

1. Click **DNS Server** → **Desktop** → **IP Configuration**
   - IP: `192.168.1.2`
   - Mask: `255.255.255.0`
   - Gateway: `192.168.1.1`

2. Click **Services** → **DNS**
   - Toggle DNS Service: **On**
   - Add the following A records:

| Name               | Type | Address       |
|--------------------|------|---------------|
| www.lab.local      | A    | 192.168.1.3   |
| webserver.lab.local| A    | 192.168.1.3   |
| dns.lab.local      | A    | 192.168.1.2   |

3. Click **Add** after each record

### Web Server — Packet Tracer GUI

1. Click **Web Server** → **Desktop** → **IP Configuration**
   - IP: `192.168.1.3`
   - Mask: `255.255.255.0`
   - Gateway: `192.168.1.1`

2. Click **Services** → **HTTP**
   - Toggle HTTP Service: **On**
   - Toggle HTTPS Service: **On** (optional)
   - Edit the default `index.html` page:

```html
<html>
  <head><title>Lab Web Server</title></head>
  <body>
    <h1>Welcome to the Lab Network</h1>
    <p>DNS and HTTP are working correctly.</p>
  </body>
</html>
```

### PC1 & PC2 — IP Configuration

**PC1 — Desktop → IP Configuration:**
- IP: `192.168.1.10`
- Mask: `255.255.255.0`
- Gateway: `192.168.1.1`
- DNS Server: `192.168.1.2`

**PC2 — Desktop → IP Configuration:**
- IP: `192.168.1.11`
- Mask: `255.255.255.0`
- Gateway: `192.168.1.1`
- DNS Server: `192.168.1.2`

---

## ✅ Verification

**From PC1 — Command Prompt:**
```
ping 192.168.1.3            ! Direct IP ping to web server
ping www.lab.local          ! Tests DNS resolution + connectivity
nslookup www.lab.local      ! Should return 192.168.1.3
nslookup dns.lab.local      ! Should return 192.168.1.2
```

**From PC1 — Web Browser:**
- Open **Desktop → Web Browser**
- Type `http://www.lab.local` → should load your custom HTML page
- Type `http://192.168.1.3` → should also load (direct IP)

**Expected results:**
- `nslookup www.lab.local` → `192.168.1.3` ✅
- Browser loads page via hostname ✅
- Browser loads page via IP ✅

---

## 💡 Key Concepts

| Concept | Detail |
|---------|--------|
| DNS A Record | Maps a hostname to an IPv4 address |
| DNS port | UDP/TCP 53 |
| HTTP port | TCP 80 |
| HTTPS port | TCP 443 |
| Resolution order | PC checks local cache → DNS server → returns IP |
| Without DNS | You can still reach server by IP — DNS is name resolution only |

**How a browser request works step by step:**
1. User types `www.lab.local` in browser
2. PC sends DNS query to `192.168.1.2` (UDP port 53)
3. DNS server responds with `192.168.1.3`
4. PC sends HTTP GET request to `192.168.1.3` (TCP port 80)
5. Web server responds with HTML page
6. Browser renders the page
