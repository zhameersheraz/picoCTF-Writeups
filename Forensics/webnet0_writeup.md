# WebNet0 — picoCTF Writeup

**Challenge:** WebNet0  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 350  
**Flag:** `picoCTF{nongshim.shrimp.crackers}`  
**Platform:** picoCTF 2019  
**Author:** Jason  
**Writeup by:** zham

---

## Description

> We found this packet capture and key. Recover the flag.

**Attachments:** `capture.pcap` (encrypted TLS traffic) + `picopico.key` (RSA private key)

---

## Hints

> 1. Try using a tool like Wireshark.
> 2. How can you decrypt the TLS stream?

---

## Background Knowledge (Read This First!)

### The easy version of WebNet1

This is the simpler of the two WebNet challenges. Same mechanics: encrypted TLS pcap + a private RSA key, and the goal is to recover the flag from inside the session. The key difference: **the flag is in the HTTP response header**, not hidden in a binary file transferred over the connection.

### Why this works

The pcap negotiates `TLS_RSA_WITH_AES_256_GCM_SHA384` (or similar static-RSA key exchange), which means the pre-master secret is RSA-encrypted in the `ClientKeyExchange` message. With the server's private RSA key in `picopico.key`, anyone who has the pcap can retroactively recover the session keys and decrypt every Application Data record — including the HTTP request and response.

The standard tool for this is Wireshark: `Edit → Preferences → Protocols → TLS → RSA keys list → Add` and point it at `picopico.key`. Once the key is loaded, the green "Application Data" packets turn readable as HTTP, and the flag is in the response.

---

## Solution — Step by Step

### Step 1 — Confirm the setup

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file capture.pcap picopico.key
capture.pcap:  pcap capture file, microsecond ts (little-endian) - version 2.4
picopico.key:  PEM RSA private key
```

A single TLS session in the pcap, an RSA private key. Same setup as WebNet1.

### Step 2 — Load the key in Wireshark

```
Wireshark → Edit → Preferences → Protocols → TLS → RSA keys list → Edit
  IP address:    172.31.22.220   (or any)
  Port:          443
  Protocol:      http
  Key File:      /path/to/picopico.key
  → OK
```

Re-open `capture.pcap`. The "Application Data" packets in the TLS stream turn green and are now decrypted HTTP.

### Step 3 — Follow the HTTP stream

`Right-click any decrypted packet → Follow → TLS Stream`.

The decrypted response looks like this:

```
HTTP/1.1 200 OK
Date: Fri, 23 Aug 2019 15:56:36 GMT
Server: Apache/2.4.29 (Ubuntu)
Last-Modified: Mon, 12 Aug 2019 16:50:05 GMT
ETag: "5ff-58fee50dc3fb0-gzip"
Accept-Ranges: bytes
Vary: Accept-Encoding
Content-Encoding: gzip
Pico-Flag: picoCTF{nongshim.shrimp.crackers}
Content-Length: 821
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html
```

The flag is right there in the `Pico-Flag:` response header.

```
picoCTF{nongshim.shrimp.crackers}
```

### CLI alternative (no GUI)

If you don't have Wireshark, `ssldump` is the standard CLI tool:

```
ssldump -r capture.pcap -k picopico.key -d
```

Scroll down past the handshake messages — the flag will appear in the dumped HTTP response.

---

## What Happened Internally (Timeline)

1. The challenge author set up an Apache server with TLS enabled, using a self-signed certificate. The matching RSA private key was saved as `picopico.key` and given to the player.
2. A client (the picoCTF script) made a normal HTTPS request. Apache's response carried a custom `Pico-Flag:` HTTP header containing the flag.
3. The TLS session used static RSA key exchange, so the entire session is retroactively decryptable by anyone who holds the server's private key.
4. You, the solver, take the private key, feed it to Wireshark/tshark/ssldump, follow the HTTP stream, and read the `Pico-Flag:` header.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `Wireshark` | GUI decryption via TLS preferences + key file | Easy |
| `ssldump` | CLI alternative — `ssldump -r capture.pcap -k picopico.key -d` | Easy |
| `tshark` | `tshark -r capture.pcap -o 'ssl.keys_list:...,picopico.key' -Y http` | Medium |

---

## Key Takeaways

- **Decrypt the TLS, follow the stream, read the headers.** This is the canonical "TLS key exchange" forensics pattern. It appears in picoCTF, HTB, and many other CTF series. Once you have the private key, the rest is just reading the decrypted stream.
- **Custom HTTP headers are common flag carriers.** The picoCTF platform uses `Pico-Flag:` as a convention for the easy version of the challenge. Look for any non-standard header (`X-Flag`, `Flag`, `Pico-Flag`, `Authorization: picoCTF{...}`) — that's almost always where the easy flag is.
- **WebNet1 is the harder sequel.** The same setup, but the flag moves from the HTTP header into a JPEG file transferred over the same connection. The decoy `Pico-Flag: picoCTF{this.is.not.your.flag.anymore}` is left in place to throw you off. Lesson: even after finding one flag, keep digging — check the response body and any binary payloads.
- **The wordplay is "nongshim shrimp crackers".** Nongshim is a major Korean food brand; their shrimp crackers are a globally recognized snack. The challenge is essentially a joke about how much effort (decrypting TLS, finding the right header) you can put into recovering a reference to a cheap snack. The follow-up WebNet1 doubles down on this joke by hiding the same kind of "meme flag" inside a vulture JPEG.
