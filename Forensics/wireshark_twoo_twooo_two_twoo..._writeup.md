# Wireshark twoo twooo two twoo... — picoCTF Writeup

**Challenge:** Wireshark twoo twooo two twoo...  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{dns_3xf1l_ftw_deadbeef}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Can you find the flag?

**Hints shown in challenge:**

> 1. Did you really find *the* flag?
> 2. Look for traffic that seems suspicious.

---

## Background Knowledge (Read This First!)

### What is a `.pcapng` file?

A `.pcapng` (Packet CAPture Next Generation) file is a binary file that records network traffic. Every packet that crossed a network interface is saved with a timestamp, the raw bytes of every protocol header, and the payload. `.pcapng` is the modern, multi-section successor to the original `.pcap` format. The two formats are similar enough that every tool in this writeup treats them interchangeably. The file still starts with a 24-byte global header (magic bytes `0A 0D 0D 0A` in pcapng, or `D4 C3 B2 A1` in classic pcap) followed by a stream of per-packet or per-block records.

For beginners, the important takeaways are:

- A `.pcapng` is just a regular binary file. You can `cat` it, `strings` it, `grep` it, or `xxd` it like any other blob.
- A packet capture is essentially a wiretap — every byte that went across a network is recorded, including the *content* of unencrypted protocols like HTTP, DNS, FTP, SMTP, and so on.
- Encrypted protocols (TLS, HTTPS, SSH, Kerberos-encrypted WinRM) look like random noise in the capture. The plaintext is gone the moment TLS finishes the handshake.

### Tools that read pcap/pcapng files

- **Wireshark** — the GUI tool everyone names first. Click around, follow streams, filter by protocol. This is the tool the challenge title is hinting at.
- **tshark** — the command-line version of Wireshark. Same parser, same display filters, no GUI. Great for scripting. This is what I will use for the main solve.
- **tcpdump** — the old-school CLI packet sniffer. It ships on Kali by default and can also read pcap files: `tcpdump -r file.pcapng`.
- **scapy** — a Python library. Load a pcap and walk packets programmatically. Useful for scripted analysis, but overkill for a 100-point challenge.

### The "Protocol Hierarchy" view

When you first open a pcap, you do not know what is inside. The fastest way to get a map of the whole file is the **Protocol Hierarchy** statistics, which in tshark is `-q -z io,phs`. It prints a tree of every protocol layer (Ethernet, IP, TCP, HTTP, TLS, ...) with packet and byte counts at each level. For a beginner, this is the best "what kind of traffic is this?" view. If you see a lot of HTTP, you know to look for plaintext requests and responses. If you see a lot of DNS, you know to look for query names. If you see a lot of TLS, the data is encrypted and you cannot read it without keys.

### HTTP basics

HTTP is a request/response protocol. The client sends a request line like `GET /flag HTTP/1.1`, the server replies with a status line like `HTTP/1.1 200 OK`, then headers, then a body. A `200 OK` means "here is what you asked for". The body of a successful `GET` is usually an HTML page, but it can also be a single line of plain text — which is exactly the kind of thing the previous shark challenge in this series served up.

### What is DNS, really?

DNS (Domain Name System) is the "phone book of the internet". Every time your computer wants to talk to a server by name (for example `www.example.com`), it asks a DNS resolver "what is the IP address of this name?" and gets an `A` (IPv4) or `AAAA` (IPv6) record back. A normal DNS query looks like this:

```
;; QUESTION SECTION:
;www.example.com.   IN   A
;; ANSWER SECTION:
www.example.com.    3600  IN  A  93.184.216.34
```

The query is a single UDP packet, usually to port 53 of a configured resolver (8.8.8.8, 1.1.1.1, the ISP's resolver, etc.). The response is a single UDP packet back. DNS is usually **not** encrypted, which is the whole reason it is interesting in a forensics context.

### DNS exfiltration — smuggling data over a side channel

Because DNS packets are tiny, mostly unencrypted, and almost always allowed out of corporate networks ("we can't break the internet by blocking DNS"), attackers have been abusing them as a **data exfiltration channel** for over a decade. The trick is simple:

1. The malware on the victim host has some data it wants to leak (a flag, a password, a credit card, whatever).
2. It chops the data into short chunks.
3. It registers a domain it controls, for example `reddshrimpandherring.com`.
4. For each chunk, it does a DNS query like `<chunk>.reddshrimpandherring.com`.
5. The chunk is encoded somehow — most often as **base64** so it stays inside the legal DNS character set (letters, digits, hyphens, max 63 chars per label).
6. The attacker controls the authoritative nameserver for `reddshrimpandherring.com`, so the resolver dutifully asks them, the chunks land in the attacker's DNS logs, and the malware's C2 has its data.

To a defender looking at a packet capture, this looks like a strange burst of DNS traffic to a domain they have never seen before, with random-looking alphanumeric labels. That is exactly the suspicious traffic the hint is pointing at.

### Why "the flag" matters here

The first hint — "Did you really find *the* flag?" — is the giveaway. A naïve `strings shark2.pcapng | grep pico` will pull out **89 different** strings that match the `picoCTF{...}` pattern. They are all 73 bytes long. They are all valid 64-character hex hashes. None of them are the flag. They are **decoys** served by a Python/Werkzeug server on `18.217.1.57` over and over again, presumably to make a `strings`-only solve fail. The real flag is somewhere else, in a place the `strings` approach cannot reach: the subdomains of the DNS queries to a domain you have never seen before.

### Why "two two two two" is the title

The challenge title is a riff on the band War's 1973 hit "Low Rider", the same way that the sister challenge "Wireshark doo dooo do doo..." is a riff on the Police's "De Do Do Do, De Da Da Da". The "two two two two" is just flavor — there is no literal `port 2222` or `stream 2` to look for. The hint is the real map.

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the pcap

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/wireshark-twoo && cd /work/wireshark-twoo

┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ curl -sL -o shark2.pcapng \
    "https://challenge-files.picoctf.net/c_wily_courier/b58071e429eda96e426f3014da6b80b8c659a29ee6aaf85631610940574ea18d/shark2.pcapng"

┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ ls -la shark2.pcapng
-rw-r--r-- 1 zham zham 3520112 Jul 13 17:18 shark2.pcapng
```

I keep the artifact in a fresh folder so any output files I dump do not pollute the rest of my filesystem. About 3.5 MB — small enough to fit in memory, large enough to hide a flag in.

### Step 2 — Confirm what we have

```
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ file shark2.pcapng
shark2.pcapng: pcapng capture file - version 1.0
```

A real pcapng. Good.

### Step 3 — Get the lay of the land: protocol hierarchy

```
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ tshark -r shark2.pcapng -q -z io,phs
===================================================================
Protocol Hierarchy Statistics
Filter: 

eth                                      frames:4831 bytes:3355920
  ip                                     frames:4829 bytes:3355822
    tcp                                  frames:3276 bytes:3120750
      tls                                frames:71 bytes:115780
        tcp.segments                     frames:2 bytes:6576
      http                               frames:802 bytes:1879844
        tcp.segments                     frames:299 bytes:1605841
        mime_multipart                   frames:309 bytes:194144
          tcp.segments                   frames:309 bytes:194144
        data-text-lines                  frames:91 bytes:23987
          tcp.segments                   frames:90 bytes:23696
        xml                              frames:1 bytes:579
    udp                                  frames:1553 bytes:235072
      gquic                              frames:41 bytes:11668
      dns                                frames:1512 bytes:223404
  arp                                    frames:2 bytes:98
===================================================================
```

What this tells me at a glance:

- **802 HTTP frames** — there is plaintext HTTP traffic in here. Same WinRM/Windows-Event-Forwarding noise as the previous shark challenge, plus a lot of `data-text-lines` responses.
- **309 MIME multipart frames** — almost all of them are `application/HTTP-Kerberos-session-encrypted`, which means WinRM/WSMan traffic with a Kerberos-encrypted body. Unreadable without the session key.
- **91 `data-text-lines` frames** — these are short, line-based text bodies. Most of them are the fake flags served by the decoy server.
- **71 TLS frames** — encrypted HTTPS, unreadable.
- **1512 DNS frames over UDP** — that is a *lot* of DNS. About 30% of the whole pcap by packet count. A 3.5 MB capture should not normally have that much DNS — that is worth a second look.

The DNS count is the anomaly. Most picoCTF pcaps I have seen have maybe a dozen DNS frames. 1512 is screaming "something is exfiltrating via DNS here".

### Step 4 — The wrong rabbit hole: a `strings` shortcut

Because the previous challenge ("Wireshark doo dooo do doo...") was a `strings` one-liner, my first instinct was the same:

```
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ strings shark2.pcapng | grep -iE 'pico|flag'
GET /flag HTTP/1.1
picoCTF{bfe48e8500c454d647c55a4471985e776a07b26cba64526713f43758599aa98b}
picoCTF{bda69bdf8f570a9aaab0e4108a0fa5f64cb26ba7d2269bb63f68af5d98b98245}
picoCTF{fe83bcb6cfd43d3b79392f6a4232685f6ed4e7a789c2ce559cf3c1ab6adbe34b}
picoCTF{711d3893d90f100c15e10ef4842abeed3a830f8237c1257cd47389646da97810}
picoCTF{3cf1e22d489fcfb6bb312a34f46c8699989ed043406134331452d11ce73cd59e}
picoCTF{b4cc138bb0f7f9da7e35085e349555aa6d00bdca3b021c1fe8663c0a422ce0d7}
picoCTF{41b8a1a796bd8d202016f75bc5b38889e9ea06007e6b22fc856d380fb7573133}
... (89 in total)
```

Eighty-nine different `picoCTF{...}` strings, all 64-char hex hashes, all 73 bytes long. I tried submitting the first one. Wrong. I tried the last one. Wrong. The first hint makes sense now: **"Did you really find *the* flag?"** — meaning, did I really find *the* one, or just *a* one? The decoys are real-looking on purpose.

Time to back away from `strings` and look at the actual traffic.

### Step 5 — Find the HTTP responses and the "fake flag" server

```
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ tshark -r shark2.pcapng -Y "http.response" -T fields -e frame.number -e http.response.code -e http.server 2>/dev/null | sort -u
52      200     Werkzeug/1.0.1 Python/3.6.9
26      401     Microsoft-HTTPAPI/2.0
35      200     Microsoft-HTTPAPI/2.0
57      200     Werkzeug/1.0.1 Python/3.6.9
... 89 more Werkzeug 200s ...
4336    200     EC2ws
4347    404     EC2ws
```

Three server banners show up:

- **Microsoft-HTTPAPI/2.0** — the WinRM/WSMan server (`wef.windomain.local:5985`). The `401` is the auth challenge, the `200` is the Kerberos auth handshake. All the multipart responses on this port are the WEF push traffic, encrypted with Kerberos. Noise.
- **Werkzeug/1.0.1 Python/3.6.9** — a Python Flask app. This is the decoy server. It serves a tiny "under construction" page on `GET /` and a fake `picoCTF{...}` flag on every `GET /flag`. 89 fake flags, all 73 bytes, all hashes. Distraction.
- **EC2ws** — Amazon EC2 instance metadata service (`169.254.169.254`). Returns an IMDSv2 token then a 404 for the role name. Noise.

So all the obvious HTTP is either encrypted WinRM, fake flags, or AWS metadata. The flag is not in any of them.

### Step 6 — Look at the DNS traffic

This is the part where I trust the second hint: "Look for traffic that seems suspicious."

```
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ tshark -r shark2.pcapng -Y "dns" -T fields -e dns.qry.name 2>/dev/null | sort -u | head -20
++FVriJb.reddshrimpandherring.com
++FVriJb.reddshrimpandherring.com.us-west-1.ec2-utilities.amazonaws.com
++FVriJb.reddshrimpandherring.com.windomain.local
+9eXNk0X.reddshrimpandherring.com
+9eXNk0X.reddshrimpandherring.com.us-west-1.ec2-utilities.amazonaws.com
+9eXNk0X.reddshrimpandherring.com.windomain.local
... (251 unique labels, all subdomains of reddshrimpandherring.com) ...
```

A domain I have never seen before — `reddshrimpandherring.com` — with 251 unique random-looking labels. The fact that every query has three variants (the bare domain, the `us-west-1.ec2-utilities.amazonaws.com` suffix used by AWS DNS resolvers, and the `windomain.local` internal suffix) is a Windows DNS resolver artifact; it does not change the data. The signal is the random labels.

That smell is unmistakable: **DNS exfiltration**. Each label is a chunk of the flag, encoded so it survives DNS character set rules.

### Step 7 — Find the resolver split

A clean exfiltration capture usually has the exfil traffic going to a *different* resolver than the normal DNS. The decoy and the exfil go to different places, and that is the discriminator. Let me ask tshark for the destination IPs of every DNS query:

```
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ tshark -r shark2.pcapng -Y "dns.flags.response == 0" -T fields -e ip.dst -e dns.qry.name 2>/dev/null \
    | awk '{print $1}' | sort | uniq -c
    735 18.217.1.57
    735 8.8.8.8
```

Aha. Every DNS query in the capture goes to **one of two destinations**:

- **8.8.8.8** — Google Public DNS, the normal recursive resolver.
- **18.217.1.57** — the *same* IP as the fake-flag server. That is the rogue DNS resolver the malware is using.

Each query is duplicated — once to 8.8.8.8 and once to 18.217.1.57. That is the local DNS resolver duplicating requests for some reason (most likely the local NIC has two DNS servers configured and Windows is "speeding things up" by querying both). What matters is that the queries to 18.217.1.57 are the **malware's** queries, and the queries to 8.8.8.8 are the **legit resolver's** queries. The malware picks the one it controls and hands the data to it directly.

I want only the data destined for the rogue resolver.

### Step 8 — Pull the labels from the rogue-resolver queries

```
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ tshark -r shark2.pcapng -Y "dns and ip.dst == 18.217.1.57" \
    -T fields -e frame.number -e dns.qry.name 2>/dev/null \
    | sed 's/\.reddshrimpandherring\.com.*//' \
    | awk '!seen[$2]++ {print $1, $2}'
1633 cGljb0NU
2042 RntkbnNf
2444 M3hmMWxf
3140 ZnR3X2Rl
3429 YWRiZWVm
3969 fQ==
```

A handful of unique labels (the rest are retransmits or duplicate queries). Each label is 6 or 8 characters of base64. The `awk '!seen[$2]++'` keeps the first occurrence of each unique label in frame order — that is the order the chunks were sent.

Concatenated in order:

```
cGljb0NU + RntkbnNf + M3hmMWxf + ZnR3X2Rl + YWRiZWVm + fQ==
= cGljb0NURntkbnNfM3hmMWxfZnR3X2RlYWRiZWVmfQ==
```

That is 60 characters. Multiple of 4. The `fQ==` is a base64-style padding suffix, so this is a single base64 blob.

### Step 9 — Decode the base64

```
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ echo "cGljb0NURntkbnNfM3hmMWxfZnR3X2RlYWRiZWVmfQ==" | base64 -d
picoCTF{dns_3xf1l_ftw_deadbeef}
```

There it is. The base64 decodes to the real flag.

### Step 10 — Submit

```
picoCTF{dns_3xf1l_ftw_deadbeef}
```

Solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | Downloaded `shark2.pcapng` into `/work/wireshark-twoo` | Clean working directory |
| 0:00 | `file shark2.pcapng` | Confirmed valid pcapng, 3.5 MB |
| 0:01 | `tshark -q -z io,phs` | Saw 802 HTTP frames, 71 TLS frames, and 1512 DNS frames. DNS count was the anomaly |
| 0:02 | `strings \| grep pico` | Got 89 fake flags. All 73 bytes, all 64-char hex. The hint suddenly made sense |
| 0:03 | `tshark -Y "http.response" -T fields -e http.server` | Saw Werkzeug (decoy), Microsoft-HTTPAPI (WinRM), EC2ws (IMDS). Flag was not in any of them |
| 0:04 | `tshark -Y "dns" -T fields -e dns.qry.name \| sort -u` | Found `*.reddshrimpandherring.com` with 251 unique random labels. DNS exfiltration |
| 0:05 | `tshark -Y "dns.flags.response == 0" -T fields -e ip.dst \| sort \| uniq -c` | Found the split: 735 queries to 8.8.8.8, 735 to 18.217.1.57 (the decoy server IP) |
| 0:05 | `tshark -Y "dns and ip.dst == 18.217.1.57" -T fields -e frame.number -e dns.qry.name \| ... \| awk '!seen[$2]++'` | Pulled the 6 unique base64 labels in frame order |
| 0:06 | Concatenated the labels into `cGljb0NURntkbnNfM3hmMWxfZnR3X2RlYWRiZWVmfQ==` | One base64 blob |
| 0:06 | `echo "..." \| base64 -d` | Got `picoCTF{dns_3xf1l_ftw_deadbeef}` |
| 0:06 | Submitted `picoCTF{dns_3xf1l_ftw_deadbeef}` | Challenge marked solved |

Internally, the pcap captures a Windows 10 host (`192.168.38.104`, build 18363) that has been compromised. Three things are happening in parallel:

1. **Fake flag noise.** The browser on the victim visits `http://18.217.1.57/flag` 89 times. Each request is answered with a different `picoCTF{...}` decoy from a Python/Werkzeug server. This is the "bait" — the `strings` approach will pick one of these up and waste your time.
2. **DNS exfiltration of the real flag.** A piece of malware on the host reads the real flag from somewhere, chops it into 6 base64 chunks of 8 chars each (plus a 4-char `fQ==` padding chunk), and emits them as DNS queries to the rogue resolver at `18.217.1.57`, which is the *same* IP as the fake flag server. The query name in each case is `<chunk>.reddshrimpandherring.com`. The Windows DNS resolver helpfully duplicates the queries to the legit 8.8.8.8 as well.
3. **Windows Event Forwarding push.** The host is a Windows Event Collector client pushing events to `wef.windomain.local:5985` over 5 WSMAN subscriptions (`/2`, `/29`, `/36`, `/37`, `/43`). The bodies are Kerberos-encrypted at the application layer. Unreadable, and not where the flag is.

The job is to ignore the bait, ignore the encrypted noise, recognize the exfiltration pattern in the DNS, and reconstruct the base64. The protocol hierarchy view, the `awk '!seen[$2]++'` dedup-in-order trick, and the `ip.dst` split are the three techniques that make this fast.

---

## Alternative Methods

### Method 1 — Wireshark GUI walk-through

If you prefer clicking, the Wireshark version of the same solve:

1. Open `shark2.pcapng` in Wireshark.
2. In the display filter bar, type `dns and ip.dst == 18.217.1.57`. Wireshark shows only the 735 DNS queries to the rogue resolver. They all look like `<random>.reddshrimpandherring.com`.
3. Right-click any of the queries → **Copy** → **...as Filter** → **...and Bytes (Hex Stream)**. You now have the raw bytes of one query. The base64 chunk is the leftmost label.
4. To get the chunks *in order*, switch the columns to show the **Frame number** and the **Qry Name** side by side (`View → Column Preferences → Add → Field: dns.qry.name`). Sort by frame number. Walk down the list and copy the unique labels into a text file in order: `cGljb0NU`, `RntkbnNf`, `M3hmMWxf`, `ZnR3X2Rl`, `YWRiZWVm`, `fQ==`.
5. Concatenate, then from a terminal: `echo 'cGljb0NURntkbnNfM3hmMWxfZnR3X2RlYWRiZWVmfQ==' | base64 -d`.

Same answer, more mouse work.

### Method 2 — One-liner with `tshark`, `awk`, and `base64`

If I want the entire solve to be a single shell pipeline, I can chain everything together:

```
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ tshark -r shark2.pcapng -Y "dns and ip.dst == 18.217.1.57" \
    -T fields -e frame.number -e dns.qry.name 2>/dev/null \
    | sed 's/\.reddshrimpandherring\.com.*//' \
    | awk '!seen[$2]++ {printf "%s", $2}' \
    | base64 -d ; echo
picoCTF{dns_3xf1l_ftw_deadbeef}
```

The pipeline:

- `tshark` prints `<frame>\t<qry_name>` for every DNS query to the rogue resolver.
- `sed` strips the `.reddshrimpandherring.com` suffix.
- `awk '!seen[$2]++'` keeps only the first occurrence of each unique label, in input order, and prints them all on one line (`printf` instead of `print` so there is no newline between chunks).
- `base64 -d` decodes the concatenated chunks.
- `; echo` adds a trailing newline so the prompt does not glue to the flag.

This is the version I would script up if I had a folder full of these pcaps to triage.

### Method 3 — `apackets.com` online viewer

A really quick no-install option is to upload the pcap to [apackets.com](https://apackets.com/) and click on the **DNS** page in the left sidebar. The viewer lists every DNS query in a sortable table, grouped by domain. Scroll to `reddshrimpandherring.com`, eyeball the labels in order, copy them out by hand, and base64-decode. Takes about two minutes and zero terminal commands. Good fallback if `tshark` is not installed.

### Method 4 — Python with `scapy` (for the script-inclined)

For completeness, here is a Python version that uses scapy to do the same thing:

```python
# exfil_decode.py
# Run: python3 exfil_decode.py shark2.pcapng
from scapy.all import rdpcap, DNS, IP

pcap_path = "shark2.pcapng"
ROGUE_RESOLVER = "18.217.1.57"
DOMAIN = "reddshrimpandherring.com"

packets = rdpcap(pcap_path)
chunks = {}  # frame_number -> label
for i, pkt in enumerate(packets, start=1):
    if not pkt.haslayer(DNS):
        continue
    if not pkt.haslayer(IP):
        continue
    if pkt[IP].dst != ROGUE_RESOLVER:
        continue
    if pkt[DNS].qr != 0:  # only queries, not responses
        continue
    qname = pkt[DNS].qd.qname.decode()
    if not qname.endswith(DOMAIN + "."):
        continue
    label = qname[: -(len(DOMAIN) + 2)]  # strip ".<domain>."
    chunks[i] = label

import base64
concat = "".join(chunks[k] for k in sorted(chunks))
print(base64.b64decode(concat).decode())
```

```
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ nano exfil_decode.py
# (paste the above, Ctrl+O / Enter / Ctrl+X to save)
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ pip install scapy 2>&1 | tail -1
┌──(zham㉿kali)-[/work/wireshark-twoo]
└─$ python3 exfil_decode.py shark2.pcapng
picoCTF{dns_3xf1l_ftw_deadbeef}
```

Same answer, more setup. I would not bother for a single challenge, but if I were writing a triage tool for the SOC this is the shape it would take.

### Method 5 — The `strings` trap (do not do this)

This is the cautionary tale. A `strings` + `grep pico` sweep of the pcap pulls out 89 different `picoCTF{...}` strings, and a beginner instinct is to brute-force-submit them to the picoCTF scoreboard. That works in theory, but:

- 89 submissions is a lot of submissions. The picoCTF scoreboard will rate-limit you long before you finish.
- The first 88 are wrong. The 89th might also be wrong — they are all decoys.
- You have not actually learned anything about the pcap. You just got lucky, and on the next exfiltration challenge (with 1000 chunks instead of 6) you will be back to square one.

The right move is to **ignore** the `strings` output the moment you see 89 of them and switch to protocol-level analysis. The decoys are bait. Do not take the bait.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Download `shark2.pcapng` from the challenge URL into the working directory | Easy |
| `file` | Confirm the downloaded blob is a real pcapng file | Easy |
| `tshark -q -z io,phs` | Get a one-shot map of every protocol layer in the pcap (Ethernet, IP, TCP, HTTP, TLS, DNS, ...) | Easy |
| `strings \| grep pico` | Initial sweep — pulled out 89 fake flags. Used as a *negative* result to confirm "the real flag is not in any HTTP body" | Easy |
| `tshark -Y "http.response" -T fields -e http.server` | List every HTTP response with its server banner. Spotted Werkzeug (decoy), Microsoft-HTTPAPI (WinRM), EC2ws (IMDS) | Easy |
| `tshark -Y "dns" -T fields -e dns.qry.name \| sort -u` | List every unique DNS query name. Found `*.reddshrimpandherring.com` with 251 random labels — the exfiltration signal | Easy |
| `tshark -Y "dns.flags.response == 0" -T fields -e ip.dst \| sort \| uniq -c` | Find the resolver split: 735 queries to 8.8.8.8, 735 to the rogue 18.217.1.57 | Medium |
| `tshark -Y "dns and ip.dst == 18.217.1.57" -T fields -e frame.number -e dns.qry.name` | Extract the exfil queries in frame order, with the chunks visible | Easy |
| `sed 's/\.reddshrimpandherring\.com.*//'` | Strip the domain suffix from each query name to leave only the base64 label | Easy |
| `awk '!seen[$2]++ {print $2}'` | Deduplicate the labels *while preserving input order*, so the chunks reassemble correctly | Medium |
| `base64 -d` | Decode the concatenated base64 blob to the real flag | Easy |
| `tshark ... \| sed ... \| awk ... \| base64 -d` (alt) | One-line version of the whole solve | Medium |
| Wireshark GUI (alt) | Open pcap, filter `dns and ip.dst == 18.217.1.57`, read the qry name column, copy labels by hand | Easy |
| `apackets.com` (alt) | Online pcap viewer with a built-in DNS table view. No install needed | Easy |
| `scapy` (alt) | Python library for programmatic pcap walking. Overkill for one challenge, but the right shape for a triage script | Medium |
| `strings \| grep pico` (trap) | Tried first, got 89 decoys. Used to *prove* the decoys exist, then discarded | n/a |

---

## Key Takeaways

- **`strings` is a starting point, not a finishing tool.** It is great for finding URLs, error messages, and obvious embedded strings, but it cannot see data that lives in protocol fields (DNS labels, HTTP headers, etc.) unless they happen to also appear in plaintext elsewhere. The 89 fake flags in the HTTP bodies are a perfect example: `strings` finds them, but they are not the answer.
- **A 5-second protocol hierarchy scan tells you where the flag is hiding.** `tshark -q -z io,phs` on a 3.5 MB pcap takes less than a second and gives you a tree of every protocol with byte counts. When one protocol sticks out — here, 1512 DNS frames in a capture that should have a dozen — that is your next lead.
- **When the obvious answer is "too easy to be right", it usually is.** 89 identical-shaped `picoCTF{...}` strings in one pcap is not a coincidence. The first hint ("Did you really find *the* flag?") is the puzzle author telling you the `strings` approach is the trap.
- **DNS exfiltration is one of the most common — and most boring — real-world data theft techniques.** Every SOC analyst sees it eventually. The tell is always the same: a domain the org does not own, with random-looking alphanumeric labels, queried in bursts. Once you know the pattern, you can spot it in a pcap in under a minute.
- **The `ip.dst` split is the discriminator for "which side is the malware?".** When a Windows box has two DNS resolvers configured, every legit query is duplicated to both. The malware, on the other hand, queries *only* its rogue resolver. Filtering to `dns and ip.dst == <rogue_ip>` cuts the noise roughly in half and isolates the data channel.
- **`awk '!seen[$2]++ {print $2}'` is the swiss-army knife for "deduplicate while preserving order".** It is the only `awk` one-liner I have used more times than `awk '{print $1}'`. Worth memorizing.
- **Base64 over DNS labels is the de-facto exfil encoding.** DNS labels can only contain `[A-Za-z0-9-]` (max 63 chars), and base64 fits that exactly. The `==` padding at the end of a chunk is the giveaway — real DNS never has `=` in a label, so anything ending in `=` is exfil data.
- **Encrypted protocols are visible but unreadable.** The 309 Kerberos-encrypted WSMAN frames look like valid HTTP, with status codes, content types, and bodies — but the bodies are `multipart/encrypted` and cannot be decoded without the Kerberos session key. Do not waste time trying to crack them. Filter them out and look for the plaintext sibling.
- **The first thing to do on a picoCTF Wireshark challenge is read both hints out loud.** "Did you really find *the* flag?" means there are decoys. "Look for traffic that seems suspicious" means there is one protocol you are ignoring. Both hints together point straight at the DNS.

### Flag wordplay decode

```
picoCTF{dns_3xf1l_ftw_deadbeef}
       ||  ||||| |||
       ||  ||||| ftw = "for the win" (classic IRC / 4chan slang)
       ||  3xf1l = "exfil" (3 → E, 1 → I — leet-speak for "exfiltration")
       dns = domain name system, the protocol being abused
       deadbeef = classic 0xDEADBEEF, the most famous magic
                  number in debugging lore — the same suffix as
                  the previous shark challenge, "peek-a-boo"
```

The whole chunk reads as **"DNS exfil for the win, deadbeef"** — a celebration of the technique. The `dns` prefix tells you the technique, `3xf1l` leet-speak-flattens it to "exfil", `ftw` celebrates that the technique works, and `deadbeef` is the picoCTF house signature for these Wireshark challenges (it also appeared in the previous one). The challenge author is being cheerful about teaching you a real-world post-exploitation trick: if you can run arbitrary code on a host and DNS is allowed out, you can exfiltrate any small blob — including a flag — through the resolver.
