File analyzed: 2026-02-28-traffic-analysis-exercise.pcap

Where you can find it: https://www.malware-traffic-analysis.net/training-exercises.html

the purpose is to find:

the infected host's IP
the infected host's MAC address
the infected host's hostname
the infected host's username and the full name of that user
Every step below shows the process of this pcap analysis using wireshark

# Setup
I am assuming that you already use an os that has wireshark pre-installed

About the pcap file: it came zipped, protected with a password, unzip it this way

unzip -P infected_20260228 2026-02-28-traffic-analysis-exercise.zip

# Troubleshooting
I have personally encountered an annoying problem with wireshark filters where a protocol was not being detected at all, if you, later, find yourslef having the same problem with any type of protocol I advise you to check the troubleshooting file

# PCAP analysis
## 1: See what protocols exist
Before filtering anything, we should get an overview of the capture to get a general idea

<img width="1504" height="775" alt="Image" src="https://github.com/user-attachments/assets/3ae9c020-1e2b-44fe-b381-faf93dce7279" />

<img width="712" height="487" alt="Image" src="https://github.com/user-attachments/assets/ab556dfd-0c99-40b0-98d8-a36dc371c0b3" />

<img width="686" height="81" alt="Image" src="https://github.com/user-attachments/assets/47512dff-1a64-4585-b608-984e05c21d36" />

<img width="871" height="121" alt="Image" src="https://github.com/user-attachments/assets/d4d0d6eb-5789-4e62-aa53-e6832fbaf8f5" />

<img width="947" height="301" alt="Image" src="https://github.com/user-attachments/assets/487dd9c4-8e33-4da2-ae0b-cce6c154d378" />

<img width="950" height="376" alt="Image" src="https://github.com/user-attachments/assets/aab4f374-fb24-42a7-a40c-4d6571f7e96d" />


--- 
<img width="1288" height="90" alt="Image" src="https://github.com/user-attachments/assets/99fabacd-6d49-4ee5-bb2a-c20f6d2a3370" />

<img width="764" height="71" alt="Image" src="https://github.com/user-attachments/assets/11e983c5-fb28-4a56-9611-97d74db010c8" />
