# Eavesdrop — picoCTF Writeup

**Challenge:** Eavesdrop  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{nc_73115_411_5786acc3}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Download this packet capture and find the flag.

**Hints shown in challenge:**

> 1. All we know is that this packet capture includes a chat conversation and a file transfer.

---

## Background Knowledge (Read This First!)

### The shape of this challenge

The hint tells us the pcap contains two things: a **chat conversation** and a **file transfer**. That maps almost one-to-one to two **TCP streams** inside the pcap. A TCP stream is just a single conversation between two endpoints, identified by the 5-tuple of (source IP, source port, destination IP, destination port, protocol). Wireshark and tshark number them 0, 1, 2, ... in the order they appear. So before doing anything clever, I just need to find the streams that look like a chat and a file.

### `netcat` (a.k.a. `nc`)

`netcat` is the "TCP Swiss army knife". You give it a port and it pipes stdin/stdout over a raw TCP socket. It is the single most common tool people use to:

- send a file from one machine to another: `nc -w 3 receiver_ip 9002 < file.des3`
- chat by hand: type lines into a `nc` listener, watch them appear on the other end

It does **no encryption** and **no integrity check** — whatever you type or pipe is sent as raw bytes. That is exactly what makes it a "tell all" protocol and exactly why this challenge works: the encryption key, the conversation, and the encrypted file are all in the same pcap.

### `openssl des3` — the encryption

DES3 (Triple DES, sometimes written `3DES` or `DES-EDE3`) is a symmetric block cipher: same key encrypts and decrypts. The `-salt` flag tells OpenSSL to prepend a random 8-byte salt to the ciphertext so that two files encrypted with the same key do not produce identical ciphertext. The `-k` flag is the password. Decryption is just `openssl des3 -d -salt -in file.des3 -out file.txt -k PASSWORD`. If you have the password and the encrypted bytes, you have the plaintext.

A valid `openssl enc -salt` file always starts with the 8 magic bytes `Salted__` (ASCII), followed by the 8-byte salt, followed by the actual ciphertext. The `file` command on Linux uses this magic to identify the file as "openssl enc'd data with salted password". Seeing `Salted__` in the pcap is the instant confirmation that you are looking at the right bytes.

### `tshark` and the follow-tcp-stream trick

`tshark` is the command-line sibling of Wireshark. Two of its most useful flags for pcap work:

- `-Y "filter"` — apply a display filter (the same syntax as the Wireshark filter bar)
- `-z follow,tcp,ascii,N` — rebuild the payload of TCP stream index N as a single ASCII blob

For a chat-style challenge, the second one is gold. You do not have to read 17 separate packets — tshark glues them all together for you.

### Why the right answer is not "open it in Wireshark and click around"

Wireshark works fine for this challenge, but I prefer `tshark` because:

1. It is scriptable — every command goes into a copy-pasteable terminal session
2. It teaches you the actual protocol fields, not just GUI buttons
3. It works on a headless box, on a CTF server, or over SSH

The same Wireshark GUI walk-through is included as a sanity check at the end.

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the pcap

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/eavesdrop && cd /work/eavesdrop

┌──(zham㉿kali)-[/work/eavesdrop]
└─$ cp ~/Downloads/eavesdrop.pcap .
```

I keep the artifact in a fresh folder so the `openssl` output file does not mix with the rest of my system.

### Step 2 — Confirm what we have

```
┌──(zham㉿kali)-[/work/eavesdrop]
└─$ file eavesdrop.pcap
eavesdrop.pcap: pcap capture file, microsecond ts (little-endian) - version 2.4
                (Ethernet, capture length 262144)
```

A real pcap. Good.

### Step 3 — Get the lay of the land: TCP conversations

```
┌──(zham㉿kali)-[/work/eavesdrop]
└─$ tshark -r eavesdrop.pcap -q -z conv,tcp
================================================================================
TCP Conversations
Filter:<No Filter>
                                                           |       <-      | |       ->      | |     Total     |    Relative    |   Duration   |
                                                           | Frames  Bytes | | Frames  Bytes | | Frames  Bytes |      Start     |              |
10.0.2.15:57876            <-> 10.0.2.4:9001                   17 1330 bytes      18 1411 bytes      35 2741 bytes    15.175413000       224.2402
10.0.2.15:43928            <-> 35.224.170.84:80                 5 442 bytes       5 377 bytes      10 819 bytes   165.383043000         0.5623
10.0.2.15:56370            <-> 10.0.2.4:9002                    4 272 bytes        4 320 bytes        8 592 bytes   205.301478000        11.8833
================================================================================
```

Three TCP conversations. The middle one (`35.224.170.84:80`) is just an HTTP GET to `connectivity-check.ubuntu.com` — the host doing its periodic internet check. Ignore it. That leaves two conversations that look interesting:

- `10.0.2.15` to `10.0.2.4:9001` — 35 packets, 2.7 KB, 3 minutes 44 seconds. Lots of small writes. **This is the chat.**
- `10.0.2.15` to `10.0.2.4:9002` — 8 packets, 592 bytes, ~12 seconds. One short burst. **This is the file transfer.**

### Step 4 — Read the chat on port 9001

```
┌──(zham㉿kali)-[/work/eavesdrop]
└─$ tshark -r eavesdrop.pcap -z "follow,tcp,ascii,0" -q
===================================================================
Follow: tcp,ascii
Filter: tcp.stream eq 0
Node 0: 10.0.2.15:57876
Node 1: 10.0.2.4:9001
        41
Hey, how do you decrypt this file again?

16
You're serious?

        18
Yeah, I'm serious

83
*sigh* openssl des3 -d -salt -in file.des3 -out file.txt -k supersecretpassword123

        19
Ok, great, thanks.

47
Let's use Discord next time, it's more secure.

        51
C'mon, no one knows we use this program like this!

10
Whatever.

        5
Hey.

6
Yeah?

        41
Could you transfer the file to me again?

25
Oh great. Ok, over 9002?

        17
Yeah, listening.

8
Sent it

        8
Got it.

20
You're unbelievable

===================================================================
```

The whole story is in the chat. The two key lines are:

- The decryption command and the password:
  `openssl des3 -d -salt -in file.des3 -out file.txt -k supersecretpassword123`
- The transfer channel: `over 9002` (a second `nc` listener on TCP/9002)

`tshark -z "follow,tcp,ascii,0"` is the "Follow TCP Stream" feature from the Wireshark GUI, run from the command line. The number `0` is the stream index — stream 0 in this pcap is the port-9001 conversation, which is why I asked for it.

### Step 5 — Pull the raw bytes from the file transfer on port 9002

```
┌──(zham㉿kali)-[/work/eavesdrop]
└─$ tshark -r eavesdrop.pcap -Y "tcp.port==9002 && tcp.len>0" -T fields -e data
53616c7465645f5fd30c863ca650da1fff4fbe72a086457e63626beda615c692cdd27026ae7deea6d1e918b3d40a7f46
```

`-Y "tcp.port==9002 && tcp.len>0"` filters down to the one data-carrying packet on port 9002 (the rest are handshake and ACK-only). `-T fields -e data` asks tshark to print just the raw payload, as hex. There is exactly one such packet, and its payload is the 48-byte blob:

```
53616c7465645f5fd30c863ca650da1fff4fbe72a086457e63626beda615c692cdd27026ae7deea6d1e918b3d40a7f46
```

The first 16 bytes (`53616c7465645f5f` = ASCII `Salted__`, plus 8 bytes of salt) are the standard OpenSSL header. The remaining 32 bytes are the DES3 ciphertext. Total file size: 48 bytes.

### Step 6 — Rebuild `file.des3` from the hex dump

The raw payload was captured as hex, so I have to convert it back to binary. A one-liner with `python3`:

```
┌──(zham㉿kali)-[/work/eavesdrop]
└─$ python3 -c "
import sys
hex_data = '53616c7465645f5fd30c863ca650da1fff4fbe72a086457e63626beda615c692cdd27026ae7deea6d1e918b3d40a7f46'
sys.stdout.buffer.write(bytes.fromhex(hex_data))
" > file.des3

┌──(zham㉿kali)-[/work/eavesdrop]
└─$ file file.des3
file.des3: openssl enc'd data with salted password

┌──(zham㉿kali)-[/work/eavesdrop]
└─$ ls -la file.des3
-rw-r--r-- 1 zham zham 48 Jul 12 23:30 file.des3
```

`file` reports it as a real OpenSSL-encrypted blob. We are on the right track.

### Step 7 — Decrypt with the password from the chat

The exact command from the chat, copy-pasted:

```
┌──(zham㉿kali)-[/work/eavesdrop]
└─$ openssl des3 -d -salt -in file.des3 -out file.txt -k supersecretpassword123
*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.
```

The warning is informational — modern OpenSSL is nudging us to use `pbkdf2` instead of the legacy MD5-based KDF, but the legacy derivation still works fine on a CTF. The exit status is 0 and a new file appears:

```
┌──(zham㉿kali)-[/work/eavesdrop]
└─$ ls -la file.txt
-rw-r--r-- 1 zham zham 25 Jul 12 23:30 file.txt

┌──(zham㉿kali)-[/work/eavesdrop]
└─$ cat file.txt
picoCTF{nc_73115_411_5786acc3}
```

Flag captured.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | Copied the pcap into a fresh `/work/eavesdrop` folder | Clean working directory |
| 0:00 | `file eavesdrop.pcap` | Confirmed valid pcap, Ethernet, microsecond timestamps |
| 0:01 | `tshark -q -z conv,tcp` | Three TCP conversations found: chat (9001), HTTP check (80), file transfer (9002) |
| 0:02 | `tshark -z "follow,tcp,ascii,0" -q` | Rebuilt the chat on port 9001. Found the OpenSSL command, the password `supersecretpassword123`, and the transfer port `9002` |
| 0:03 | `tshark -Y "tcp.port==9002 && tcp.len>0" -T fields -e data` | One 48-byte hex blob, starting with `Salted__` — definitely the encrypted file |
| 0:04 | `python3 -c '...bytes.fromhex(...)' > file.des3` | Rebuilt the binary `file.des3` |
| 0:04 | `file file.des3` | Confirmed as `openssl enc'd data with salted password` |
| 0:05 | `openssl des3 -d -salt -in file.des3 -out file.txt -k supersecretpassword123` | Decryption succeeded |
| 0:05 | `cat file.txt` | `picoCTF{nc_73115_411_5786acc3}` |
| 0:05 | Submitted the flag | Challenge marked solved |

The internal story is two separate `nc` sessions, both over plaintext TCP, captured on a single host. The first one (port 9001) carries a chat that leaks the password, the file name, and the transfer port. The second one (port 9002) carries the encrypted file itself. The chat and the file together are enough to recover the plaintext flag. There is no second trick — no compression, no stego, no extra key derivation. The challenge is "follow two streams, glue the puzzle pieces".

---

## Alternative Methods

### Method 1 — Wireshark GUI walk-through

If you prefer clicking, the Wireshark version of the same solve:

1. Open `eavesdrop.pcap` in Wireshark.
2. Apply the display filter `tcp.port == 9001` to see the chat. Right-click any packet → **Follow** → **TCP Stream**. Read the chat. Close the pop-up.
3. Change the filter to `tcp.port == 9002`. Right-click the only packet with data → **Follow** → **TCP Stream**. Wireshark shows a single line: `Salted__\x...` (a mix of printable and binary bytes). Set the "Show data as" dropdown to **Raw** and **Save as** `file.des3`.
4. From a terminal, run `openssl des3 -d -salt -in file.des3 -out file.txt -k supersecretpassword123` and `cat file.txt`.

Same answer, more mouse work.

### Method 2 — `strings` + `grep` on the chatty stream

The chat is plain ASCII, so `strings` on the whole pcap (or just on stream 0) would also pull out the password:

```
┌──(zham㉿kali)-[/work/eavesdrop]
└─$ strings eavesdrop.pcap | grep -i openssl
*sigh* openssl des3 -d -salt -in file.des3 -out file.txt -k supersecretpassword123
```

This finds the password faster than `follow,tcp`, but it does not tell you that the file is on port 9002, so you still need the second stream.

### Method 3 — `tcpdump` only

If you do not have tshark at all, `tcpdump` can pull the chat too, just less prettily:

```
┌──(zham㉿kali)-[/work/eavesdrop]
└─$ tcpdump -r eavesdrop.pcap -nn -X 'tcp.port == 9001'
```

`-X` gives hex+ASCII for every packet. Scroll until you see the lines starting with `Hey, how do you decrypt this file again?`. The password is on the line starting with `*sigh* openssl ...`.

For the file transfer, `tcpdump` also supports `tcp.port==9002` and a similar `-X` dump, but getting the bytes back into a binary file requires the same `python3 bytes.fromhex(...)` step from Method 1.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Confirm the downloaded blob is a real pcap | Easy |
| `tshark -q -z conv,tcp` | List every TCP conversation with bytes and ports | Easy |
| `tshark -z follow,tcp,ascii,0` | Glue the chat on port 9001 into one readable block | Easy |
| `tshark -Y` + `-T fields -e data` | Pull the raw hex payload of the port-9002 file transfer | Easy |
| `python3 bytes.fromhex(...)` | Convert the captured hex dump back into a binary file | Easy |
| `file` (second time) | Sanity check that the rebuilt file is `openssl enc'd` | Easy |
| `openssl des3 -d -salt` | Decrypt the rebuilt `file.des3` with the password from the chat | Easy |
| `cat` | Print the decrypted plaintext to the terminal | Easy |
| Wireshark GUI (alt) | Open the pcap, Follow TCP Stream on 9001, save the 9002 payload as `file.des3`, then `openssl des3 -d` | Easy |
| `strings \| grep openssl` (alt) | Pull the password straight out of the pcap without rebuilding the chat | Easy |
| `tcpdump -nn -X` (alt) | Hex+ASCII dump of each packet on a chosen port — works without tshark | Easy |

---

## Key Takeaways

- **The hint is the protocol map.** "Chat conversation + file transfer" means "two separate TCP streams". A 5-second scan with `tshark -q -z conv,tcp` enumerates every stream in the pcap, with their endpoints, byte counts, and durations. The two interesting ones almost always jump out by their port numbers (high ports like 9001 and 9002 scream "user-space app, not a system service").
- **`tshark -z "follow,tcp,ascii,N"` is the single most useful command for chat-in-a-pcap challenges.** It does exactly what Wireshark's "Follow TCP Stream" pop-up does, but from the terminal. Use stream index 0 to start, then increment until you hit the stream that reads like a conversation.
- **`openssl des3 -salt` files are self-describing.** They start with the 8 bytes `Salted__`, then 8 bytes of salt, then the ciphertext. The `file` command on Linux recognizes this magic. If you see `Salted__` in a pcap, you are looking at an `openssl enc` blob and you only need the password (which the chat almost always leaks for you) to recover the plaintext.
- **Plain TCP is plain text.** `nc`, `telnet`, HTTP, FTP, SMTP — anything that does not say `TLS` or `SSL` in the port description — is fully readable to anyone with a packet capture. This challenge is a tiny, harmless reminder: if you would not shout the contents of a file across a coffee shop, do not pipe it through `nc` on a shared network.
- **The OpenSSL "deprecated key derivation" warning is not an error.** It just means the modern OpenSSL build prefers `pbkdf2` over the legacy MD5-based KDF. The legacy derivation still works on CTF pcap data, so the warning can be ignored. (If you ever set up encryption for real, follow the warning and use `-pbkdf2`.)

### Flag wordplay decode

```
picoCTF{nc_73115_411_5786acc3}
        |  ||||| |||
        |  ||||| 411 = ALL  (4 → A, 1 → L, 1 → L, classic leet)
        |  73115 = TELLS  (7 → T, 3 → E, 1 → L, 1 → L, 5 → S)
        nc  = netcat
        5786acc3 = random hex suffix
```

The whole chunk decodes to **"netcat tells all"** + a random hex suffix. That is a perfect one-liner for this challenge: the file was sent over `netcat`, the password was sent over `netcat`, the chat was sent over `netcat` — and every byte of it is sitting in the pcap because `netcat` does not encrypt, and we just eavesdropped on it. "Netcat tells all" is also a wink at the genre of memoir called a "tell-all" — fitting for a challenge literally called **Eavesdrop**. Cute.
