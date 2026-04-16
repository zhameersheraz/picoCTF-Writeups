# Rogue Tower — picoCTF Writeup

**Challenge:** Rogue Tower  
**Category:** Forensics  
**Difficulty:** Medium  
**Flag:** `picoCTF{r0gu3_c3ll_t0w3r_a7310be3}`  

---

## Description

> A suspicious cell tower has been detected in the network. Analyze the captured network traffic to identify the rogue tower, find the compromised device, and recover the exfiltrated flag.

**Hint shown in challenge:** `Look for unauthorized test network broadcasts on UDP port 55000`
**Attachment:** `rogue_tower.pcap`

---

## Background Knowledge (Read This First!)

### What is a PCAP?

A `.pcap` file is a **network traffic recording** — it stores every packet captured on the network. Tools like **Wireshark** or **tshark** are used to inspect them.

### What is a Rogue Cell Tower?

A **rogue (fake) cell tower** — also called an IMSI catcher or Stingray — impersonates a legitimate cell tower. Nearby devices connect to it thinking it is real, allowing the attacker to intercept traffic.

### How do cell towers announce themselves?

Legitimate carriers broadcast on **UDP port 55000** with messages like:

```
CARRIER: Verizon PLMN=310410 CELLID=16274
CARRIER: AT&T PLMN=310410 CELLID=16275
```

A rogue tower broadcasts with suspicious values:

```
UNAUTHORIZED-TEST-NETWORK PLMN=00101 CELLID=92771
```

### What is PLMN?

**PLMN (Public Land Mobile Network)** is a 6-digit code identifying a mobile carrier. `PLMN=00101` is a test/fake network — no real carrier uses this.

### What is XOR encryption?

The flag was encrypted using **XOR** — each byte of the plaintext is XORed with a repeating key. The key used here was the last 8 digits of the compromised device's IMSI number.

### ⚠️ Two Important Notes

**Note 1 — Install tshark if not present**
```
sudo apt-get install tshark
```

**Note 2 — No extra Python libraries needed**
Only Python's built-in `base64` and `struct` are used!

---

## Solution — Step by Step

### Step 1 — Find the rogue tower broadcast

Filter UDP port 55000 traffic in tshark:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ tshark -r rogue_tower.pcap -Y "udp.port == 55000" \
    -T fields -e ip.src -e ip.dst -e data.data
```

Output:

```
192.168.1.1    255.255.255.255   434152524945523a20566572697a6f6e...
192.168.1.1    255.255.255.255   434152524945523a204154265420504c...
192.168.99.1   255.255.255.255   554e415554484f52495a45442d544553...
```

Decode the hex payloads in Python:

```python
payloads = [
    '434152524945523a20566572697a6f6e20504c4d4e3d3331303431302043454c4c49443d3136323734',
    '434152524945523a204154265420504c4d4e3d3331303431302043454c4c49443d3136323735',
    '554e415554484f52495a45442d544553542d4e4554574f524b20504c4d4e3d30303130312043454c4c49443d393237373120',
]
for p in payloads:
    print(bytes.fromhex(p).decode())
```

Output:

```
CARRIER: Verizon PLMN=310410 CELLID=16274
CARRIER: AT&T PLMN=310410 CELLID=16275
UNAUTHORIZED-TEST-NETWORK PLMN=00101 CELLID=92771
```

**Rogue tower identified:** `192.168.99.1`

### Step 2 — Find the compromised device

Check which device connected to the rogue tower's IP (`198.51.100.205`) instead of the legitimate carrier (`198.51.100.7`):

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ tshark -r rogue_tower.pcap -Y "http" \
    -T fields -e ip.src -e ip.dst -e http.host -e http.request.uri
```

Output:

```
10.100.200.246   198.51.100.7    network.carrier.com   /api/register
10.100.245.9     198.51.100.7    network.carrier.com   /api/register
10.100.130.239   198.51.100.7    network.carrier.com   /api/register
10.100.210.36    198.51.100.7    network.carrier.com   /api/register
10.100.204.98    198.51.100.205  network.carrier.com   /api/register  ← rogue!
10.100.204.98    198.51.100.205  198.51.100.205        /upload
...
```

**Compromised device:** `10.100.204.98`

Check its DNS query to find its IMSI:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ tshark -r rogue_tower.pcap -Y "dns" -T fields -e ip.src -e dns.qry.name
```

```
10.100.204.98   device-310410728284734.network.com
```

**Device IMSI:** `310410728284734`

### Step 3 — Recover the exfiltrated data

The compromised device sent 6 chunked HTTP POST uploads to the rogue server:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ tshark -r rogue_tower.pcap -Y "http" \
    -T fields -e ip.src -e http.request.uri -e http.file_data \
    | grep "/upload"
```

Output:

```
10.100.204.98   /upload   516c46525633646a64
10.100.204.98   /upload   5539414346564e4232
10.100.204.98   /upload   685142313555625577
10.100.204.98   /upload   455141424762566b46
10.100.204.98   /upload   437755485556454252
10.100.204.98   /upload   513d3d
```

### Step 4 — Reassemble and decrypt the flag

Create the solve script:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano solve.py
```

Paste this:

```python
import base64

# Step 1: Reassemble the hex chunks into one Base64 string
chunks = [
    '516c46525633646a64',
    '5539414346564e4232',
    '685142313555625577',
    '455141424762566b46',
    '437755485556454252',
    '513d3d',
]
combined = ''.join(bytes.fromhex(c).decode() for c in chunks)
print("Combined Base64:", combined)

# Step 2: Decode Base64 to get the encrypted bytes
encrypted = base64.b64decode(combined)
print("Encrypted (hex):", encrypted.hex())

# Step 3: XOR decrypt using last 8 digits of IMSI as key
key = b'28284734'
decrypted = bytes(encrypted[i] ^ key[i % len(key)] for i in range(len(encrypted)))
print("Flag:", decrypted.decode())
```

Run it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 solve.py
Combined Base64: QlFRV3djdU9ACFVNB2hQB15UbUwEQABGbVkFCwUHUVEBRQ==
Encrypted (hex): 425151577763754f4008554d076850075e546d4c044000466d59050b050751510145
Flag: picoCTF{r0gu3_c3ll_t0w3r_a7310be3}
```

✅ Got the flag! 🎯

### How was the XOR key found?

A **known-plaintext attack** was used. Since the flag must start with `picoCTF{`, the first 8 bytes of the encrypted data were XORed against `picoCTF{` to reveal the key:

```python
known = b'picoCTF{'
key_start = bytes(encrypted[i] ^ known[i] for i in range(8))
# → b'28284734'  (last 8 digits of the device IMSI 310410728284734)
```

---

## Packet Summary

| Packet | Source IP | Destination | Event |
|--------|-----------|-------------|-------|
| 1–2 | 192.168.1.1 | 255.255.255.255 | Legitimate carrier broadcasts (Verizon, AT&T) |
| 14 | 192.168.99.1 | 255.255.255.255 | **Rogue tower broadcast** (UNAUTHORIZED-TEST-NETWORK) |
| 15 | 10.100.204.98 | 8.8.8.8 | Compromised device DNS lookup |
| 16 | 10.100.204.98 | 198.51.100.205 | Device registers with **rogue** server |
| 17–22 | 10.100.204.98 | 198.51.100.205 | **6 chunked flag uploads** to rogue server |

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| tshark | Filter and extract fields from PCAP | ⭐⭐ Medium |
| Python `base64` | Decode the Base64 layer | ⭐ Easy |
| Python XOR | Decrypt the flag using IMSI key | ⭐⭐ Medium |
| Known-plaintext attack | Recover the XOR key from `picoCTF{` prefix | ⭐⭐⭐ Hard |

---

## Key Takeaways

- **Rogue cell towers** broadcast on UDP port 55000 — always filter this port first in telecom forensics
- `PLMN=00101` is a test network code — any device connecting to it is immediately suspicious
- Devices that register to a **different IP** than other devices on the same network are compromised
- **IMSI numbers** are embedded in DNS hostnames — check DNS queries to identify devices
- **XOR encryption** with a key derived from network metadata (like IMSI digits) is a common CTF exfiltration pattern
- A **known-plaintext attack** on the `picoCTF{` prefix reveals the XOR key instantly
