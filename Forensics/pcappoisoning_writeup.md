# PcapPoisoning — picoCTF Writeup

**Challenge:** PcapPoisoning  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{P64P_4N4L7S1S_SU55355FUL_f621fa37}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> How about some hide and seek heh?
>
> Download this file and find the flag.

**Hints shown in challenge:** *None — this challenge has no hints.*

---

## Background Knowledge (Read This First!)

### What is a `.pcap` file?

A `.pcap` (Packet CAPture) file is a binary file that records network traffic — every packet that crossed a network interface is saved with a timestamp, the raw bytes of every protocol header, and the payload. Tools that work with pcaps include:

- **Wireshark** — GUI, click-and-explore
- **tshark** — command-line version of Wireshark
- **tcpdump** — older CLI tool, very stable, ships on Kali by default
- **Python `scapy`** — load a pcap and walk packets programmatically

The file format starts with a 24-byte global header (magic `D4 C3 B2 A1` in little-endian or `A1 B2 C3 D4` in big-endian) followed by a stream of per-packet records. Each record has a 16-byte header (timestamp, captured length, original length) followed by the raw packet bytes. So if you open a pcap in a hex editor, you can clearly see this structure — and you can also just `cat` or `strings` it like any other file.

### "Poisoning" — what's the trick?

The challenge title hints at **ARP cache poisoning** or **DNS poisoning** — the kind of attack where an attacker injects bogus packets so other machines trust the wrong mapping. The pcap here is full of suspicious crafted packets:

- A packet with **IP protocol 0 (HOPOPT)** carrying what looks like an ARP request from `02:42:ac:10:00:02` claiming `172.16.0.2` — a classic cache-poisoning bait packet
- A **TCP SYN flood** to thousands of ports, each with the same 22-byte payload
- A **fake FTP login** (`username root    password toor`) decoy
- A **DNS packet** from `127.0.0.1` to `127.0.0.1` with a weird Base64-looking payload

The "poisoning" is mostly noise to distract you. The flag is sitting right next to one of those decoy packets — you just have to look.

### Three searches to learn

For any pcap challenge, the three commands I run first are:

```
file capture.pcap         # what kind of pcap is this
strings capture.pcap      # printable ASCII sequences ≥ 4 chars
tcpdump -r capture.pcap -nn  # packet-by-packet summary
```

`strings` is the lazy-but-effective move: every protocol layer that embeds text — DNS names, HTTP headers, FTP banners, IRC messages, raw ASCII payloads — will surface as a "string". A picoCTF flag is itself an ASCII string, so `strings capture.pcap | grep pico` will find it almost instantly.

`tcpdump -nn` shows packet-by-packet summaries (no name resolution, raw IPs/ports). Adding `-X` shows the hex+ASCII dump of each packet. Adding `-v` shows the IP/transport headers in detail.

### A pcap file is *also* a regular file

One thing beginners sometimes forget: a `.pcap` is just a binary blob. You can `grep` it, `strings` it, `xxd` it, or even pipe it through `tr`. The packet boundaries don't matter for text search — `strings` will simply concatenate every run of printable bytes it sees.

---

## Solution — Step by Step

### Step 1 — Download and inspect the pcap

I make a working folder and pull the file. The challenge gives the file as `trace.pcap`; I rename it to `capture.pcap` to match my usual convention.

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/pcap-poisoning && cd /work/pcap-poisoning

┌──(zham㉿kali)-[/work/pcap-poisoning]
└─$ wget https://artifacts.picoctf.net/c_titan/109/trace.pcap
--2026-07-01 20:36:00--  https://artifacts.picoctf.net/c_titan/109/trace.pcap
Resolving artifacts.picoctf.net (artifacts.picoctf.net)... 18.155.173.16
Connecting to artifacts.picoctf.net (artifacts.picoctf.net)|18.155.173.16|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 106715 (104K)
Saving to: 'trace.pcap'

trace.pcap                  100%[==================================>] 104.22K  --.-KB/s    in 0.05s

┌──(zham㉿kali)-[/work/pcap-poisoning]
└─$ mv trace.pcap capture.pcap
```

### Step 2 — Confirm the file type

```
┌──(zham㉿kali)-[/work/pcap-poisoning]
└─$ file capture.pcap
capture.pcap: pcap capture file, microsecond ts (little-endian) - version 2.4
              (Raw IPv4, capture length 65535)
```

It is a real pcap. Now I want to know roughly what's inside.

### Step 3 — Get a packet-by-packet summary

```
┌──(zham㉿kali)-[/work/pcap-poisoning]
└─$ tcpdump -r capture.pcap -nn 2>&1 | head -10
reading from file capture.pcap, link-type IPV4 (Raw IPv4), snapshot length 65535
02:26:11.393946 IP 10.253.0.55 > 10.253.0.6:  ip-proto-0 28
02:26:11.394799 IP 10.253.0.55.1337 > 10.253.0.6.22: Flags [S], seq 1000, win 8192, length 0
02:26:11.395329 IP 10.253.0.55 > 10.253.0.6: ICMP echo request, id 0, seq 0, length 8
02:26:11.395803 IP 172.16.0.2.20 > 10.253.0.6.21: Flags [S], seq 0:34, win 8192, length 34: FTP:     username root    password toor [|ftp]
02:26:11.396448 IP 127.0.0.1.53 > 127.0.0.1.53: 26946 zoneInit+ [b2&3=0x7761] [30289a] [22350q] [12626n] [18277au] [|domain]
02:26:11.396945 IP 10.253.0.55.20 > 192.168.5.5.500: Flags [S], seq 0:22, win 8192, length 22
02:26:11.396945 IP 10.253.0.55.20 > 192.168.5.5.501: Flags [S], seq 0:22, win 8192, length 22
02:26:11.396945 IP 10.253.0.55.20 > 192.168.5.5.502: Flags [S], seq 0:22, win 8192, length 22
02:26:11.396945 IP 10.253.0.55.20 > 192.168.5.5.503: Flags [S], seq 0:22, win 8192, length 22
02:26:11.396945 IP 10.253.0.55.20 > 192.168.5.5.504: Flags [S], seq 0:22, win 8192, length 22
```

Already some interesting oddities:

- A packet using **IP protocol 0** (HOPOPT) — basically never seen in real life
- An **FTP login attempt** with `root / toor` (a famous default credential pair, classic decoy)
- A **DNS packet** with what looks like a Base64 string in the name field
- A massive **TCP SYN scan** to sequential ports `500`, `501`, `502`, ... all carrying a 22-byte payload

That's 1511 packets of mostly noise. Time to grep for the flag.

### Step 4 — The lazy (and effective) move: `strings` + `grep pico`

I don't even need Wireshark for this one. Every ASCII string in the file is right there in `strings`, including the flag.

```
┌──(zham㉿kali)-[/work/pcap-poisoning]
└─$ strings capture.pcap | grep -i pico
iBwaWNvQ1RGe1Flag is close=C~
picoCTF{P64P_4N4L7S1S_SU55355FUL_f621fa37}C~
```

Bingo. Two interesting hits:

1. **`picoCTF{P64P_4N4L7S1S_SU55355FUL_f621fa37}`** — the flag itself
2. **`iBwaWNvQ1RGe1Flag is close=`** — a hint packet from the DNS layer that says (literally) "the flag is close"

The trailing `C~` in both lines is just the next packet's record header bleeding into the printable run — `strings` doesn't care about packet boundaries.

### Step 5 — Confirm where the flag lives in the pcap

Let me look at the packet that actually carries the flag so I can describe it in the writeup:

```
┌──(zham㉿kali)-[/work/pcap-poisoning]
└─$ tcpdump -r capture.pcap -nn -X 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x7069636F' 2>&1
```

(That BPF filter asks tcpdump for any TCP segment whose 4 bytes at the payload offset equal the ASCII `pico` — `0x7069636F`. You can also just `grep -B1 pico` if you want it simpler.)

Output:

```
02:26:11.495231 IP 172.16.0.2.20 > 10.253.0.6.21: Flags [S], seq 0:42, win 8192, length 42:
                  FTP: picoCTF{P64P_4N4L7S1S_SU55355FUL_f621fa37} [|ftp]
	0x0000:  4500 0052 0001 0000 4006 c390 ac10 0002  E..R....@.......
	0x0010:  0afd 0006 0014 0015 0000 0000 0000 0000  ................
	0x0020:  5002 2000 b019 0000 7069 636f 4354 467b  P.......picoCTF{
	0x0030:  5036 3450 5f34 4e34 4c37 5331 535f 5355  P64P_4N4L7S1S_SU
	0x0040:  3535 3335 3546 554c 5f66 3632 3166 6133  55355FUL_f621fa3
	0x0050:  377d                                 7}.
```

The flag is sitting inside a **fake FTP packet**: source `172.16.0.2:20`, destination `10.253.0.6:21`, with the ASCII text `picoCTF{...}` as the FTP payload (42 bytes total, which is the length of the flag plus the `SYN`+`seq`+`win` etc. headers around it). This is the second FTP-looking packet in the file; the first one carried the decoy `root/toor` credentials.

### Step 6 — Submit the flag

```
picoCTF{P64P_4N4L7S1S_SU55355FUL_f621fa37}
```

Paste it into the picoCTF submission box and the challenge is solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | `wget` the file, rename to `capture.pcap` | 104 KB pcap on disk |
| 0:00 | `file capture.pcap` | Confirmed valid pcap, Raw IPv4 |
| 0:01 | `tcpdump -r capture.pcap -nn \| head` | Saw the decoy traffic: HOPOPT, FTP login, DNS, SYN flood |
| 0:01 | Counted packets: `tcpdump -r ... \| wc -l` | 1511 packets total — mostly SYN-flood noise |
| 0:01 | `strings capture.pcap \| grep -i pico` | Flag appeared in 2 lines along with a `Flag is close=` hint |
| 0:02 | BPF filter `tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x7069636F` to confirm | Flag is in a fake FTP packet from `172.16.0.2:20` |
| 0:02 | Submitted the flag | Challenge marked solved |

The "internally" detail that matters: the file holds **one real flag**, **one DNS hint packet** saying "the flag is close", **one decoy FTP login**, **one ARP-poisoning bait** in an IP-protocol-0 packet, and **~1500 SYN-flood packets** that are pure noise. The hint packet and the real flag live in adjacent packets — exactly what the "Flag is close=" message claims.

---

## Alternative Methods

### Method 1 — Wireshark GUI search (the intended way)

The challenge description hints at "hide and seek", and the original picoCTF solve video shows using Wireshark. Open `capture.pcap` in Wireshark and:

1. Press `Ctrl + F` to open the **Find** bar
2. Switch the dropdown from **Display filter** to **Packet bytes**
3. Type `pico` (or the full `picoCTF{` prefix)
4. Click **Find**

Wireshark will jump straight to the FTP packet containing the flag. You can also right-click the packet → **Follow → TCP Stream** to see the flag in context, although here the "stream" is just the single packet so `Follow` will not add anything.

### Method 2 — Wireshark display filter

If you want Wireshark to *only* show the matching packets instead of jumping to them:

1. In the display-filter bar at the top, type:
   ```
   tcp contains "picoCTF{"
   ```
2. Press `Enter`. Wireshark will reduce the view to just the one TCP segment whose payload contains the flag bytes.

Wireshark's `contains` keyword is a quick way to search the payload of any protocol without writing a BPF offset expression.

### Method 3 — tshark command-line

If you have `tshark` installed (the CLI version of Wireshark), one command does everything:

```
┌──(zham㉿kali)-[/work/pcap-poisoning]
└─$ tshark -r capture.pcap -Y 'tcp contains "picoCTF"' -T fields -e data
7069636f4354467b503634505f344e344c375331535f53553535353546554c5f66363231666133373137... 
```

The `-Y` flag is the display filter, `-T fields -e data` asks for the hex of the TCP payload. The output `7069636f4354467b...` is the ASCII hex of `picoCTF{...}` — pipe it through `xxd -r -p` if you want it as raw bytes:

```
┌──(zham㉿kali)-[/work/pcap-poisoning]
└─$ tshark -r capture.pcap -Y 'tcp contains "picoCTF"' -T fields -e data | xxd -r -p
picoCTF{P64P_4N4L7S1S_SU55355FUL_f621fa37}
```

Clean one-liner that works in any terminal.

### Method 4 — scapy Python script

If you prefer Python (or want to do further analysis programmatically), `scapy` is the standard library:

```
┌──(zham㉿kali)-[/work/pcap-poisoning]
└─$ pip3 install --break-system-packages scapy
```

Then a tiny script:

```
┌──(zham㉿kali)-[/work/pcap-poisoning]
└─$ nano scan.py
```

```python
#!/usr/bin/env python3
from scapy.all import rdpcap, TCP, Raw

for pkt in rdpcap("capture.pcap"):
    if pkt.haslayer(TCP) and pkt.haslayer(Raw):
        payload = pkt[Raw].load
        if b"picoCTF{" in payload:
            print(payload.decode(errors="replace"))
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`, then run:

```
┌──(zham㉿kali)-[/work/pcap-poisoning]
└─$ python3 scan.py
picoCTF{P64P_4N4L7S1S_SU55355FUL_f621fa37}
```

`scapy` walks every packet, asks for the TCP and Raw layers, and prints any payload that contains the flag prefix. Same logic as `grep pico`, but typed in Python so you can extend it (e.g., dump all DNS names, count SYN packets, filter by IP, etc.) without ever leaving the script.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `wget` | Download the pcap from picoCTF's artifact server | Easy |
| `file` | Confirm the downloaded blob is a real pcap | Easy |
| `tcpdump -r` | Read a pcap and get a packet-by-packet summary | Easy |
| `strings` | Pull every ASCII run ≥ 4 chars out of the pcap as a binary blob | Easy |
| `grep pico` | One-grep search for the flag prefix | Easy |
| `tcpdump -X` (alt) | Hex+ASCII dump of each packet to read the FTP payload manually | Easy |
| Wireshark GUI (alt) | Open the pcap, `Ctrl+F` for `pico`, jump to the flag | Easy |
| `tshark` (alt) | CLI Wireshark — `-Y 'tcp contains "picoCTF"'` filter and dump the data field | Medium |
| `scapy` (alt) | Python packet parser for scripted pcap analysis | Medium |

---

## Key Takeaways

- **`strings` + `grep` is your fastest pcap-search move.** A pcap is a regular binary file. ASCII content — DNS names, HTTP headers, FTP payloads, IRC text, raw embedded files — all appear as plain "strings". When you suspect a flag is hiding as readable text (like `picoCTF{...}`), `strings capture.pcap | grep picoCTF` finds it in under a second. This is true for **every** pcap challenge where the flag is embedded as text.
- **Decoys and "poisoning" traffic are noise, not signal.** ARP-poisoning bait (IP-protocol-0 with embedded ARP), fake FTP credentials, SYN-flood decoys — these are red herrings designed to waste your time if you stare at every packet. Use `tcpdump` to count and categorize, then jump to the text search. Most CTF pcaps have a 90/10 split: 90% junk traffic, 10% interesting packet that contains the flag.
- **Wireshark's `contains` keyword is gold.** Both the GUI search and the tshark display filter support `protocol contains "text"`. So `tcp contains "picoCTF{"` or `frame contains "pico"` works without any BPF offset calculation. Use `tcp contains`, `dns.qry.name contains`, `http.request.uri contains`, `frame contains`, etc. depending on where you expect the data to live.
- **Follow the hint.** The DNS packet in this file literally says `Flag is close=`. The flag was placed in the next interesting packet after the hint — exactly as the hint advertises. When a pcap challenge author writes a clue into the traffic, assume it is true. Hunt the neighbouring packets first.
- **Print the count of packets early.** `tcpdump -r capture.pcap | wc -l` tells you how much you are dealing with. 1511 packets sounds scary, but a quick `strings | grep` makes it a 5-second solve. Don't open Wireshark until you know whether you need it.

### Flag wordplay decode

```
picoCTF{P64P_4N4L7S1S_SU55355FUL_f621fa37}
   |   |   |     |       |
       P64P     = PCAP (P-C-A-P → P64P leet)
             4N4L7S1S = ANALYSIS (4=A, 7=T, 1=I, etc.)
                       SU55355FUL = SUCCESSFUL
                                  f621fa37 = random hex suffix
```

So the whole flag decodes to **"PCAP analysis successful"** — a wink at the fact that the entire solve really is just pcap analysis: open it, read it, find the flag.
