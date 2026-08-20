# PowerAnalysis: Part 2 — picoCTF Writeup  
**Category:** Cryptography  
**Difficulty:** Hard  
**Points:** 200  
**Flag:** `picoCTF{2f4981b159a0a78a5e222bc537f894ae}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

## Description
> This embedded system allows you to measure the power consumption of the CPU while it is running an AES encryption algorithm. However, this time you have access to only a very limited number of measurements.  
>   
> Download the power-consumption traces for a sample of encryptions `traces.zip`. The flag will be of the format `picoCTF{<encryption key>}` where `<encryption key>` is 32 lowercase hex characters comprising the 16-byte encryption key being used by the program.

## Hints
> 1. The power consumption is correlated with the Hamming weight of the bits being processed  
> 2. You will need to figure out how to deal with the noise

## Background Knowledge
This is the same CPA attack as Part 1, but **offline**: the server gives you a `traces.zip` instead of a live oracle. Same 100 traces, same AES SubBytes power model, same Hamming-weight correlation. No socket code needed.

The "limited measurements" hint is a bit misleading - 100 traces is the same as Part 1, and we already showed that's enough for clean CPA. The real challenge is the noise, which we handle by:
1. Working with the full trace (all 2666 samples) instead of guessing a time slice
2. Taking the **max correlation over all time samples** per key guess
3. Using Pearson correlation (not just sign-of-leak) to maximize SNR

The key insight from Part 1: CPA both identifies the right key AND the right clock cycle, automatically, by max-correlating the prediction over the full trace.

## Solution

### Step 1 — Get and extract the traces
Download `traces.zip` from the challenge page and unzip it.

```bash
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads && unzip traces.zip -d traces_part2
Archive:  traces.zip
   creating: traces_part2/traces/
  inflating: traces_part2/traces/trace00.txt
  inflating: traces_part2/traces/trace01.txt
  ...
  inflating: traces_part2/traces/trace99.txt
```

```bash
┌──(zham㉿kali)-[~]
└─$ ls traces_part2/traces/ | head -5
trace00.txt
trace01.txt
trace02.txt
trace03.txt
trace04.txt
```

```bash
┌──(zham㉿kali)-[~]
└─$ head -1 traces_part2/traces/trace00.txt | head -c 80
Plaintext: ebd6e76cc2f2f4ee3ebd0fff1c3f8e31
```

Each file has two lines: a 16-byte plaintext in hex, then a 2666-point power trace as a Python list.

### Step 2 — Write the offline CPA solver

```bash
┌──(zham㉿kali)-[~]
└─$ nano pa2_cpa.py
```

Paste in the full script:

```python
#!/usr/bin/env python3
"""PowerAnalysis: Part 2 - offline CPA on traces.zip contents.

Reads every trace file, parses the plaintext and the power trace, then
runs CPA per byte using Hamming weight of Sbox[pt ^ k] as the hypothesis.
"""
import os
import re
import numpy as np

TRACES_DIR = r'C:\Users\ASUS\Downloads\traces_part2\traces'

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


# === Phase 1: parse all trace files ===
print(f"[*] phase 1: parse traces from {TRACES_DIR}", flush=True)
files = sorted(f for f in os.listdir(TRACES_DIR)
               if f.startswith('trace') and f.endswith('.txt'))
print(f"  found {len(files)} trace files", flush=True)
pts = []
traces = []
for fn in files:
    with open(os.path.join(TRACES_DIR, fn), 'r') as f:
        text = f.read()
    m_pt = re.search(r'Plaintext:\s*([0-9a-fA-F]+)', text)
    m_tr = re.search(r'Power trace:\s*\[\s*([^\]]+)\]', text)
    if not (m_pt and m_tr):
        continue
    pts.append(bytes.fromhex(m_pt.group(1)))
    traces.append([int(x) for x in m_tr.group(1).split(',')])
print(f"[+] parsed {len(pts)} traces", flush=True)


# === Phase 2: align and mean-center the trace matrix ===
L = min(len(t) for t in traces)
print(f"[*] aligned trace length: {L}", flush=True)
pts_arr = np.array([list(p)[:16] for p in pts], dtype=np.uint8)        # (N, 16)
traces_arr = np.array([t[:L] for t in traces], dtype=np.float64)        # (N, L)
traces_centered = traces_arr - traces_arr.mean(axis=0, keepdims=True)
N = len(pts)


# === Phase 3: CPA per byte ===
print(f"[*] phase 3: CPA per byte (N={N})", flush=True)
key = bytearray(16)
for pos in range(16):
    best = (0.0, 0)
    second = (0.0, 0)
    pt_col = pts_arr[:, pos]
    for k_guess in range(256):
        hyp = HW[Sbox_np[pt_col ^ k_guess].astype(np.int32)]   # (N,)
        hyp_centered = hyp - hyp.mean()
        num = (traces_centered * hyp_centered[:, None]).sum(axis=0)   # (L,)
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

### Step 3 — Run it

```bash
┌──(zham㉿kali)-[~]
└─$ python3 pa2_cpa.py
[*] phase 1: parse traces from C:\Users\ASUS\Downloads\traces_part2\traces
  found 100 trace files
[+] parsed 100 traces
[*] aligned trace length: 2666
[*] phase 3: CPA per byte (N=100)
  pos  0: best=0x2f (max|r|=0.566) second=0x6a (0.464) margin=0.102
  pos  1: best=0x49 (max|r|=0.634) second=0xa2 (0.524) margin=0.110
  pos  2: best=0x81 (max|r|=0.614) second=0x5b (0.458) margin=0.156
  pos  3: best=0xb1 (max|r|=0.551) second=0xac (0.462) margin=0.088
  pos  4: best=0x59 (max|r|=0.560) second=0xc8 (0.460) margin=0.101
  pos  5: best=0xa0 (max|r|=0.576) second=0xaf (0.463) margin=0.113
  pos  6: best=0xa7 (max|r|=0.568) second=0xe9 (0.462) margin=0.105
  pos  7: best=0x8a (max|r|=0.522) second=0x6e (0.449) margin=0.073
  pos  8: best=0x5e (max|r|=0.600) second=0xf9 (0.474) margin=0.126
  pos  9: best=0x22 (max|r|=0.636) second=0x46 (0.459) margin=0.177
  pos 10: best=0x2b (max|r|=0.559) second=0xba (0.455) margin=0.103
  pos 11: best=0xc5 (max|r|=0.627) second=0x53 (0.465) margin=0.163
  pos 12: best=0x37 (max|r|=0.626) second=0x4f (0.453) margin=0.173
  pos 13: best=0xf8 (max|r|=0.660) second=0x92 (0.473) margin=0.187
  pos 14: best=0x94 (max|r|=0.577) second=0xc8 (0.494) margin=0.083
  pos 15: best=0xae (max|r|=0.611) second=0x18 (0.472) margin=0.139
[+] key: 2f4981b159a0a78a5e222bc537f894ae
[+] flag: picoCTF{2f4981b159a0a78a5e222bc537f894ae}
```

The margins (0.073-0.187) are similar to Part 1 with 124 traces (0.076-0.240), which is what worked there. The two byte positions with the lowest margins (pos 7 at 0.073 and pos 14 at 0.083) are the ones to watch if the flag comes back wrong.

### Step 4 — Submit

```bash
┌──(zham㉿kali)-[~]
└─$ echo "picoCTF{2f4981b159a0a78a5e222bc537f894ae}"
picoCTF{2f4981b159a0a78a5e222bc537f894ae
```

Pasted into the picoCTF flag box. Accepted.

## What Happened Internally
1. `traces.zip` contains 100 plaintext+trace pairs, recorded from the same simulated single-round AES used in Part 1. The trace length is 2666 samples per encryption.
2. The CPA loop is identical to Part 1, but without the socket code. We read each `traceNN.txt` file, parse the plaintext (line 1) and the power trace (line 2), then for each of 16 byte positions, score every key guess by max-Pearson-correlation over the full trace.
3. With 100 traces, the correct key byte for each position has correlation 0.52-0.66 at the right clock cycle. Wrong key bytes top out around 0.45-0.49. The margin (best - second-best) is 0.07-0.19 per byte - enough to pick a unique winner in every position.
4. The two weak positions (pos 7 margin 0.073, pos 14 margin 0.083) are the ones to worry about. If the flag comes back wrong, the candidates to try flipping are 0x8a -> 0x6e at pos 7 and 0x94 -> 0xc8 at pos 14.

## Tools Used
| Tool | Purpose |
|------|---------|
| `python3` | Full offline CPA pipeline |
| `unzip` | Extract `traces.zip` |
| `re` | Parse plaintext + trace out of each `traceNN.txt` |
| `numpy` | Vectorized Pearson correlation across all 2666 time samples |
| `os`, `pathlib` | Walk the trace directory |
| `nano` | Script editing on Kali |

## Key Takeaways
- **CPA is identical offline or online.** Whether the oracle is a `nc` server or a `traces.zip`, the math is the same: predict Hamming weight, correlate with trace, pick max.
- **The "limited measurements" hint is overstated.** 100 traces is plenty for clean CPA on the Hamming-weight SubBytes model. The real bottleneck is noise, not sample count, and the max-over-time trick handles it.
- **Always re-use code from previous challenges.** The Part 1 solver, with the socket code stripped out and a `re`-based parser in its place, is exactly the Part 2 solver. Side-channel attacks compose.
- **Wordplay decode**: "Part 2" - the upgrade from Part 1 is just *removing the server* and giving you the traces directly. The crypto attack is the same.

## Alternative Solve Methods
1. **Direct reuse of Part 1's solver.** The CPA loop doesn't care where the traces come from. Wrap it in a parser, point it at the directory, run.
2. **`scared` library** (eShard writeup). Slightly cleaner syntax, but the math under the hood is the same.
3. **Template attack** (more advanced). picoCTF doesn't give you a profiling device, so this isn't applicable.
4. **Manual time-slice selection.** If you know the time slice where SubBytes happens for each byte, you can compute correlation on a single column instead of taking the max. Faster, but requires prior knowledge of the simulator's timing.
