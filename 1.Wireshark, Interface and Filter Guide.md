Analysing packets is something, reading the interface efficiently is somehting else

This explains the Wireshark window: 
- what each section is for
- what the colors mean
- the fastest way to scan a capture without getting lost

---
# The Interface

## 1. The Four Main Sections

Wireshark's main window is split into four zones. Learning to move between them in the right order is most of what "reading Wireshark" means.

```
┌─────────────────────────────────────────────┐
│  Menu bar / Toolbar                          │
├─────────────────────────────────────────────┤
│  Filter bar (display filter)                 │
├─────────────────────────────────────────────┤
│  Packet List Pane                            │  ← 1. we scan here first
├─────────────────────────────────────────────┤
│  Packet Details Pane                         │  ← 2. we drill in here
├─────────────────────────────────────────────┤
│  Packet Bytes Pane                           │  ← 3. we verify raw bytes here
├─────────────────────────────────────────────┤
│  Status bar                                  │
└─────────────────────────────────────────────┘
```

### Filter bar
Where you type display filters (example: `http`, `ip.addr==10.0.0.5`, `tcp.port==443`)

- Green background = valid syntax
- Red = invalid

A raw capture is unreadable without filtering

### Packet List Pane (top)
One row per packet, columns typically: `No.`, `Time`, `Source`, `Destination`, `Protocol`, `Length`, `Info`.

- Columns can be deleted, added, edited, reordered, and more.
  
### Packet Details Pane (middle)
A collapsible tree showing the selected packet listed into protocol layers (Frame → Ethernet → IP → TCP/UDP → Application layer). 

Each layer can be expanded to see individual fields (flags, sequence numbers, headers, etc..)

### Packet Bytes Pane (bottom)
The raw hex and ASCII dump of the selected packet. As you click fields in the Details pane above, the corresponding bytes highlight here (this works both ways). 

- This is useful when you need to see exact byte values, and it can also be used, in some cases, to find the details you are searching for more easily, rather than openening each layer of the details pane (if somehting is written there and you suspect it to be what you are looking for, you just click on it)

### Status bar (very bottom)
Shows the current filter status, packet counts (displayed vs. total), and profile name, it tells whether we're looking at a filtered subset or the whole capture

---

## 2. What the Colors Mean

Wireshark applies **coloring rules** to the Packet List Pane automatically. These are the defaults (View → Coloring Rules to customize or inspect):

| Color | Meaning |
|---|---|
| **Light purple** | TCP packets |
| **Light blue** | UDP packets |
| **Black with red text** | TCP packets with checksum errors |
| **Black background** | TCP RST (reset) packets, connection being torn down abruptly |
| **Dark red / bright red** | Errors, malformed packets, or connection resets|
| **Green** | HTTP traffic |
| **Light yellow** | Routing protocols (examples: OSPF, RIP, etc..) |
| **Gray** | TCP SYN/FIN/ACK-only packets (handshake overhead, no payload) |
| **Bright yellow/orange highlight** on a single row | Currently selected / conversation-related packet after you click something |

**Read strategy for colors:** 
- Don't try to memorize every color meaning first
- Scan for anything that stands out as *dark red or black*, since that's where you will find errors, resets, or retransmissions live.
- Everything in purple/blue/gray is "normal traffic flow" and can usually be skimmed unless you're looking for something specific.
- You can always hover over a row and check the status bar, or right-click → "Colorize Conversation" to temporarily highlight all packets in one exchange, which is one of the fastest ways to visually isolate a single conversation from a noisy capture.

---

## 3. How to Read a Capture

Rather than scrolling top to bottom, you can:

1. **Filter before you scroll:** do not try to read a raw, unfiltered capture line-by-line. Start with something like `ip.addr==<host>` or `tcp.port==<port>` to isolate the relevant conversation
   
2. **Use Statistics → Conversations first:** this shows you a table of every IP/TCP/UDP conversation with byte counts and duration, it helps to indentify *where the traffic is* before you start checking individual packets
   
3. **Use Statistics → Protocol Hierarchy:** this helps to see what protocols make up the capture as a percentage. This instantly tells you if you're looking at mostly HTTP, DNS, TLS, etc., without reading a single row
   
4. **Scan the Info column more often:** the Info column in the **Packet List** shpws a fast readable summary (such as: "SYN", "GET /index.html", "DNS query"), this helps with undertstanding what happened in a certain row

5. **Follow the stream:** do not hesitate to right-click any packet → Follow → TCP/UDP/HTTP Stream, by doping so you will reassembles the whole conversation into readable text instead of individual packets, far faster than reading packet-by-packet

6. **Only then open Packet Details:** expand the tree only for the  packet you actually need to inspect closely (that can be for: checking flags, TTL, a specific header value, etc..)

7. **Bytes pane is a last resort.** you can check the raw hex/ASCII view when you specifically need to confirm exact byte values (verifying a payload signature or suspected corruption)

---

# Filter Cheat Sheet

## 1. Filter syntax basics

| Operator | Meaning | Example |
| --- | --- | --- |
| `==` or `eq` | equals | `ip.addr == 10.0.0.5` |
| `!=` or `ne` | not equal | `ip.addr != 10.0.0.5` |
| `>` `<` `>=` `<=` | comparison | `frame.len > 1000` |
| `&&` or `and` | logical AND | `ip.src == 10.0.0.5 && tcp.port == 80` |
| `\|` or `or` | logical OR | `tcp.port == 80 \| tcp.port == 443` |
| `!` or `not` | logical NOT | `!arp` |
| `contains` | substring match | `http.host contains "google"` |
| `matches` | regex match | `http.host matches "^www\."` |
| `in {}` | value in a set | `tcp.port in {80 443 8080}` |
| `[n]` | byte or bit slice | `eth.src[0:3] == 00:0c:29` |

## 2 IP and Ethernet 

```
ip.addr == 10.0.0.2
ip.src == 10.0.0.2
ip.dst == 10.0.0.2
ip.addr == 10.0.0.0/24
!(ip.addr == 10.0.0.0/8 or ip.addr == 192.168.0.0/16 or ip.addr == 172.16.0.0/12)
eth.addr == aa:bb:cc:dd:ee:ff
eth.src == aa:bb:cc:dd:ee:ff
eth.dst == aa:bb:cc:dd:ee:ff
ip.ttl < 10
ip.flags.mf == 1
ip.frag_offset > 0
```

- the negated private range filter is for isolating traffic going out to the public internet

## 3. TCP and UDP

```
tcp
udp
tcp.port == 443
tcp.srcport == 443
tcp.dstport == 443
tcp.port in {80 443 8080 8443}
tcp.flags.syn == 1 && tcp.flags.ack == 0
tcp.flags.syn == 1 && tcp.flags.ack == 1
tcp.flags.reset == 1
tcp.flags.fin == 1
tcp.analysis.retransmission
tcp.analysis.duplicate_ack
tcp.analysis.zero_window
tcp.analysis.lost_segment
tcp.analysis.out_of_order
tcp.stream == 5
tcp.len > 0
udp.length > 0
```

`tcp.analysis.*` filters are also a way to detect network problems, such as :
- retransmissions
- zero windows
- out of order packets
- lost segments

## 4.HTTP and web traffic

```
http
http.request
http.response
http.request.method == "GET"
http.request.method == "POST"
http.response.code == 200
http.response.code == 404
http.response.code >= 400
http.host == "example.com"
http.host contains "example"
http.request.uri contains "login"
http.user_agent contains "curl"
http.cookie
http.authorization
http.file_data
media_type == "image/jpeg"
```

## 5. TLS and HTTPS

```
tls
tls.handshake.type == 1
tls.handshake.type == 2
tls.handshake.type == 11
tls.handshake.extensions_server_name
tls.handshake.extensions_server_name contains "example"
tls.record.version == 0x0303
tls.alert_message
tls.handshake.ciphersuite
```

- handshake type 1 is Client Hello
- handshake type 2 is Server Hello
- type 11 is Certificate
- `extensions_server_name` is the SNI field that helps identifying which site was requested even when the rest of the session is encrypted

## 6. DNS

```
dns
dns.flags.response == 0
dns.flags.response == 1
dns.qry.name == "example.com"
dns.qry.name contains "example"
dns.qry.type == 1
dns.qry.type == 28
dns.qry.type == 5
dns.a
dns.flags.rcode != 0
dns.count.answers > 0
```

- query type 1 is A
- query type 28 is AAAA
- query type 5 is CNAME
- `dns.flags.rcode != 0` finds failed lookups (NXDOMAIN and similar), therefore it is used to spot malware trying dead domains

## 7. ARP

```
arp
arp.opcode == 1
arp.opcode == 2
arp.duplicate-address-detected
arp.src.proto_ipv4 == 10.0.0.5
```

- opcode 1 is request
- opcode 2 is reply
- duplicate address filter flags ARP spoofing or IP conflicts

## 8. DHCP

```
dhcp
bootp
dhcp.option.dhcp == 1
dhcp.option.dhcp == 2
dhcp.option.dhcp == 3
dhcp.option.dhcp == 5
dhcp.option.hostname
dhcp.option.requested_ip_address
```

- DHCP message type 1 is Discover
- DHCP message type 2 is Offer
- DHCP message type  3 is Request
- DHCP message type 5 is ACK
- old captures may show as `bootp` and not `dhcp`

## 9. ICMP

```
icmp
icmp.type == 8
icmp.type == 0
icmp.type == 3
icmp.type == 11
```

- icmp type 8 is echo request (ping)
- icmp type 0 is echo reply
- icmp type 3 is destination unreachable
- icmp type 11 is time exceeded (traceroute).

## 10. Authentication protocols

```
kerberos
kerberos.msg_type == 10
kerberos.msg_type == 11
kerberos.msg_type == 13
kerberos.CNameString
kerberos.realm
ntlmssp
ntlmssp.auth.username
ntlmssp.auth.domain
smb
smb2
smb2.cmd == 0
smb2.filename
ldap
ldap.authentication
```

Kerberos message types: 
- 10 is AS-REQ, 11 is AS-REP, 13 is TGS-REP
- SMB2 command 0 is Negotiate Protocol
