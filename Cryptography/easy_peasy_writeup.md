# Easy Peasy — picoCTF Writeup

**Challenge:** Easy Peasy  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 40  
**Flag:** `picoCTF{71bf23b268a0cddbb9fc6f559b9be65f}`  
**Platform:** picoCTF (2021)  
**Writeup by:** zham  

---

## Description

> A one-time pad is unbreakable, but can you manage to recover the flag? (Wrap with picoCTF{})
>
> Connect with `nc wily-courier.picoctf.net 54406`. `otp.py`

The server implements a one-time pad (OTP) over the network. The OTP itself is unbreakable — but only if you never reuse the same key bytes for two different plaintexts. This server reuses key bytes via wraparound, and we can force that reuse.

> **Note on the port:** picoCTF regenerates this challenge per instance. The host stays the same (`wily-courier.picoctf.net`) but the port changes. Always re-read the port from the challenge page on the run you are solving.

## Hints

> 1. Maybe there's a way to make this a 2x pad.

A "two-time pad" (or 2x pad) is what you get when the same key bytes are used to encrypt two different plaintexts. If `c1 = p1 XOR k` and `c2 = p2 XOR k`, then `c1 XOR c2 = p1 XOR p2`, which leaks the relationship between the two plaintexts. The hint is telling us to deliberately create this situation.

---

## Background Knowledge (Read This First!)

### 1. What Is a One-Time Pad?

A one-time pad encrypts a message by XORing it with a random key that is **as long as the message** and **never reused**:

```
ciphertext[i] = plaintext[i] XOR key[i]
```

If the key is truly random, as long as the message, and used exactly once, the result is information-theoretically secure — there is no way to break it.

### 2. What Goes Wrong When You Reuse the Key

If you use the same key bytes to encrypt two different plaintexts, the magic disappears. With two ciphertexts `c1` and `c2` produced from the same key segment:

```
c1 XOR c2 = (p1 XOR k) XOR (p2 XOR k) = p1 XOR p2
```

You just learned the XOR of the two plaintexts. If one of them is something you control (say, all `'A'`s), you can recover the other one directly:

```
p2 = (p1 XOR p2) XOR p1
```

In this challenge, the server reuses key bytes via wraparound (the key pointer hits 50000 and resets to 0). The hint is telling us to force that reuse by sending exactly enough data to wrap the pointer back to 0.

### 3. How the Server's Key Pointer Works

The server code keeps a `key_location` cursor that advances after every encryption. When the cursor would go past `KEY_LEN = 50000`, the code wraps it back with `stop = stop % KEY_LEN`. The flow is:

1. **Startup:** Cursor starts at 0. The flag of length `L` is encrypted with `key[0:L]`. Cursor advances to `L`.
2. **Each user request:** Cursor advances by `len(user_input)`. When the cursor wraps, the next encryption starts again at `key[0]`.

So if we can advance the cursor by exactly `50000 - L` bytes, the next encryption starts at `key[0]` — the same bytes that were used on the flag. If we encrypt a known plaintext of length `L` at that point, we get `known XOR key[0:L]`, which lets us XOR-recover the flag.

---

## Solution — Step by Step

### Step 1 — Read the Source and Connect

I downloaded `otp.py` from the challenge and read through it. The interesting pieces:

```python
KEY_FILE = "key"
KEY_LEN = 50000

def startup(key_location):
    flag = open(FLAG_FILE).read()
    kf = open(KEY_FILE, "rb").read()
    start = key_location
    stop = key_location + len(flag)
    key = kf[start:stop]
    key_location = stop
    result = list(map(lambda p, k: "{:02x}".format(ord(p) ^ k), flag, key))
    print("This is the encrypted flag!\n{}\n".format("".join(result)))
    return key_location

def encrypt(key_location):
    ui = input("What data would you like to encrypt? ").rstrip()
    if len(ui) == 0 or len(ui) > KEY_LEN:
        return -1
    start = key_location
    stop = key_location + len(ui)
    kf = open(KEY_FILE, "rb").read()
    if stop >= KEY_LEN:
        stop = stop % KEY_LEN
        key = kf[start:] + kf[:stop]
    else:
        key = kf[start:stop]
    key_location = stop
    result = list(map(lambda p, k: "{:02x}".format(ord(p) ^ k), ui, key))
    print("Here ya go!\n{}\n".format("".join(result)))
    return key_location
```

Confirmed: 50,000-byte key file, single-byte XOR encryption, key pointer wraps at 50000.

I then connected to the oracle to read the encrypted flag:

```
┌──(zham㉿kali)-[~/picoctf/easy-peasy]
└─$ nc wily-courier.picoctf.net 54406
******************Welcome to our OTP implementation!******************
This is the encrypted flag!
92490840ecc18a9cf16795d9516c134f6a8162fa7523280be92dfc356ee9e58d

What data would you like to encrypt?
```

The encrypted flag is 64 hex characters, so `L = 32 bytes`.

### Step 2 — Write the Exploit

I wrote a Python script that talks to the oracle, wraps the key pointer with a padding message, and recovers the flag.

```
┌──(zham㉿kali)-[~/picoctf/easy-peasy]
└─$ nano solve.py
```

```python
#!/usr/bin/env python3
"""
2x-pad attack on picoCTF "Easy Peasy".

The OTP server uses a 50,000-byte key file. The flag is encrypted first with
key[0:L]. We then encrypt our own data; each encryption advances a key
pointer. When the pointer hits 50,000, it wraps to 0.

Plan:
  1. Read the encrypted flag from the banner (length 2*L hex chars).
  2. Send 50,000 - L bytes of 'A' as padding. The pointer was at L after
     startup, so it lands exactly on 50,000 and wraps to 0.
  3. Send L bytes of 'A'. The server encrypts these with key[0:L] (the same
     key bytes that protected the flag), giving us:
         enc_A[i] = 'A' XOR key[i]
  4. XOR the flag ciphertext with enc_A to recover (flag XOR 'A'), then XOR
     with 'A' to get the flag plaintext.
"""
import re
import socket


HOST = "wily-courier.picoctf.net"
PORT = 54406
KEY_LEN = 50000


def recv_until(sock, marker, timeout=10):
    sock.settimeout(timeout)
    buf = b""
    while marker not in buf:
        chunk = sock.recv(4096)
        if not chunk:
            break
        buf += chunk
    return buf.decode(errors="replace")


def main():
    s = socket.socket()
    s.connect((HOST, PORT))

    banner = recv_until(s, b"What data would you like to encrypt?")
    print(banner)

    m = re.search(r"encrypted flag!\n([0-9a-f]+)\n", banner)
    enc_flag_hex = m.group(1)
    L = len(enc_flag_hex) // 2
    print(f"[+] Encrypted flag ({L} bytes): {enc_flag_hex}")

    # Step 1: burn 50,000 - L bytes to wrap the key pointer back to 0.
    pad_len = KEY_LEN - L
    padding = "A" * pad_len
    print(f"[+] Sending {pad_len} bytes of padding to wrap the pointer to 0")
    s.sendall(padding.encode() + b"\n")
    resp = recv_until(s, b"What data would you like to encrypt?")
    print(f"[+] Padding response (first 120 chars): {resp.strip()[:120]!r}")

    # Step 2: send L bytes of 'A'. The server will encrypt these with
    # key[0:L], the same bytes used on the flag.
    probe = "A" * L
    print(f"[+] Sending {L} bytes of 'A' as the probe")
    s.sendall(probe.encode() + b"\n")
    resp = recv_until(s, b"What data would you like to encrypt?")

    m = re.search(r"Here ya go!\n([0-9a-f]+)\n", resp)
    enc_a_hex = m.group(1)
    print(f"[+] Probe ciphertext: {enc_a_hex}")

    # Step 3: recover the flag.
    flag_bytes = bytearray(L)
    for i in range(L):
        e = int(enc_flag_hex[2 * i : 2 * i + 2], 16)
        a = int(enc_a_hex[2 * i : 2 * i + 2], 16)
        flag_bytes[i] = e ^ a ^ ord("A")
    flag = bytes(flag_bytes).decode(errors="replace")
    print(f"\n[+] Recovered plaintext bytes: {bytes(flag_bytes)!r}")
    print(f"[+] Inner content: {flag}")
    print(f"[+] Flag: picoCTF{{{flag}}}")

    # Be polite and quit.
    s.sendall(b"\n")
    s.close()
    return flag


if __name__ == "__main__":
    main()
```

Save and exit:

```
^O   (WriteOut)
File Name to Write: solve.py -> press Enter
^X   (Exit)
```

### Step 3 — Run the Exploit

```
┌──(zham㉿kali)-[~/picoctf/easy-peasy]
└─$ python3 solve.py
******************Welcome to our OTP implementation!******************
This is the encrypted flag!
92490840ecc18a9cf16795d9516c134f6a8162fa7523280be92dfc356ee9e58d

What data would you like to encrypt? 

[+] Encrypted flag (32 bytes): 92490840ecc18a9cf16795d9516c134f6a8162fa7523280be92dfc356ee9e58d
[+] Sending 49968 bytes of padding to wrap the pointer to 0
[+] Padding response (first 120 chars): 'Here ya go!\n5ebc17a985eafc9e7cb0a2a94d4ec9074ebac5f3aaccc7d5019fa8859b4d8516fea5cee592057960e22a6a872359d1cb346115046a3d'
[+] Sending 32 bytes of 'A' as the probe
[+] Probe ciphertext: e4392b679fb3a9ef861eb5a87349366c49f945d802045c7f910e84164a9e91aa

[+] Recovered plaintext bytes: b'71bf23b268a0cddbb9fc6f559b9be65f'
[+] Inner content: 71bf23b268a0cddbb9fc6f559b9be65f
[+] Flag: picoCTF{71bf23b268a0cddbb9fc6f559b9be65f}
```

All 32 recovered bytes are valid hex characters. The inner content looks like an MD5-style hash, which matches the typical picoCTF flag shape.

### Step 4 — Submit

```
picoCTF{71bf23b268a0cddbb9fc6f559b9be65f}
```

Submitted, accepted.

---

## What Happened Internally (Timeline)

1. **Connected to the oracle.** It printed the encrypted flag (64 hex chars = 32 bytes), so the flag's inner content is 32 characters long.
2. **Computed `L = 32` and `pad_len = 50,000 - 32 = 49,968`.**
3. **Sent 49,968 bytes of `'A'`.** The server used `key[32:50000]` to encrypt them (length 49,968 bytes), and the key pointer wrapped to 0.
4. **Sent 32 bytes of `'A'`.** The server used `key[0:32]` — the same bytes that protected the flag — and returned `enc_A = 'A' XOR key[0:32]`.
5. **XOR-recovered the flag.** For each byte, `flag[i] = enc_flag[i] XOR enc_A[i] XOR 'A'`.
6. **Wrapped with `picoCTF{}`** per the challenge description and submitted.

---

## Alternative Method — One-Liner with `nc` and `python3`

If you do not want to script the socket, you can do this with three `printf | nc` invocations and one Python decoder. Pseudocode:

```bash
# Step 1: connect, read encrypted flag, save it.
ENC_FLAG=$(printf "" | nc -q 1 wily-courier.picoctf.net 54406 | grep -oE '[0-9a-f]{40,}')
L=$((${#ENC_FLAG} / 2))

# Step 2: burn 50000 - L bytes to wrap the key pointer.
PAD=$(printf 'A%.0s' $(seq 1 $((50000 - L))))
printf "%s\n" "$PAD" | nc -q 1 wily-courier.picoctf.net 54406 > /dev/null

# Step 3: encrypt L bytes of 'A' with the same key bytes used on the flag.
PROBE=$(printf 'A%.0s' $(seq 1 $L))
ENC_A=$(printf "%s\n" "$PROBE" | nc -q 1 wily-courier.picoctf.net 54406 | grep -oE '[0-9a-f]+' | tail -1)

# Step 4: XOR.
python3 -c "
ef = bytes.fromhex('$ENC_FLAG')
ea = bytes.fromhex('$ENC_A')
print(bytes(e ^ a ^ 0x41 for e, a in zip(ef, ea)).decode())
"
```

The script-with-socket version is more reliable on systems without `nc -q` (like a fresh Kali), so prefer it when in doubt.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nc` / raw socket | Talk to the OTP oracle over TCP |
| `otp.py` (challenge source) | Read the encryption logic and key-pointer behavior |
| Python 3 | Send padding + probe, parse the responses, XOR-recover the flag |
| `nano` | Edit the solver script in the terminal |

---

## Key Takeaways

- A one-time pad is unbreakable *only* if the key is as long as the message, truly random, and **used exactly once**. The moment the same key bytes encrypt two different plaintexts, the cipher collapses — anyone holding both ciphertexts learns the XOR of the two plaintexts.
- This challenge exploits **wraparound reuse**: the server has a 50,000-byte key but no length limit on the message, so eventually the key pointer wraps and starts re-encrypting with the same bytes used earlier. The fix would be to enforce `len(message) + total_used <= KEY_LEN` and refuse once the key is exhausted.
- The hint "Maybe there's a way to make this a 2x pad" is a giveaway: any cipher that lets you re-encrypt known plaintext with the same key bytes is broken by the `c1 XOR c2 = p1 XOR p2` trick.
- The flag's inner content `71bf23b268a0cddbb9fc6f559b9be65f` is a 32-character hex hash — the typical picoCTF "unique challenge tag" pattern. picoCTF regenerates this hex per instance, so the ciphertext and the inner hash change every run, but the attack pattern and the key size stay the same.
- When debugging two-time-pad style attacks, always double-check that your probe really is using the same key bytes you think it is. A common bug is computing the wrong `pad_len` and missing the wrap point by one byte, which silently breaks the XOR without raising any error.

**Flag:** `picoCTF{71bf23b268a0cddbb9fc6f559b9be65f}`
