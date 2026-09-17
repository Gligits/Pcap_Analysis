File analyzed: 2026-02-28-traffic-analysis-exercise.pcap

Where you can find it: https://www.malware-traffic-analysis.net/training-exercises.html

the purpose is to find:

- the infected host's IP
- the infected host's MAC address
- the infected host's hostname
- the infected host's username and the full name of that user
Every step below shows the process of this pcap analysis using wireshark

# Setup
I am assuming that you already use an os that has wireshark pre-installed

About the pcap file: it came zipped, protected with a password, unzip it this way

````bash
unzip -P infected_20260228 2026-02-28-traffic-analysis-exercise.zip
````

# Troubleshooting
I have personally encountered an annoying problem with wireshark filters where a protocol was not being detected at all, if you, later, find yourslef having the same problem with any type of protocol I advise you to check the troubleshooting file

# PCAP analysis
## 1: See what protocols exist
Before filtering anything, we should get an overview of the capture to get a general idea

```md
Wireshark: 
this is a statistics command, not a filter, you should go to:
Statistics -> Protocol Hierarchy in the menu bar
```

<img width="1504" height="775" alt="Image" src="https://github.com/user-attachments/assets/3ae9c020-1e2b-44fe-b381-faf93dce7279" />

This shows: `dhcp`, `dns`, `nbns`, `http`, `kerberos`, `smb2`, `tls`, `dcerpc` (including `samr`)
This means that we are facing a Windows Active Directory environment (Kerberos, LDAP, SMB2, SAMR)
so:
- hostname and username questions can be answered from domain authentication traffic

There's also a reasonable amount of plain HTTP to inspect for malicious activity

## 2: Find candidate internal hosts by conversation volume
```md
Wireshark: 
also a statistics command, not a filter, so you should go to:
Statistics -> Conversations -> IPv4 tab
then click the Bytes or Packets column header to sort
```
Note: you can do this by analysing the DNS packets and trying to find the internal host that is generating all the DNS traffic 
(a normal LAN has many machines, but usually only the client actively resolving name after name is the interesting one, while a DC mostly just answers)


Result:  
there are one internal IP that is a heavy external talker, `10.2.28.88` 

<img width="712" height="487" alt="Image" src="https://github.com/user-attachments/assets/ab556dfd-0c99-40b0-98d8-a36dc371c0b3" />

we confirm this by applying a DNS filter 

```md
Wireshark filter bar:
dns.flags.response == 0
```

After applying the filter we expand **Domain Name System** in the packet details pane for each row's queried name.

Note: there is no built-in count/sort in the packet list itself, so for a large capture use Statistics, DNS to see query volume (for this one it can, be scanned visually since the filtered list is already much shorter)

<img width="913" height="425" alt="Image" src="https://github.com/user-attachments/assets/cd6b17b8-310d-4853-a4a8-c167bd39de1e" />

Result:

- `10.2.28.88` queried some normal Microsoft domains, but also queried a long list of odd, short, randomly worded domains with unusual top level domains

we know that legitimate everyday browsing does not generate a long list of very random-looking domains across unusual TLDs 

we can also confirm that no other internal IP shows up as a source at all!

This is confirmed independently by the DHCP exchange at the very start of the capture, where 10.2.28.88 is explicitly the address being handed out by the DHCP server (10.2.28.1) to this machine

<img width="764" height="71" alt="Image" src="https://github.com/user-attachments/assets/11e983c5-fb28-4a56-9611-97d74db010c8" />


==**10.2.28.88 is with no doubt the infected host**==

## 4: Getting the MAC address
We know the infected IP is 10.2.28.88, we will try to get its MAC with ARP (since every host on the LAN periodically answers "who has the IP X" with an is-at reply containing its hardware address)

<img width="945" height="169" alt="Image" src="https://github.com/user-attachments/assets/72e83bf4-3df9-41b3-a3f5-796a0905ca7f" />

We can confirù that there has been no spoofing because the DHCP Discover packets sent by 10.2.28.88 (we have seen these packets earlier) also carry this MAC as their Ethernet source, and the DHCP server's Offer is addressed to that same MAC, so the ARP result and the DHCP handshake prove there's no MAC/IP conflict anywhere in the capture 

you can also verify it this  way 

```md
Wireshark filter bar:
arp.duplicate-address-frame
```
<img width="613" height="101" alt="image" src="https://github.com/user-attachments/assets/c7b35f65-c9ba-4df8-b8a7-ab9a64bd3cac" />

we can clearly see Zero packets returned 


==**00:19:d1:b2:4d:ad is the MAC address of the infected machine**==

## 5: Getting the hostname
The client announces its own name to the DHCP server as part of asking for a lease
```md
Wireshark filter bar:
dhcp.type == 1
or just
dhcp
```

<img width="950" height="358" alt="image" src="https://github.com/user-attachments/assets/43c215de-d22a-4d92-af21-929b0c56bcc2" />

aftre expandinf the 2nd frame we can see 
the client mac address : 00:19:d1:b2:4d:ad
the hostname: DESKTOP-TEYQ2NR
the requested ip address: 10.2.28.88

which the DHCP server gave to machine (DHCP ACK, frame 104)

==**DESKTOP-TEYQ2NR is the hostname of the infected machine**==

## 6: Getting the user account name from the infected Windows machine

since this is an Active Directory environment, the client authenticates to the domain controller via Kerberos, the client name (cname) in a Kerberos AS-REQ (the very first step of a Kerberos login) 

````md
Wireshark filter bar:
kerberos.msg_type == 10 && ip.src == 10.2.28.88
Expand **Kerberos -> as-req -> cname -> cname-string -> CNameString** and **Kerberos -> as-req -> cname -> realm** in the packet details pane for each result
````

<img width="949" height="362" alt="image" src="https://github.com/user-attachments/assets/d6f24004-3ac3-40fa-b8d1-7a332fcfc524" />

&nbsp;

==**the username is brolf**==

## 7: Full name of the user from the user account 

with the account name (brolf) in hand, the way find the full name is to rely on the SAMR (Security Account Manager Remote protocol), it is the RPC interface Windows uses internally, and it's fully visible on the wire since it isn't encrypted at this layer

````md
Wireshark filter bar:
samr && ip.addr==10.2.28.88
````

<img width="946" height="393" alt="image" src="https://github.com/user-attachments/assets/7a2fcabe-4d6f-4356-9826-e37b33e7fbf3" />

==**the full name is Becka Rolf**==

--- 
# Further Investigation

## HTTP request behavior

```md
General wireshark filter :
http.request && ip.src == 10.2.28.88
```

Note: it is okey if you do not see the same colums as shown in my screenshot it is totally fine you can add whatever columns you want by yourself.

I added custom columns for `http.host` and `http.user_agent` so i can scan them visually without opening every packet, yoiu just have to right click each field in the packet details pane and choose **Apply as Column**.

<img width="938" height="409" alt="Image" src="https://github.com/user-attachments/assets/b68b47cd-8531-4668-8ec6-8b25a9cb3bff" />

Note: The grey rows are SSDP, it may seem strange since we are filtering for http requests, but the SSDP is showing because it uses an HTTP-like request format, particularly (M-SEARCH * HTTP/1.1)

and 239.255.255.250:1900 is the standard multicast address/port used for SSDP/UPnP discovery, therefore these requests are considered network/service discovery traffic, not web browsing

For the other requests, the User-Agent in is **NetSupport Manager **, which is a legitimate remote-management and remote-control software. However, the machine 10.2.28.88 is repeatedly communicating with the external IP 45.131.214.85 while identifying itself as NetSupport Manager

But Why is this machine using NetSupport Manager to communicate with this particular external server? this leads us to question if NetSupport Manager was intentionally installed and used on the machine

---
<img width="947" height="301" alt="Image" src="https://github.com/user-attachments/assets/487dd9c4-8e33-4da2-ae0b-cce6c154d378" />

