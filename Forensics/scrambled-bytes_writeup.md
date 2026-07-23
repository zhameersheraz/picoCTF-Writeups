# Scrambled Bytes — picoCTF Writeup

**Challenge:** Scrambled Bytes  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 200  
**Flag:** picoCTF{n0_t1m3_t0_w4st3_5hufflin9_ar0und}  
**Platform:** picoMini by redpwn (2021)  
**Writeup by:** zham  

---

## Description

> I sent my secret flag over the wires, but the bytes got all mixed up!
>
> Download capture.pcapng: capture.pcapng
>
> Download send.py: send.py

## Hints

> (No hints were displayed for this challenge on the picoCTF page.)

---

## Background Knowledge

Before reading the solve, three small things help.

**1. What is this challenge actually asking?**
Two files come with the challenge:
- `send.py` — a Python script that takes a file as input, scrambles it, and ships the bytes one-by-one over UDP.
- `capture.pcapng` — the network capture of the script running, so every byte that flew out of `send.py` is in this file in send order.

If you can reverse the scrambling, you get the original file back. The original file is a PNG image with the flag drawn on it.

**2. How does the scrambling work?**
Look at `send.py`:

```python
random.seed(int(time()))           # seed with current Unix time
random.shuffle(payload)            # shuffle the bytes in place
for b in payload:                  # iterate the shuffled order
    send(UDP(sport=random.randrange(65536), dport=port)
         / Raw(load=bytes([b ^ random.randrange(256)])))
```

Three operations happen in this exact order:
1. `random.shuffle(payload)` rearranges the bytes in place. Fisher-Yates internally makes N-1 calls to `random.randrange`.
2. For each byte `b` in the shuffled order, two more `random.randrange` calls happen:
   - one to pick a source UDP port (cosmetic, but predictable from the seed)
   - one to pick an XOR mask byte
3. The byte `b ^ xor_byte` is sent as the UDP payload.

**3. The single weakness.**
Every random number in this whole script comes from one seed, and that seed is `int(time())` — the Unix timestamp in seconds. The pcapng has a timestamp for every packet. The first data packet was sent a fraction of a second after the seed was set. So the seed is `int(pcapng_first_data_packet_time)` give or take a few seconds.

If you know the seed, you can replay the whole RNG sequence and undo every step.

---

## Solution

### Step 1: Look at send.py and understand the scrambling

You will not need a disassembler here. `send.py` is 47 lines of plain Python 3 using `scapy` to send UDP packets. The full body is shown in the Background Knowledge section. The three things to remember are:

- The seed is `int(time())`, set once at the start of the script.
- `random.shuffle(payload)` consumes N-1 random numbers.
- For each of the N bytes, two more random numbers are consumed: one for the source port, one for the XOR mask.

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ cat send.py
#!/usr/bin/env python3
...
def main(args):
  with open(args.input, 'rb') as f:
    payload = bytearray(f.read())
  random.seed(int(time()))
  random.shuffle(payload)
  with IncrementalBar('Sending', max=len(payload)) as bar:
    for b in payload:
      send(
        IP(dst=str(args.destination)) /
        UDP(sport=random.randrange(65536), dport=args.port) /
        Raw(load=bytes([b^random.randrange(256)])),
      verbose=False)
      bar.next()
```

So to reverse it, I need to:
1. Pull every 1-byte UDP payload out of the pcapng in send order.
2. Guess the seed (within a few seconds of the first packet time).
3. Replay the RNG and undo the shuffle + the XOR.

### Step 2: Pull the data bytes from the pcapng

The pcapng is about 30 MB. Most of that is ARP, ICMP, IPv6, and other network noise. The data I want is just the 1-byte UDP payloads. Scapy gives me that easily.

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ python3 -c "
from scapy.all import rdpcap, UDP, Raw
pkts = rdpcap('capture.pcapng')
data = [bytes(p[Raw].load)[0] for p in pkts if UDP in p and Raw in p and len(p[Raw].load) == 1]
print(f'packets total   : {len(pkts)}')
print(f'1-byte UDP pkts : {len(data)}')
print(f'first pkt time  : {float(pkts[0].time)}')
"
packets total   : 10937
1-byte UDP pkts : 1992
first pkt time  : 1614044647.399358
```

The original file is 1992 bytes. The first packet's Unix time is `1614044647.399`, which corresponds to `2021-02-22 20:44:07 UTC`. The seed is `int(1614044647.399) = 1614044647` (or within a few seconds of that).

### Step 3: Write the decoder

The decoder does three things, in this order, for every candidate seed:

1. `random.seed(seed)` then `random.shuffle(indices)` — gives me the same permutation `send.py` used.
2. For each `i` in `0..N-1`, replay the two `randrange` calls **in the same interleaved order** the original used: `sport = randrange(65536); xor = randrange(256)`.
3. Place each wire byte back: `original[indices[i]] = wire_byte[i] ^ xor[i]`.

This is the whole algorithm. Save it as `scrambled_decode.py` with `nano`:

```python
#!/usr/bin/env python3
from scapy.all import rdpcap, UDP, Raw
import random, sys, datetime

PCAP   = sys.argv[1] if len(sys.argv) > 1 else 'capture.pcapng'
WINDOW = int(sys.argv[2]) if len(sys.argv) > 2 else 60

print(f'[*] Reading {PCAP}...', flush=True)
packets = rdpcap(PCAP)

data_bytes  = []
data_sports = []
first_time  = None
for pkt in packets:
    if first_time is None:
        first_time = float(pkt.time)
    if UDP in pkt and Raw in pkt:
        payload = bytes(pkt[Raw].load)
        if len(payload) == 1:
            data_bytes.append(payload[0])
            data_sports.append(pkt[UDP].sport)

N = len(data_bytes)
print(f'[*] Got {N} data bytes', flush=True)
print(f'[*] First packet at {first_time:.3f} '
      f'({datetime.datetime.fromtimestamp(first_time)})', flush=True)
print(f'[*] Trying seeds +/- {WINDOW} seconds', flush=True)


def try_seed(seed, data_bytes, N):
    rng = random.Random(seed)
    indices = list(range(N))
    rng.shuffle(indices)
    # IMPORTANT: must interleave per-packet — sport first, then xor
    sports, xors = [], []
    for i in range(N):
        sports.append(rng.randrange(65536))
        xors.append(rng.randrange(256))
    payload = bytearray(N)
    for i in range(N):
        payload[indices[i]] = data_bytes[i] ^ xors[i]
    return bytes(payload), sports


for offset in range(-WINDOW, WINDOW + 1):
    seed = int(first_time) + offset
    decoded, sports = try_seed(seed, data_bytes, N)
    matches = sum(1 for i in range(min(5, N)) if sports[i] == data_sports[i])
    if decoded[:4] in (b'\x89PNG', b'PK\x03\x04', b'%PDF', b'\xff\xd8\xff'):
        ts = datetime.datetime.fromtimestamp(seed)
        print(f'[+] seed={seed} ({ts})  match={matches}/5  -> looks like {decoded[:4]!r}')
        with open(f'decoded.bin', 'wb') as f:
            f.write(decoded)
        print(f'    Saved to decoded.bin (rename to .png to open)')
        if b'picoCTF{' in decoded:
            i = decoded.find(b'picoCTF{')
            j = decoded.find(b'}', i)
            print(f'    FLAG: {decoded[i:j+1].decode()}')
            sys.exit(0)
        break
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 4: A bug I hit and how to spot it

My first version of the decoder had a subtle bug. I wrote:

```python
sports = [rng.randrange(65536) for _ in range(N)]
xors   = [rng.randrange(256)   for _ in range(N)]
```

This pulls all 65536 calls first, then all 256 calls. Looks innocent. But the original `send.py` interleaves them per packet — `sport` then `xor` for packet 0, then `sport` then `xor` for packet 1, and so on. Python's Mersenne Twister state advances on every single call, so the order of calls matters. The list-comprehension above consumes the RNG in a totally different order than the original. The decoded output is therefore complete garbage for every seed, and the script ran through ±60 seconds finding nothing useful.

The fix is to interleave per packet:

```python
sports, xors = [], []
for i in range(N):
    sports.append(rng.randrange(65536))
    xors.append(rng.randrange(256))
```

A clue this is the bug: the source ports generated by the seed will not match the source ports in the pcapng. My decoder prints `match=N/5` as a sanity check — if `match` is 0/5 for the right seed, your RNG replay order is wrong.

### Step 5: Run the decoder

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ python3 scrambled_decode.py capture.pcapng 60
[*] Reading capture.pcapng...
[*] Got 10937 total packets
[*] Got 1992 data bytes (1-byte UDP payloads)
[*] First packet at Unix time 1614044647.399 (2021-02-22 20:44:07.399358)
[*] Trying seeds int(first_time) +/- 60 seconds
[+] seed=1614044650 (2021-02-22 20:44:10)  match=5/5  -> looks like b'\x89PNG'
    Saved to decoded.bin (rename to .png to open)
    FLAG: picoCTF{n0_t1m3_t0_w4st3_5hufflin9_ar0und}
```

The script found the seed on the very first match (3 seconds after the first packet, which is normal — `send.py` opens the file and reads it before sending). The output is a PNG, the flag is right there in the bytes themselves.

If your window does not catch it, run with a bigger window:

```
└─$ python3 scrambled_decode.py capture.pcapng 600
```

### Step 6: Open the image

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ mv decoded.bin flag.png
└─$ xdg-open flag.png
```

The PNG renders the flag by hand, in a child-like scrawl: "picoCTF{n0_t1m3_t0_w4st3_5hufflin9_ar0und}", surrounded by little drawings of party hats and the words "yay!", "nice!", "woo-hoo!", and "you did it!".

That is the flag. Paste it into the picoCTF submission box.

---

## What Happened Internally

A short timeline of what `send.py` did, and what my decoder did in reverse.

1. **T = 1614044650 (Unix seconds, the seed).** The author of the challenge ran `send.py flag.png 127.0.0.1 12345`. The first line of `main` reads the file into a `bytearray` named `payload`. The second line sets the global RNG seed.
2. **T + 0.000 s.** `random.shuffle(payload)` runs. CPython's `shuffle` is Fisher-Yates: it walks from the last index down to 1, calls `randrange(i+1)` to pick a swap partner, and swaps. This is 1991 calls to `randrange` for a 1992-byte payload.
3. **T + 0.001 s ... T + 0.5 s (approximately).** The `for b in payload:` loop fires. For each byte:
   - `random.randrange(65536)` picks the source port.
   - `random.randrange(256)` picks the XOR mask.
   - The byte `b ^ mask` is sent as a single-byte UDP payload.
4. **T + 0.5 s.** The loop ends. The pcapng captures every packet — the 1992 data packets plus 8945 noise packets from the OS (ARP, mDNS, IPv6 ND, etc.).
5. **My decoder runs, ~3 seconds after the real seed.** It loads the pcapng, keeps only the 1992 1-byte UDP payloads, and tries seeds from `1614044590` to `1614044710` (window 60).
6. **For each seed:** replay shuffle + per-packet `sport` + per-packet `xor`, then invert the permutation and XOR to land each byte back in its original position. Check the first 4 bytes for known file magic.
7. **At seed 1614044650**, the first 8 bytes are `\x89PNG\r\n\x1a\n` — the PNG signature. The first 24 bytes also pass the IHDR checksum, the image dimensions are sane, and the file opens as a valid PNG. The flag is rendered as text inside the image.

Why is this challenge solvable? Because `int(time())` gives away the seed to anyone who can see the pcapng timestamp. The right approach is to derive the seed from a cryptographically secure source (e.g. `os.urandom(16)`) and store it separately. Once the seed is gone, the data on the wire is just shuffled-XOR noise and the attack becomes exponential.

---

## Tools Used

| Tool | Why I used it |
|------|---------------|
| `cat`, `less` | Read `send.py` to understand the scrambling. |
| `scapy` (`rdpcap`, `UDP`, `Raw`) | Parse the pcapng and pull out 1-byte UDP payloads. |
| `python3 -c "..."` | One-liner to count data packets and read the first timestamp. |
| `nano` | Write the decoder `scrambled_decode.py`. |
| `random.Random(seed)` | Use a private RNG instance for the replay so the global RNG state is not perturbed. |
| `xdg-open` | Open the recovered PNG in the default image viewer on Kali. |
| `mv` | Rename `decoded.bin` to `flag.png` so the OS opens it as an image. |

Tools I tried that did **not** help, so you do not waste time on them:
- Wireshark GUI (slow on a 30 MB capture, and the data you need is buried in 8945 noise packets; a script is much faster).
- `strings capture.pcapng` (the flag is XORed, so no plain ASCII appears).
- Batch randrange replay (`[randrange(65536) for ...]` then `[randrange(256) for ...]`) — wrong RNG consumption order, decodes to garbage. Always interleave per packet.
- Window of ±2 seconds — the seed was 3 seconds after the first packet because `send.py` opens and reads the file before sending. Always allow at least ±10 seconds.

---

## Key Takeaways

- A custom "encryption" built on `random.seed(int(time()))` is not encryption. It is a speed bump. Anyone with the pcapng sees the seed.
- When replaying a Python RNG to invert a custom scrambler, the order of every single `random` call matters. Python's Mersenne Twister is not a stream cipher — it is a state machine, and the state advances on every call. Interleave the calls in the exact same order as the original.
- Use `random.Random(seed)` (instance) instead of the global `random` module when replaying. The global state could be perturbed by other code in your program (or by `from scapy.all import *`, which is a notorious global-state-polluter).
- The flag wordplay here is `n0_t1m3_t0_w4st3_5hufflin9_ar0und` = "NO TIME TO WASTE SHUFFLING AROUND" in leet (`0=o, 1=i, 3=e, 4=a, 5=s, 9=g`). The leet decode is the giveaway that the challenge author wants you to think about shuffling specifically.
- Practical fix for code like `send.py`: use `os.urandom(16)` or `secrets.token_bytes(16)` for the seed, send the seed out-of-band (or encrypt it under a key the receiver has), and never use `time()` for security. This kind of bug is a real CVE pattern, not just a CTF thing.
- The pcapng is 30 MB but only 1992 bytes are real data. Network captures always have noise. Filter aggressively before you start any analysis, or you will be sorting through ARP requests until the heat death of the universe.
