File analyzed: `2026-08-09-traffic-analysis-exercise.pcap`

the purpose is to find:

- the infected host's IP
- the infected host's MAC address
- the infected host's hostname
- the infected host's username and the full name of that user.

Every step below shows the process of this pcap analysis using wireshark, in addition to the equivalent tshark commands, so you can follow along in either tool.

* * *

To avoid any confusion, keep in mind that few steps use tshark's `-z` statistics or `-V` full-packet-detail options, which have no filter bar text if you use wireshark, those are marked with the matching menu action instead.

* * *

# Setup

I am assuming that you already use an os that has wireshark pre-installed.  
About tshark (if not installed on your os) you can install it with:

```bash
apt-get update
apt-get install -y tshark
(for debian based distributions)
```

About the pcap file: it came zipped, protected with a password, unzip it this way

```bash
unzip -P infected_20260809 2026-08-09-traffic-analysis-exercise.zip
```

# Troubleshooting

I have personally encountered an annoying problem with wireshark filters where a protocol was not being detected at all, if you, later, find yourslef having the same problem with any type of protocol I advise you to check the troubleshooting file

# PCAP analysis

## 1: See what protocols exist

Before filtering anything, we should get an overview of the capture to get a general idea

```bash
tshark -r 2026-08-09-traffic-analysis-exercise.pcap -q -z io,phs
```

```md
Wireshark: 
this is a statistics command, not a filter, you should go to:
Statistics -> Protocol Hierarchy in the menu bar
```

What this showed: `dhcp`, `dns`, `nbns`, `http`, `kerberos`, `ldap`, `smb2`, `tls`, `dcerpc` (including `samr`).

This means that we are facing a Windows Active Directory environment (Kerberos, LDAP, SMB2, SAMR)  
so:

- hostname and username questions can be answered from domain authentication traffic
- there's a reasonable amount of plain HTTP to inspect for malicious activity

## 2: Find candidate internal hosts by conversation volume

```bash
tshark -r 2026-08-09-traffic-analysis-exercise.pcap -q -z conv,ip
```

```md
Wireshark: 
also a statistics command, not a filter, so you should go to:
Statistics -> Conversations -> IPv4 tab
then click the Bytes or Packets column header to sort
```

Result:  
there are two internal IPs that are heavy external talkers, `172.16.8.53` and `172.16.8.49`, each with many separate conversations to external IPs.

A third internal IP, `172.16.8.8`, talked to the other two internally, this one should refer to the domain controller, it is not a client.

At this point we have two candidates, yet, no confirmed one.

## 3: Comparing DNS activity for each candidate

```bash
tshark -r 2026-08-09-traffic-analysis-exercise.pcap -Y "dns.flags.response==0" -T fields -e ip.src -e dns.qry.name | sort | uniq -c | sort -rn
```

```md
Wireshark filter bar:
dns.flags.response == 0
```

After applying the filter, read the **Source** column and expand **Domain Name System** in the packet details pane for each row's queried name. There is no built-in count/sort in the packet list itself, so for a large capture use **Statistics, DNS** to see query volume, or scan visually since the filtered list is already much shorter.

<img width="1620" height="831" alt="Image" src="https://github.com/user-attachments/assets/e011eb32-6840-4f8a-a421-b1981bf6708b" />
<img width="321" height="154" alt="Image" src="https://github.com/user-attachments/assets/6a4e853c-f5cc-487b-8196-47de02f1fb78" />
<img width="261" height="205" alt="Image" src="https://github.com/user-attachments/assets/7191005d-7561-480f-ac0e-2fd23af81848" />
Result:

- `172.16.8.53` mostly queried normal Microsoft/Bing/MSN domains, ordinary background traffic for a Windows machine.
- `172.16.8.49` queried the same normal domains, but also queried a long list of odd, short, randomly worded domains with unusual top level domains: `www.z61gqw.beer`, `www.www-bet456.co`, `www.vjscloudjsns.beer`, `www.p3x63q.garden`, `www.moxom.online`, `www.kentmediallc.com`, `www.earthframe.site`, `www.21207628.shop`, and others

we know that egitimate everyday browsing does not generate a long list of very random-looking domains across unusual TLDs like `.beer`, `.garden`, `.shop`, `.site`, `.online`.

==**172.16.8.49 seems to be the infected host**==

## 4: Confirming with HTTP request behavior

```bash
tshark -r 2026-08-09-traffic-analysis-exercise.pcap -Y "http.request && ip.src==172.16.8.49" -T fields -e frame.number -e ip.dst -e http.host -e http.request.uri -e http.user_agent
```

```md
General wireshark filter :
http.request && ip.src == 172.16.8.49
```

Note: it is okey if you do not see the same colums as shown in my screenshot it is totally fine you can add whatever columns you want by yourself.

I added custom columns for `http.host` and `http.user_agent` so i can scan them visually without opening every packet, yoiu just have to right click each field in the packet details pane and choose **Apply as Column**.

<img width="1912" height="946" alt="Image" src="https://github.com/user-attachments/assets/258ddb60-035f-4a6b-aaf5-55c38236da89" />
<img width="1878" height="876" alt="Image" src="https://github.com/user-attachments/assets/877d0ddd-1e41-4054-90c6-b61e920ab18f" />
Result:

- `172.16.8.49` made repeated HTTP requests to a rotating set of the odd domains found in **step 3**, each with a short random looking path (`/lqjm/`, `/8nw8/`, `/r7l3/`, `/hut9/`, `/7qex/`, `/v2r8/`, `/irpw/`, `/ujvq/`), each hit multiple times, and each carrying a matching pair of encrypted looking query parameters across every domain.
    
- All of these requests shared the same outdated user agent: `Mozilla/5.0 (Windows NT 6.2; rv:39.0) Gecko/20100101 Firefox/39.0`.
    

This confirms a certain pattern where one process on this machine is cycling through a list of domains with the same request structure and the same tracking parameters.  
That can be an indication of a browser hijacker, a malware redirector, or maybe ad fraud chain, but definetly not a person browsing normally

==**from this we can conclude that the infected IP address is indeed 172.16.8.49**==

## 5: Get the MAC address

```bash
tshark -r 2026-08-09-traffic-analysis-exercise.pcap -Y "ip.src==172.16.8.49" -T fields -e eth.src | sort -u
```

```md
Wireshark filter bar:
ip.src == 172.16.8.49
then: 
click any result and expand **Ethernet II (you can stop here because it appears) -> Source** in the packet details pane.
```

<img src=":/5a4b4df792044803be3c81eb114eb9d3" alt="a37c0d5cd446319a0da1fe7358cf9ce7.png" width="824" height="295" class="jop-noMdConv">

==**the MAC address is 00:12: f0:28:d4:34**==

## 6: Get the hostname

```bash
tshark -r 2026-08-09-traffic-analysis-exercise.pcap -Y "dhcp" -T fields -e ip.src -e ip.dst -e dhcp.option.dhcp -e dhcp.option.hostname -e dhcp.option.requested_ip_address
```

```md
Wireshark filter bar:
dhcp
then:
Expand **Dynamic Host Configuration Protocol** in the packet details pane
chack **Option: (12) Host Name** and **Option: (50) Requested IP Address** 
Note:these only appear on Request and Inform packets, not every DHCP packet.
```

Result:  
A DHCP Request packet where `dhcp.option.hostname` was `DESKTOP-5NLV63K` and `dhcp.option.requested_ip_address` was `172.16.8.49`, tying the hostname directly to the IP address we already confirmed.  
this will be confirmed in **Step 7's Kerberos output**

![944ff1f6d2fe78b1ebfaab87a439e79e.png](:/0ff003c29e104ef08635beb29fa282a9)

==**the hostname is `DESKTOP-5NLV63K`.**==

## Step 7: Get the username

```bash
tshark -r 2026-08-09-traffic-analysis-exercise.pcap -Y "kerberos.msg_type==10 && ip.src==172.16.8.49" -T fields -e ip.src -e kerberos.CNameString -e kerberos.realm | sort -u
```

```md
Wireshark filter bar:
kerberos.msg_type == 10 && ip.src == 172.16.8.49
Expand **Kerberos -> as-req -> cname -> cname-string -> CNameString** and **Kerberos -> as-req -> cname -> realm** in the packet details pane for each result

```

![2a2cab548881036208d1cdac911b666e.png](:/bc30491e1ba64e6786b5a306a5cfb76d)  
Result:  
we see two distinct client principal names from this IP:

- one was `desktop-5nlv63k$`, the machine account (note the trailing dollar sign, which always marks a computer account rather than a person)
- the other was `rvance`, with realm `FIRSTTOLAST.TECH`, the actual user account authenticating from this machine.

If NTLMSSP traffic had been present for this host, `ntlmssp.auth.username` would have been an equally valid way to confirm this same value, but in this capture the Kerberos AS-REQ was the source.

&nbsp;==**the username is rvance**==

## 8: Getting the full name of the user

Username alone does not give a full name, in this capture, the full name was leaked through SAMR (a protocol used for Active Directory account management) when the client queried its own account details.

First, we locate the SAMR QueryUserInfo exchange for this host:

```bash
tshark -r 2026-08-09-traffic-analysis-exercise.pcap -Y "samr && ip.addr==172.16.8.49" -T fields -e frame.number -e ip.src -e ip.dst -e _ws.col.Info
```

Wireshark filter bar:

```
samr && ip.addr == 172.16.8.49
Search on the **Info** column for each result to find the line that says **QueryUserInfo response**
```

this produces a numbered sequence of SAMR operations (Connect5, EnumDomains, LookupDomain, OpenDomain, LookupNames, OpenUser, QueryUserInfo, and so on), we note the frame number of the `QueryUserInfo response` line ( you can get the details in the packet detail pane just by clicking that row, the search by frame number that comes next is optional in wireshark)

![95f7942710d8f937610a0df51b4c35a1.png](:/faa92348fb67410a9396f4254bc90480)

Then read that specific packet in full detail:

```bash
tshark -r 2026-08-09-traffic-analysis-exercise.pcap -Y "frame.number==<that_frame_number>" -V | grep -i -A2 -B2 "full name"
```

```md
Wireshark: 
type frame.number == <that_frame_number> in the filter bar
then: in the packet details pane, expand the SAMR response fields down into **QueryUserInfo, Info21, Full Name**.
```

![d82fc0fbf9f3b228de21ba77627d8550.png](:/af79bdb5d16e400986379f05bd826b54)  
Result: the response packet contained both `Account Name: rvance` and `Full Name: Raymond Vance` together in the same SAMR structure, directly linking the username to a real name.

&nbsp;==**the full name is `Raymond Vance`.**==

## Why this host and not the other candidate

`172.16.8.53` (hostname `DESKTOP-A8WD4E2`) was also a heavy external talker, but every domain it contacted was a legitimate Microsoft, Bing, or MSN service, and its HTTP traffic showed normal, varied requests rather than a repeating pattern.

`172.16.8.49` had a combination of randomly named external domains, repeated identical request paths, matching encrypted query parameters across unrelated domains, and a single outdated browser user agent tying it all together.  
(byte or packet count alone would abviously not have distinguished the two)
