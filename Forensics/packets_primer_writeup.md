# Packets Primer — picoCTF Writeup

**Challenge:** Packets Primer  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{p4ck37_5h4rk_ceccaa7f}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Download the packet capture file and use packet analysis software to find the flag.

**Hints shown in challenge:**

> 1. Wireshark, if you can install and use it, is probably the most beginner friendly packet analysis software product.

---

## Background Knowledge (Read This First!)

### What is a `.pcap` file?

A `.pcap` (Packet CAPture) file is a binary file that records network traffic — every packet that crossed a network interface is saved with a timestamp, the raw bytes of every protocol header, and the payload. The file format starts with a 24-byte global header (magic bytes `D4 C3 B2 A1` in little-endian or `A1 B2 C3 D4` in big-endian) followed by a stream of per-packet records. Each record has a 16-byte header (timestamp, captured length, original length) followed by the raw packet bytes.

One important thing for beginners: a `.pcap` is just a regular binary file. You can `cat` it, `strings` it, `grep` it, or `xxd` it just like any other blob on disk. You do not strictly need a special tool to read a pcap — you just need one to *parse* it nicely.

### Tools that read pcaps

- **Wireshark** — the GUI tool everyone names first. Click around, follow streams, filter by protocol. The challenge hint points at Wireshark on purpose because it is genuinely the most beginner-friendly option.
- **tshark** — the command-line version of Wireshark. Same parser, same display filters, no GUI. Great for scripting.
- **tcpdump** — the old-school CLI packet sniffer/reader. It ships on Kali by default. `tcpdump -r file.pcap` reads a pcap and prints one summary line per packet. Add `-X` for hex+ASCII dump, add `-nn` to skip name resolution (so you see raw IPs and ports).
- **scapy** — a Python library. Load a pcap and walk packets programmatically. Useful when you want to script logic over the traffic.

### A TCP "stream" in Wireshark

A TCP connection is not a single packet — it is a conversation. It starts with a three-way handshake:

1. Client sends `SYN` (start)
2. Server replies `SYN + ACK` (okay, start)
3. Client sends `ACK` (got it)

After the handshake, both sides can send `PSH + ACK` packets carrying the actual data. When a Wireshark user says "follow the TCP stream", they mean: glue together every payload byte from the client and the server, in order, ignoring the headers. That is the most useful single feature in Wireshark for a beginner because it shows you the *content* of a conversation instead of dozens of header-heavy packets.

### What the `PSH` flag means

`PSH` is short for "push". When a sender sets the `PSH` flag, it is telling the receiver "do not buffer this — deliver it to the application right now". In practice, every TCP packet that carries application data has the `PSH` flag set, and handshake packets do not. So in `tcpdump` output, `[S]` = SYN (start of handshake), `[S.]` = SYN+ACK, `[.]` = ACK-only, `[P.]` = PSH+ACK (data packet). Seeing `[P.]` is your cue that the packet has a real payload to inspect.

---

## Solution — Step by Step

### Step 1 — Download the pcap

I make a working folder and pull the file from picoCTF's artifact server.

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/packets-primer && cd /work/packets-primer

┌──(zham㉿kali)-[/work/packets-primer]
└─$ wget https://artifacts.picoctf.net/c/295/dump.pcap
--2026-07-12 22:00:00--  https://artifacts.picoctf.net/c/295/dump.pcap
Resolving artifacts.picoctf.net (artifacts.picoctf.net)... 18.155.173.16
Connecting to artifacts.picoctf.net (artifacts.picoctf.net)|18.155.173.16|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 778 (778B)
Saving to: 'dump.pcap'

dump.pcap          100%[==================================>]       778  --.-KB/s    in 0s
```

Tiny file — only 778 bytes. picoCTF is being gentle.

### Step 2 — Confirm the file type

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ file dump.pcap
dump.pcap: pcap capture file, microsecond ts (little-endian) - version 2.4
           (Ethernet, capture length 262144)
```

Real pcap, microsecond timestamps, little-endian, classic Ethernet framing. Good.

### Step 3 — Get a packet-by-packet summary

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ tcpdump -r dump.pcap -nn
reading from file dump.pcap, link-type EN10MB (Ethernet), snapshot length 262144
19:26:35.244594 IP 10.0.2.15.48750 > 10.0.2.4.9000: Flags [S], seq 669832374, win 64240, options [...], length 0
19:26:35.245490 IP 10.0.2.4.9000 > 10.0.2.15.48750: Flags [S.], seq 3173423547, ack 669832375, win 65160, options [...], length 0
19:26:35.245600 IP 10.0.2.15.48750 > 10.0.2.4.9000: Flags [.], ack 1, win 502, options [...], length 0
19:26:35.245819 IP 10.0.2.15.48750 > 10.0.2.4.9000: Flags [P.], seq 1:61, ack 1, win 502, options [...], length 60
19:26:35.246625 IP 10.0.2.4.9000 > 10.0.2.15.48750: Flags [.], ack 61, win 509, options [...], length 0
19:26:40.265000 ARP, Request who-has 10.0.2.15 tell 10.0.2.4, length 46
19:26:40.265048 ARP, Reply 10.0.2.15 is at 08:00:27:af:39:9f, length 28
19:26:40.276530 ARP, Request who-has 10.0.2.4 tell 10.0.2.15, length 28
19:26:40.277416 ARP, Reply 10.0.2.4 is at 08:00:27:93:ce:73, length 46
```

Eight packets total. Reading the timeline:

1. `[S]` from `10.0.2.15:48750` to `10.0.2.4:9000` — client opens a TCP connection to port 9000 on the server
2. `[S.]` back from the server — TCP handshake step 2
3. `[.]` from the client — TCP handshake step 3 (connection is now open)
4. `[P.]` from the client, **length 60** — the client just pushed 60 bytes of application data to the server
5. `[.]` from the server — just an ACK, no data, so the server did not reply to whatever was sent
6–9. Four ARP packets — regular "who has IP X" lookups, totally unrelated to our TCP stream

So the conversation is: open a TCP connection to port 9000, send 60 bytes, and close it. The 60 bytes are the interesting part.

### Step 4 — Look at the actual payload

`tcpdump -X` shows a hex+ASCII dump of every packet. I pipe it through `less` so I can scroll.

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ tcpdump -r dump.pcap -nn -X 2>&1 | less
```

The fourth packet (the `[P.]` one) is the one that matters. The relevant slice of the hex dump looks like this:

```
19:26:35.245819 IP 10.0.2.15.48750 > 10.0.2.4.9000: Flags [P.], seq 1:61, ack 1, win 502, options [...], length 60
        0x0000:  4500 0070 50c2 4000 4006 d1b3 0a00 020f  E..pP.@.@.......
        0x0010:  0a00 0204 be6e 2328 27ec d4b7 bd26 99bc  .....n#(.'.&..n#
        0x0020:  8018 01f6 1875 0000 0101 080a 8dcf e965  .....u.........e
        0x0030:  68f0 f1c3 7020 6920 6320 6f20 4320 5420  h...p.i.c.o.C.T.
        0x0040:  4620 7b20 7020 3420 6320 6b20 3320 3720  F.{.p.4.c.k.3.7.
        0x0050:  5f20 3520 6820 3420 7220 6b20 5f20 6320  _.5.h.4.r.k._.c.
        0x0060:  6520 6320 6320 6120 6120 3720 6620 7d0a  e.c.c.a.a.7.f.}.
```

Reading the right-hand ASCII column on the last three lines:

```
p.i.c.o.C.T.F.{.p.4.c.k.3.7._.5.h.4.r.k._.c.e.c.c.a.a.7.f.}.
```

Every printable character is separated from the next by a literal space byte (0x20). Strip the spaces:

```
picoCTF{p4ck37_5h4rk_ceccaa7f}
```

That is the flag.

### Step 5 — Cross-check with Wireshark (the intended way)

The challenge hint points at Wireshark, so let me reproduce the solve in the GUI as well.

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ wireshark dump.pcap &
```

In Wireshark:

1. The pcap loads with the 8 packets listed in the top pane.
2. Press `Ctrl + F` to open the **Find** bar.
3. Switch the dropdown from **Display filter** to **Packet bytes**.
4. Type `pico` in the search field, leave the case-insensitive box as default.
5. Click **Find**.

Wireshark jumps straight to packet 4, the `[PSH, ACK]` from `10.0.2.15` to `10.0.2.4:9000`. The bottom pane shows the packet bytes; the ASCII column reads:

```
p i c o C T F { p 4 c k 3 7 _ 5 h 4 r k _ c e c c a a 7 f }
```

Or, more usefully, right-click the packet → **Follow → TCP Stream**. Wireshark will show the entire conversation as a single ASCII block in a pop-up window. The pop-up will look like this (one direction only — the server never replied, so there is nothing on the bottom half):

```
p i c o C T F { p 4 c k 3 7 _ 5 h 4 r k _ c e c c a a 7 f }
```

The flag is the contents of the stream, with the inter-character spaces stripped.

### Step 6 — Submit the flag

```
picoCTF{p4ck37_5h4rk_ceccaa7f}
```

Paste it into the picoCTF submission box and the challenge is solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | `wget` the file from `artifacts.picoctf.net` | 778-byte pcap on disk |
| 0:00 | `file dump.pcap` | Confirmed valid pcap, Ethernet, microsecond timestamps |
| 0:01 | `tcpdump -r dump.pcap -nn` | Saw 8 packets: 3-way handshake, 1 data packet, 4 ARP |
| 0:01 | Spotted `[P.]` with `length 60` | The data packet is the only one with application payload |
| 0:02 | `tcpdump -r dump.pcap -nn -X` | Hex+ASCII dump of the payload, ASCII column reads `p.i.c.o.C.T.F.{.p.4.c.k.3.7._.5.h.4.r.k._.c.e.c.c.a.a.7.f.}` |
| 0:02 | Stripped the inter-character spaces | Recovered `picoCTF{p4ck37_5h4rk_ceccaa7f}` |
| 0:03 | Cross-check in Wireshark GUI: `Ctrl+F` → Packet bytes → `pico` | Wireshark jumped to the same packet, confirmed the flag |
| 0:03 | Cross-check via Follow → TCP Stream | The full stream contents are exactly the flag string |
| 0:03 | Submitted the flag | Challenge marked solved |

The internal detail that matters: the pcap holds **one single TCP conversation** between `10.0.2.15` and `10.0.2.4:9000`. The client opens a connection, sends 60 bytes (a spaced-out ASCII version of the flag), and never gets a reply. Four unrelated ARP packets round out the file. There is no encryption, no obfuscation, no file carving — the flag is right there in the first data packet, written as plain ASCII with one space between every character. The whole challenge is "open the pcap, find the conversation, read the data".

---

## Alternative Methods

### Method 1 — `strings` + `grep` (the lazy move)

A `.pcap` is a regular binary file, so `strings` will pull out every run of printable ASCII bytes. The flag is a printable ASCII string, so this works:

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ strings dump.pcap | grep -i pico
p i c o C T F { p 4 c k 3 7 _ 5 h 4 r k _ c e c c a a 7 f }
```

The flag is on a single line, with the inter-character spaces still in place. Strip them and you are done:

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ strings dump.pcap | grep -i pico | tr -d ' '
picoCTF{p4ck37_5h4rk_ceccaa7f}
```

`tr -d ' '` deletes every space character. The remaining bytes are the flag. This is by far the fastest method once you remember that pcaps are just files.

### Method 2 — tshark with a display filter

If you have `tshark` installed (the CLI version of Wireshark), a single command dumps just the payload of the matching packet:

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ tshark -r dump.pcap -Y 'tcp contains "picoCTF"' -T fields -e tcp.payload
70:20:69:20:63:20:6f:20:43:20:54:20:46:20:7b:20:70:20:34:20:63:20:6b:20:33:20:37:20:5f:20:35:20:68:20:34:20:72:20:6b:20:5f:20:63:20:65:20:63:20:63:20:61:20:61:20:37:20:66:20:7d:0a
```

The `-Y 'tcp contains "picoCTF"'` filter keeps only TCP segments whose payload contains the substring `picoCTF`. The `-T fields -e tcp.payload` asks tshark to print just the payload, as colon-separated hex bytes. Pipe through `xxd -r -p` to convert hex back to raw ASCII:

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ tshark -r dump.pcap -Y 'tcp contains "picoCTF"' -T fields -e tcp.payload | xxd -r -p
p i c o C T F { p 4 c k 3 7 _ 5 h 4 r k _ c e c c a a 7 f }
```

One-liner, no GUI, works on any headless box. Same inter-character spaces, same answer after `tr -d ' '`.

### Method 3 — Follow TCP stream from the command line

Wireshark-style "Follow TCP Stream" is also available in tshark. It glues the payload of every packet in a single TCP conversation into one text block:

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ tshark -r dump.pcap -z 'follow,tcp,ascii,0' -q
===================================================================
Follow: tcp,ascii
Filter: tcp.stream eq 0
Node 0: 10.0.2.15:48750
Node 1: 10.0.2.4:9000
                              31
===================================================================
p i c o C T F { p 4 c k 3 7 _ 5 h 4 r k _ c e c c a a 7 f }
===================================================================
```

`-z 'follow,tcp,ascii,0'` adds a "follow TCP stream" expert info for stream index 0. `-q` keeps tshark quiet about every other packet. The output prints the full conversation as a single ASCII block. The `31` near the top is the byte count of the client-to-server direction (0x31 = 49 printable bytes plus the trailing newline).

### Method 4 — scapy Python script

If you prefer Python (or want to do further analysis programmatically), `scapy` is the standard library for pcap work.

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ pip3 install --break-system-packages scapy
```

Then a tiny script:

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ nano read_stream.py
```

```python
#!/usr/bin/env python3
from scapy.all import rdpcap, TCP, Raw

pkts = rdpcap("dump.pcap")

# Glue every TCP payload (client -> server) into one buffer
stream = b""
for pkt in pkts:
    if pkt.haslayer(TCP) and pkt.haslayer(Raw):
        stream += bytes(pkt[Raw].load)

# The flag was sent with one space between every printable byte
flag = stream.replace(b" ", b"").decode().strip()
print(flag)
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`, then run:

```
┌──(zham㉿kali)-[/work/packets-primer]
└─$ python3 read_stream.py
picoCTF{p4ck37_5h4rk_ceccaa7f}
```

`rdpcap("dump.pcap")` reads every packet into a list. We filter for packets that have a TCP header *and* a Raw layer (payload), concatenate the payloads, strip the spaces, and print. The same logic as `strings | grep | tr`, just in Python so you can extend it (filter by IP, count handshake retries, extract all DNS names, etc.) without ever leaving the script.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `wget` | Download the pcap from picoCTF's artifact server | Easy |
| `file` | Confirm the downloaded blob is a real pcap | Easy |
| `tcpdump -r` | Read a pcap and get a packet-by-packet summary | Easy |
| `tcpdump -X` | Hex+ASCII dump of each packet — read the flag bytes manually | Easy |
| `tr -d ' '` | Strip the inter-character spaces from the spaced-out ASCII | Easy |
| `strings` + `grep` (alt) | Pull every printable ASCII run out of the pcap and search for `pico` | Easy |
| Wireshark GUI (alt) | Open the pcap, `Ctrl+F` for `pico`, jump to the flag, optionally Follow → TCP Stream | Easy |
| `tshark` (alt) | CLI Wireshark — `-Y 'tcp contains "picoCTF"'` filter and dump the TCP payload as hex | Medium |
| `tshark -z follow,tcp` (alt) | Glue the whole TCP stream into one ASCII block from the command line | Medium |
| `scapy` (alt) | Python packet parser for scripted pcap analysis | Medium |

---

## Key Takeaways

- **A `.pcap` is a regular binary file.** You can `strings` it, `grep` it, `xxd` it. If a flag is sitting in a pcap as printable ASCII (which is the common case for "intro to packets" challenges), `strings file.pcap | grep picoCTF` finds it in under a second. You do not always need Wireshark.
- **The TCP handshake tells you what is interesting.** `[S]` is the start, `[S.]` is the server agreeing, `[.]` is the final ack, and `[P.]` is a data packet. In a small pcap like this one, the only `[P.]` is the one carrying the flag. Spotting it from `tcpdump -nn` is faster than opening Wireshark.
- **Wireshark's `Find` is the GUI answer to `grep`.** `Ctrl+F`, switch the dropdown from "Display filter" to "Packet bytes", type your search term, click Find. Wireshark jumps straight to the matching packet. This is the intended solve path for any beginner-friendly pcap challenge that points you at Wireshark in its hint.
- **Follow TCP Stream is the most useful Wireshark feature for a beginner.** Right-click a packet → Follow → TCP Stream. Wireshark will glue every payload byte from that conversation into one ASCII pop-up, which is exactly what you want when the flag is sent as the body of a single TCP connection.
- **Watch out for "spaced-out" encodings.** A surprising number of beginner CTF challenges hide a flag by putting one space between every printable character. It still greps as the flag string, but the spaces are noise you have to strip. `tr -d ' '` is the usual cleanup.

### Flag wordplay decode

```
picoCTF{p4ck37_5h4rk_ceccaa7f}
        |    |    |       |
        |    |    |       ceccaa7f = random hex suffix
        |    |    5h4rk = shark (5 → s, 4 → a, leet-speak)
        |    p4ck37 = packet (4 → a, 3 → e, 7 → t)
        p4ck37_5h4rk = "packet shark"
```

So the whole flag decodes to **"packet shark"** + a random hex suffix — a little wink at the fact that the whole solve really is just "use a packet shark (Wireshark) to read the pcap". Cute.
