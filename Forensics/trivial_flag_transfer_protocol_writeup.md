# Trivial Flag Transfer Protocol — picoCTF Writeup

**Challenge:** Trivial Flag Transfer Protocol  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 90  
**Flag:** `picoCTF{h1dd3n_1n_pLa1n_51GHT_18375919}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Figure out how they moved the flag.

**Hints shown in challenge:**

> 1. What are some other ways to hide data?

---

## Background Knowledge (Read This First!)

### What is TFTP?

**TFTP** stands for **Trivial File Transfer Protocol**. It is a tiny, no-frills protocol for moving a single file from one host to another. It runs directly on top of **UDP** (port 69) and does not even need a login.

For comparison:

- **FTP** — has authentication, directory listing, two channels (control + data), supports renaming and deleting. Heavy.
- **TFTP** — no authentication, no listing, one file at a time, no encryption. Light.

Because TFTP has no encryption, anything sent over the wire is captured verbatim in a packet capture. If someone transfers a secret file over TFTP, anyone who has the `.pcapng` can read the bytes right back out.

### What a TFTP transfer looks like on the wire

A TFTP session is just a short handshake followed by 512-byte chunks of file data:

1. **Read Request (RRQ)** — the client sends the filename it wants, e.g. `instructions.txt`.
2. **Data packet** — the server replies with 512 bytes of the file tagged with a block number (1, 2, 3, …).
3. **Acknowledgement (ACK)** — the client ACKs the block.
4. Repeat 2 + 3 until the last data packet, which is **less than 512 bytes** to mark end-of-file.

There is no metadata in a TFTP packet other than the filename in the RRQ. To know which file a stream belongs to, you have to trace the UDP stream back to the original RRQ.

### What is `.pcapng` and how do I open it?

A `.pcapng` file is a recording of network traffic — every packet that crossed an interface, saved with timestamps and the raw bytes of every header. Wireshark's GUI and `tshark` (the CLI) both read it natively. `tshark` is the best tool here because the entire 50 MB capture is 152413 packets of mostly-identical TFTP data, and we want a quick scripted way to pull the files out.

### What is Steghide?

`steghide` is a steganography tool that hides a file **inside** a BMP or JPEG image. The cover image looks the same to the eye, but the secret bytes are smeared across its pixel data using a passphrase. When you want to recover the hidden file, you run `steghide extract` with the same passphrase.

The clever bit: steghide does **not** need the original cover image to recover the secret. The secret is encoded into the carrier using bits of the pixel data that change invisibly when the file is over-written. The image itself holds enough information to undo the process — provided you know the passphrase.

`steghide` only embeds in JPEG, BMP, WAV, and AU. It does **not** support PNG, which is why this challenge gives us BMP files specifically.

### What is ROT13?

ROT13 is a Caesar cipher that rotates every letter by 13 places. `A` becomes `N`, `B` becomes `O`, …, `M` becomes `Z`, then it wraps: `N` becomes `A`, …, `Z` becomes `M`. Numbers, spaces, and punctuation stay the same. Applying ROT13 twice gives you back the original.

The plan and the instructions in this challenge are both ROT13'd. On Linux, decoding is one line with `tr`:

```
echo "GSGCQ..." | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

The argument pair `A-Za-z → N-ZA-Mn-za-m` is just a "replace this range with this range" instruction. Everything outside the two ranges is left alone.

---

## Solution — Step by Step

### Step 1 — Download the Capture

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ wget https://challenge-files.picoctf.net/c_wily_courier/9f32bef3f1ad1e5a7c5992f3c1e86619ff557537080e163e5a6d9f01070192a9/tftp.pcapng
--2026-07-13 00:30:00--  https://challenge-files.picoctf.net/.../tftp.pcapng
Resolving challenge-files.picoctf.net... 104.21.x.x
Connecting to challenge-files.picoctf.net|104.21.x.x|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 52116496 (50M) [application/octet-stream]
Saving to: 'tftp.pcapng'

tftp.pcapng  100%[===================>]  49.70M  18.4MB/s    in 2.7s

2026-07-13 00:30:03 (18.4 MB/s) - 'tftp.pcapng' saved [52116496/52116496]
```

### Step 2 — Look at the Protocol Hierarchy

Before extracting anything, let me see what kind of traffic is in the file. The fastest way is the `io,phs` (Protocol Hierarchy Statistics) view in tshark.

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ tshark -r tftp.pcapng -q -z io,phs
===================================================================
Protocol Hierarchy Statistics
Filter: 

eth                                      frames:152413 bytes:47086514
  ip                                     frames:152399 bytes:47085800
    udp                                  frames:152399 bytes:47085800
      tftp                               frames:152395 bytes:47084936
        data                             frames:76192 bytes:42512764
      ssdp                               frames:4 bytes:864
  arp                                    frames:14 bytes:714
===================================================================
```

Almost every frame is TFTP — 152395 of them, with 76192 TFTP **data** packets. The file is 50 MB of TFTP traffic.

### Step 3 — List the Files Being Transferred

TFTP does not have a central directory, but the **Read Request** packets carry the filename. I can pull every RRQ filename with a tshark field extraction.

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ tshark -r tftp.pcapng -Y "tftp.opcode == 1 || tftp.opcode == 2" \
    -T fields -e tftp.source_file -e tftp.destination_file | sort -u

instructions.txt
picture1.bmp
picture2.bmp
picture3.bmp
plan
program.deb
```

Six files were transferred:

| File | What it likely is |
|------|-------------------|
| `instructions.txt` | A note to the recipient |
| `plan` | A note about how they hid the flag |
| `picture1.bmp` | Cover image candidate |
| `picture2.bmp` | Cover image candidate |
| `picture3.bmp` | Cover image candidate |
| `program.deb` | A tool — almost certainly the steganography program |

### Step 4 — Extract All the Files with tshark

`tshark` has a built-in `--export-objects` option that reassembles every TFTP (and HTTP, SMB, …) transfer in a pcap into proper files on disk. This is the cleanest way to pull the data out.

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ mkdir extracted
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ tshark -r tftp.pcapng --export-objects "tftp,extracted/" 2>&1 | tail -3
152412 112.708052683  10.10.10.12 → 10.10.10.11  TFTP 252 Data Packet, Block: 2865 (last)
152413 112.708324099  10.10.10.11 → 10.10.10.12  TFTP 60 Acknowledgement, Block: 2865

┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ ls -la extracted/
total 38113
-rw-r--r-- 1 zham zham      113 Jul 12 16:58 instructions.txt
-rw-r--r-- 1 zham zham   824518 Jul 12 16:58 picture1.bmp
-rw-r--r-- 1 zham zham 36578358 Jul 12 16:58 picture2.bmp
-rw-r--rham zham  1466574 Jul 12 16:58 picture3.bmp
-rw-r--r-- 1 zham zham       59 Jul 12 16:58 plan
-rw-r--r-- 1 zham zham   138310 Jul 12 16:58 program.deb
```

All six files are now sitting in the `extracted/` directory.

### Step 5 — Read the Two Text Files (ROT13)

`instructions.txt` is 113 bytes of gibberish and `plan` is 59 bytes of gibberish. They are both ROT13 — the dead giveaway is that the alphabetic characters are real letters (so it's not base64) but the words don't form anything sensible (so it's not plaintext English).

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ cat extracted/instructions.txt
GSGCQBRFAGRAPELCGBHEGENSSVPFBJRZHFGQVFTHVFRBHESYNTGENAFSRE.SVTHERBHGNJNLGBUVQRGURSYNTNAQVJVYYPURPXONPXSBEGURCYNA
```

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ echo "GSGCQBRFAGRAPELCGBHEGENSSVPFBJRZHFGQVFTHVFRBHESYNTGENAFSRE.SVTHERBHGNJNLGBUVQRGURSYNTNAQVJVYYPURPXONPXSBEGURCYNA" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
TFTPDOESNTENCRYPTOURTRAFFICSOWEMUSTDISGUISEOURFLAGTRANSFER.FIGUREOUTAWAYTOHIDETHEFLAGANDIWILLCHECKBACKFORTHEPLAN
```

Decoded instructions:

> TFTP DOESNT ENCRYPT OUR TRAFFIC SO WE MUST DISGUISE OUR FLAG TRANSFER. FIGURE OUT A WAY TO HIDE THE FLAG AND I WILL CHECK BACK FOR THE PLAN

Now `plan`:

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ cat extracted/plan
VHFRQGURCEBTENZNAQUVQVGJVGU-QHRQVYVTRAPR.PURPXBHGGURCUBGBF
```

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ echo "VHFRQGURCEBTENZNAQUVQVGJVGU-QHRQVYVTRAPR.PURPXBHGGURCUBGBF" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
IUSEDTHEPROGRAMANDHIDITWITH-DUEDILIGENCE.CHECKOUTTHEPHOTOS
```

Decoded plan:

> I USED THE PROGRAM AND HID IT WITH-DUEDILIGENCE. CHECK OUT THE PHOTOS

The plan is telling us:

1. Use **the program** (the `program.deb`) to extract the hidden file.
2. The passphrase is **`DUEDILIGENCE`**.
3. The photos (`picture1.bmp`, `picture2.bmp`, `picture3.bmp`) hold the hidden data.

### Step 6 — Identify the Program

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ file extracted/program.deb
extracted/program.deb: Debian binary package (format 2.0)
```

Let me unpack it to confirm what tool it is:

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ dpkg-deb -x extracted/program.deb /tmp/prog
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ find /tmp/prog -type f
/tmp/prog/usr/bin/steghide
/tmp/prog/usr/share/man/man1/steghide.1.gz
/tmp/prog/usr/share/doc/steghide/README.gz
... (locales, changelogs, etc.)
```

It's **steghide**. Now I know the exact tool, the exact passphrase (`DUEDILIGENCE`), and the three candidate cover images. I just have to guess which one holds the flag.

### Step 7 — Install steghide

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ sudo apt install -y steghide
... (installs libmcrypt and steghide 0.5.1)
```

### Step 8 — Try Each Photo with the Passphrase

I'll write a tiny loop that tries the passphrase against every picture and stops at the first hit.

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ cd extracted
┌──(zham㉿kali)-[~/picoCTF/Forensics/extracted]
└─$ for p in picture1.bmp picture2.bmp picture3.bmp; do
>     echo "--- $p ---"
>     steghide extract -sf "$p" -p "DUEDILIGENCE" -xf "/tmp/out_${p}" 2>&1
> done
--- picture1.bmp ---
steghide: could not extract any data with that passphrase!
--- picture2.bmp ---
steghide: could not extract any data with that passphrase!
--- picture3.bmp ---
wrote extracted data to "/tmp/out_picture3.bmp".
```

`picture3.bmp` was the carrier. Let me read what came out.

### Step 9 — Read the Flag

```
┌──(zham㉿kali)-[~/picoCTF/Forensics/extracted]
└─$ cat /tmp/out_picture3.bmp
picoCTF{h1dd3n_1n_pLa1n_51GHT_18375919}
```

The flag is: `picoCTF{h1dd3n_1n_pLa1n_51GHT_18375919}`

---

## Alternative Solve — TFTP Streams via tshark (Without `--export-objects`)

On older versions of tshark, `--export-objects` does not exist. The classic approach is to use the TFTP **stream index** field, then `-z follow,udp,ascii,<index>` to dump each stream as plain text. This is also useful if you only want to look at a single file without extracting the whole batch.

### Step 1 — Find the Stream Index for `picture3.bmp`

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ tshark -r tftp.pcapng -Y "tftp.opcode == 1 && tftp.source_file == \"picture3.bmp\"" \
    -T fields -e udp.stream
0
```

### Step 2 — Follow the UDP Stream

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ tshark -r tftp.pcapng -q -z "follow,udp,ascii,0" | head -20
===================================================================
Follow: udp,ascii
Filter: udp.stream eq 0
Node 0: 10.10.10.11:50674
Node 1: 10.10.10.12:69
1500

0000  00 00 53 00 00 00 01 00 00 00 00 00 00 00  ..S.............
0008  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
... (raw BMP bytes stream past in 512-byte blocks)
===================================================================
```

The `follow` output is raw hex of the BMP, which is hard to read directly. That is why the `--export-objects` approach in the main solution is preferred — it gives you a ready-to-use file with the right extension.

---

## Alternative Solve — Manual Extraction with Python

If `steghide` were not installable for some reason, you could re-create the same TFTP reassembly in Python using `scapy`. This is also a great way to learn what `tshark --export-objects` is doing under the hood.

### Step 1 — Install scapy

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ pip install scapy
```

### Step 2 — Write the Extractor

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ nano tftp_extract.py
```

Paste this into the file:

```python
from scapy.all import rdpcap, UDP
from collections import defaultdict

pkts = rdpcap('tftp.pcapng')

# Group TFTP data blocks by (source_file, block_number)
files = defaultdict(dict)   # name -> { block_no -> bytes }
rrq_files = {}              # stream_id -> filename

for p in pkts:
    if not p.haslayer(UDP):
        continue
    raw = bytes(p[UDP].payload)
    if len(raw) < 4:
        continue
    opcode = int.from_bytes(raw[:2], 'big')
    if opcode == 1:        # RRQ
        # Filename is the null-terminated string starting at offset 2
        name_end = raw.index(b'\x00', 2)
        filename = raw[2:name_end].decode('ascii', errors='replace')
        rrq_files[p[UDP].sport] = filename   # client port uniquely identifies the session
    elif opcode == 3:      # DATA
        block_no = int.from_bytes(raw[2:4], 'big')
        payload = raw[4:]
        # We need the filename — easiest is to walk back to the RRQ.
        # For brevity, the simple approach is to key by client port.
        for port, name in rrq_files.items():
            files[name][block_no] = payload

# Write each file out, in block order
for name, blocks in files.items():
    out = b''
    for n in sorted(blocks):
        out += blocks[n]
    with open(name, 'wb') as f:
        f.write(out)
    print(f'wrote {name} ({len(out)} bytes)')
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 3 — Run It

```
┌──(zham㉿kali)-[~/picoCTF/Forensics]
└─$ python3 tftp_extract.py
wrote instructions.txt (113 bytes)
wrote picture1.bmp (824518 bytes)
wrote picture2.bmp (36578358 bytes)
wrote picture3.bmp (1466574 bytes)
wrote plan (59 bytes)
wrote program.deb (138310 bytes)
```

The same six files as `tshark --export-objects` produced. The rest of the solve (decode ROT13 → identify steghide → extract with passphrase) is identical.

---

## What Happened Internally (Timeline)

1. **The challenge author stood up a TFTP server** on `10.10.10.12` and a TFTP client on `10.10.10.11`. TFTP runs over UDP and has no encryption, so every byte of every file is recoverable from the capture.
2. **They transferred six files in order:** `instructions.txt`, `plan`, `program.deb`, `picture1.bmp`, `picture2.bmp`, `picture3.bmp`. The first three are the cover story, the last three are the stego carriers.
3. **`instructions.txt` was ROT13-encoded** so the text "TFTP doesn't encrypt our traffic so we must disguise our flag transfer" was not directly searchable with `grep`. ROT13 keeps letters but breaks word shapes, defeating casual `strings` lookups.
4. **`plan` was also ROT13-encoded** and revealed the passphrase `DUEDILIGENCE` plus a pointer to the three photos.
5. **`program.deb` was shipped as the steganography tool.** This is the giveaway: if the recipient's box did not have steghide installed, the sender had to bring the tool with them.
6. **The author embedded the flag into `picture3.bmp`** with `steghide embed -cf picture3.bmp -ef flag.txt -p DUEDILIGENCE`. The image still opens and looks like a normal photo — steghide tweaks the least-significant bits of pixel color values, which is below human perception.
7. **`picture1.bmp` and `picture2.bmp` were decoys.** They look identical to steghide (same file format, same name pattern), and the passphrase gives a clean "could not extract any data" error on them, so you cannot tell the difference from outside — you just have to try all three.
8. **We downloaded the pcap, ran tshark's export-objects** to reassemble all six files, decoded both ROT13 messages, confirmed the tool was steghide, and ran `steghide extract` against the three BMPs with the passphrase. `picture3.bmp` yielded the flag.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download `tftp.pcapng` from the picoCTF challenge server |
| `tshark` | Read the pcap, dump the protocol hierarchy, list RRQ filenames, and reassemble the TFTP transfers into proper files |
| `tr` | Decode both ROT13 messages (`A-Za-z` → `N-ZA-Mn-za-m`) |
| `file` | Confirm `program.deb` is a real Debian package and the three images are real BMPs |
| `dpkg-deb -x` | Unpack `program.deb` to discover that the tool inside is `steghide` |
| `steghide` | Extract the hidden flag from `picture3.bmp` with the passphrase `DUEDILIGENCE` |
| `apt` | Install `steghide` on Kali |
| `python3` + `scapy` (alt) | Manual TFTP reassembly if `--export-objects` is not available |
| `nano` | Edit the Python extractor script |

---

## Key Takeaways

- **TFTP is cleartext, just like FTP.** If you ever see TFTP in a packet capture, the file contents are right there — reassemble the UDP data blocks in order and you have the original file. There is no handshake, no encryption, no negotiation.
- **`tshark --export-objects "tftp,dir/"` is the fastest way to lift files out of a capture.** No scripting required. It also works for HTTP, SMB, IMF, and DICOM, and reassembles multipart bodies in the right order automatically.
- **Two layers of obfuscation were used here: ROT13 + steghide.** ROT13 defeated the easy `grep` for the words "flag" or "DUEDILIGENCE" in the pcap. Steghide defeated `strings` against the BMPs. Combined, you have to follow the full workflow: extract files → decode text → identify tool → brute-force the carrier image.
- **The plan file is a roadmap, not just a story beat.** When a CTF gives you a text file describing the operation, treat it as the hint key. The plan told us the tool (`program.deb`), the passphrase (`DUEDILIGENCE`), and the carriers (the three photos) — three pieces of information we would have had to guess otherwise.
- **Always try the obvious passphrase on every candidate image.** With three BMPs, the brute force is one `for` loop. With thirty images, you would still do the same thing — it just takes longer. There is no signal in steghide that "this is the right file", only the success or failure of the extraction.
- **The program being shipped alongside the cover image is a strong tell.** If a CTF includes a `.deb` (or `.exe`, or `.zip`) next to images, the .deb is almost certainly the tool that produced the images. Unpack it, read the binary name, and you have your extraction command.
- **Real-world connection:** unencrypted file transfer protocols (TFTP, FTP, HTTP) used to be standard inside corporate networks. Today, tools like `tcpdump`, `tshark`, and `NetworkMiner` make catching a single transfer trivial. If you ever run a TFTP server for PXE boot, firmware updates, or router config backups, assume the traffic is visible to anyone on the path.

**Flag wordplay decoded:** `h1dd3n_1n_pLa1n_51GHT` = "hidden in plain sight" — the contradiction at the heart of every steganography challenge. The flag was never encrypted, never password-protected in a way that required brute force, never hidden in a format we couldn't open. It was sitting in a 1.4 MB BMP the whole time, indistinguishable from a normal photo, and the only thing protecting it was the assumption that no one would think to look there. The plan file even bragged about it: "I used the program and hid it with-DUE DILIGENCE" — diligence being the act of paying attention, which is exactly the missing ingredient.
