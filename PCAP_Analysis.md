# 1: Basics of Networking

## 1.1 Devices need addresses

Every device on a network ( laptop, printer, phone, smart device) needs an address so other devices know where to send data

There are two addresses every device has at once:

- **IP address**: a logical address, like `172.16.8.20`, it can change (a laptop gets a new one on a different network).
- **MAC address**: a physical address burned into the network card itself, like `00:12:f0:28:d4:34`, it belongs to that specific piece of hardware.


## 1.2 IP address ranges, private vs public

Some IP ranges are reserved for use *inside* private networks only (homes, offices), and are never used directly on the internet:

- `10.0.0.0` to `10.255.255.255`
- `172.16.0.0` to `172.31.255.255`
- `192.168.0.0` to `192.168.255.255`

Any address outside these ranges is a **public IP**, a real address reachable on the internet
This distinction helps a lot in investigations, because an internal (private) machines talking to public IPs is usually where malicious activity shows up

## 1.3 A packet, and a capture (pcap)

Every piece of data sent over a network travels in small chunks called **packets**. 
A packet has layers:

```
[ Ethernet: who's the physical sender/receiver? ]
  [ IP: what's the logical sender/receiver address? ]
    [ TCP or UDP: what "conversation" or port is this part of? ]
      [ Application data: the actual content, example an HTTP request ]
```

A **pcap file** is just a recording of many packets, captured off a network, saved (in rows) to a file so you can inspect them later. 
That's what Wireshark opens and displays.

---

# 2: The protocols you'll  see

These network protocols are the ones that leak identity information or malicious behavior:

## 2.1 ARP, "who has this IP address? (helps searching for the MAC)"

When a device wants to send data to an IP address on the same local network, it first has to find out which MAC address owns that IP, it broadcasts a question: "who has 172.16.8.20?" and the device with that address replies with its MAC address

- ARP replies are a reliable way to map an IP address to a MAC address, since the device is directly telling you its own hardware address
- 
## 2.2 DHCP, "give me an IP address (helps finding the hostname)"

When a device joins a network (boots up, connects to Wi-Fi), it doesn't have an IP address yet, it broadcasts a request asking for one, a server on the network (often the router) replies with an assigned address.

Windows machines, when making this request, commonly include their own computer name in the request, as a courtesy so the network can identify them, which is very useful for investigation.

- DHCP **request** packets often contain the device's hostname, one of the easiest ways to find a machine's name

## 2.3 DNS, "what's the address for this website name? (helps identifying suspicous activity)"

We use names (`google.com`), computers use IP addresses? and DNS is the system that translates one into the other. 
Before any device connects to a website, it  asks a DNS server "what's the IP address for this domain name?" first

Every domain a device has ever tried to reach shows up here, including malware trying to reach its control server. 

- Reviewing DNS queries can be the fastest way to notice something suspicious, since infections most of the time need to look up a domain name before connecting

## 2.4 HTTP and HTTPS, "give me this webpage"

HTTP is the protocol web browsers (and a lot of malware) use to request and receive content
Remember that an HTTP request includes: 
- which website (Host)
- which specific page or resource (URI)
- information about the software making the request (User-Agent)

HTTPS is the same idea, but encrypted, so the content is hidden from anyone watching the network, except for one field, the domain name being contacted, which is still visible in a preliminary handshake step (called TLS), even though everything after that is encrypted

- Repeated requests to unusual domains, with a mismatched or outdated User-Agent, or a suspicious repeating URL pattern, are a sign of malware communicating with its operator (called command and control, aka "C2").

## 2.5 NBNS, "who has this name on the network?"

NBNS (NetBIOS Name Service) is an older Windows protocol used to look up and announce machine names on a local network, separate from DNS

- Windows machines send NBNS traffic constantly in the background, mostly to resolve names to IPs on the LAN itself
- It doesn't always contain a specific hostname though, sometimes it only has  the domain/workgroup name instead of the machine name, depending on the query type captured

## 2.6 The Browser protocol (helps with the hostname)

The Browser protocol is a  Windows service used for network neighborhood discovery, machines periodically announce themselves on the LAN so others can see them

- A **Host Announcement** packet includes the sender's own hostname directly in it
- A **Browser Election Request** is a different kind of packet, machines voting on who manages browsing duties, it does not contain a hostname 

This is useful specifically when DHCP isn't present in a capture (if you are searching for the hostname)
---

# 3: What Active Directory is 

Active Directory (AD) is Microsoft's system for managing a network of Windows computers and user accounts as one organized whole, instead of every machine having its own separate list of users

## 3.1 The domain controller

One (or a few) servers act as the **domain controller (DC)**:
- It holds the master list of every user account
- every computer account
-  every permission
for the whole organization

Every login, every permission check, goes through this server.

In a packet capture, the domain controller is usually the internal IP address that all the *other* internal machines are constantly talking to (for logins, permission checks, name lookups), **while rarely if ever talking to the outside internet itself**

## 3.2 User accounts vs computer accounts

In AD, both **people** and **computers** get accounts:

- A user account looks like a normal username,`meow`
- A computer account is the hostname with a dollar sign at the end, `DESKTOP-4HKB967K$`

This distinction matters a lot when reading authentication traffic (if you see a name ending in `$`, that's the machine itself checking in, not a person)

## 3.3 Kerberos, and how logins happen

When a user logs into a Windows machine that's part of a domain, the machine doesn't just take their word for it, it talks to the domain controller using a protocol called **Kerberos** to prove the user's identity and get a "ticket" that acts like a temporary ID card for accessing other resources.

The very first message in this exchange (called an AS-REQ, Authentication Service Request) contains the username of the person or computer requesting access. 

- This is one of the two most reliable places to find a real username in a capture.

## 3.4 NTLM, the older login method

NTLM is an older Windows authentication protocol, still used in many environments alongside or instead of Kerberos, especially for things like file share access. 

- An NTLM exchange (specifically the final step, called NTLMSSP_AUTH) often contains the username, the domain name, and the hostname of the machine, all together in one message.

## 3.5 SAMR, looking up account details

- SAMR is a protocol Windows machines use to ask the domain controller for details about an account, things like a user's actual real-world name (their "Full Name" field), separate from their login username. 


