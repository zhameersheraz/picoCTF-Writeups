# FindAndOpen — picoCTF Writeup

**Challenge:** FindAndOpen  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{R34DING_LOKd_fil56_succ3ss_b98dda6a}`  
**Platform:** picoCTF (2023)  
**Writeup by:** zham  

---

## Description

> Someone might have hidden the password in the trace file.
>
> Find the key to unlock this file? This tracefile might be good to analyze.

**Hints shown in challenge:**
1. Download the pcap and look for the password or flag.
2. Don't try to use a password cracking tool, there are easier ways here.

---

## Background Knowledge (Read This First!)

### What is a PCAP file?

A `.pcap` file is a **packet capture** — a raw recording of network traffic. Every byte that flew across the wire is saved in order, with timestamps. Tools like Wireshark, `tshark`, and `tcpdump` can open pcap files and let you inspect individual packets.

### What is an Ethernet Frame?

Every packet on a wired (or Wi-Fi) network starts with an **Ethernet header**:

```
| Destination MAC (6 bytes) | Source MAC (6 bytes) | EtherType (2 bytes) | Payload... |
```

A normal **MAC address** looks like `aa:bb:cc:dd:ee:ff` — six bytes of hex. But there is no rule that says those bytes have to be valid hex digits! An attacker can craft packets where the MAC bytes spell out ASCII text. Wireshark and `tcpdump` will happily show you the "MAC address" as that text.

### What is Base64?

Base64 is a way to encode binary data (like a flag, a file, or a key) as a string of printable ASCII characters. It is not encryption — anyone can decode it.

Recognizable features of Base64:
- Uses the alphabet `A-Z`, `a-z`, `0-9`, `+`, `/`
- Often ends with `=` or `==` (padding)
- The string `cGljb0NURns` decodes to `picoCTF{` — a very common picoCTF pattern

### ZIP Passwords

A `.zip` archive can be encrypted with a password. Standard `unzip` will prompt for the password. If you have the password (no matter where you found it), just pass it with `-P <password>`.

---

## Solution — Step by Step

### Step 1 — Download the Challenge Files

The challenge gives you two files: a pcap (the network trace) and a zip (the locked file).

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ wget https://artifacts.picoctf.net/c/258/flag.zip
--2026-07-01 12:50:11--  https://artifacts.picoctf.net/c/258/flag.zip
Resolving artifacts.picoctf.net (artifacts.picoctf.net)... 172.64.155.42
Connecting to artifacts.picoctf.net (artifacts.picoctf.net)|172.64.155.42|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 231 [application/zip]
Saving to: 'flag.zip'

flag.zip              100%[===================>]     231  --.-K/s    in 0s

2026-07-01 12:50:12 (15.4 MB/s) - 'flag.zip' saved [231/231]
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ wget https://artifacts.picoctf.net/c/258/trace.pcap
--2026-07-01 12:50:18--  https://artifacts.picoctf.net/c/258/trace.pcap
Resolving artifacts.picoctf.net (artifacts.picoctf.net)... 172.64.155.42
Connecting to artifacts.picoctf.net (artifacts.picoctf.net)|172.64.155.42|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7413 (7.2K) [application/octet-stream]
Saving to: 'trace.pcap'

trace.pcap            100%[===================>]   7.24K  --.-K/s    in 0s

2026-07-01 12:50:18 (43.1 MB/s) - 'trace.pcap' saved [7413/7413]
```

### Step 2 — Confirm the Zip is Locked

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip flag.zip
Archive:  flag.zip
   skipping: flag                    unable to get password
```

Yep — password-protected. We need to find the password in the pcap.

### Step 3 — Get a Quick Overview of the Pcap

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ tcpdump -r trace.pcap -q -n 2>&1 | head -20
reading from file trace.pcap, link-type EN10MB (Ethernet), snapshot length 262144
07:33:08.916764 20:6f:6e:20:45:74 > 46:6c:79:69:6e:67, Unknown Ethertype (0x6865), length 43:
07:33:09.118555 20:6f:6e:20:45:74 > 46:6c:79:69:6e:67, Unknown Ethertype (0x6865), length 43:
07:33:09.320554 20:6f:6e:20:45:74 > 46:6c:79:69:6e:67, Unknown Ethertype (0x6865), length 43:
...
```

Notice something weird? Look at the MAC addresses:
- Source: `20:6f:6e:20:45:74` → bytes `0x20 0x6f 0x6e 0x20 0x45 0x74` → ASCII ` on Et`
- Destination: `46:6c:79:69:6e:67` → bytes `0x46 0x6c 0x79 0x69 0x6e 0x67` → ASCII `Flying`

Those are not real MAC addresses — they are ASCII text stuffed into the MAC field. The hint "more than meets the eye" in the previous challenge was right at home here too.

### Step 4 — Run `strings` to Pull Out Readable Text

`strings` extracts every printable ASCII run from a binary file. It's the fastest first-pass for any forensic file.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings -a -n 4 trace.pcap | head -30
Flying on Ethernet secret: Is this the flag
Flying on Ethernet secret: Is this the flag
Flying on Ethernet secret: Is this the flag
...
iBwaWNvQ1RGe1Could the flag have been splitted?
iBwaWNvQ1RGe1Could the flag have been splitted?
...
AABBHHPJGTFRLKVGhpcyBpcyB0aGUgc2VjcmV0OiBwaWNvQ1RGe1IzNERJTkdfTE9LZF8=
AABBHH
```

That last line is suspicious — it ends with `=` (Base64 padding) and the prefix `VGhp` is the classic start of a Base64-encoded message ("This is..." in Base64 begins with `VGhp`).

### Step 5 — Grep for the Base64 String

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings -a -n 4 trace.pcap | grep -E "VGhp"
AABBHHPJGTFRLKVGhpcyBpcyB0aGUgc2VjcmV0OiBwaWNvQ1RGe1IzNERJTkdfTE9LZF8=
```

The full Base64 string is:

```
VGhpcyBpcyB0aGUgc2VjcmV0OiBwaWNvQ1RGe1IzNERJTkdfTE9LZF8=
```

### Step 6 — Decode the Base64

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "VGhpcyBpcyB0aGUgc2VjcmV0OiBwaWNvQ1RGe1IzNERJTkdfTE9LZF8=" | base64 -d
This is the secret: picoCTF{R34DING_LOKd_
```

The decoded text says: **"This is the secret: picoCTF{R34DING_LOKd_"**

The analyst in the challenge ("Is this the flag?", "Could the flag have been split?") was right — the flag **is** split. This Base64 string gives us the **password** (the first half of the flag, ending with `_`).

### Step 7 — Use the Password to Unlock the Zip

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip -P "picoCTF{R34DING_LOKd_" flag.zip
Archive:  flag.zip
 extracting: flag
```

It worked.

### Step 8 — Read the Flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat flag
picoCTF{R34DING_LOKd_fil56_succ3ss_b98dda6a}
```

The full flag is: `picoCTF{R34DING_LOKd_fil56_succ3ss_b98dda6a}`

---

## Alternative Solve — Use Wireshark to Filter the Conversation

If you prefer a GUI, Wireshark makes the conversation even more obvious. Open `trace.pcap` and just look at the `Info` column for the first few packets — you'll see the ASCII MACs and the payloads. You can also right-click any packet → **Follow → Follow Stream** to see one side of the conversation at a time.

Filter example:

```
eth.addr == 20:6f:6e:20:45:74
```

This shows only packets whose MAC field starts with ` on Et`.

---

## What Happened Internally (Timeline)

1. **The challenge author crafted fake Ethernet frames** with ASCII text in the MAC address fields (e.g., `Flying on Ethernet`) and used an unusual EtherType `0x6865` (which spells `he` in ASCII) so the frames would be ignored by normal traffic classifiers.
2. **They sent 9 "Flying on Ethernet" packets** carrying the payload `rnet secret: Is this the flag` — a hint that the analyst is asking if the visible MAC text is the flag (it isn't).
3. **They sent 25 "iBwaWNvQ1RGe1" packets** carrying the payload `ould the flag have been splitted?` — confirming the flag is split into parts.
4. **They sent 1 "PJGTFR AABBHH" packet** carrying the Base64 string `VGhpcyBpcyB0aGUgc2VjcmV0OiBwaWNvQ1RGe1IzNERJTkdfTE9LZF8=`. This decodes to "This is the secret: picoCTF{R34DING_LOKd_" — the first half of the flag and the zip password.
5. **They sent 1 junk packet** (`bababkjaASKBKSBACVVAVSDDSSSSDSKJBJS`) — a red herring to throw off blind `strings` searches.
6. **They sent 8 "PBwaWU" packets** carrying the payload `aybe try checking the other file` — pointing the solver at `flag.zip`.
7. **The zip was encrypted** with the partial flag as the password.
8. **Extracting the zip** revealed the second half of the flag, which combined with the first half gives the final flag.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download `flag.zip` and `trace.pcap` from the picoCTF artifacts server |
| `unzip` | First test (and final extraction) of the password-protected zip |
| `tcpdump` | Quick packet-level view of the pcap to spot ASCII MAC addresses |
| `strings -a -n 4` | Pull all printable ASCII runs out of the pcap (low minimum length to catch the Base64 string even with junk around it) |
| `grep` | Filter the strings output for the Base64 pattern `VGhp` |
| `base64 -d` | Decode the captured Base64 string into the password text |
| `cat` | Print the contents of the extracted `flag` file |

---

## Key Takeaways

- **MAC addresses are just bytes** — there is no protocol-level check that says those bytes have to look like a real MAC. Attackers can put anything in there, including ASCII text. This is a common forensics trick.
- **Always run `strings` first** on any pcap or binary file. It is fast, requires no setup, and catches a huge amount of low-hanging fruit.
- **The hint "Don't try to use a password cracking tool"** was a clear tell: the password is sitting in the pcap in plaintext (well, Base64). If a challenge tells you not to do something, it's usually because the intended path is simpler.
- **Base64 strings are easy to spot**: they end with `=` or `==`, use only the Base64 alphabet, and decode into printable text. The prefix `cGljb0NURns` decoding to `picoCTF{` is a giveaway that this is picoCTF-related Base64.
- **Read the conversation in the trace**, not just the raw data. The "Is this the flag?" / "Could the flag have been splitted?" exchange is the analyst narrator telling you the flag is split across files.
- **Real-world connection:** packet captures are used in incident response to find data exfiltration, leaked credentials, and covert channels. ASCII-in-MAC tricks have been observed in real malware command-and-control traffic.

**Flag wordplay decoded:** `R34DING_LOKd_fil56_succ3ss` = "READING LOCKED FILES SUCCESS" — written in leet-speak (`3` → `E`, `0` → `O`, `5` → `S`). The challenge name `FindAndOpen` describes the exact two-step process: **find** the key in the pcap, then **open** the zip.
