# Wireshark Showed Zero HTTP Packets
There was a pcap file that genuinely contains HTTP traffic (confirmed independently with tshark) showed **0 results** for every HTTP filter in Wireshark, even though the file loaded correctly and the packet count matched.

Filtering for `http` in Wireshark gave:

```
Packets: 22473 · Displayed: 0 (0.0%)
```

The file loaded fully (22473 packets, correct total), but not a single one was recognized as HTTP.

## 1: Wrong filter or typo

- maybe the filter itself had a typo, an invisible character, or was combined with a leftover condition from an earlier search.
- i cleared the filter bar completely, retype `http`, check that the bar turned green (valid syntax) and confirm the total packet count.
- filter was valid (green) and packet count was correct (22473 total), so this is not a typo problem.

## 2: The file has no plain HTTP only encrypted TLS traffic

- some captures genuinely have zero HTTP because everything is HTTPS. If true, it is just a fact about the file.
- i independently reopened the exact same file with tshark and searched for HTTP directly.
- the file did contain HTTP, 330 separate HTTP frames, this proved the data was there
- so the problem is on the Wireshark side and not the file

## 3: The HTTP dissector was turned off in settings

- Wireshark has a setting (Analyze, Enabled Protocols) where individual protocols can be switched off, if HTTP was unchecked there, Wireshark would see the raw data but refuse to label it as HTTP
- but i opened Analyze --> Enabled Protocols, searched for HTTP, and made sure the checkbox was ticked

## 4: Confirming  with an exact known packet

- instead of testing with a generic filter, i tested with the one specific packet (frame 1880) that we already knew, from the tshark side, was genuinely HTTP
- filtered `frame.number == 1880` and looked at what Wireshark labeled that packet as
- it showed as plain TCP, not HTTP, even though this exact packet was confirmed to be an HTTP request moments earlier using tshark on the same file, this means that Wireshark's HTTP recognition was broken for this session, not that the data was missing or the filter was wrong.

## 5: Port not mapped to HTTP, try manual mapping

- maybe Wireshark just wasn't automatically associating port 80 traffic with the HTTP dissector, and forcing it manually (Decode As) would work as a quick per-session fix.
- so i right-clicked frame 1880, chose Decode As, and looked for HTTP in the list of protocols to manually assign.
- HTTP wasn't even available as an option in that list
- this is not just a port-mapping issue, the HTTP dissector wasn't just misconfigured, it appeared to not be properly available at all in the current session.

## A new configuration profile

Wireshark stores all its settings (enabled protocols, preferences, column setup, filters, colors) in a folder called a "profile." 
If that folder gets corrupted(a setting gets written incorrectly, a plugin fails to load, a preference file becomes malformed) individual features can break, even ones that look fine when we check their checkbox.

so i created a brand new, completely default configuration profile (Edit, Configuration Profiles, click +), switched to it, and reopened the same pcap file.

so, the original Wireshark profile (its saved settings folder) had become corrupted in a way that specifically broke HTTP protocol recognition, while everything else (file loading, packet counts, other protocols, the filter syntax engine itself) kept working normally. 

Switching to a new profile bypassed the corrupted settings entirely and let Wireshark's default, working HTTP dissector take over again.

So if a specific protocol filter returns zero results in Wireshark, but an independent check (another tool, or a known specific packet) proves the data is there, the fastest thing to test early is a brand new configuration profile, fast, effective, i recommend.
