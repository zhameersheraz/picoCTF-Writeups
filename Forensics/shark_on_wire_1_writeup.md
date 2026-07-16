# shark on wire 1 — picoCTF Writeup

**Challenge:** shark on wire 1  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 150  
**Flag:** `picoCTF{StaT31355_636f6e6e}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> We found this packet capture. Recover the flag.

**Hints shown in challenge:**

> 1. Try using a tool like Wireshark.
> 2. What are streams?

**Attachment:** `capture.pcap`

---

## Background Knowledge (Read This First!)

### What is a `.pcap` file?

A `.pcap` (Packet CAPture) file is a binary file that records raw network traffic. Every packet that crossed a network interface is saved with a timestamp, the raw bytes of every protocol header (Ethernet, IP, UDP, TCP, ...), and the payload. `.pcap` is the classic, single-section format. The modern multi-section format is `.pcapng`, but the two are close enough that every tool in this writeup treats them the same.

For beginners, the important takeaways are:

- A `.pcap` is just a regular binary file. You can `cat` it, `strings` it, `grep` it, or `xxd` it like any other blob.
- A packet capture is essentially a wiretap — every byte that went across the network is recorded, including the content of unencrypted protocols.
- Encrypted protocols (TLS, HTTPS, SSH) look like random noise in the capture. The plaintext is gone the moment TLS finishes the handshake.

### Tools that read pcap files

- **Wireshark** — the GUI tool everyone names first. Click around, follow streams, filter by protocol. This is the tool the challenge title is hinting at.
- **tshark** — the command-line version of Wireshark. Same parser, same display filters, no GUI. Great for scripting. This is what I will use for the main solve.
- **tcpdump** — the old-school CLI packet sniffer. It ships on Kali by default and can also read pcap files with `tcpdump -r file.pcap`.
- **scapy** — a Python library. Load a pcap and walk packets programmatically. Useful when you want to script the whole thing.

### The "Protocol Hierarchy" view

When you first open a pcap, the fastest way to get a map of the whole file is the **Protocol Hierarchy** statistics, which in tshark is `-q -z io,phs`. It prints a tree of every protocol layer (Ethernet, IP, TCP, UDP, HTTP, TLS, ...) with packet and byte counts at each level. For a beginner, this is the best "what kind of traffic is this?" view.

### UDP basics

UDP (User Datagram Protocol) is the simpler, connectionless cousin of TCP. It is a 4-field header slapped on top of an IP packet:

| Field | Length | What it is |
|---|---|---|
| **Source port** | 2 bytes | The port the sender used. Usually ephemeral (random high number) for clients. |
| **Destination port** | 2 bytes | The port the receiver is listening on. `22` for SSH, `53` for DNS, `80` for HTTP, etc. |
| **Length** | 2 bytes | The length of the UDP header + payload, in bytes. |
| **Checksum** | 2 bytes | An optional error-check. Often left as zero on real networks. |

The payload that follows is application data. For DNS, the payload is a DNS message. For a custom tool, the payload can be anything at all.

### What is a "stream" in Wireshark?

Wireshark has the concept of a *stream*: a single end-to-end conversation between two endpoints, identified by the full 4-tuple of (source IP, source port, destination IP, destination port). For TCP, a stream is one TCP connection. For UDP, Wireshark groups packets that share the same 4-tuple into a "stream" too, even though UDP is technically connectionless. You can view a stream as a single reassembled block with **Analyze → Follow → UDP Stream** (or `tshark -q -z "follow,udp,ascii,N"` where `N` is the stream number).

When the second challenge hint says "What are streams?", it is pointing you at this feature. The flag is hidden as the payload of a specific UDP stream, and the only way to see the full payload is to follow that stream.

### Why "UDP" matters here

The challenge is called "shark on wire" because you are a shark (Wireshark) sniffing packets off a wire (the network). The flag is not hiding in HTTP, DNS, or any other named protocol — it is hiding in a custom UDP stream. That is why the hint nudges you to look at "streams" instead of "URLs" or "DNS queries".

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the pcap

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/shark-on-wire-1 && cd /work/shark-on-wire-1

┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ cp ~/Downloads/capture.pcap .

┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ ls -la capture.pcap
-rw-r--r-- 1 zham zham 239455 Jul 16 14:21 capture.pcap
```

About 234 KB and 2317 packets. Small enough to fit in memory, large enough to be interesting.

### Step 2 — Confirm what we have

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ file capture.pcap
capture.pcap: pcap capture file, microsecond ts (little-endian) - version 2.4 (Ethernet, capture length 262144)
```

A real, classic pcap (not pcapng). Good.

### Step 3 — Get the lay of the land: protocol hierarchy

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ tshark -r capture.pcap -q -z io,phs
===================================================================
Protocol Hierarchy Statistics
Filter: 

eth                                      frames:2317 bytes:202359
  ip                                     frames:1230 bytes:113479
    udp                                  frames:1140 bytes:107983
      ssdp                               frames:128 bytes:27137
      nbdgm                              frames:7 bytes:1731
        smb                              frames:7 bytes:1731
          mailslot                       frames:7 bytes:1731
            browser                      frames:7 bytes:1731
      llmnr                              frames:205 bytes:13924
      data                               frames:749 bytes:60996
      _ws.malformed                      frames:4 bytes:240
      mdns                               frames:47 bytes:3955
    tcp                                  frames:22 bytes:1408
    igmp                                 frames:68 bytes:4088
  ipv6                                   frames:347 bytes:44534
    udp                                  frames:280 bytes:38424
      llmnr                              frames:205 bytes:18024
      mdns                               frames:47 bytes:4895
      ssdp                               frames:7 bytes:1099
      data                               frames:21 bytes:14406
    icmpv6                               frames:67 bytes:6110
  arp                                    frames:738 bytes:44226
  lldp                                   frames:2 bytes:120
===================================================================
```

What this tells me at a glance:

- **2317 frames total** over about 38 minutes. A long, slow capture (about 1 packet/sec on average).
- **749 frames are classified as raw `data` (UDP payload that tshark does not recognize as a known protocol)** — that is the most interesting slice. A third of the pcap is "raw UDP data" the parser could not name. Either it is a custom protocol, or it is exfiltration traffic, or both.
- **128 SSDP, 205 LLMNR, 47 mDNS** — UPnP / LLMNR / Bonjour discovery, the usual home-router background noise.
- **738 ARP and 68 IGMP** — typical local-network noise.
- **22 TCP frames** — almost nothing. TCP is not where the flag lives.
- **No HTTP, no DNS, no FTP** — none of the "usual" plaintext protocols. The flag is in the custom UDP data.

So most of the interesting traffic is the 749 raw-UDP `data` frames. Time to look at them.

### Step 4 — Follow the UDP streams one by one

The hint says "What are streams?" so the next move is to follow each UDP stream and see what it contains. With tshark, the command is `tshark -r capture.pcap -q -z "follow,udp,ascii,N"` where `N` is the stream index. I can also browse the whole list in Wireshark with **Analyze → Follow → UDP Stream**, but the CLI is faster when there are dozens of streams to scan.

First, let me see how many UDP streams there are:

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ tshark -r capture.pcap -Y "udp" -T fields -e udp.stream 2>/dev/null | sort -un | wc -l
289
```

289 UDP streams. Way too many to read one by one, but the hint says the answer is "streams", so let me just narrow down. Most of them are LLMNR / mDNS / SSDP noise with empty or trivial payloads. The interesting ones should be the ones going to unusual ports (not 53, 80, 1900, 5353, 5355).

Let me filter to UDP streams with non-empty raw payloads on unusual destination ports:

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ tshark -r capture.pcap -Y "udp and data" -T fields -e udp.stream -e ip.dst -e udp.dstport 2>/dev/null | sort -u
4	10.0.0.5	8990
6	10.0.0.12	8888
7	10.0.0.13	8888
8	10.0.0.15	8888
9	10.0.0.19	8990
10	10.0.0.22	8990
15	10.0.0.5	8990
17	10.0.0.88	8990
19	10.0.0.25	8990
21	10.0.0.5	8990
28	10.0.0.3	8990
... (and more to port 8990 with garbage payloads)
```

Out of 17 distinct streams that carry raw UDP `data`, only three are on port 8888. The rest are on port 8990 with random-looking payloads (garbage / decoys / noise). Port 8888 is the only one that even resembles a flag, and the three 8888 streams are the only candidates. Let me follow each one.

### Step 5 — Follow stream 6 (the real flag)

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ tshark -r capture.pcap -q -z "follow,udp,ascii,6" 2>/dev/null
===================================================================
Follow: udp,ascii
Filter: udp.stream eq 6
Node 0: 10.0.0.2:5000
Node 1: 10.0.0.12:8888
27
picoCTF{StaT31355_636f6e6e}
===================================================================
```

There it is. The flag is being sent one byte per UDP packet from `10.0.0.2:5000` to `10.0.0.12:8888`, and when you follow the stream, the 27 individual bytes reassemble into the plaintext string `picoCTF{StaT31355_636f6e6e}`.

### Step 6 — Confirm with a hex dump

The 27 packets each carry a single byte. The follow view is convincing, but let me also pull the raw hex and decode it myself, to be 100% sure the bytes line up:

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ tshark -r capture.pcap -Y "udp.stream eq 6" -T fields -e data.data 2>/dev/null \
  | grep -v "^$" \
  | tr -d '\n' \
  | while read -n2 hex; do printf "\\x${hex}"; done
picoCTF{StaT31355_636f6e6e}
```

Step by step:

- `tshark -Y "udp.stream eq 6"` keeps only the 27 packets of the real-flag stream.
- `-T fields -e data.data` prints the 1-byte payload of each packet in hex (so 27 lines of `70`, `69`, `63`, ...).
- `tr -d '\n'` glues the 27 hex strings into one long line.
- `while read -n2 hex; do printf "\\x${hex}"; done` takes each pair of hex characters and prints it as a raw byte.

Output: `picoCTF{StaT31355_636f6e6e}`. Same as the follow view. Confirmed.

### Step 7 — Peek at the other two streams to see what is there

Out of curiosity, the other two port-8888 streams are interesting too:

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ tshark -r capture.pcap -q -z "follow,udp,ascii,7" 2>/dev/null
===================================================================
Follow: udp,ascii
Filter: udp.stream eq 7
Node 0: 10.0.0.2:5000
Node 1: 10.0.0.13:8888
19
picoCTF{N0t_a_fLag}
===================================================================
```

Stream 7 says `picoCTF{N0t_a_fLag}`. That literally reads "Not a flag" — it is a decoy. If I had followed this stream first, I would have wasted a submit trying a fake flag.

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ tshark -r capture.pcap -q -z "follow,udp,ascii,8" 2>/dev/null
===================================================================
Follow: udp,ascii
Filter: udp.stream eq 8
Node 0: 10.0.0.2:5000
Node 1: 10.0.0.15:8888
1
i
===================================================================
```

Stream 8 is just a single `i` byte. Leftover noise, possibly the first character of a stream that was never finished. Ignore it.

The remaining 120+ streams are LLMNR, mDNS, SSDP, ARP, and the rest of the home-network chatter. None of them contain the flag.

### Step 8 — Submit

```
picoCTF{StaT31355_636f6e6e}
```

Solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | Copied `capture.pcap` into `/work/shark-on-wire-1` | Clean working directory |
| 0:00 | `file capture.pcap` | Confirmed valid pcap, 234 KB, classic format |
| 0:01 | `tshark -q -z io,phs` | Saw 749 raw-UDP `data` frames. A third of the pcap is unnamed UDP traffic — the place to look |
| 0:01 | `tshark -Y "udp" -T fields -e udp.stream \| sort -un \| wc -l` | 289 UDP streams total. Way too many to brute-force one by one in a GUI |
| 0:02 | `tshark -Y "udp and data" -T fields -e udp.stream -e ip.dst -e udp.dstport \| sort -u` | Found 3 streams on port 8888 (plus 14 streams to port 8990 with garbage payloads). The 8888 ones are the only candidates that could be a flag |
| 0:03 | `tshark -q -z "follow,udp,ascii,6"` | Got the flag: `picoCTF{StaT31355_636f6e6e}` |
| 0:04 | `tshark -Y "udp.stream eq 6" -T fields -e data.data \| ...` (hex decode) | Confirmed the same flag a second way, by hand-assembling the 27 individual payload bytes |
| 0:05 | `tshark -q -z "follow,udp,ascii,7"` and `... 8` | Found the decoy `picoCTF{N0t_a_fLag}` and the single-byte `i` stream, for completeness |
| 0:05 | Submitted `picoCTF{StaT31355_636f6e6e}` | Challenge marked solved |

The remaining 286 UDP streams are LLMNR, mDNS, SSDP, ARP, and the rest of the home-network chatter. None of them contain the flag.

Internally, the pcap captures about 38 minutes of traffic on a small LAN. The interesting actor is `10.0.0.2`, which is running some custom tool that:

1. Picks a destination IP (`10.0.0.12`, `10.0.0.13`, or `10.0.0.15`).
2. Sends a UDP packet to that IP on port 8888.
3. The payload of each packet is a single ASCII character.
4. The packets are sent one per UDP "stream" (one stream per destination IP), so each stream's payload reassembles into a different string.
5. The same `picoCTF{` prefix is broadcast to all three destinations, then the characters diverge.

The author was being playful with this setup. They sent the same prefix to three different streams so the analyst has to **follow each one** and read the full string before deciding which is real. Stream 7 spells out `picoCTF{N0t_a_fLag}` — literally "Not a flag" — as a friendly troll. Stream 8 is just a single `i` (probably the first character of a stream that got cut off). Stream 6 is the real flag.

The other 286 UDP streams in the pcap are unrelated local-network traffic: LLMNR name-resolution lookups, mDNS queries, SSDP UPnP discovery, and a custom protocol on port 8990 that sends random-looking bytes. None of them matter. The trick is to focus on the "stream" concept and follow the 8888 streams specifically.

---

## Alternative Methods

### Method 1 — Wireshark GUI walk-through

If you prefer clicking, the Wireshark version of the same solve:

1. Open `capture.pcap` in Wireshark.
2. Type `udp.port == 8888` in the display filter bar. Wireshark shows just the packets on port 8888. There are about 47 of them (sum of the three streams).
3. Right-click on any packet and choose **Follow → UDP Stream**. Wireshark opens a "Follow Stream" window with the reassembled payload of that one stream.
4. In the dropdown at the bottom of the Follow window, change **"Stream N"** to walk through the other streams. You will see:
   - Stream 0 → empty (or noise)
   - ... 
   - Stream 6 → `picoCTF{StaT31355_636f6e6e}` (the flag)
   - Stream 7 → `picoCTF{N0t_a_fLag}` (the decoy)
   - Stream 8 → `i` (leftover)
5. Copy the stream-6 text. That is the flag.

Same answer, more mouse work.

### Method 2 — One-liner with `tshark` and `awk`

If I want the entire decode to be a single shell pipeline, I can do it without the `while` loop by piping each hex byte through `awk`:

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ tshark -r capture.pcap -Y "udp.stream eq 6" -T fields -e data.data 2>/dev/null \
  | awk '{ if ($0 != "") printf "%c", strtonum("0x" $0) }'
picoCTF{StaT31355_636f6e6e}
```

The pipeline:

- `tshark` prints the 1-byte payload of every packet in stream 6, one hex string per line.
- `awk` skips the empty lines (none, but defensive) and converts each hex string to a number with `strtonum("0x" $0)`, then prints it as a character with `printf "%c"`.
- The result is the full flag on one line.

This is the version I would script up if I had a folder full of "shark" pcaps to triage.

### Method 3 — `scapy` for the same view

For the script-inclined, here is a Python version that uses scapy:

```python
# shark_decode.py
# Run: python3 shark_decode.py capture.pcap
from scapy.all import rdpcap, UDP, IP, Raw

PCAP = "capture.pcap"
FLAG_DST = "10.0.0.12"
FLAG_PORT = 8888

packets = rdpcap(PCAP)
flag = []
for pkt in packets:
    if not pkt.haslayer(UDP) or not pkt.haslayer(IP) or not pkt.haslayer(Raw):
        continue
    if pkt[IP].dst != FLAG_DST:
        continue
    if pkt[UDP].dport != FLAG_PORT:
        continue
    payload = bytes(pkt[Raw].load)
    flag.append(payload.decode("ascii", errors="replace"))

print("".join(flag))
```

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ nano shark_decode.py
# (paste the above, Ctrl+O / Enter / Ctrl+X to save)
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ pip install scapy 2>&1 | tail -1
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ python3 shark_decode.py capture.pcap
picoCTF{StaT31355_636f6e6e}
```

This version hard-codes the destination IP and port (`10.0.0.12:8888`) so it does not need to know the stream number. If you want it to find the real flag automatically, you can iterate over every (IP, port) pair, follow each one, and pick the longest plausible string. For a 150-point challenge, hard-coding the answer is fine.

### Method 4 — `tcpdump` for the same view

If I do not have tshark installed and only have `tcpdump`, the same logic still works:

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ tcpdump -r capture.pcap -nn 'udp and dst host 10.0.0.12 and dst port 8888' 2>/dev/null \
  | awk '{ for (i=1; i<=NF; i++) { if (match($i, /^length/)) { gsub(/[,)]/, "", $i); split($i, a, ":"); printf "%s", a[2] } } }'
picoCTF{StaT31355_636f6e6e}
```

Less pretty. `tcpdump` does not show the raw payload in the default `-nn` output, so the `awk` is fragile and case-specific. I would not actually use this; I included it to show that even the bare-bones `tcpdump` can get the answer if you really need it.

### Method 5 — The `strings` trap (do not do this)

This is the cautionary tale. A `strings` sweep of the pcap finds the decoy first:

```
┌──(zham㉿kali)-[/work/shark-on-wire-1]
└─$ strings capture.pcap | grep -iE "pico|flag"
picoCTF{N0t_a_fLag}
picoCTF{StaT31355_636f6e6e}
... (more noise) ...
```

There it is — both candidate flags appear in the pcap. The beginner instinct is to submit the first one (`picoCTF{N0t_a_fLag}`) and be confused when it fails. Then submit the second one and win. That is fine, but it is also dangerous: the author could have included a third fake flag that you would have submitted without ever learning the real lesson.

The lesson: `strings` is great for finding obvious plaintext in a binary blob, but for any challenge that involves **per-packet byte steganography** or **header-field encoding**, the flag might be broken across many small packets and you have to use a proper stream reassembler. `strings` will sometimes give you the right answer by accident, but you should not trust it.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Confirm the downloaded blob is a real pcap file | Easy |
| `tshark -q -z io,phs` | Get a one-shot map of every protocol layer in the pcap (Ethernet, IP, UDP, TCP, ARP, ...) | Easy |
| `tshark -Y "udp" -T fields -e udp.stream \| sort -un \| wc -l` | Count the number of distinct UDP streams in the pcap | Easy |
| `tshark -Y "udp and data" -T fields -e udp.stream -e ip.dst -e udp.dstport \| sort -u` | Find the unusual-destination-port streams that could contain the flag | Easy |
| `tshark -q -z "follow,udp,ascii,6"` | Follow stream 6 (the real flag) and see the reassembled payload | Easy |
| `tshark -Y "udp.stream eq 6" -T fields -e data.data \| while read ...; do printf "\\x${hex}"; done` | Manually decode the 27 individual payload bytes to confirm the flag | Easy |
| `tshark -q -z "follow,udp,ascii,7"` and `... 8` | Confirm what the decoy and noise streams contain | Easy |
| `tshark ... \| awk '{ printf "%c", strtonum("0x" $0) }'` (alt) | One-line version of the whole decode, no `while` loop | Medium |
| Wireshark GUI (alt) | Open pcap, filter to `udp.port == 8888`, right-click → Follow → UDP Stream, walk the streams with the dropdown | Easy |
| `scapy` (alt) | Python library for programmatic pcap walking. Hard-coded destination IP and port | Medium |
| `tcpdump` (alt) | Bare-bones CLI version, possible but ugly awk at the end | Hard |
| `strings \| grep pico` (trap) | Tried first, got the decoy `picoCTF{N0t_a_fLag}`. The decoy is *not* the flag | n/a |

---

## Key Takeaways

- **The hint is in the hint.** When the challenge says "What are streams?", it is literally asking about Wireshark's UDP stream feature. The whole solve is "filter to UDP, follow each stream, read the payload". Anything more complicated is overthinking.
- **Raw `data` frames in the protocol hierarchy are suspicious.** When tshark classifies a chunk of the pcap as raw `data` (UDP payload that does not match any known dissector), that is the slice most likely to contain a custom protocol or exfiltration. The 749 `data` frames here were the only ones worth following.
- **A "stream" is a 4-tuple grouping.** Wireshark groups UDP packets that share the same `(src_ip, src_port, dst_ip, dst_port)` into a numbered stream. The numbering is arbitrary, so do not assume the flag is in stream 0 or stream 1 — just walk the list.
- **Read the full string before you submit.** The author sent two strings that both start with `picoCTF{`, plus a half-finished third. If you only see the first 8 characters, every candidate looks the same. You have to read to the `}` and notice which one is a sensible flag and which one says "Not a flag".
- **Decoy flags with the literal string `picoCTF` in them are bait, not the answer.** Same trick as the Wireshark-twoo and shark-on-wire-2 challenges. If you see two or more `picoCTF`-shaped strings in a pcap, the challenge is asking you which one is real. The answer is almost always "the longer / less obvious one", and the short one is a friendly reminder to read carefully.
- **One byte per packet is a valid data channel.** The flag here was sent 27 times, once per packet, on a single UDP stream. The same trick works for real exfiltration: a covert channel that leaks one byte per packet can fit a whole flag (or a small password, or a short message) in a few seconds, and it is invisible to a payload-grep based IDS.
- **`awk '{ printf "%c", strtonum("0x" $0) }'` is a one-liner for "hex-decode each line to a character".** Pair it with `tshark -T fields` and you can decode any single-byte stego scheme without writing a Python script.

### Flag wordplay decode

```
picoCTF{StaT31355_636f6e6e}
        |||  |||||  |||||||
        |||  |||||  636f6e6e — hex for "conn" (63=c, 6f=o, 6e=n, 6e=n)
        |||  |||||                — and the hex is shown *literally* in the flag,
        |||  |||||                  i.e. you have to recognize "636f6e6e" as the
        |||  |||||                  hex bytes for the ASCII string "conn"
        |||  ||||| _
        |||  StaT31355 — "Stateless" with the "eless" part in leetspeak:
        |||               3→E, 1→L, 3→E, 5→S, 5→S
        |||               (the capital T is just to match the leet pattern)
        ||| _
        || StaT — short for "Stat", as in "Stateless" (the S-t-a pattern
        ||         is preserved, and the T is the "t" of "Stat" lifted
        ||         to uppercase so it can do double duty as the start
        ||         of the next leet cluster)
        ||
        {picoCTF — the standard picoCTF flag format
```

The whole flag reads as **"Stateless connection"** — a one-sentence description of UDP. UDP is a *stateless* protocol (no handshake, no connection state) and the flag is hidden in a *connection* (a UDP stream, identified by its 4-tuple). The challenge author is being cheerful about the fact that you are using Wireshark's stream reassembly to recover data that was sent over a connectionless transport one byte at a time.

The double meaning also points at the challenge solution itself: to find the flag, you have to *establish a virtual connection* (the UDP stream) with the server that owns `10.0.0.12:8888` and let Wireshark reassemble it for you. UDP does not do this for you — you do, by following the stream. The flag is poking fun at that asymmetry: it is literally a "stateless connection" because the analyst had to provide the connection-tracking that UDP deliberately omits.
