# Surfing the Waves — picoCTF Writeup

**Challenge:** Surfing the Waves  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 150  
**Flag:** picoCTF{mU21C_1s_1337_5db6b85e}  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> While you're going through the FBI's servers, you stumble across their incredible taste in music. One you found is particularly interesting, see if you can find the flag!
>
> Download main.wav: main.wav

## Hints

> 1. Music is cool, but what other kinds of waves are there?
> 2. Look deep below the surface

---

## Background Knowledge

Before reading the solve, three things help.

**1. What is a WAV file, really?**
A `.wav` file is just a wrapper around raw integer samples. Each sample is a 16-bit signed integer (typically) representing the amplitude of the audio signal at a point in time. A 1-second mono file at 2736 Hz has 2736 samples. A 16-bit sample can hold any value from -32768 to +32767. The samples are interpreted as amplitude, but they are *just numbers*. There is nothing in the WAV format that forces them to *be* audio — they could encode anything.

**2. "Other kinds of waves" — what does the hint mean?**
Audio is one kind of wave — a pressure wave in air. There are also:
- Electromagnetic waves (radio, light, x-rays)
- Water waves (the challenge title is a pun — "surfing" is a water activity)
- Digital "waves" of voltage on a wire
- Seismic waves through the ground

The hint is asking: "what if the audio is not actually audio, but a *different kind* of wave carrying data?" The answer here is that the data is encoded as a sequence of discrete amplitude *levels* — not a continuous sound wave.

**3. "Look deep below the surface" — what does that mean?**
The audio looks noisy in a normal player. The signal-to-noise ratio is bad. But if you *amplify* the differences (look "deeper"), a pattern emerges: every sample is one of exactly 16 distinct quantized levels, plus 10 units of random noise. The quantization is the data; the noise is the camouflage.

---

## Solution

### Step 1: Look at the raw sample values

The hint says "look deep below the surface" — that means skip the audio rendering and read the integer values directly. Open the file in Python:

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "
from scipy.io import wavfile
_, data = wavfile.read('main.wav')
print('First 6 samples:', data[:6].tolist())
print('Length:', len(data))
"
First 6 samples: [2002, 2507, 2001, 1508, 2006, 8500]
Length: 2736
```

Already suspicious. Real audio samples are not nice round numbers like 2002, 2507, 2001, 1508. They are continuous values that come out of a microphone. These values are quantized to 16 distinct levels.

### Step 2: Strip the noise and find the 16 levels

The last two digits of each sample are random noise (a value 0-9 added to disguise the pattern). The first two digits give the actual level. Truncate:

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "
from scipy.io import wavfile
_, data = wavfile.read('main.wav')
rounded = [int(str(s)[:2]) for s in data]
print('Unique rounded values:', sorted(set(rounded)))
"
Unique rounded values: [10, 15, 20, 25, 30, 35, 40, 45, 50, 55, 60, 65, 70, 75, 80, 85]
```

Exactly 16 unique values. 16 is also the number of hex digits (0-9, a-f). Strong signal that the levels encode hex nibbles.

### Step 3: Map the 16 levels to 16 hex digits

Sort the levels and assign rank 0-15, then map to hex:

| Level | Rank | Hex |
|-------|------|-----|
| 10    | 0    | `0` |
| 15    | 1    | `1` |
| 20    | 2    | `2` |
| 25    | 3    | `3` |
| 30    | 4    | `4` |
| 35    | 5    | `5` |
| 40    | 6    | `6` |
| 45    | 7    | `7` |
| 50    | 8    | `8` |
| 55    | 9    | `9` |
| 60    | 10   | `a` |
| 65    | 11   | `b` |
| 70    | 12   | `c` |
| 75    | 13   | `d` |
| 80    | 14   | `e` |
| 85    | 15   | `f` |

Note that the levels are exactly `1000 + nibble * 500`, so the conversion is `nibble = (level - 1000) / 500`. Either form works.

### Step 4: Build the hex string and decode

Save the following as `solve.py`:

```python
from scipy.io import wavfile

sr, data = wavfile.read('main.wav')

# Build the hex string
hex_str = ''
for v in data:
    level = int(str(int(v))[:2])        # truncate to 2 digits
    nibble = (level - 10) // 5         # 10→0, 15→1, 20→2, ..., 85→15
    hex_str += hex(nibble)[2:]          # strip the '0x' prefix

# Decode as ASCII
print(bytes.fromhex(hex_str).decode('utf-8', errors='replace'))
```

Run it:

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
#!/usr/bin/env python3
import numpy as np
from scipy.io.wavfile import write
from binascii import hexlify
from random import random

with open('generate_wav.py', 'rb') as f:
    content = f.read()
    f.close()

# Convert this program into an array of hex values
hex_stuff = (list(hexlify(content).decode("utf-8")))

# Loop through the each character, and convert the hex a-f characters to 10-15
for i in range(len(hex_stuff)):
    if hex_stuff[i] == 'a':
        hex_stuff[i] = 10
    elif hex_stuff[i] == 'b':
        hex_stuff[i] = 11
    elif hex_stuff[i] == 'c':
        hex_stuff[i] = 12
    elif hex_stuff[i] == 'd':
        hex_stuff[i] = 13
    elif hex_stuff[i] == 'e':
        hex_stuff[i] = 14
    elif hex_stuff[i] == 'f':
        hex_stuff[i] = 15

    # To make the program actually audible, 100 hertz is added from the beginning,
    # then the number is multiplied by 500 hertz
    # Plus a cheeky random amount of noise
    hex_stuff[i] = 1000 + int(hex_stuff[i]) * 500 + (10 * random())


def sound_generation(name, rand_hex):
    # The hex array is converted to a 16 bit integer array
    scaled = np.int16(np.array(hex_stuff))
    # Sci Pi then writes the numpy array into a wav file
    write(name, len(hex_stuff), scaled)
    randomness = rand_hex


# Pump up the music!
# print("Generating main.wav...")
# sound_generation('main.wav')
# print("Generation complete!")

# Your ears have been blessed# picoCTF{mU21C_1s_1337_5db6b85e}
```

The output is the Python source code that *generated* `main.wav` — the encoder. And at the very end, the flag is the last comment in the source:

`picoCTF{mU21C_1s_1337_5db6b85e}`

The hash `5db6b85e` is per-instance, so the suffix will be different if you spin up a fresh challenge. The prefix is the same: `picoCTF{mU21C_1s_1337_...}`.

---

## What Happened Internally

A short timeline of what the challenge author did, and what I did to reverse it.

1. **The author wrote a Python script** (`generate_wav.py`) that reads its own source file, hex-encodes it, then converts each hex character (0-9 and a-f) to one of 16 discrete levels (10, 15, 20, 25, ..., 85) and adds a small random offset to disguise the pattern.
2. **The script writes the result to a 16-bit mono WAV at 2736 Hz.** The sample rate is irrelevant — it just has to be fast enough to hold all the hex characters. The script's own source is 1368 hex chars (684 bytes), so 1368 samples are needed, plus a few hundred for the trailing comment. 2736 is the closest round number that fits.
3. **The file is `main.wav`.** When opened in any audio player, it sounds like a sustained, buzzy tone (because all the samples cluster in the 1000-8500 range, which is well above the "center" of a 16-bit signed sample at 0). The author added the cheeky comment `# Your ears have been blessed` to make fun of anyone who actually tries to listen to it.
4. **I opened the file with `scipy.io.wavfile.read`** to get the raw integer samples. The first six values (2002, 2507, 2001, 1508, 2006, 8500) immediately looked suspicious — too clean to be real audio.
5. **I truncated to two digits and found 16 unique values.** The number 16 was the dead giveaway: that is the size of the hex alphabet. 16 levels → 16 hex digits → a hex string → ASCII.
6. **I mapped each level to its rank in the sorted list, got a hex string of 2736 characters, and decoded it as ASCII.** The first character was `#` (0x23), so the result was clearly a Python source file, not the flag itself.
7. **I read the decoded source.** It was a self-referential encoder — a script that contains itself. The flag was the last line of the source: a comment. I extracted it with a regex.

The "very deep below the surface" hint was literal: the data is not in the audio (surface) but in the *integer values* of the samples (deep underneath). The audio is the camouflage.

---

## Tools Used

| Tool | Why I used it |
|------|---------------|
| `scipy.io.wavfile.read` | Get raw 16-bit integer samples without any audio interpretation. |
| Python `set()` and `sorted()` | Find the 16 unique quantized levels. |
| `bytes.fromhex(hex_str).decode()` | Decode the assembled hex string as ASCII. |
| `re.search(r'picoCTF\{...\}', decoded)` | Extract the flag from the comment at the end of the decoded source. |

Tools I tried that did **not** help, so you do not waste time on them:
- `sox main.wav -n spectrogram` — the spectrogram is just a smooth blob of bright energy in the lower frequencies. No text, no image, no hidden message in the spectrum.
- `Audacity Spectrogram view` — same. The challenge is not a spectrogram image.
- `strings main.wav` — the flag is hex-encoded, so the literal string `picoCTF{` does not appear anywhere in the file.
- LSB of the audio samples — every sample's low byte is essentially random noise (the `10 * random()` term in the encoder), so the LSB carries no data.
- `play main.wav` (or any audio player) — sounds like a sustained buzz, no Morse code, no DTMF, no speech. The audio is camouflage, not the payload.

---

## Key Takeaways

- A `.wav` file is just an array of integers. The "audio" interpretation is one possible rendering; you can read the samples as any data you want. Whenever an audio challenge looks like noise, skip the audio tools and go straight to the integer values.
- A small, fixed number of distinct sample levels is a giant red flag that the audio is *quantized data*, not real audio. Real audio uses the full dynamic range; encoding schemes use a small alphabet (hex digits here, but it could be 8 levels for an octal encoding, 26 for letters, etc.).
- The challenge is a *self-referential encoder* — the script that produces `main.wav` is itself the content of `main.wav`. This is a fun puzzle but also a real anti-forensics technique: if someone finds the file and reverses it, they get a Python script that documents exactly how the encoding works, with no separate "decoder" to write.
- The flag wordplay is `mU21C_1s_1337` = "MUSIC IS 1337" in leet speak. `1337 = leet`, and the leet transformation of "music" is `mU21C` (U=u, 2=Z→S or just stylization, 1=I, C=C). The author is making a meta-comment that "music" (audio) is the leet (1337) way to smuggle data — true for this challenge, true for steganography in general.
- Practical fix for code like this encoder: if you want to hide data in audio, do not quantize to visible levels. Use the LSB of the *real audio* and the data — that way the audio sounds unchanged and the quantization does not produce nice round numbers like 1000 + nibble * 500. The encoder here was very obvious because the levels were exact multiples of 500.
- The "deep below the surface" hint is a *great* hint because it tells you to skip the obvious (audio) and go to the underlying (integer samples). On a real forensics case, the equivalent move is: skip the user-facing rendering, look at the raw data bytes.
- The PicoMini 2021 challenge "scambled-bytes" used a similar trick: take integer-looking data, realize it is a cipher, reverse the cipher. The pattern is recurring: picoCTF loves "look at the raw values" challenges.
