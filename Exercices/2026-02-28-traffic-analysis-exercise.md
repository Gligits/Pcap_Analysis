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

we know that egitimate everyday browsing does not generate a long list of very random-looking domains across unusual TLDs 

==**10.2.28.88 is much suspected to be the infected host**==

## 4: Confirming with HTTP request behavior

```md
General wireshark filter :
http.request && ip.src == 10.2.28.88
```

Note: it is okey if you do not see the same colums as shown in my screenshot it is totally fine you can add whatever columns you want by yourself.

I added custom columns for `http.host` and `http.user_agent` so i can scan them visually without opening every packet, yoiu just have to right click each field in the packet details pane and choose **Apply as Column**.

<img width="938" height="409" alt="Image" src="https://github.com/user-attachments/assets/b68b47cd-8531-4668-8ec6-8b25a9cb3bff" />

Note: The grey rows are SSDP, it may seem strange since we are filtering for http requests, but the SSDP is showing because it uses an HTTP-like request format, particularly (M-SEARCH * HTTP/1.1)

and 239.255.255.250:1900 is the standard multicast address/port used for SSDP/UPnP discovery, therefore these requests are considered network/service discovery traffic, not web browsing

<img width="686" height="81" alt="Image" src="https://github.com/user-attachments/assets/47512dff-1a64-4585-b608-984e05c21d36" />


<img width="947" height="301" alt="Image" src="https://github.com/user-attachments/assets/487dd9c4-8e33-4da2-ae0b-cce6c154d378" />

<img width="950" height="376" alt="Image" src="https://github.com/user-attachments/assets/aab4f374-fb24-42a7-a40c-4d6571f7e96d" />


--- 
<img width="1288" height="90" alt="Image" src="https://github.com/user-attachments/assets/99fabacd-6d49-4ee5-bb2a-c20f6d2a3370" />

<img width="764" height="71" alt="Image" src="https://github.com/user-attachments/assets/11e983c5-fb28-4a56-9611-97d74db010c8" />
