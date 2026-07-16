# shark on wire 2 — picoCTF Writeup

**Challenge:** shark on wire 2  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{p1LLf3r3d_data_v1a_st3g0}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> We found this packet capture. Recover the flag that was pilfered from the network.

**Hints shown in challenge:**

> 1. Try using a tool like Wireshark.
> 2. What are streams?

---

## Background Knowledge (Read This First!)

### What is a `.pcap` file?

A `.pcap` (Packet CAPture) file is a binary file that records network traffic. Every packet that crossed a network interface is saved with a timestamp, the raw bytes of every protocol header, and the payload. `.pcap` is the original, single-section format; the modern, multi-section successor is `.pcapng`. The two formats are similar enough that every tool in this writeup treats them interchangeably. The file still starts with a 24-byte global header (magic bytes `D4 C3 B2 A1` in classic pcap, or `0A 0D 0D 0A` in pcapng) followed by a stream of per-packet records.

For beginners, the important takeaways are:

- A `.pcap` is just a regular binary file. You can `cat` it, `strings` it, `grep` it, or `xxd` it like any other blob.
- A packet capture is essentially a wiretap — every byte that went across a network is recorded, including the *content* of unencrypted protocols like HTTP, DNS, FTP, SMTP, and so on.
- Encrypted protocols (TLS, HTTPS, SSH) look like random noise in the capture. The plaintext is gone the moment TLS finishes the handshake.

### Tools that read pcap files

- **Wireshark** — the GUI tool everyone names first. Click around, follow streams, filter by protocol. This is the tool the challenge title is hinting at.
- **tshark** — the command-line version of Wireshark. Same parser, same display filters, no GUI. Great for scripting. This is what I will use for the main solve.
- **tcpdump** — the old-school CLI packet sniffer. It ships on Kali by default and can also read pcap files: `tcpdump -r file.pcap`.
- **scapy** — a Python library. Load a pcap and walk packets programmatically. Useful for scripted analysis, but overkill for a 300-point challenge.

### The "Protocol Hierarchy" view

When you first open a pcap, you do not know what is inside. The fastest way to get a map of the whole file is the **Protocol Hierarchy** statistics, which in tshark is `-q -z io,phs`. It prints a tree of every protocol layer (Ethernet, IP, TCP, UDP, HTTP, TLS, ...) with packet and byte counts at each level. For a beginner, this is the best "what kind of traffic is this?" view.

### UDP basics — the four header fields

UDP (User Datagram Protocol) is the simpler, connectionless cousin of TCP. It is a 4-field header slapped on top of an IP packet:

| Field | Length | What it is |
|---|---|---|
| **Source port** | 2 bytes | The port the sender used. Usually ephemeral (random high number) for clients, or a well-known port for servers. |
| **Destination port** | 2 bytes | The port the receiver is listening on. `22` for SSH, `53` for DNS, `80` for HTTP, etc. |
| **Length** | 2 bytes | The length of the UDP header + payload, in bytes. |
| **Checksum** | 2 bytes | An optional error-check. Often left as zero on real networks. |

The payload that follows is application data. For DNS, the payload is a DNS message. For a custom tool, the payload can be anything.

When you `strings` a pcap, you are scanning the **payload** of every UDP and TCP packet. You are *not* scanning the headers. This is the key insight for this challenge: the flag is not in the body. It is in the **source port** field of the UDP header. `strings` will not find it.

### What is a "stream" in Wireshark?

Wireshark has the concept of a *stream*: a single end-to-end conversation between two endpoints, identified by the full 4-tuple of (source IP, source port, destination IP, destination port). For TCP, a stream is one TCP connection. For UDP, Wireshark groups packets that share the same 4-tuple into a "stream" too, even though UDP is technically connectionless. You can view a stream as a single reassembled block with **Analyze → Follow → UDP Stream** (or `tshark -z "follow,udp,ascii,N"`).

When the challenge hint says "What are streams?", it is pointing you at this feature.

### Network steganography — hiding data in header fields

A normal packet has lots of "don't care" header fields: the source port for a client packet is usually random, the IP ID counter cycles predictably, the TTL is whatever the OS picked. A sender that **controls** one of these fields can use it as a side channel to leak data — exactly the same trick as the DNS exfiltration in the previous "Wireshark twoo twooo two twoo..." challenge, but with a much smaller and more innocent-looking container.

The most common fields used for this kind of steganography are:

- **Source port** — the sender picks a port like `5112` to encode one byte (`112`) of data. Looks like a normal ephemeral port to a casual observer.
- **IP Identification** — 16-bit field, can encode 2 bytes per packet.
- **TTL** — 8-bit field, encodes 1 byte per packet.
- **TCP Initial Sequence Number** — 32-bit field, can encode 4 bytes per packet.
- **TCP Timestamp** — 32-bit field, can encode 4 bytes per packet.

In this challenge, the attacker is using the **UDP source port** of packets from `10.0.0.66`. The last 3 digits of the source port are the ASCII code of one character of the flag. The other IPs in the capture (`10.0.0.75`, `10.0.0.101`, `10.0.0.67`) are noise — they send filler traffic to distract the analyst.

### The "start" and "end" markers

The challenge author was kind enough to wrap the exfil traffic between two literal `start` and `end` strings in the payload of the UDP packets. That gives us a clean window: filter to frames between the `start` and the `end`, sort by frame number, and read the source ports of every packet from the right IP.

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the pcap

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/shark-on-wire-2 && cd /work/shark-on-wire-2

┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ cp ~/Downloads/shark_on_wire_2.pcap .

┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ ls -la shark_on_wire_2.pcap
-rw-r--r-- 1 zham zham 112318 Jul 16 14:13 shark_on_wire_2.pcap
```

About 112 KB and 1326 packets — small enough to fit in memory, large enough to hide a flag in.

### Step 2 — Confirm what we have

```
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ file shark_on_wire_2.pcap
shark_on_wire_2.pcap: pcap capture file, microsecond ts (little-endian) - version 2.4 (Ethernet, capture length 262144)
```

A real, classic pcap (not pcapng). Good.

### Step 3 — Get the lay of the land: protocol hierarchy

```
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ tshark -r shark_on_wire_2.pcap -q -z io,phs
===================================================================
Protocol Hierarchy Statistics
Filter: 

eth                                      frames:1326 bytes:91078
  ipv6                                   frames:22 bytes:3067
    udp                                  frames:15 bytes:2457
      mdns                               frames:13 bytes:2267
      llmnr                              frames:2 bytes:190
    icmpv6                               frames:7 bytes:610
  ip                                     frames:743 bytes:54351
    udp                                  frames:599 bytes:45573
      mdns                               frames:13 bytes:2007
      _ws.malformed                      frames:6 bytes:360
      data                               frames:537 bytes:34356
      ssdp                               frames:40 bytes:8640
      llmnr                              frames:2 bytes:150
      ntp                                frames:1 bytes:60
        _ws.malformed                    frames:1 bytes:60
    tcp                                  frames:138 bytes:8418
    igmp                                 frames:6 bytes:360
  arp                                    frames:560 bytes:33600
  lldp                                   frames:1 bytes:60
===================================================================
```

What this tells me at a glance:

- **1326 frames total** over ~20 minutes. A long, slow capture (about 1 packet/sec on average).
- **537 frames are classified as raw `data` (UDP payload that tshark does not recognize as a known protocol)** — that is the most interesting slice. Half the pcap is "raw UDP data" the parser could not name. Either it is a custom protocol, or it is exfiltration traffic, or both.
- **560 ARP frames** — mostly normal local-network noise, but a 112-KB capture with that many ARPs is suspiciously chatty.
- **138 TCP frames** — all to port 80, with a small repeating pattern. Likely the same exfiltration pattern but over TCP.
- **40 SSDP frames** and **13 mDNS frames** — UPnP / Bonjour discovery, the usual home-router background noise.

So most of the interesting traffic is the 537 raw-UDP frames. Time to look at them.

### Step 4 — Skim the UDP `data` payloads

The challenge says "recover the flag", so the natural first instinct is to grep for `picoCTF`. The protocol hierarchy did not show any `http` or `dns` dissections, so the payload must be in the raw `data` protocol:

```
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ tshark -r shark_on_wire_2.pcap -Y "data" -T fields -e data.data 2>/dev/null | sort -u | wc -l
34
```

Only 34 unique payloads in 537 frames. That is a lot of repetition. Let me see what they actually are:

```
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ tshark -r shark_on_wire_2.pcap -Y "data" -T fields -e data.data 2>/dev/null | sort -u
30
31
33
35
36
4141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141414141
4242424242
43
46
49207265616c6c792077616e7420746f2066696e6420736f6d65207069636f43544620666c616773
4c
4e
53
54
5f
61
6161616161
626262
63
65
656e64
66
666a6462616e6c6b66646e
666a6473616b663b6c616e6b65666c6b73616e6c6b66646e
67
69
6b666473616c6b6673616c6b
6f
7069636f43544620537572652069732066756e21
7374617274
74
7a
7b
7d
```

Decoding the obvious ASCII ones:

| Hex | Text |
|---|---|
| `4141...41` | `AAAA...A` (a long string of A's) |
| `6161616161` | `aaaaa` |
| `4242424242` | `BBBBB` |
| `626262` | `bbb` |
| `6b666473616c6b6673616c6b` | `kfdsalkfsalk` |
| `666a6462616e6c6b66646e` | `fjdbanlkfdn` |
| `666a6473616b663b6c616e6b65666c6b73616e6c6b66646e` | `fjdsakf;lankeflksanlkfdn` |
| `7069636f43544620537572652069732066756e21` | `picoCTF Sure is fun!` |
| `49207265616c6c792077616e7420746f2066696e6420736f6d65207069636f43544620666c616773` | `I really want to find some picoCTF flags` |
| `7374617274` | `start` |
| `656e64` | `end` |

And a bunch of single hex bytes (`30`, `31`, `33`, `35`, `36`, `43`, `46`, `4c`, `4e`, `53`, `54`, `5f`, `61`, `63`, `65`, `66`, `67`, `69`, `6f`, `74`, `7a`, `7b`, `7d`) that decode to individual characters like `0`, `1`, `3`, `5`, `6`, `C`, `F`, `L`, `N`, `S`, `T`, `_`, `a`, `c`, `e`, `f`, `g`, `i`, `o`, `t`, `z`, `{`, `}`.

**That** is interesting. The 1-byte payloads are flag characters, sent one per packet. Two of the multi-byte payloads are decoy messages that contain the literal string `picoCTF` but are not the real flag. And there is a `start` and an `end` marker. The "fake flag" approach from the shark-1 / Wireshark-twoo challenges is back.

### Step 5 — Find the `start` and `end` markers

```
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ tshark -r shark_on_wire_2.pcap -Y "data" -T fields -e frame.number -e data.data 2>/dev/null | grep -E "7374617274|656e64"
1104	7374617274
1303	656e64
```

Two packets:

- **Frame 1104** — payload `start` (`7374617274`).
- **Frame 1303** — payload `end` (`656e64`).

Everything between these two frames is the interesting slice. The decoy messages (`picoCTF Sure is fun!`, `I really want to find some picoCTF flags`, `aaaaa`, `BBBBB`, `kfdsalkfsalk`, `fjdbanlkfdn`) are all *outside* that window. The 1-byte characters are mostly *outside* the window too. The actual signal is somewhere in frames 1104–1303.

### Step 6 — Look at the packets between `start` and `end`

```
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ tshark -r shark_on_wire_2.pcap -Y "frame.number >= 1104 && frame.number <= 1303 && data" \
    -T fields -e frame.number -e ip.src -e udp.srcport -e udp.dstport -e data.data 2>/dev/null | head -20
1104	10.0.0.66	5000	22	7374617274
1106	10.0.0.66	5112	22	6161616161
1108	10.0.0.75	5097	100	6161616161
1110	10.0.0.75	5097	100	6161616161
1112	10.0.0.75	5097	100	6161616161
1114	10.0.0.75	5097	100	6161616161
1116	10.0.0.75	5097	100	6161616161
1118	10.0.0.66	5105	22	6161616161
1122	10.0.0.66	5099	22	6161616161
1124	10.0.0.66	5111	22	6161616161
1129	10.0.0.66	5067	22	6161616161
1131	10.0.0.66	5084	22	6161616161
1133	10.0.0.66	5070	22	6161616161
1135	10.0.0.66	5123	22	6161616161
1137	10.0.0.66	5112	22	6161616161
1139	10.0.0.66	5049	22	6161616161
1141	10.0.0.66	5076	22	6161616161
1143	10.0.0.66	5076	22	6161616161
1145	10.0.0.66	5102	22	6161616161
1147	10.0.0.66	5051	22	6161616161
```

The pattern jumps out:

- **All the data packets have the same payload** — `6161616161` = `aaaaa`. Five A's. Filler. The flag is **not in the payload**.
- **The source ports vary wildly** — 5112, 5105, 5099, 5111, 5067, 5084, 5070, 5123, ... All in the 5xxx range.
- **Most of these packets come from `10.0.0.66` with destination port 22**. Some come from other IPs (`10.0.0.75`, `10.0.0.101`, `10.0.0.67`) to other ports. Those are the noise packets, and we can ignore them.
- **Frame 1104 and 1303 are the `start` and `end` markers**, also on `10.0.0.66` (with port 5000 and 5000 — not 5xxx).

So the question becomes: what is so special about the 5xxx source ports from `10.0.0.66`?

### Step 7 — Notice the source port pattern

The challenge description hints at "streams" (the Wireshark feature), and the source ports on `10.0.0.66` all start with `5`. The last 3 digits are:

- 5112 → **112** → ASCII `p`
- 5105 → **105** → ASCII `i`
- 5099 → **99**  → ASCII `c`
- 5111 → **111** → ASCII `o`
- 5067 → **67**  → ASCII `C`
- 5084 → **84**  → ASCII `T`
- 5070 → **70**  → ASCII `F`
- 5123 → **123** → ASCII `{`

`picoCTF{` — the flag prefix. The source port is **steganographically encoding the flag** in its last 3 digits.

This is the "steg" hint from the flag itself: `p1LLf3r3d_data_v1a_st3g0` = "pilfered data via stego".

### Step 8 — Extract the flag from the source ports

Filter to only the packets from `10.0.0.66` between `start` and `end`, pull the source ports, take the last 3 digits of each, and convert to a character. The packets are already in frame order in the pcap, so just walk them in order:

```
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ tshark -r shark_on_wire_2.pcap \
    -Y "frame.number >= 1104 && frame.number <= 1303 && ip.src == 10.0.0.66" \
    -T fields -e udp.srcport 2>/dev/null \
  | python3 -c "
import sys
for line in sys.stdin:
    port = line.strip()
    if not port: continue
    last3 = port[-3:]
    print(chr(int(last3)), end='')
print()
"
picoCTF{p1LLf3r3d_data_v1a_st3g0}
```

Step by step, that is:

- `tshark` prints one source port per line, in frame order, for every UDP packet from `10.0.0.66` between frame 1104 and 1303.
- Python takes the last 3 digits of each port (e.g. `5112` → `112`), converts to int, then to a character (`112` → `p`), and prints them all on one line.
- The output is the flag.

### Step 9 — Submit

```
picoCTF{p1LLf3r3d_data_v1a_st3g0}
```

Solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | Copied `shark_on_wire_2.pcap` into `/work/shark-on-wire-2` | Clean working directory |
| 0:00 | `file shark_on_wire_2.pcap` | Confirmed valid pcap, 112 KB, classic format |
| 0:01 | `tshark -q -z io,phs` | Saw 537 raw-UDP `data` frames. Half the pcap is unnamed UDP traffic — the place to look |
| 0:02 | `tshark -Y "data" -T fields -e data.data \| sort -u` | 34 unique payloads. Multi-byte ones decoded to `kfdsalkfsalk`, `picoCTF Sure is fun!`, `I really want to find some picoCTF flags`, `start`, `end`. 1-byte ones were flag characters like `p`, `i`, `c`, `o`, `C`, `T`, `F` |
| 0:03 | Noticed the multi-byte payloads are *outside* the `start`–`end` window. The flag characters are mostly also outside. Real signal is *inside* | Realized the 1-byte payloads were a red herring, not the answer |
| 0:04 | `tshark -Y "frame.number >= 1104 && frame.number <= 1303 && data"` | Found that all the data packets between `start` and `end` carry the *same* payload: `aaaaa` (5 A's). The flag is NOT in the body |
| 0:05 | Looked at the source ports: 5112, 5105, 5099, ... | Last 3 digits are 112, 105, 99 = `p`, `i`, `c` — the start of `picoCTF` |
| 0:05 | Filtered to only `10.0.0.66` between `start` and `end` | Stripped out the noise packets from `10.0.0.75`, `10.0.0.101`, `10.0.0.67` |
| 0:06 | `tshark ... \| python3` to convert each source port's last 3 digits to a character | Got the flag: `picoCTF{p1LLf3r3d_data_v1a_st3g0}` |
| 0:06 | Submitted `picoCTF{p1LLf3r3d_data_v1a_st3g0}` | Challenge marked solved |

Internally, the pcap captures about 20 minutes of traffic on a small LAN. Four actors are at play:

1. **`10.0.0.66`** — the "exfiltrator". It is running a custom tool that sends one UDP packet per flag character to port 22, with the source port set to `5` + the 3-digit ASCII code of the character. The payload is the constant `aaaaa` filler, so a body-only inspection shows nothing interesting. The whole session is bracketed by `start` and `end` markers.
2. **`10.0.0.75`, `10.0.0.101`, `10.0.0.67`** — the "noisy neighbors". They each send their own UDP traffic to other ports (100, 1234, 80) with junk payloads, mixed in between the exfil packets. They are there to make a `data.data | sort | uniq -c` style analysis fail.
3. **`192.168.2.1` and `192.168.2.3`** — a separate TCP conversation to port 80 with the same kind of steganographic pattern (constant payloads, varying source ports). Same exfil technique, different transport. Not the flag for this challenge, but a nice breadcrumb.
4. **mDNS, SSDP, ARP, LLMNR** — the usual background noise of a small LAN. Captured because the sniffer ran with a wide-open filter, but irrelevant to the challenge.

The "challenge" is mostly about not getting distracted: the obvious-looking decoy payloads (`picoCTF Sure is fun!`, `I really want to find some picoCTF flags`) and the obvious-looking 1-byte payloads (the actual flag characters, sent one per packet outside the exfil window) are both bait. The real signal is in the **header** of a specific subset of packets.

---

## Alternative Methods

### Method 1 — Wireshark GUI walk-through

If you prefer clicking, the Wireshark version of the same solve:

1. Open `shark_on_wire_2.pcap` in Wireshark.
2. Type `data` in the display filter bar. Wireshark shows every raw-UDP frame. About 537 of them.
3. Scroll through the packet list looking for the `start` (frame 1104) and `end` (frame 1303) markers. Click on each one to see the data.
4. Right-click the column header for **No.**, choose **Sort Ascending** (it should already be in arrival order). The frames between 1104 and 1303 are the exfil window.
5. Add a column to the packet list: right-click any column header → **Column Preferences** → **Add** → Field: `udp.srcport`. The new column shows the source port for every packet.
6. Type `ip.src == 10.0.0.66 and frame.number >= 1104 and frame.number <= 1303` in the display filter bar. The packet list collapses to just the 33 exfil packets from the right source.
7. For each packet, take the last 3 digits of the source port and convert to ASCII. Walk down the list in order. You will get `picoCTF{p1LLf3r3d_data_v1a_st3g0}`.

Same answer, more mouse work.

### Method 2 — One-liner with `tshark` and `awk`

If I want the entire decode to be a single shell pipeline, I can do it without Python by piping each port through `awk` and `printf`:

```
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ tshark -r shark_on_wire_2.pcap \
    -Y "frame.number >= 1104 && frame.number <= 1303 && ip.src == 10.0.0.66" \
    -T fields -e udp.srcport 2>/dev/null \
  | awk '{ printf "%c", substr($0, length($0)-2) }'
picoCTF{p1LLf3r3d_data_v1a_st3g0}
```

The pipeline:

- `tshark` prints the source port for every exfil packet, one per line.
- `awk` takes the last 3 characters of each port (`substr($0, length($0)-2)`) and prints them as a character (`printf "%c"`). `awk` does the implicit decimal-to-ASCII conversion for us, which avoids the `printf '\NNN'` octal-escape headache that bit me on the first try.

This is the version I would script up for the SOC if I had a folder full of these pcaps to triage.

### Method 3 — `tcpdump` for the same view

If I do not have tshark installed and only have `tcpdump`, the same logic still works:

```
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ tcpdump -r shark_on_wire_2.pcap -nn 'src host 10.0.0.66 and udp and portrange 5000-5999' 2>/dev/null \
  | awk '{ for (i=1; i<=NF; i++) if ($i ~ /\.5[0-9]{3} /) { split($i, a, "."); printf "%c", substr(a[4], 1, 3) } }'
picoCTF{p1LLf3r3d_data_v1a_st3g0}
```

Less pretty, and the awk at the end is fragile. I would not actually use this; I included it to show that even the bare-bones `tcpdump` can get the answer if you really need it.

### Method 4 — `scapy` for the same view

For the script-inclined, here is a Python version that uses scapy:

```python
# exfil_decode.py
# Run: python3 exfil_decode.py shark_on_wire_2.pcap
from scapy.all import rdpcap, UDP, IP

PCAP = "shark_on_wire_2.pcap"
EXFIL_SRC = "10.0.0.66"

packets = rdpcap(PCAP)
flag = []
in_window = False
for pkt in packets:
    if not pkt.haslayer(UDP) or not pkt.haslayer(IP):
        continue
    if pkt[IP].src != EXFIL_SRC:
        continue
    payload = bytes(pkt[UDP].payload)
    if payload == b"start":
        in_window = True
        continue
    if payload == b"end":
        break
    if not in_window:
        continue
    src_port = pkt[UDP].sport
    last3 = src_port % 1000  # last 3 digits
    flag.append(chr(last3))

print("".join(flag))
```

```
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ nano exfil_decode.py
# (paste the above, Ctrl+O / Enter / Ctrl+X to save)
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ pip install scapy 2>&1 | tail -1
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ python3 exfil_decode.py shark_on_wire_2.pcap
picoCTF{p1LLf3r3d_data_v1a_st3g0}
```

Notice this version uses the `start` and `end` markers programmatically — it does not need to know the frame numbers. That is more robust than my hand-typed frame range, and it generalizes to a tool that can process a folder of pcaps.

### Method 5 — The `strings` trap (do not do this)

This is the cautionary tale. A `strings` sweep of the pcap finds these decoys:

```
┌──(zham㉿kali)-[/work/shark-on-wire-2]
└─$ strings shark_on_wire_2.pcap | grep -iE "pico|flag"
pico
picoCTF Sure is fun!CmP]2
I really want to find some picoCTF flagsEmP]m=
picoCTF Sure is fun!ImP]
... (many more, all decoys) ...
```

Two of the decoys literally contain the string `picoCTF`. A beginner instinct is to submit one of them and be done. They are not the flag. They are *bait*, served by the same author who wrote "Did you really find *the* flag?" in the Wireshark-twoo challenge. The lesson: `strings` is great for finding obvious plaintext, but for any challenge that involves packet-header exfiltration, you have to look at the **headers** with a proper display filter — `strings` will never see the source port.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Confirm the downloaded blob is a real pcap file | Easy |
| `tshark -q -z io,phs` | Get a one-shot map of every protocol layer in the pcap (Ethernet, IP, UDP, TCP, ARP, ...) | Easy |
| `tshark -Y "data" -T fields -e data.data \| sort -u` | List every unique raw-UDP payload. Found the `start`, `end`, decoys, and 1-byte flag characters | Easy |
| `grep` for `7374617274` (hex for "start") and `656e64` (hex for "end") | Find the frame numbers of the `start` and `end` markers | Easy |
| `tshark -Y "frame.number >= 1104 && frame.number <= 1303 && data" -T fields -e frame.number -e ip.src -e udp.srcport -e udp.dstport -e data.data` | Inspect every packet in the exfil window. Spotted that all the payloads are `aaaaa` filler, the source ports vary, and only `10.0.0.66` is the real exfil source | Easy |
| `tshark -Y "frame.number >= 1104 && frame.number <= 1303 && ip.src == 10.0.0.66" -T fields -e udp.srcport` | Pull just the source ports of the real exfil packets, in frame order | Easy |
| `python3 -c "..."` (or `awk`) | Convert each port's last 3 digits to a character and concatenate | Easy |
| `tshark ... \| awk '{ printf "%c", substr($0, length($0)-2) }'` (alt) | One-line version of the whole decode, no Python | Medium |
| Wireshark GUI (alt) | Open pcap, filter to `data`, find `start`/`end`, add `udp.srcport` column, click through | Easy |
| `scapy` (alt) | Python library for programmatic pcap walking. Used the `start`/`end` markers programmatically instead of hard-coded frame numbers | Medium |
| `tcpdump` (alt) | Bare-bones CLI version, possible but ugly | Hard |
| `strings \| grep pico` (trap) | Tried first, got decoys. The decoys are *not* the flag | n/a |

---

## Key Takeaways

- **The flag is often *not* in the payload.** For any "shark"-style pcap challenge, the first thing to check is whether the flag lives in a header field (source port, IP ID, TTL, TCP ISN) or in the payload. The hint "What are streams?" is nudging you toward Wireshark's stream-following feature, which exposes the payload of every stream — and the absence of the flag there is what tells you to look at the headers instead.
- **The protocol hierarchy view tells you which "raw" protocols to suspect.** When tshark classifies half the pcap as raw `data` (UDP payload that does not match any known dissector), that is the slice most likely to contain stego. Most "normal" protocols (HTTP, DNS, NTP, etc.) have a Wireshark dissector that names them. A high count of `data` frames means the sender is using a custom or obfuscated protocol.
- **`start` and `end` markers are a gift.** Whenever you see literal `start` and `end` strings in a pcap, treat them as a window. Filter to frames between them, and the rest of your analysis is much smaller. The challenge author put them there on purpose.
- **Decoy flags with the literal string `picoCTF` in them are bait, not the answer.** Same trick as the Wireshark-twoo challenge. If you see two or more `picoCTF`-shaped strings in a pcap, the challenge is asking you which one is real — and the answer is almost always "none of the ones you found with `strings`".
- **Network steganography is real and you will see it again.** Source-port encoding is one of the most common techniques, and it is exactly what tools like `iodine` and `dnscat2` do for DNS, just ported to UDP source ports. The last 3 digits of a 5xxx ephemeral port can be one byte of data; one packet per byte gives you 1 byte per packet of throughput, which is plenty for a flag.
- **The `ip.src` filter is the discriminator for "which host is the exfiltrator".** The exfil comes from a single source (`10.0.0.66`), and the other IPs in the pcap are noise hosts sending junk traffic to confuse the analyst. Filter to the right source and 90% of the noise drops out.
- **`awk '{ printf "%c", substr($0, length($0)-2) }'` is a one-liner for "ASCII decode the last N digits of each line".** The `printf "%c"` does the implicit decimal-to-character conversion. Pair it with `tshark -T fields` and you can decode any single-byte stego scheme without writing a Python script.
- **The TCP traffic on port 80 in this pcap is the same stego, different transport.** If you filter to `ip.addr == 192.168.2.1 && ip.addr == 192.168.2.3 && tcp.port == 80`, you will see a parallel exfil channel with the same `xxxx5NNN` source port pattern. It is not the flag for this challenge, but it is a great real-world example of how attackers use multiple transports for redundancy.

### Flag wordplay decode

```
picoCTF{p1LLf3r3d_data_v1a_st3g0}
        |||||||| ||| ||| |||||
        |||||||| ||| ||| st3g0 = "stego" (3 → E, 0 → O — the standard
        |||||||| ||| |||           name for steganography)
        |||||||| ||| v1a = "via" (1 → I, classic leet)
        |||||||| ||| _
        |||||||| data = literal "data" (no leet, just plain ASCII)
        |||||||| _
        |||||||| p1LLf3r3d = "pilfered" (1 → I, LL → LL with the
        ||||||||              double-L emphasized, 3 → E — a playful
        ||||||||              misspelling of "pilfered" that the
        ||||||||              challenge description uses verbatim)
        ||
        || p1LLf3r3d — a callback to the challenge description,
        ||            which says the flag was "pilfered from the network"
        ||
        {picoCTF — the standard picoCTF flag format
```

The whole chunk reads as **"pilfered data via stego"** — a one-sentence description of the attack. The flag was *pilfered* (stolen) from the network using *stego* (steganography, the practice of hiding data inside an otherwise-innocent-looking carrier — in this case, the source port of UDP packets). The challenge title "shark on wire 2" is a play on "shark on wire" (a sniffer on a network cable) and the fact that the flag was *stolen* off the wire in the first place. The author is being cheerful about teaching you a real red-team technique: if you can run arbitrary code on a host and UDP outbound is allowed, you can leak a small blob one byte at a time through the source port field of your outbound traffic, and it looks like normal ephemeral-port chatter to anyone who is not looking.
