File analyzed: `2026-09-11-traffic-analysis-exercise.pcap`
the purpose is to find:

- the IP address of the infected Windows client?
- the MAC address of the infected Windows client?
- the host name of the infected Windows client?
- the user account name from the infected Windows client?
- the full name of the user from the user account?


Every step below shows the process of this pcap analysis using wireshark, in addition to the equivalent tshark commands, so you can follow along in either tool.

* * *

To avoid any confusion, keep in mind that few steps use tshark's `-z` statistics or `-V` full-packet-detail options, which have no filter bar text if you use wireshark, those are marked with the matching menu action instead.

* * *

# Setup

```bash
apt-get update
apt-get install -y tshark
(for debian based distributions)
```

About the pcap file: it came zipped, protected with a password, unzip it this way

```bash
unzip -P infected_20260911 2026-09-11-traffic-analysis-exercise.zip
```

# Troubleshooting

I have, earlier, with another pcap file encountered an annoying problem with wireshark filters where a protocol was not being detected at all, if you, during the investigation, find yourslef having the same problem with any type of protocol I advise you to check the troubleshooting file

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


This showed: `DNS, TLS, QUIC, HTTP, SMB2, Kerberos, LDAP, SAMR, NBNS,` and a protocol called `browser.` 
This is a Windows AD environment (Kerberos, LDAP, SMB2, SAMR), with **no DHCP** present

<img width="1588" height="721" alt="Image" src="https://github.com/user-attachments/assets/051ef58c-f96b-4ec6-89a4-b0039ed46a2b" />

<img width="1654" height="904" alt="Image" src="https://github.com/user-attachments/assets/4a3f70b3-b2ed-4f0d-945d-56ebdb463460" />

<img width="1192" height="211" alt="Image" src="https://github.com/user-attachments/assets/ed263923-abf3-439f-9f4e-7aaed40701f8" />

<img width="1876" height="937" alt="Image" src="https://github.com/user-attachments/assets/39a39cf6-6dbd-4161-a954-4748463c8c0c" />

<img width="1254" height="265" alt="Image" src="https://github.com/user-attachments/assets/947a9c6a-b938-4bb0-afc8-f34049b18d48" />

<img width="1899" height="712" alt="Image" src="https://github.com/user-attachments/assets/215cbc95-1a2d-4001-b729-8e12a5777577" />

<img width="944" height="335" alt="Image" src="https://github.com/user-attachments/assets/c0fe9e7c-5523-4d1b-b3dc-025559008023" />

<img width="944" height="362" alt="Image" src="https://github.com/user-attachments/assets/9a91c54e-813d-420f-8b27-66cb15e09091" />
