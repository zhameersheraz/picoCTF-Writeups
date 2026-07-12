# Wireshark doo dooo do doo... — picoCTF Writeup

**Challenge:** Wireshark doo dooo do doo...  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 50  
**Flag:** `picoCTF{p33kab00_1_s33_u_deadbeef}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Can you find the flag?

**Hints shown in challenge:**

> (no hints were provided for this challenge — the title *is* the hint: it is a tongue-in-cheek riff on "Wireshark", the most well-known packet analysis tool in the world)

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
- **scapy** — a Python library. Load a pcap and walk packets programmatically. Useful for scripted analysis, but overkill for a 50-point challenge.

### The "Protocol Hierarchy" view

When you first open a pcap, you do not know what is inside. The fastest way to get a map of the whole file is the **Protocol Hierarchy** statistics, which in tshark is `-q -z io,phs`. It prints a tree of every protocol layer (Ethernet, IP, TCP, HTTP, TLS, ...) with packet and byte counts at each level. For a beginner, this is the best "what kind of traffic is this?" view. If you see a lot of HTTP, you know to look for plaintext requests and responses. If you see a lot of TLS, the data is encrypted and you cannot read it without keys.

### The HTTP `GET /` request and `200 OK` response

HTTP is a request/response protocol. The client sends a request line like `GET / HTTP/1.1`, the server replies with a status line like `HTTP/1.1 200 OK`, then headers, then a body. A `200 OK` means "here is what you asked for". The body of a successful `GET /` is usually an HTML page, but it can also be a single line of plain text — which is exactly what this challenge serves.

### ROT13 — the Caesar cipher by another name

ROT13 is a substitution cipher that rotates every letter 13 places through the alphabet. `A` becomes `N`, `B` becomes `O`, ..., `M` becomes `Z`, then it wraps: `N` becomes `A`, ..., `Z` becomes `M`. The trick is that applying ROT13 twice gives you back the original. Numbers and symbols are not changed.

ROT13 is the de-facto "spoiler tag" of the early internet — Usenet, email subject lines, and CTF challenges use it so that the text is readable enough to recognize but not readable enough to spoil. On Linux, decoding a chunk of ROT13 is a one-liner with the `tr` command:

```
echo "Gur synt vf ..." | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

The argument pair `A-Za-z → N-ZA-Mn-za-m` says: "for every letter in A-Z, replace it with the letter 13 places later; for every letter in a-z, replace it with the letter 13 places later". Everything else is left alone.

### Why the right answer is "filter by HTTP, then read the body"

Wireshark is a fine choice for this challenge — open the file, type `http` in the display filter bar, scroll to the one response that is `text/html`, expand it, read the body, decode the ROT13. The tshark walkthrough below does the same thing from the terminal, with copy-pasteable commands and no GUI required. The Wireshark GUI version is included as an alternative at the end.

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the pcap

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/wireshark-doo && cd /work/wireshark-doo

┌──(zham㉿kali)-[/work/wireshark-doo]
└─$ cp ~/Downloads/shark1.pcapng .
```

I keep the artifact in a fresh folder so any output files I dump do not pollute the rest of my filesystem.

### Step 2 — Confirm what we have

```
┌──(zham㉿kali)-[/work/wireshark-doo]
└─$ file shark1.pcapng
shark1.pcapng: pcapng capture file - version 1.0

┌──(zham㉿kali)-[/work/wireshark-doo]
└─$ ls -la shark1.pcapng
-rw-r--r-- 1 zham zham 1558808 Jul 12 22:00 shark1.pcapng
```

A real pcapng. About 1.5 MB — small, but it is going to be packed with encrypted Windows event-forwarding traffic.

### Step 3 — Get the lay of the land: protocol hierarchy

```
┌──(zham㉿kali)-[/work/wireshark-doo]
└─$ tshark -r shark1.pcapng -q -z io,phs
===================================================================
Protocol Hierarchy Statistics
Filter: 

eth                                      frames:987 bytes:1524814
  ip                                     frames:985 bytes:1524716
    tcp                                  frames:985 bytes:1524716
      http                               frames:288 bytes:1158734
        tcp.segments                     frames:142 bytes:1078777
        mime_multipart                   frames:142 bytes:78526
          tcp.segments                   frames:142 bytes:78526
        data-text-lines                  frames:2 bytes:695
      tls                                frames:43 bytes:75441
        tcp.segments                     frames:2 bytes:6576
  arp                                    frames:2 bytes:98
===================================================================
```

What this tells me at a glance:

- **288 HTTP frames** — there is plaintext HTTP traffic in here. The flag is almost certainly inside one of those.
- **142 of those HTTP frames are MIME multipart with Kerberos-encrypted body** — that is Windows Event Forwarding (WEF) traffic, going from `192.168.38.104` to a Windows Event Collector on port 5985 (the WinRM-over-HTTP default). It is encrypted at the message level with Kerberos, so even though it sits on top of HTTP, the bodies are unreadable ciphertext.
- **2 frames are `data-text-lines`** — these are the two HTTP responses whose bodies are short, line-based text. One of them is the flag.
- **43 TLS frames** — encrypted HTTPS, also unreadable.

So the strategy is: ignore all the encrypted stuff, focus on the two `data-text-lines` HTTP responses, and read their bodies.

### Step 4 — Find the two readable HTTP responses

The fastest way is to filter to HTTP responses and skip the Kerberos-encrypted ones:

```
┌──(zham㉿kali)-[/work/wireshark-doo]
└─$ tshark -r shark1.pcapng -Y "http.response" -T fields -e frame.number -e http.response.code -e http.content_type 2>/dev/null | grep -v Kerberos
827      200     text/html
964      200     text/plain
```

Two unencrypted HTTP responses:

- **Frame 827** — `text/html`, 200 OK. This is suspicious: a 200 with `text/html` content type means the server returned a tiny HTML page. PicoCTF loves to serve the flag on the landing page of a remote server.
- **Frame 964** — `text/plain`, 200 OK. This is the AWS instance metadata response (`169.254.169.254` is the link-local AWS metadata IP). Body is the single word `none`. Ignore it.

Frame 827 is the one I care about.

### Step 5 — Find the request that triggered frame 827

A `200` is the response to a request. To know which request, I look at the IP endpoints:

```
┌──(zham㉿kali)-[/work/wireshark-doo]
└─$ tshark -r shark1.pcapng -Y "ip.addr==18.222.37.134" -T fields -e frame.number -e ip.src -e ip.dst -e http.request.method -e http.request.uri -e http.response.code 2>/dev/null
819     192.168.38.104   18.222.37.134
820     192.168.38.104   18.222.37.134
821     18.222.37.134    192.168.38.104
822     192.168.38.104   18.222.37.134
823     192.168.38.104   18.222.37.134   GET     /
824     18.222.37.134    192.168.38.104
825     192.168.38.104   18.222.37.134
826     18.222.37.134    192.168.38.104
827     18.222.37.134    192.168.38.104                           200
828     192.168.38.104   18.222.37.134
```

This is a normal HTTP request/response dance:

- **Frame 823** — `GET /` from the host to `18.222.37.134` (the server). The browser-style user-agent in the headers (Chrome 84 on Windows 10) tells me the host was a normal user machine.
- **Frame 827** — the server's `200 OK` response.

So the host `192.168.38.104` browsed to `http://18.222.37.134/` and the server returned something interesting. Time to read the body.

### Step 6 — Extract the body of frame 827

`-T fields -e http.file_data` tells tshark to print just the reassembled HTTP body as one hex blob. For a `text/html` response, tshark does not have a fancy HTML parser, so it just dumps the raw bytes as a single hex string.

```
┌──(zham㉿kali)-[/work/wireshark-doo]
└─$ tshark -r shark1.pcapng -Y "frame.number == 827" -T fields -e http.file_data 2>/dev/null
Gur synt vf cvpbPGS{c33xno00_1_f33_h_qrnqorrs}\n
```

There it is. The body of the page is a single line:

```
Gur synt vf cvpbPGS{c33xno00_1_f33_h_qrnqorrs}
```

This is **ROT13** — the "spoiler tag" Caesar cipher. "Gur synt vf" is ROT13 for "The flag is", and the bracket contents are the encoded flag. The encoding is visible if you know what to look for: `cvpbPGS` looks like it should be `picoCTF` but every letter is rotated 13 positions.

### Step 7 — Decode the ROT13

The classic Linux way to decode ROT13 is the `tr` command:

```
┌──(zham㉿kali)-[/work/wireshark-doo]
└─$ echo "Gur synt vf cvpbPGS{c33xno00_1_f33_h_qrnqorrs}" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
The flag is picoCTF{p33kab00_1_s33_u_deadbeef}
```

Decoded plaintext:

```
The flag is picoCTF{p33kab00_1_s33_u_deadbeef}
```

Flag captured.

### Step 8 — Submit

I copy the flag (without the "The flag is " prefix and without the trailing period) into the picoCTF submission box:

```
picoCTF{p33kab00_1_s33_u_deadbeef}
```

Solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | Copied `shark1.pcapng` into `/work/wireshark-doo` | Clean working directory |
| 0:00 | `file shark1.pcapng` | Confirmed valid pcapng, 1.5 MB |
| 0:01 | `tshark -q -z io,phs` | Saw 288 HTTP frames and 43 TLS frames. Decided HTTP is the place to look |
| 0:02 | `tshark -Y "http.response" ... \| grep -v Kerberos` | Found two readable responses: frame 827 (`text/html`) and frame 964 (`text/plain`) |
| 0:02 | `tshark -Y "ip.addr==18.222.37.134" -T fields ...` | Confirmed frame 823 is the matching `GET /` request to `18.222.37.134` |
| 0:03 | `tshark -Y "frame.number == 827" -T fields -e http.file_data` | Got the body: `Gur synt vf cvpbPGS{c33xno00_1_f33_h_qrnqorrs}` |
| 0:03 | Recognized ROT13 by the shape of `cvpbPGS` vs `picoCTF` | No tool call, just pattern recognition |
| 0:04 | `echo "..." \| tr 'A-Za-z' 'N-ZA-Mn-za-m'` | Decoded to `The flag is picoCTF{p33kab00_1_s33_u_deadbeef}` |
| 0:04 | Submitted `picoCTF{p33kab00_1_s33_u_deadbeef}` | Challenge marked solved |

Internally, the pcap captured a single HTTP request from a Windows 10 host (`192.168.38.104`, Chrome 84) to a remote Ubuntu/Apache server (`18.222.37.134`). The server returned a 1-line page containing the ROT13-encoded flag. The vast majority of the 1.5 MB capture is unrelated encrypted Windows Event Forwarding traffic (HTTP multipart with Kerberos-encrypted bodies) — that is noise. The signal is a single `GET /` request, a single `200 OK` response, and a single ROT13 line. The "challenge" is mostly about not getting distracted by the encrypted stuff and recognizing ROT13 on sight.

---

## Alternative Methods

### Method 1 — Wireshark GUI walk-through

If you prefer clicking, the Wireshark version of the same solve:

1. Open `shark1.pcapng` in Wireshark.
2. Type `http.response && !http.content_type contains "Kerberos"` in the display filter bar. Wireshark shows exactly two packets: 827 (`text/html`) and 964 (`text/plain`).
3. Click on frame 827. In the packet details pane, expand **Hypertext Transfer Protocol** → **Line-based text data: text/html**. The body is `Gur synt vf cvpbPGS{c33xno00_1_f33_h_qrnqorrs}`.
4. From a terminal, run `echo 'Gur synt vf cvpbPGS{c33xno00_1_f33_h_qrnqorrs}' | tr 'A-Za-z' 'N-ZA-Mn-za-m'`. The flag is in the output.

Same answer, more mouse work.

### Method 2 — `tshark` "Follow TCP Stream" on the right stream

Instead of hunting for a specific frame number, you can ask tshark to glue the entire TCP conversation into one blob:

```
┌──(zham㉿kali)-[/work/wireshark-doo]
└─$ tshark -r shark1.pcapng -Y "ip.addr==18.222.37.134" -z "follow,tcp,ascii,0" -q 2>/dev/null
===================================================================
Follow: tcp,ascii
Filter: tcp.stream eq 0
Node 0: 18.222.37.134:80
Node 1: 192.168.38.104:64093
        GET / HTTP/1.1
Host: 18.222.37.134
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.105 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,...
        HTTP/1.1 200 OK
Date: Mon, 10 Aug 2020 01:51:45 GMT
Server: Apache/2.4.29 (Ubuntu)
Last-Modified: Fri, 07 Aug 2020 00:45:02 GMT
Content-Length: 52
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html
Gur synt vf cvpbPGS{c33xno00_1_f33_h_qrnqorrs}
===================================================================
```

The full request, the headers, and the encoded body are all in one block. The `Content-Length: 52` header confirms the body is exactly 52 bytes (including the trailing newline) — small enough to read in one go.

### Method 3 — `tcpdump` for the body

If you do not have tshark at all, `tcpdump` works too, just less prettily:

```
┌──(zham㉿kali)-[/work/wireshark-doo]
└─$ tcpdump -r shark1.pcapng -nn -X 'tcp and src host 18.222.37.134 and port 80' 2>/dev/null
```

`-X` prints hex + ASCII for every packet. The packet with the `200 OK` response will contain the encoded flag in the ASCII column on the right. You will need to scroll past the hex header dump to find it.

### Method 4 — `strings` and a hint of luck

This one almost works but does not quite get you the answer, so it is worth showing as a cautionary tale:

```
┌──(zham㉿kali)-[/work/wireshark-doo]
└─$ strings -n 8 shark1.pcapng | grep -iE 'pico|flag|ctf'
peek-a-boo
```

`strings` does find the substring `peek-a-boo` somewhere in the capture — that is not the flag, but it is the un-ROT13ed version of `p33kab00` from inside the flag, so it is a real breadcrumb. However, `strings` does *not* find `picoCTF` in the pcap, because the literal string `picoCTF` never appears in cleartext — it only appears as the ROT13 ciphertext `cvpbPGS`. So `strings` alone is not enough; you still need to filter to the HTTP response and decode ROT13 by hand.

The lesson: `strings` is great for finding URLs, paths, error messages, and embedded files, but it is hopeless against a stream cipher (or even a Caesar cipher) embedded in a single short HTTP body.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Confirm the downloaded blob is a real pcapng file | Easy |
| `tshark -q -z io,phs` | Get a one-shot map of every protocol layer in the pcap (Ethernet, IP, TCP, HTTP, TLS, ...) | Easy |
| `tshark -Y "http.response" -T fields` | List every HTTP response with its frame number, status code, and content type | Easy |
| `grep -v Kerberos` | Strip out the noisy WinRM/Kerberos-encrypted responses from the list | Easy |
| `tshark -Y "ip.addr==18.222.37.134"` | Find the request that triggered the response of interest | Easy |
| `tshark -Y "frame.number == 827" -T fields -e http.file_data` | Extract the raw body of the HTTP response in a single hex/ASCII blob | Easy |
| `tr 'A-Za-z' 'N-ZA-Mn-za-m'` | Decode the ROT13-encoded flag back to plaintext | Easy |
| Wireshark GUI (alt) | Open the pcap, filter to `http.response && !http.content_type contains "Kerberos"`, click packet 827, read the body in the packet details pane | Easy |
| `tshark -z "follow,tcp,ascii,0"` (alt) | Glue the entire HTTP conversation (request + response + headers + body) into one readable block | Easy |
| `tcpdump -nn -X` (alt) | Hex+ASCII dump of every packet — works without tshark, just less pretty | Easy |
| `strings \| grep` (alt) | Tried and failed to find `picoCTF` because the flag is ROT13 encoded in the body, not the raw text | n/a |

---

## Key Takeaways

- **The protocol hierarchy view is the first thing to check on a new pcap.** `tshark -q -z io,phs` gives you a tree of every protocol in the capture with byte counts. It tells you instantly whether the pcap is full of HTTP (readable), TLS (encrypted), DNS, SMB, or some mix. A 5-second scan tells you where to spend the next 5 minutes.
- **Most of the bytes in a real-world pcap are usually noise.** A 1.5 MB capture is not 1.5 MB of useful data — most of it is encrypted Windows event forwarding, TLS handshakes, ARP, retries, ACKs, and so on. The signal is often a single packet. The job is to filter the noise out, not to read every byte.
- **"Text in a single-line HTTP response" is the most common flag format for picoCTF's pcap challenges.** The server returns a one-line body with the flag (or an encoded version of it) on `GET /`. Filtering to `http.response.code == 200` and looking at the body is the muscle-memory move.
- **ROT13 is the de-facto CTF "spoiler tag".** It is a Caesar cipher with a shift of 13, and it is trivially reversible with `tr 'A-Za-z' 'N-ZA-Mn-za-m'`. If you see flag-shaped text that does not match the usual `picoCTF{...}` pattern, try ROT13 first. The shape of the encoded prefix `cvpbPGS` next to the un-encoded numbers and braces is a dead giveaway.
- **Encrypted protocols are visible but unreadable.** The 142 Kerberos-encrypted HTTP frames in this pcap look like valid HTTP — they have status codes, content types, and bodies — but the bodies are `multipart/encrypted` and cannot be decoded without the Kerberos session key. Do not waste time trying to crack them. Filter them out and look for the plaintext sibling.
- **`strings` is a starting point, not a finishing tool.** It is great for finding URLs and error messages, but it cannot break a substitution cipher. For pcap challenges, `strings` is a "let me skim for obvious things" tool; the real work is always done by `tshark` (or Wireshark) with the right display filter.

### Flag wordplay decode

```
picoCTF{p33kab00_1_s33_u_deadbeef}
        ||||||  |  |||  |  ||||||||
        ||||||  |  |||  |  deadbeef = classic 0xDEADBEEF, the most famous
        ||||||  |  |||  |             magic number in debugging lore
        ||||||  |  |||  u = "you" (single letter, leet-style)
        ||||||  |  s33 = "see" (3 → E, classic leet)
        ||||||  1 = "I" or just the digit 1
        |||||| _ = literal underscore
        p33kab00 = "peek-a-boo" (3 → E, 0 → O, 00 → OO — the
                   "peek-a-boo" breadcrumb that `strings` finds
                   in the pcap)
```

The whole chunk reads as **"peek-a-boo, I see you, deadbeef"** — a cute play on the fact that the flag was hidden in plain sight inside an HTTP response, and that the act of capturing and reading the pcap literally lets you peek at someone else's network traffic. The `0xDEADBEEF` suffix is a nod to the long tradition of using deadbeef, cafebabe, and similar hex strings as "magic numbers" in code — fitting for a challenge about pulling bytes out of a wiretap and making sense of them.
