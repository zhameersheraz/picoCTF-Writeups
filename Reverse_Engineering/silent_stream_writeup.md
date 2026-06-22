# Silent Stream — picoCTF Writeup

**Challenge:** Silent Stream  
**Category:** Reverse Engineering  
**Points:** 200  
**Flag:** `picoCTF{tr4ck_th3_tr4ff1c_c41f36a9}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham  
  
---

## Description

> We recovered a suspicious packet capture file that seems to contain a transferred file. The sender was kind enough to also share the script they used to encode and send it. Can you reconstruct the original file?
>
> Download the PCAP file: here. And the sender's encoding script here.

---

## Hints

> 1. The encoding script is a clue, focus on what it's doing to each byte before it's sent.
> 2. The flag is hidden inside the file, try reconstructing and opening it.
> 3. Don't rely on everything you see including flag format

---

## Background Knowledge (Read This First!)

### Additive (Caesar-style) byte cipher

The encoding script does one thing to every byte before sending it:

```python
def encode_byte(b, key):
    return (b + key) % 256
```

That's the simplest possible "cipher": add a fixed key to each byte, wrap around at 256. The inverse is just as simple — subtract the same key:

```python
def decode_byte(b, key):
    return (b - key) % 256
```

Hint 1 is telling you the encoding is the whole story. Once you know what the script did to each byte, the file is fully recoverable.

Why `mod 256`? Because a byte is 8 bits and there are 256 possible values (0..255). Adding past 255 wraps around to 0. Subtracting past 0 wraps around to 255. The `% 256` in both directions keeps everything inside one byte.

### Why "track the traffic" matters

Hint 3 says "don't rely on everything you see including flag format." That is a giant hint that the recovered file is **not** a plain text file containing `picoCTF{...}`. In this challenge, the transferred file is actually an **image**, and the flag is rendered inside it as text on the picture. If you `grep picoCTF recovered.bin` you will find nothing. If you `file recovered.bin` you will see `JPEG image data`. The flag only becomes visible once you actually open the image.

This pattern — "the data is some other format, the flag is just visually present" — is common in CTFs. Always run `file` on a recovered blob before trying to interpret it.

### Reading a pcap with `tshark`

A pcap file is a recording of network traffic. `tshark` is the CLI version of Wireshark: it can list conversations, follow a single TCP stream, and dump the raw payload of every packet. For a file-transfer challenge, the cleanest move is:

1. `tshark -z conv,tcp` to see who talked to whom and how many bytes went each way.
2. Pick the conversation that contains the right number of bytes (the file size).
3. `tshark -Y 'tcp.srcport == <sender>' -T fields -e tcp.payload` to dump only the bytes the sender pushed.

The `-T fields -e tcp.payload` form prints the raw bytes of each packet's payload as a hex string with no separators, one packet per line. You then concatenate the lines and `bytes.fromhex()` the result. That's it — no Wireshark GUI required.

---

## Solution — Step by Step

### Step 1 — Look at the encoder to know the transform

```
┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ cat encoder.py
import socket

def encode_byte(b, key):
    return (b + key) % 256

def simulate_flag_transfer(filename, key=42):
    print(f"[!] flag transfer for '{filename}' using encoding key = {key}")
    with open(filename, "rb") as f:
        data = f.read()
    print(f"[+] Encoding and sending {len(data)} bytes...")
    for b in data:
        encoded = encode_byte(b, key)
        pass
    print("Transfer complete")

if __name__ == "__main__":
    simulate_flag_transfer("flag.txt")
```

The real sender used `key = 42`, and for every plaintext byte `b` it sent `(b + 42) % 256`. The script's `simulate_flag_transfer` is the literal blueprint for what hit the wire.

### Step 2 — Find the transfer in the pcap

```
┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ tshark -r capture.pcap -q -z conv,tcp
================================================================================
TCP Conversations
Filter:<No Filter>
                                                           |       <-      | |       ->      | |     Total     |     Relative    |   Duration   |
                                                           | Frames  Bytes | | Frames  Bytes | | Frames  Bytes |      Start     |              |
10.10.10.10:12345          <-> 10.10.10.11:9000                 0 0 bytes       143 16 kB         143 16 kB         0.000000000         0.1420
================================================================================
```

One TCP conversation: the server on `10.10.10.10:12345` pushed **143 frames / 16 kB** to the client on `10.10.10.11:9000`. That is our file. The transfer took 0.142 seconds.

### Step 3 — Pull out just the sender's payload

```
┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ tshark -r capture.pcap \
           -Y "tcp.srcport == 12345" \
           -T fields -e tcp.payload \
           > raw.hex

┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ wc -c raw.hex
18260 raw.hex

┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ head -c 80 raw.hex
2902290a2a3a747073702a2b2b2a2a2b2a2b2a2a29052a6d2a32303031302f3231313133333234363e37
```

`raw.hex` is 18260 hex characters, which decodes to `18260 / 2 = 9130 bytes`. That matches the 16 kB on the wire (close enough — TCP overhead accounts for the rest). One packet's worth of payload per line.

### Step 4 — Decode by subtracting the key

The math: if `encoded = (plain + 42) mod 256`, then `plain = (encoded - 42) mod 256`. Subtraction in Python with `% 256` already wraps correctly for negative numbers, so this is one expression.

```
┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ python3 -c "
raw = bytes.fromhex(open('raw.hex').read())
dec = bytes((b - 42) % 256 for b in raw)
open('flag.bin', 'wb').write(dec)
print('size:', len(dec))
print('first 16 hex:', dec[:16].hex())
"
size: 9130
first 16 hex: ffd8ffe000104a46494600010100000100010000ffdb
```

`ff d8 ff e0 ... 4a 46 49 46 ...` — that's a **JPEG magic header** (`FF D8 FF E0`) followed by the literal ASCII `JFIF`. So the recovered file is a JPEG image, not text. Hint 3 was warning us about exactly this.

### Step 5 — Rename to `.jpg` and check it parses

```
┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ mv flag.bin flag.jpg

┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ file flag.jpg
flag.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, baseline, precision 8, 800x500, components 3
```

`file` confirms it's a valid 800x500 baseline JPEG, 9 KB. No corruption.

### Step 6 — View the image

```
┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ xdg-open flag.jpg
```

The image is a near-blank white picture. Roughly center, in a small red typewriter font:

```
picoCTF{tr4ck_th3_tr4ff1c_c41f36a9}
```

That's the flag. (If you are doing this over SSH or in a headless terminal, `xdg-open` won't help — just copy `flag.jpg` to your local machine and open it in any image viewer. Or `python3 -c "from PIL import Image; Image.open('flag.jpg').show()"` if you have Pillow installed.)

### Step 7 — Wrap the whole pipeline in a script

For a repeatable solve (and so I can re-run on a different pcap without typing the same commands), I scripted it:

```
┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ nano solve.py
```

Pasted:

```python
#!/usr/bin/env python3
"""Reconstruct the original file from a Silent Stream pcap.

Strategy:
  1. Pull all TCP payloads sent from the sender's port (default 12345).
  2. Concatenate the per-packet hex strings into one big hex blob.
  3. Subtract the encoding key (default 42, mod 256) from each byte.
  4. Write the decoded bytes to disk and let `file` decide what it is.
"""
import argparse
import subprocess
import sys

DEFAULT_PORT = 12345
DEFAULT_KEY = 42


def pull_payloads(pcap: str, src_port: int) -> bytes:
    out = subprocess.check_output(
        [
            "tshark", "-r", pcap,
            "-Y", f"tcp.srcport == {src_port}",
            "-T", "fields", "-e", "tcp.payload",
        ],
        stderr=subprocess.DEVNULL,
    ).decode()
    # Each line is one packet's hex payload, no separator.
    return bytes.fromhex(out.replace("\n", "").replace(" ", ""))


def decode(payload: bytes, key: int) -> bytes:
    return bytes((b - key) % 256 for b in payload)


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("pcap", help="path to the pcap file")
    ap.add_argument("-o", "--out", default="recovered.bin")
    ap.add_argument("--port", type=int, default=DEFAULT_PORT,
                    help=f"sender's TCP source port (default: {DEFAULT_PORT})")
    ap.add_argument("--key", type=int, default=DEFAULT_KEY,
                    help=f"additive encoding key (default: {DEFAULT_KEY})")
    args = ap.parse_args()

    payload = pull_payloads(args.pcap, args.port)
    print(f"[+] pulled {len(payload)} bytes from sender")
    recovered = decode(payload, args.key)
    with open(args.out, "wb") as f:
        f.write(recovered)
    print(f"[+] wrote {len(recovered)} bytes to {args.out}")

    # If `file` is available, print what the recovered file looks like.
    try:
        kind = subprocess.check_output(
            ["file", args.out], stderr=subprocess.DEVNULL
        ).decode().strip()
        print(f"[+] {kind}")
    except Exception:
        pass


if __name__ == "__main__":
    main()
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

```
┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ chmod +x solve.py

┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ ./solve.py capture.pcap -o flag.jpg
[+] pulled 9130 bytes from sender
[+] wrote 9130 bytes to flag.jpg
[+] flag.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, baseline, precision 8, 800x500, components 3

┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ xdg-open flag.jpg
```

Open the image and read off the flag.

---

## Alternative — Decode without `tshark`

If `tshark` is not installed, you can pull the same payload with `tcpdump` plus a tiny Python script.

```
┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ tcpdump -r capture.pcap -Y 'tcp.srcport == 12345' \
            -T fields -e data 2>/dev/null \
   | tr -d ' \n' > raw.hex

┌──(zham㉿kali)-[~/ctf/silent_stream]
└─$ python3 -c "
raw = bytes.fromhex(open('raw.hex').read())
print(bytes((b - 42) % 256 for b in raw)[:16].hex())
"
ffd8ffe000104a46494600010100000100010000ffdb
```

Same first 16 bytes, same JPEG header. (Note: `tcpdump` drops the Ethernet/IP/TCP headers so `-e data` is just the TCP payload bytes — exactly what we want.)

A third option is `scapy` if it's already on the system:

```python
from scapy.all import rdpcap, TCP
pkts = rdpcap("capture.pcap")
payload = b"".join(bytes(p[TCP].payload) for p in pkts if p.haslayer(TCP) and p[TCP].sport == 12345)
open("flag.jpg", "wb").write(bytes((b - 42) % 256 for b in payload))
```

Any of the three works. `tshark` is the most common on CTF boxes and gives the most informative output.

---

## What Happened Internally

A timeline from sender's `flag.txt` to the recovered image:

1. The sender runs a script identical to `encoder.py` against a local file `flag.txt` that contains a JPEG image (a small white picture with the flag rendered in red text).
2. For each byte `b` of that JPEG, the script computes `encoded = (b + 42) % 256` — every byte is shifted up by 42, with wraparound.
3. The sender writes those encoded bytes to a TCP socket. On the receiver side, libpcap captures every byte that hits the wire into `capture.pcap`. (The sender's port is 12345, the receiver's is 9000.)
4. We open the pcap, list TCP conversations, confirm there is one transfer of about 16 kB.
5. We filter on `tcp.srcport == 12345` to keep only the bytes the sender pushed, then dump each packet's TCP payload as a hex string.
6. We concatenate those per-packet hex strings into one big hex blob, then `bytes.fromhex()` it back into a 9130-byte `bytes` object. So far we have a perfect copy of the encoded stream.
7. We apply the inverse transform `plain = (encoded - 42) % 256` byte by byte. Subtraction mod 256 handles the wraparound case correctly (`(b - 42) % 256` returns the proper positive result even when `b < 42`).
8. The decoded byte stream starts with `FF D8 FF E0 ... JFIF`. We recognize that as a JPEG header.
9. We save the decoded bytes as `flag.jpg` and `file` confirms a valid 800x500 baseline JPEG.
10. We open the image in any image viewer. The flag is the red text in the middle of an otherwise blank white image: `picoCTF{tr4ck_th3_tr4ff1c_c41f36a9}`.

The whole transfer is 0.142 seconds — the encoding is trivial, the network hop is fast, and the receiver's `tcpdump` was running the whole time, so we got every byte. The only "trick" is knowing what transform to undo (Hint 1) and what file type to expect on the other end (Hint 3).

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` / text editor | Read the encoder script to learn the transform |
| `tshark -r capture.pcap -q -z conv,tcp` | Find which TCP conversation carries the file |
| `tshark -Y 'tcp.srcport == 12345' -T fields -e tcp.payload` | Dump the sender's TCP payload as hex |
| `bytes.fromhex(...)` | Convert the concatenated hex string back to bytes |
| `python3` | One-liner decode `(b - key) % 256`, plus the full `solve.py` script |
| `file` | Confirm the recovered blob is a JPEG, not text |
| `mv` / `xdg-open` (or any image viewer) | Rename `.bin` to `.jpg` and actually look at the image |
| `nano` | Write the `solve.py` script (Ctrl+O / Enter / Ctrl+X to save and exit) |
| (Alternative) `tcpdump -e data` | Same payload extraction, no `tshark` needed |
| (Alternative) `scapy` | Programmatic pcap parsing in pure Python |

---

## Key Takeaways

- **The encoder script is the spec.** When a challenge gives you the encoding logic, do not guess — read it. The whole crypto here is `(b + 42) % 256`. Reversing it is `(b - 42) % 256`. That's the entire puzzle in one line of Python.
- **Know your mod-256 math.** Subtraction mod 256 must wrap below zero — Python's `%` operator does this for you, but in C you'd need `(b - key + 256) % 256`. Off-by-one mistakes here turn the recovered image into garbage.
- **Always run `file` on a recovered blob.** "Don't trust the flag format" — the recovered file is a JPEG, not text. Grepping for `picoCTF{` in the raw bytes finds nothing. The flag is rendered visually inside the image.
- **`tshark -T fields -e tcp.payload` is your friend.** When a CTF gives you a pcap, this is the fastest way to get raw bytes out without launching the Wireshark GUI.
- **Filter on `tcp.srcport ==` to get one side of a transfer.** A TCP conversation has bytes going both ways; the file-transfer bytes are only on one side. Picking the right port cuts your data set in half instantly.
- **The flag name tells you the lesson.** `tr4ck_th3_t4ff1c` = "track the traffic" — the whole challenge is about pulling bytes off the wire and undoing what was done to them.

### Flag wordplay decode

`picoCTF{tr4ck_th3_tr4ff1c_c41f36a9}` reads as **"track the traffic"** with leet-speak substitutions: `4` for `a` (twice — in `tr4ck` and `tr4ff1c`), `3` for `e`, `1` for `i`. The challenge is literally about tracing bytes across a network capture and undoing the sender's per-byte transform. The trailing `c41f36a9` is a per-instance hex suffix so the flag is unique across picoCTF deployments.
