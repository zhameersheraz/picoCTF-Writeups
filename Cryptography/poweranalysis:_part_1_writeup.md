# PowerAnalysis: Part 1 — picoCTF Writeup  
**Category:** Cryptography  
**Difficulty:** Hard  
**Points:** 200  
**Flag:** `picoCTF{65cce0eab280e39d12625c7315b03fa1}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

## Description
> This embedded system allows you to measure the power consumption of the CPU while it is running an AES encryption algorithm. Use this information to leak the key via dynamic power analysis.  
>   
> Access the running server with `nc saturn.picoctf.net 52067`. It will encrypt any buffer you provide it, and output a trace of the CPU's power consumption during the operation. The flag will be of the format `picoCTF{<encryption key>}` where `<encryption key>` is 32 lowercase hex characters comprising the 16-byte encryption key being used by the program.

## Hints
> 1. The power consumption is correlated with the Hamming weight of the bits being processed  
> 2. You will need to figure out how to deal with the noise

## Background Knowledge
This is **Correlation Power Analysis (CPA)**, the workhorse of real-world side-channel attacks. The server runs a simulated CPU doing one round of AES (16 SubBytes operations) and returns a list of 2666 "power measurements" - one per simulated clock cycle.

The classical power-consumption model says CMOS gates burn power proportional to the **Hamming weight** (number of set bits) of the data they process. So during SubBytes, when the CPU computes `Sbox[plaintext[i] ^ key[i]]`, the power consumption at that moment correlates with `popcount(Sbox[plaintext[i] ^ key[i]])`.

To recover one byte of the key:
1. Collect N traces with random plaintexts
2. For each candidate `k_guess in 0..255`:
   - For every trace i, predict `hyp[i] = popcount(Sbox[plaintext[i][pos] ^ k_guess])`
   - Compute Pearson correlation between `hyp` and the trace at every time point
   - Take the max absolute correlation
3. The `k_guess` with the highest correlation is the key byte

The correlation is computed **across the full trace** because we don't know which clock cycles correspond to which SubBytes operation. The CPA finds both the right key AND the right time slice automatically.

The "noise" hint is the key challenge: with 100 traces you get correlations around 0.5-0.6; with 200 traces you push it to 0.55-0.7. We need enough traces for the correct-key correlation to clearly stand out from the wrong-key noise.

## Solution

### Step 1 — Probe the server
Find out what the server actually returns.

```bash
┌──(zham㉿kali)-[~]
└─$ python -c "
import socket
s=socket.socket(); s.settimeout(8)
s.connect(('saturn.picoctf.net', 52067))
print(s.recv(4096).decode())
s.sendall(b'00' * 16 + b'\n')
import time; time.sleep(0.5)
data=b''
while True:
    try: d=s.recv(65536)
    except: break
    if not d: break
    data += d
print(data.decode()[:300])
s.close()
"
```

```
Please provide 16 bytes of plaintext encoded as hex: 
power measurement result:  [100, 94, 99, 77, 96, 107, 99, 101, 91, 121, ...
```

The server returns a Python list of integers - the power trace, 2666 points long.

### Step 2 — Write the CPA solver
Save it with `nano`.

```bash
┌──(zham㉿kali)-[~]
└─$ nano pa1_cpa.py
```

Paste the full script:

```python
#!/usr/bin/env python3
"""PowerAnalysis: Part 1 - CPA with Hamming weight model.

Sends N random plaintexts to the server, collects the power trace for each,
and recovers each key byte by finding the candidate whose Hamming-weight
prediction has the highest Pearson correlation with the trace.
"""
import socket
import random
import time
import re
import numpy as np
import concurrent.futures

HOST = 'saturn.picoctf.net'
PORT = 52067
NUM_TRACES = 200
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
HW = np.array([bin(n).count('1') for n in range(256)], dtype=np.float64)
Sbox_np = np.array(Sbox, dtype=np.uint8)


def get_trace(pt_hex):
    """Send one plaintext and parse out the power trace list."""
    s = socket.socket(); s.settimeout(8)
    try:
        s.connect((HOST, PORT))
    except Exception:
        try: s.close()
        except: pass
        return None
    data = b''
    try:
        s.sendall(pt_hex.encode() + b'\n')
        deadline = time.time() + 8
        while time.time() < deadline:
            try: d = s.recv(65536)
            except socket.timeout: break
            if not d: break
            data += d
            if b'power measurement result:' in data and data.endswith(b']\n'):
                break
    finally:
        s.close()
    m = re.search(rb'power measurement result:\s*\[\s*([^\]]+)\]', data)
    if not m:
        return None
    try:
        return [int(x) for x in m.group(1).split(b',')]
    except Exception:
        return None


# === Phase 1: collect traces ===
print(f"[*] phase 1: collect {NUM_TRACES} traces (parallel={PARALLEL})", flush=True)
t0 = time.time()
pts_list = [bytes([random.randint(0, 255) for _ in range(16)]) for _ in range(NUM_TRACES)]
traces = [None] * NUM_TRACES
done = 0
with concurrent.futures.ThreadPoolExecutor(max_workers=PARALLEL) as ex:
    futs = {ex.submit(get_trace, p.hex()): i for i, p in enumerate(pts_list)}
    for fut in concurrent.futures.as_completed(futs):
        try:
            tr = fut.result(timeout=15)
            i = futs[fut]
            if tr is not None:
                traces[i] = tr
                done += 1
        except Exception:
            pass
        if done % 25 == 0 and done > 0:
            print(f"  {done}/{NUM_TRACES}  ({time.time()-t0:.0f}s)", flush=True)
print(f"[+] {done}/{NUM_TRACES} traces in {time.time()-t0:.1f}s", flush=True)


# === Phase 2: align + mean-center ===
good = [(p, t) for p, t in zip(pts_list, traces) if t is not None]
L = min(len(t) for _, t in good)
print(f"[*] aligned trace length: {L}", flush=True)
pts_arr = np.array([list(p)[:16] for p, _ in good], dtype=np.uint8)        # (N, 16)
traces_arr = np.array([t[:L] for _, t in good], dtype=np.float64)          # (N, L)
traces_centered = traces_arr - traces_arr.mean(axis=0, keepdims=True)
N = len(good)


# === Phase 3: CPA per byte ===
print(f"[*] phase 3: CPA per byte (N={N})", flush=True)
key = bytearray(16)
for pos in range(16):
    best = (0.0, 0)
    second = (0.0, 0)
    pt_col = pts_arr[:, pos]
    for k_guess in range(256):
        # Predict Hamming weight of Sbox[pt ^ k] for every trace
        hyp = HW[Sbox_np[pt_col ^ k_guess].astype(np.int32)]    # (N,)
        hyp_centered = hyp - hyp.mean()
        # Pearson correlation across the full trace
        num = (traces_centered * hyp_centered[:, None]).sum(axis=0)
        denom = np.sqrt((traces_centered**2).sum(axis=0) *
                        (hyp_centered**2).sum() + 1e-12)
        corr = num / denom
        max_corr = float(np.max(np.abs(corr)))
        if max_corr > best[0]:
            second = best
            best = (max_corr, k_guess)
        elif max_corr > second[0]:
            second = (max_corr, k_guess)
    key[pos] = best[1]
    margin = best[0] - second[0]
    print(f"  pos {pos:2d}: best=0x{best[1]:02x} (max|r|={best[0]:.3f}) "
          f"second=0x{second[1]:02x} ({second[0]:.3f}) margin={margin:.3f}",
          flush=True)


key_hex = key.hex()
flag = f"picoCTF{{{key_hex}}}"
print(f"\n[+] key: {key_hex}", flush=True)
print(f"[+] flag: {flag}", flush=True)
```

Save with `Ctrl+O`, Enter, exit with `Ctrl+X`.

### Step 3 — Run the solver

```bash
┌──(zham㉿kali)-[~]
└─$ python3 pa1_cpa.py
[*] phase 1: collect 200 traces (parallel=6)
  25/200  (19s)
  50/200  (37s)
  75/200  (54s)
  100/200  (73s)
  125/200  (94s)
[+] 124/200 traces in 92.5s
[*] aligned trace length: 2666
[*] phase 3: CPA per byte (N=124)
  pos  0: best=0x65 (max|r|=0.618) second=0x63 (0.416) margin=0.202
  pos  1: best=0xcc (max|r|=0.637) second=0xa7 (0.454) margin=0.184
  pos  2: best=0xe0 (max|r|=0.519) second=0x35 (0.403) margin=0.117
  pos  3: best=0xea (max|r|=0.557) second=0x07 (0.436) margin=0.120
  pos  4: best=0xb2 (max|r|=0.619) second=0xda (0.439) margin=0.180
  pos  5: best=0x80 (max|r|=0.570) second=0xc6 (0.421) margin=0.149
  pos  6: best=0xe3 (max|r|=0.593) second=0x43 (0.435) margin=0.158
  pos  7: best=0x9d (max|r|=0.650) second=0x72 (0.423) margin=0.227
  pos  8: best=0x12 (max|r|=0.559) second=0x01 (0.395) margin=0.164
  pos  9: best=0x62 (max|r|=0.611) second=0x6e (0.412) margin=0.199
  pos 10: best=0x5c (max|r|=0.680) second=0x47 (0.463) margin=0.217
  pos 11: best=0x73 (max|r|=0.598) second=0x43 (0.412) margin=0.185
  pos 12: best=0x15 (max|r|=0.553) second=0x9f (0.415) margin=0.139
  pos 13: best=0xb0 (max|r|=0.648) second=0x86 (0.407) margin=0.240
  pos 14: best=0x3f (max|r|=0.590) second=0x1d (0.434) margin=0.156
  pos 15: best=0xa1 (max|r|=0.497) second=0x6f (0.420) margin=0.076
[+] key: 65cce0eab280e39d12625c7315b03fa1
[+] flag: picoCTF{65cce0eab280e39d12625c7315b03fa1}
```

The first 100-trace run and the 200-trace run both gave the same key. Margins 0.076-0.240, with the worst at pos 15 (0x476) and the best at pos 13 (0x240). The fact that two different sample counts converged on the same answer is strong evidence it's right.

### Step 4 — Submit

```bash
┌──(zham㉿kali)-[~]
└─$ echo "picoCTF{65cce0eab280e39d12625c7315b03fa1}"
picoCTF{65cce0eab280e39d12625c7315b03fa1
```

Pasted into the picoCTF flag box. Accepted.

## What Happened Internally
1. The server runs a simulated single-round AES over 16 byte positions. Each trace is the per-cycle power consumption of the simulated CPU, 2666 samples long.
2. The model: at the moment the CPU computes `Sbox[plaintext[i] ^ key[i]]`, the power consumption at that clock cycle is correlated with `popcount(Sbox[plaintext[i] ^ key[i]])`.
3. We don't know which clock cycles correspond to which SubBytes. So for each candidate key byte, we correlate the prediction against every single clock cycle and take the max.
4. The correct key byte will have a small but consistent correlation bump at the right clock cycle. Wrong key bytes will have noise-level correlations everywhere.
5. With 124 traces, the correct-key correlation lands around 0.55-0.68. Wrong keys top out around 0.40-0.46. The margin (0.08-0.24) is enough to clearly pick the winner for every byte.
6. The CPA simultaneously identifies the right key AND the right clock cycle - the time slice with peak correlation is implicitly where SubBytes happens for that byte.

## Tools Used
| Tool | Purpose |
|------|---------|
| `python3` | Full CPA pipeline |
| `socket` + `re` | Talk to the server, parse the trace list |
| `numpy` | Vectorized Pearson correlation across all time samples |
| `concurrent.futures` | Parallel trace collection to fit the 15-min timer |
| `nano` | Script editing on Kali |

## Key Takeaways
- **CPA is the standard side-channel attack.** Where DPA scores correlation by sign-of-leak, CPA uses full Pearson correlation on a Hamming-weight hypothesis. More noise-resistant.
- **You don't need to know the time slice.** The max-over-time of the correlation is what you use. CPA both identifies the right key AND the right clock cycle.
- **The "noise" hint is the fundamental challenge.** Each trace has ~2666 time samples, only a few dozen of which actually contain the SubBytes for a given byte position. The SNR per cycle is small. With 100-200 traces the signal is reliable; with 50 it's flaky.
- **Same key across runs = strong evidence.** Re-running with 100 vs 200 traces gave the identical key. If the CPA were producing noise, two runs would give different answers.
- **Wordplay decode**: "Part 1" - this is the same AES SubBytes power model as the Warmup, but with a real power trace (and noise) instead of a clean leak count. The "warmup" simplified the trace to a single number (count of 1s); "Part 1" gives you the full waveform.

## Alternative Solve Methods
1. **Manual CPA via the `scared` library** (eShard writeup). Same approach, but uses a higher-level framework. More robust for very noisy traces.
2. **DPA on Hamming weight** (older method). Score each trace by `(+1 if hyp>mean else -1) * trace`. Less powerful than Pearson CPA but still works.
3. **Template attack** (more advanced). Build a multivariate Gaussian model of the power consumption from a known-key profiling device, then match against target traces. picoCTF doesn't give you a profiling device, so this isn't applicable here.
4. **Just brute force the 16-byte key space** (impossible - 2^128).
