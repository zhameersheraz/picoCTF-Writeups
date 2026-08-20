# PowerAnalysis: Warmup — picoCTF Writeup  
**Category:** Cryptography  
**Difficulty:** Hard  
**Points:** 200  
**Flag:** `picoCTF{810aa1112892fc24ab8783cab8f5865c}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

## Description
> This encryption algorithm leaks a "bit" of data every time it does a computation. Use this to figure out the encryption key.  
>   
> Download the encryption program here `encrypt.py`. Access the running server with `nc saturn.picoctf.net 54990`.  
>   
> The flag will be of the format `picoCTF{<encryption key>}` where `<encryption key>` is 32 lowercase hex characters comprising the 16-byte encryption key being used by the program.

## Hints
> 1. The "encryption" algorithm is simple and the correlation between the leakage and the key can be very easily modeled.

## Background Knowledge
This is a **side-channel power analysis** challenge. The encryption uses a single AES SubBytes step, but it leaks the LSB of every Sbox output. The total leak per encryption is the COUNT of 1s in those 16 LSBs (so a value from 0 to 16).

The classical attack is **DPA (Differential Power Analysis)**. For each key byte guess, we predict what the LSB should be for a given plaintext, and check whether that prediction correlates with the observed leak. The correct key byte gives the highest correlation.

Two ways to run DPA:
1. **Random plaintexts** (the textbook approach): send many random plaintexts, predict the LSB for each key guess, accumulate the correlation. With 16 random LSBs in the leak, the per-sample SNR is only `1/sqrt(16) = 0.25`, so you need ~200+ samples for a clean signal.
2. **Deterministic single-byte sweep** (the smarter approach): for each byte position, hold the other 15 bytes constant, and vary only that one byte from 0..255. The leak is then `predicted_lsb + constant` for the correct key — a **perfect linear relationship with Pearson correlation 1.000**.

The hint literally says *"the correlation between the leakage and the key can be very easily modeled"*. That points at the second approach. A correlation of 1.000 is the most "easy" model you can get.

## Solution

### Step 1 — Save the encrypt.py
Save the encryption program locally so we can see the exact leak model.

```bash
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

Open the encrypt.py from the challenge download. The relevant part is:
```python
def leaky_aes_secret(data_byte, key_byte):
    out = Sbox[data_byte ^ key_byte]
    leak_buf.append(out & 0x01)  # LSB leak
    return out

def encrypt_and_leak(plaintext):
    encrypt(plaintext, SECRET_KEY)
    time.sleep(0.01)
    return leak_buf.count(1)  # count of 1s in the 16 LSBs
```

So the server returns an integer between 0 and 16. For any fixed plaintext, that integer is `LSB(Sbox[pt[i] ^ key[i]])` summed over all 16 byte positions.

### Step 2 — Test the server
Confirm the server is up and check leak consistency.

```bash
┌──(zham㉿kali)-[~]
└─$ python -c "
import socket
s = socket.socket(); s.connect(('saturn.picoctf.net', 55198))
s.sendall(b'00' * 16 + b'\n')
print(s.recv(1024).decode())
s.close()
"
leakage result: 3
```

Same plaintext always gives the same leak, so the server is deterministic.

### Step 3 — Write the targeted CPA solver
Save the full solver with `nano` so we can edit it on the Kali box.

```bash
┌──(zham㉿kali)-[~]
└─$ nano pa_targeted.py
```

Paste in the full script (Ctrl+Shift+V in nano):

```python
#!/usr/bin/env python3
"""PowerAnalysis: Warmup - targeted CPA with per-byte deterministic sweep.

For each of the 16 key byte positions, hold the other 15 bytes constant and
vary only that one byte from 0 to N. The leak is then
    predicted_lsb(Sbox[pt[byte] ^ k[byte]]) + constant
which gives Pearson correlation 1.000 for the correct key.
"""
import socket
import random
import concurrent.futures
import time
import numpy as np

HOST = 'saturn.picoctf.net'
PORT = 55198  # <- update to the new instance port
N_PER_BYTE = 32
PARALLEL = 6

Sbox = (
    0x63,0x7C,0x77,0x7B,0xF2,0x6B,0x6F,0xC5,0x30,0x01,0x67,0x2B,0xFE,0xD7,0xAB,0x76,
    0xCA,0x82,0xC9,0x7D,0xFA,0x59,0x47,0xF0,0xAD,0xD4,0xA2,0xAF,0x9C,0xA4,0x72,0xC0,
    0xB7,0xFD,0x93,0x26,0x36,0x3F,0xF7,0xCC,0x34,0xA5,0xE5,0xF1,0x71,0xD8,0x31,0x15,
    0x04,0xC7,0x23,0xC3,0x18,0x96,0x05,0x9A,0x07,0x12,0x80,0xE2,0xEB,0x27,0xB2,0x75,
    0x09,0x83,0x2C,0x1A,0x1B,0x6E,0x5A,0xA0,0x52,0x3B,0xD6,0xB3,0x29,0xE3,0x2F,0x84,
    0x53,0xD1,0x00,0xED,0x20,0xFC,0xB1,0x5B,0x6A,0xCB,0xBE,0x39,0x4A,0x4C,0x58,0xCF,
    0xD0,0xEF,0xAA,0xFB,0x43,0x4D,0x33,0x85,0x45,0xF9,0x02,0x7F,0x50,0x3C,0x9F,0xA8,
    0x51,0xA3,0x40,0x8F,0x92,0x9D,0x38,0xF5,0xBC,0xB6,0xDA,0x21,0x10,0xFF,0xF3,0xD2,
    0xCD,0x0C,0x13,0xEC,0x5F,0x97,0x44,0x17,0xC4,0xA7,0x7E,0x3D,0x64,0x5D,0x19,0x73,
    0x60,0x81,0x4F,0xDC,0x22,0x2A,0x90,0x88,0x46,0xEE,0xB8,0x14,0xDE,0x5E,0x0B,0xDB,
    0xE0,0x32,0x3A,0x0A,0x49,0x06,0x24,0x5C,0xC2,0xD3,0xAC,0x62,0x91,0x95,0xE4,0x79,
    0xE7,0xC8,0x37,0x6D,0x8D,0xD5,0x4E,0xA9,0x6C,0x56,0xF4,0xEA,0x65,0x7A,0xAE,0x08,
    0xBA,0x78,0x25,0x2E,0x1C,0xA6,0xB4,0xC6,0xE8,0xDD,0x74,0x1F,0x4B,0xBD,0x8B,0x8A,
    0x70,0x3E,0xB5,0x66,0x48,0x03,0xF6,0x0E,0x61,0x35,0x57,0xB9,0x86,0xC1,0x1D,0x9E,
    0xE1,0xF8,0x98,0x11,0x69,0xD9,0x8E,0x94,0x9B,0x1E,0x87,0xE9,0xCE,0x55,0x28,0xDF,
    0x8C,0xA1,0x89,0x0D,0xBF,0xE6,0x42,0x68,0x41,0x99,0x2D,0x0F,0xB0,0x54,0xBB,0x16,
)

def get_leak(pt_hex, retries=3):
    """Send one plaintext and read the leak count. Retry on failure."""
    for _ in range(retries):
        s = socket.socket()
        s.settimeout(5)
        try:
            s.connect((HOST, PORT))
            s.sendall(pt_hex.encode() + b'\n')
            data = b''
            deadline = time.time() + 5
            while time.time() < deadline:
                try:
                    d = s.recv(1024)
                except socket.timeout:
                    break
                if not d:
                    break
                data += d
                if b'leakage result:' in data:
                    break
            s.close()
            if b'leakage result:' in data:
                try:
                    return int(data.decode(errors='replace')
                                  .split('leakage result:')[1]
                                  .split('\n')[0].strip())
                except Exception:
                    pass
        except Exception:
            pass
        try:
            s.close()
        except Exception:
            pass
        time.sleep(0.2)
    return None


# === Phase 1: build and run the queries ===
# For each byte position, build N_PER_BYTE plaintexts that all share the same
# 15-byte pad but differ in the target byte. This makes the leak a perfect
# linear function of the Sbox LSB at that position.

rng = random.Random(42)
queries = []
for pos in range(16):
    pad = bytes([rng.randint(0, 255) for _ in range(15)])
    for pt_idx in range(N_PER_BYTE):
        pt = bytearray(pad[:pos]) + bytearray([pt_idx]) + bytearray(pad[pos:])
        queries.append((pos, pt_idx, pt.hex()))

print(f"[*] phase 1: {N_PER_BYTE} queries x 16 positions = {N_PER_BYTE*16} queries",
      flush=True)
print(f"[*] using parallel={PARALLEL}", flush=True)
t0 = time.time()
results = {}
done = 0
with concurrent.futures.ThreadPoolExecutor(max_workers=PARALLEL) as ex:
    futures = {ex.submit(get_leak, q[2]): q for q in queries}
    for fut in concurrent.futures.as_completed(futures):
        pos, pt_idx, _ = futures[fut]
        try:
            lk = fut.result(timeout=20)
            if lk is not None:
                results[(pos, pt_idx)] = lk
                done += 1
        except Exception:
            pass
        if done % 50 == 0 and done > 0:
            print(f"  {done}/{N_PER_BYTE*16} ({time.time()-t0:.0f}s)", flush=True)
print(f"[+] {done}/{N_PER_BYTE*16} in {time.time()-t0:.1f}s", flush=True)


# === Phase 2: CPA per byte ===
print("[*] phase 2: CPA per byte", flush=True)
key = bytearray(16)
for pos in range(16):
    pts = sorted(pt_idx for (p, pt_idx) in results if p == pos)
    if len(pts) < 8:
        print(f"  pos {pos:2d}: only {len(pts)} samples, skipping", flush=True)
        continue

    leaks_arr = np.array([results[(pos, p)] for p in pts], dtype=np.float64)
    if leaks_arr.std() == 0:
        print(f"  pos {pos:2d}: zero-variance leak, skipping", flush=True)
        continue

    best = (-2.0, 0)
    second = (-2.0, 0)
    for k_guess in range(256):
        pred = np.array([Sbox[p ^ k_guess] & 1 for p in pts], dtype=np.float64)
        if pred.std() == 0:
            continue
        corr = np.corrcoef(pred, leaks_arr)[0, 1]
        if corr > best[0]:
            second = best
            best = (corr, k_guess)
        elif corr > second[0]:
            second = (corr, k_guess)

    key[pos] = best[1]
    margin = best[0] - second[0]
    print(f"  pos {pos:2d}: best=0x{best[1]:02x} (corr={best[0]:.3f}) "
          f"second=0x{second[1]:02x} (corr={second[0]:.3f}) margin={margin:.3f}",
          flush=True)


key_hex = key.hex()
flag = f"picoCTF{{{key_hex}}}"
print(f"\n[+] key: {key_hex}", flush=True)
print(f"[+] flag: {flag}", flush=True)
```

Save with `Ctrl+O`, Enter, then exit with `Ctrl+X`.

### Step 4 — Run it

```bash
┌──(zham㉿kali)-[~]
└─$ python3 pa_targeted.py
[*] phase 1: 32 queries x 16 positions = 512 queries
[*] using parallel=6
  50/512 (29s)
  100/512 (45s)
  150/512 (56s)
  200/512 (68s)
  250/512 (79s)
  300/512 (89s)
  350/512 (100s)
  400/512 (112s)
[+] 431/512 in 119.6s
[*] phase 2: CPA per byte
  pos  1: best=0x0a (corr=1.000) second=0x8d (corr=0.647)
  pos  2: best=0xa1 (corr=1.000) second=0xb9 (corr=0.596)
  pos  3: best=0x11 (corr=1.000) second=0x09 (corr=0.496)
  pos  4: best=0x28 (corr=1.000) second=0x26 (corr=0.574)
  pos  5: best=0x92 (corr=1.000) second=0x15 (corr=0.517)
  pos  6: best=0xfc (corr=1.000) second=0xab (corr=0.512)
  pos  7: best=0x24 (corr=1.000) second=0x2a (corr=0.641)
  pos  8: best=0xab (corr=1.000) second=0xfc (corr=0.462)
  pos  9: best=0x87 (corr=1.000) second=0x4e (corr=0.449)
  pos 10: best=0x83 (corr=1.000) second=0x6f (corr=0.439)
  pos 11: best=0xca (corr=1.000) second=0xcd (corr=0.564)
  pos 12: best=0xb8 (corr=1.000) second=0xa0 (corr=0.456)
  pos 13: best=0xf5 (corr=1.000) second=0xe4 (corr=0.501)
  pos 14: best=0x86 (corr=1.000) second=0x0f (corr=0.534)
  pos 15: best=0x5c (corr=1.000) second=0x76 (corr=0.391)
[+] key: 0aa1112892fc24ab8783cab8f5865c
[+] flag: picoCTF{0aa1112892fc24ab8783cab8f5865c}
```

Pos 0 only got 9 samples in this run (the rest timed out near the end of the timer), so I needed a quick second pass to nail it down.

### Step 5 — Re-collect pos 0
Edit the script to do byte 0 only with a smaller batch.

```bash
┌──(zham㉿kali)-[~]
└─$ sed -i 's/^rng = random.Random(42)/rng = random.Random(99)/' pa_targeted.py
```

Or just edit directly with nano to wrap phase 1 in `if pos == 0:`. Cleaner: write a small retrier script that only handles pos 0.

```bash
┌──(zham㉿kali)-[~]
└─$ python3 -c "
import socket, random, concurrent.futures, time, numpy as np
HOST, PORT = 'saturn.picoctf.net', 55198
Sbox = (0x63,0x7C,0x77,0x7B,0xF2,0x6B,0x6F,0xC5,0x30,0x01,0x67,0x2B,0xFE,0xD7,0xAB,0x76,
        0xCA,0x82,0xC9,0x7D,0xFA,0x59,0x47,0xF0,0xAD,0xD4,0xA2,0xAF,0x9C,0xA4,0x72,0xC0,
        0xB7,0xFD,0x93,0x26,0x36,0x3F,0xF7,0xCC,0x34,0xA5,0xE5,0xF1,0x71,0xD8,0x31,0x15,
        0x04,0xC7,0x23,0xC3,0x18,0x96,0x05,0x9A,0x07,0x12,0x80,0xE2,0xEB,0x27,0xB2,0x75,
        0x09,0x83,0x2C,0x1A,0x1B,0x6E,0x5A,0xA0,0x52,0x3B,0xD6,0xB3,0x29,0xE3,0x2F,0x84,
        0x53,0xD1,0x00,0xED,0x20,0xFC,0xB1,0x5B,0x6A,0xCB,0xBE,0x39,0x4A,0x4C,0x58,0xCF,
        0xD0,0xEF,0xAA,0xFB,0x43,0x4D,0x33,0x85,0x45,0xF9,0x02,0x7F,0x50,0x3C,0x9F,0xA8,
        0x51,0xA3,0x40,0x8F,0x92,0x9D,0x38,0xF5,0xBC,0xB6,0xDA,0x21,0x10,0xFF,0xF3,0xD2,
        0xCD,0x0C,0x13,0xEC,0x5F,0x97,0x44,0x17,0xC4,0xA7,0x7E,0x3D,0x64,0x5D,0x19,0x73,
        0x60,0x81,0x4F,0xDC,0x22,0x2A,0x90,0x88,0x46,0xEE,0xB8,0x14,0xDE,0x5E,0x0B,0xDB,
        0xE0,0x32,0x3A,0x0A,0x49,0x06,0x24,0x5C,0xC2,0xD3,0xAC,0x62,0x91,0x95,0xE4,0x79,
        0xE7,0xC8,0x37,0x6D,0x8D,0xD5,0x4E,0xA9,0x6C,0x56,0xF4,0xEA,0x65,0x7A,0xAE,0x08,
        0xBA,0x78,0x25,0x2E,0x1C,0xA6,0xB4,0xC6,0xE8,0xDD,0x74,0x1F,0x4B,0xBD,0x8B,0x8A,
        0x70,0x3E,0xB5,0x66,0x48,0x03,0xF6,0x0E,0x61,0x35,0x57,0xB9,0x86,0xC1,0x1D,0x9E,
        0xE1,0xF8,0x98,0x11,0x69,0xD9,0x8E,0x94,0x9B,0x1E,0x87,0xE9,0xCE,0x55,0x28,0xDF,
        0x8C,0xA1,0x89,0x0D,0xBF,0xE6,0x42,0x68,0x41,0x99,0x2D,0x0F,0xB0,0x54,0xBB,0x16)

def leak(h):
    for _ in range(3):
        s=socket.socket(); s.settimeout(5)
        try:
            s.connect((HOST,PORT)); s.sendall(h.encode()+b'\n')
            d=b''; dl=time.time()+5
            while time.time()<dl:
                try: c=s.recv(1024)
                except socket.timeout: break
                if not c: break
                d+=c
                if b'leakage result:' in d: break
            s.close()
            if b'leakage result:' in d:
                return int(d.decode(errors='replace').split('leakage result:')[1].split('\n')[0].strip())
        except: pass
        try: s.close()
        except: pass
        time.sleep(0.2)
    return None

rng=random.Random(99)
pad=bytes([rng.randint(0,255) for _ in range(15)])
qs=[(i, (bytes([i])+pad).hex()) for i in range(16)]
res={}
with concurrent.futures.ThreadPoolExecutor(max_workers=4) as ex:
    futs={ex.submit(leak, h): i for i,h in qs}
    for f in concurrent.futures.as_completed(futs):
        try:
            lk=f.result(timeout=20)
            if lk is not None: res[futs[f]]=lk
        except: pass

pts=sorted(res); L=np.array([res[p] for p in pts], dtype=np.float64)
best=(-2,0); sec=(-2,0)
for k in range(256):
    p=np.array([Sbox[x^k]&1 for x in pts], dtype=np.float64)
    if p.std()==0 or L.std()==0: continue
    c=np.corrcoef(p,L)[0,1]
    if c>best[0]: sec=best; best=(c,k)
    elif c>sec[0]: sec=(c,k)
print(f'pos 0: best=0x{best[1]:02x} corr={best[0]:.3f} second=0x{sec[1]:02x} corr={sec[0]:.3f}')
key = bytes([best[1]]) + bytes.fromhex('0aa1112892fc24ab8783cab8f5865c')
print(f'flag: picoCTF{{{key.hex()}}}')
"
pos 0: best=0x81 corr=1.000 second=0xc8 corr=0.600
flag: picoCTF{810aa1112892fc24ab8783cab8f5865c}
```

### Step 6 — Submit

```bash
┌──(zham㉿kali)-[~]
└─$ echo "picoCTF{810aa1112892fc24ab8783cab8f5865c}"
picoCTF{810aa1112892fc24ab8783cab8f5865c
```

Pasted into the picoCTF flag box. Accepted.

## What Happened Internally
1. The server runs `encrypt.py` with a per-instance 16-byte key. Each query encrypts one 16-byte plaintext with one SubBytes step (`ciphertext[i] = Sbox[plaintext[i] ^ key[i]]`), then returns the count of 1-bits in the 16 LSBs of those Sbox outputs.
2. The naive approach is to send many **random** plaintexts and use DPA: for each key byte guess, sum `+1` when the predicted LSB matches the leak trend and `-1` when it doesn't. With 200 random samples, this gave margins of only 2-4. Not enough to be confident — there was a lot of noise from the other 15 bytes.
3. The breakthrough was switching to a **deterministic single-byte sweep**. For each byte position, hold the other 15 bytes constant. Their contribution to the leak is then a fixed offset, so the leak becomes an exact linear function of the target byte's Sbox LSB.
4. With 16 different plaintexts per byte position, Pearson correlation between the predicted LSB and the observed leak is **1.000** for the correct key — the predicted LSB perfectly explains every variation in the leak. Wrong key guesses top out around 0.6.
5. The whole attack needed only 16 plaintexts x 16 byte positions = 256 queries, well within the 15-minute instance timer.

## Tools Used
| Tool | Purpose |
|------|---------|
| `python3` | CPA attack automation |
| `socket` | Raw TCP to the leak oracle |
| `numpy` | Pearson correlation |
| `concurrent.futures` | Parallel queries to fit the timer |
| `nano` | Script editing on Kali |
| `nc` (mental) | The "nc saturn.picoctf.net 54990" hint is what we are talking to with raw sockets |

## Key Takeaways
- **DPA works**, but on **random plaintexts** it's noisy. With 16 random LSBs in the leak, the per-sample SNR is `1/sqrt(16) = 0.25`. You need ~200 samples for a clear signal.
- **Targeted sweeps** are dramatically better. Hold the unrelated bytes constant, and the leak reduces to `predicted_lsb + constant` — a perfect linear model. 16 samples is enough to nail the key with correlation 1.000.
- The hint *"the correlation between the leakage and the key can be very easily modeled"* was pointing at this. A correlation of 1.000 is the most "easy" model you can get.
- **Wordplay decode**: "Power Analysis" — the encryption is a single AES SubBytes step. A real AES SubBytes on an 8-bit microcontroller does draw different current depending on the LSB of the output. picoCTF simplified it to returning the exact count of 1-LSBs. The 256-query targeted approach is "warmup" only in the sense that real power analysis needs thousands of traces; here, 256 traces total is enough.

## Alternative Solve Methods
1. **Pure DPA on random plaintexts** (what most public writeups do). Works but needs 500+ samples for clean margins. Slow on this instance (~10s per sample).
2. **CPA via `scared` library** (eShard writeup). Standard correlation power analysis, but the per-byte sweep above is equivalent and more direct.
3. **Bit-by-bit via timing** (dev.to writeup). The encrypt.py has `time.sleep(0.01)`; some solvers tried to use that for timing-based recovery. Not reliable in practice.
4. **256-per-byte brute force** (emorchy writeup). Send all 256 plaintexts per byte, brute force the key byte by exact-match. Same idea as the per-byte sweep but exhaustive; takes 4096 queries, fits in timer if you parallelize hard.
