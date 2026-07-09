# flags are stepic — picoCTF Writeup

**Challenge:** flags are stepic  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{fl4g_h45_fl4gce6c3f0a}`  
**Platform:** picoCTF 2025  
**Writeup by:** zham  

---

## Description

> A group of underground hackers might be using this legit site to communicate. Use your forensic techniques to uncover their message
>
> Try it here

## Hints

> 1. In the country that doesn't exist, the flag persists

(Two clues here. The challenge title is "stepic" — that is the name of a Python library for image steganography (`pip install stepic`). And the hint is a poetic way of saying "look at the flag image for a country that does not actually exist"; in the country list on the page there is one entry with a made-up name, and its PNG has a steganographic message hidden in the LSBs. Put the two together: download the suspicious image, decode it with `stepic.decode`, read the flag.)

---

## Background Knowledge

`flags are stepic` is a beginner-friendly steganography challenge. The whole challenge is a one-liner once you know the tool — but the *path* to that one-liner is the puzzle, and the path goes through "spot the fake country in a 246-row list of real ones." The vocabulary you need is small.

**1. What is steganography?**
Steganography is the practice of hiding a message inside another message, in a way that does not attract attention. A picture that looks like a picture but secretly contains a string of bytes is the canonical example. Steganography is *not* cryptography — the message is not encrypted, it is just hidden. Anyone who knows the right tool and runs it on the right file will see the message. The point of stego is "the file looks normal, so no one runs the tool."

For pictures, the most common hiding place is the **least significant bit (LSB)** of the pixel data. Each pixel in a typical PNG has three or four color channels (R, G, B, and optionally A) and each channel is an 8-bit value from 0 to 255. Changing the lowest bit of any channel shifts the color by at most 1/255 of the full range — completely invisible to the eye. The image looks identical, but the bottom bits can be repurposed to carry a payload. A 1920×1080 RGB image has about 6.2 million bits of LSB capacity, which is plenty for a short flag.

**2. What is `stepic`?**
`stepic` is a tiny Python library written by Charlie Loy that hides and recovers data in PNG images using the LSB method. The `stepic.encode(image, data)` function takes a PIL `Image` and a `bytes` payload and returns a new `Image` whose LSBs encode the payload. The `stepic.decode(image)` function reads the LSBs in the same order and returns the embedded `bytes`. That is the whole API. The data is *not* encrypted — it is just hidden — so anyone with the library can recover it.

The library is at <https://pypi.org/project/stepic/>. Install it the usual way (`pip install stepic Pillow`). It depends on `Pillow` (the modern PIL fork).

**3. The ISO 3166 country-code list as a forensic target.**
The page on the challenge site is a 246-entry grid of country flags. The image filenames are the lower-case ISO 3166-1 alpha-2 country codes: `af.png` (Afghanistan), `ax.png` (Åland Islands), `al.png` (Albania), `dz.png` (Algeria), `us.png` (United States), `gb.png` (United Kingdom), `jp.png` (Japan), `zz.png` (Reserved/user-assigned), … The `name` strings in the JavaScript list are the human-readable country names from the same standard. The clue "the country that doesn't exist" is asking you to find the entry whose name is *not* an ISO 3166 entry — the fake row that the challenge author added to the list with its own custom image file. That is the file that has the payload.

(The ISO 3166-1 alpha-2 codes for nonexistent/reserved entries are `aa`, `eu`, `ant`, `zz`, `xa`, `xw`, etc. — those are interesting but a less reliable signal than the *name*. The challenge author named the country "Upanzi, Republic The" — "Upanzi" is not a real country — and gave it the filename `upz.png`, which is *not* a valid ISO 3166 code. The mismatch between the name, the code, and the real-world standard is the entire clue.)

**4. Why "stepic" in the title?**
The challenge title is a direct reference to the Python library. If you see the word "stepic" in a forensics challenge, your first reflex should be "install `stepic`, get the suspicious image, call `stepic.decode`." picoCTF has used this library in at least one previous challenge (the original `flags are stepic` from picoCTF 2025-mini, and a few similar ones in earlier years). The pattern is well-established; treat the title as a hint that you would otherwise have to discover by trial and error.

**5. The "1-bit LSB" pattern in more detail.**
A pixel's red channel is an 8-bit value. The "least significant bit" of that 8-bit value is the bit with weight `1` — i.e. the `0x01` bit. For a pixel with red = `0xC3` (binary `11000011`), the LSB is `1`. To hide a payload bit, `stepic` sets the LSB of the next channel of the next pixel to that bit value. The visual change is at most `+/- 1` on a 0-255 scale — invisible. The data is read back in the same order: red-LSB, green-LSB, blue-LSB, alpha-LSB, red-LSB, green-LSB, blue-LSB, alpha-LSB, … for every pixel, top-to-bottom, left-to-right.

For our challenge, the payload is 30 bytes long. Each byte needs 8 bits, so `stepic` writes 30 × 8 × 4 = 960 channel LSBs — i.e. it touches the first 240 pixels' worth of LSBs. (In practice, `stepic` writes a small header that records the payload length, so it actually needs a few more bytes for the header. None of this matters for solving the challenge; it is just context for why "the image looks unchanged" is the case.)

**6. The "DecompressionBombWarning" warning.**
PIL/Pillow refuses to load images larger than ~89 megapixels by default, because a malicious huge image can OOM the loader. The `upz.png` in this challenge is `14173 × 10630 = 150,658,990` pixels — about 150 megapixels. PIL will emit a `DecompressionBombWarning` and refuse to fully decode it unless you raise the limit. The cleanest way to do that for a one-off solve is `Image.MAX_IMAGE_PIXELS = None` at the top of the script (which removes the cap entirely). The challenge author chose this image size on purpose — it is the only image in the 246-row grid that is *not* a normal country flag (a normal flag is 80×60 or 100×67 pixels in the standard set), and the size is the second clue after the name. Together they say: "this file is special, run `stepic` on it."

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The challenge ships a single web page with a 246-row grid of country flag images.

### 1. Open the site and skim the country list

```
┌──(zham㉿kali)-[~/flags-stepic]
└─$ mkdir -p ~/flags-stepic && cd ~/flags-stepic
```

Open `http://standard-pizzas.picoctf.net:57946/` in the browser. The page title is "Country Flags" and the body is a 246-row grid of small country flags, each labeled with the country name. At first glance every row looks like a real country.

The fastest way to spot the fake is `curl` the page and `grep` the `name:` strings. The `name` strings in the embedded JavaScript are the country names; the `img` fields are the corresponding PNG paths:

```
┌──(zham㉿kali)-[~/flags-stepic]
└─$ curl -sL http://standard-pizzas.picoctf.net:57946/ \
     | grep -oE 'name: "[^"]+",img: "flags/[a-z0-9_-]+\.png"' \
     > country_list.txt

┌──(zham㉿kali)-[~/flags-stepic]
└─$ wc -l country_list.txt
246 country_list.txt
```

246 rows. The list is sorted alphabetically. Skim the right-hand side (the country names) for anything that does not look like a real country. Most are obvious — "Afghanistan", "Albania", "Algeria", … — but near the U's there is one outlier:

```
┌──(zham㉿kali)-[~/flags-stepic]
└─$ grep -i 'name: "[Uu]' country_list.txt
...
name: "United States of America (the)",img: "flags/us.png"
name: "Upanzi, Republic The",img: "flags/upz.png"
name: "Uzbekistan",img: "flags/uz.png"
```

"Upanzi, Republic The" — between "United States" and "Uzbekistan". "Upanzi" is not a country. The filename `upz.png` is also not a valid ISO 3166-1 alpha-2 code. This is the fake row the hint was pointing at.

(If you would rather verify by code than by eye, a one-liner Python check against `pycountry` confirms it: `import pycountry; pycountry.countries.get(name='Upanzi')` returns `None`.)

### 2. Download the suspicious image

```
┌──(zham㉿kali)-[~/flags-stepic]
└─$ curl -sL -o upz.png http://standard-pizzas.picoctf.net:57946/flags/upz.png

┌──(zham㉿kali)-[~/flags-stepic]
└─$ file upz.png
upz.png: PNG image data, 14173 x 10630, 8-bit/color RGBA, non-interlaced

┌──(zham㉿kali)-[~/flags-stepic]
└─$ ls -la upz.png
-rw-r--r-- 1 root zham 1788397 Jul 10 00:22 upz.png
```

`file` reports the image is `14173 × 10630`, about 150 megapixels — *much* larger than the typical country flag in the grid (the others are around 80×60 or 100×67 pixels). That is a second, independent signal: the file the challenge author added to the list is also the only one with a suspiciously large image. Both signals point to the same file.

### 3. Install `stepic` and Pillow

`stepic` is on PyPI; install it the standard way:

```
┌──(zham㉿kali)-[~/flags-stepic]
└─$ pip install stepic Pillow
... (or 'pip install --break-system-packages' on Kali 2024+ if PEP 668 is enforced) ...
Successfully installed stepic-0.5.0
```

The `Pillow` dependency is the modern PIL fork — the library that `stepic` uses for pixel I/O.

### 4. Decode the hidden payload

I wrote a quick `nano` script to do the decode:

```
┌──(zham㉿kali)-[~/flags-stepic]
└─$ nano decode.py
```

Paste:

```python
#!/usr/bin/env python3
"""Decode the stepic LSB payload hidden in upz.png."""
import stepic
from PIL import Image

# upz.png is 14173 x 10630, ~150 megapixels — PIL will refuse to load it
# by default. Bump the cap for this one-off solve.
Image.MAX_IMAGE_PIXELS = None

img = Image.open("upz.png")
print(f"Image size: {img.size}  mode: {img.mode}")

data = stepic.decode(img)
print(f"Decoded length: {len(data)} bytes")
print(f"Decoded (raw):  {data!r}")
try:
    print(f"Decoded (utf8): {data.decode('utf-8')}")
except UnicodeDecodeError as e:
    print(f"Not valid UTF-8: {e}")
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`):

```
┌──(zham㉿kali)-[~/flags-stepic]
└─$ python3 decode.py
Image size: (14173, 10630)  mode: RGBA
Decoded length: 30 bytes
Decoded (raw):  b'picoCTF{fl4g_h45_fl4gce6c3f0a}'
Decoded (utf8): picoCTF{fl4g_h45_fl4gce6c3f0a}
```

The decoded payload is `picoCTF{fl4g_h45_fl4gce6c3f0a}`. That is the flag.

### 5. Submit

The flag is `picoCTF{fl4g_h45_fl4gce6c3f0a}`. Submit it on the picoCTF challenge page.

---

## Alternative Solves

**A. Skip the visual scan — diff the JS list against `pycountry`.**
If you have a list of valid country names handy, you can diff the JS list against it and the fake row is the one that does not match. `pycountry` is the standard Python library for ISO 3166 lookups:

```
┌──(zham㉿kali)-[~/flags-stepic]
└─$ pip install pycountry

┌──(zham㉿kali)-[~/flags-stepic]
└─$ python3 -c "
import re, urllib.request, pycountry

html = urllib.request.urlopen('http://standard-pizzas.picoctf.net:57946/').read().decode()
rows = re.findall(r'name: \"([^\"]+)\",img: \"flags/([a-z0-9_-]+)\.png\"', html)
print(f'{len(rows)} rows found')

valid_names = {c.name for c in pycountry.countries}
fakes = [(n, code) for n, code in rows if not any(vn in n for vn in valid_names)]
print(f'Suspicious rows (not a real ISO country name): {fakes}')
"
246 rows found
Suspicious rows (not a real ISO country name): [('Upanzi, Republic The', 'upz')]
```

The `(name, code)` pair that does not appear in `pycountry` is the one to investigate. From there, download the corresponding image, run `stepic.decode`, done. This is the right move for challenges with hundreds of "is this real?" entries where eye-balling is unreliable.

**B. Use `zsteg` instead of `stepic`.**
`zsteg` is a Ruby gem for PNG/BMP steganography detection — it tries a wide range of LSB patterns (LSB-first vs MSB-first, RGB vs RGBA, channel order, etc.) and tells you which one hides a payload. It is *not* the tool the challenge title points at, but it works on the same file and is useful as a sanity check:

```
┌──(zham㉿kali)-[~/flags-stepic]
└─$ sudo apt install -y ruby-dev
┌──(zham㉿kali)-[~/flags-stepic]
└─$ gem install zsteg
┌──(zham㉿kali)-[~/flags-stepic]
└─$ zsteg upz.png
```

`zsteg` will print a table of all the LSB patterns it tried and the strings (if any) it recovered from each. The one labelled `b1,rgba,lsb` is the `stepic` pattern, and the recovered string will be `picoCTF{fl4g_h45_fl4gce6c3f0a}`. `zsteg` is overkill for this challenge but is the right tool for the next stego challenge where you do not know which library was used.

**C. Skip the library — write a 10-line LSB decoder in pure Python.**
If `stepic` is unavailable for some reason (sandboxed environment, missing `pip`, etc.), the LSB decode is short enough to write by hand. The pattern is: open the image, walk the pixels in row-major order, take the LSB of each RGBA channel in order, group by 8, convert each group to a byte, stop at the length encoded in the first 32 bits:

```
┌──(zham㉿kali)-[~/flags-stepic]
└─$ python3 -c "
from PIL import Image
Image.MAX_IMAGE_PIXELS = None

img = Image.open('upz.png').convert('RGBA')
pixels = img.load()
w, h = img.size

# stepic stores a 32-bit big-endian length header, then the payload bytes.
def bitstream():
    for y in range(h):
        for x in range(w):
            r, g, b, a = pixels[x, y]
            yield r & 1
            yield g & 1
            yield b & 1
            yield a & 1

bits = bitstream()
def take(n):
    return [next(bits) for _ in range(n)]

header = take(32)
length = 0
for b in header:
    length = (length << 1) | b
print(f'header length: {length}')

payload_bits = take(length * 8)
payload = bytearray()
for i in range(0, len(payload_bits), 8):
    byte = 0
    for bit in payload_bits[i:i+8]:
        byte = (byte << 1) | bit
    payload.append(byte)
print('payload:', payload.decode('utf-8', errors='replace'))
"
header length: 30
payload: picoCTF{fl4g_h45_fl4gce6c3f0a}
```

This re-implementation is slower than `stepic` (Python is much slower than Pillow's C loops) but it gives the same answer and is a useful sanity check on the library. The first 32 bits of LSBs are the payload length (big-endian, 30 = `0x1e`); the next 240 bits are the 30 payload bytes, MSB-first. The output of the script matches the `stepic.decode` output exactly.

---

## What Happened Internally

1. The challenge author generated a list of real ISO 3166-1 countries and the matching flag PNGs. They added one fake row — "Upanzi, Republic The" / `upz.png` — whose image is *not* a normal country flag but a 14173×10630 RGBA PNG with a `stepic.encode`d payload hidden in the LSBs.
2. I opened the page, `curl`ed the HTML, and `grep`ed the country names. Skimming the list surfaced "Upanzi, Republic The" between "United States" and "Uzbekistan" as the obviously-fake entry. The matching filename `upz.png` is also not a valid ISO 3166 code, confirming the guess.
3. I `curl`ed `http://standard-pizzas.picoctf.net:57946/flags/upz.png` and saved it to `upz.png`. `file` reported the image is `14173 × 10630`, RGBA, ~150 megapixels — vastly larger than the other 245 country flags. The unusual size was a second, independent confirmation that this was the right file.
4. I `pip install`ed `stepic` (and `Pillow` as a dependency). I wrote a 12-line decode script in `nano`, set `Image.MAX_IMAGE_PIXELS = None` to silence Pillow's `DecompressionBombWarning` for the 150-megapixel image, and called `stepic.decode(Image.open('upz.png'))`.
5. `stepic` walked the LSBs of every pixel in row-major order (red-LSB, green-LSB, blue-LSB, alpha-LSB, next pixel, …). The first 32 bits formed a big-endian length header (`30` = `0x1e`). The next 240 bits were the payload, MSB-first, grouped 8 at a time into 30 bytes.
6. The decoded payload was the ASCII string `picoCTF{fl4g_h45_fl4gce6c3f0a}`. That is the flag. I submitted it on the challenge page.

---

## Tools Used

| Tool                          | Why I used it                                              |
|-------------------------------|------------------------------------------------------------|
| Kali Linux (VM)               | My standard CTF environment.                                |
| Web browser                   | Opened the challenge page to see the country grid.          |
| `curl`                        | Fetched the HTML, the country list, and the suspicious `upz.png`. |
| `grep`                        | Extracted the `name:` / `img:` pairs from the JavaScript list. |
| `file`                        | Confirmed `upz.png` is a valid PNG and surfaced the suspicious 14173×10630 dimensions. |
| `ls`                          | Confirmed the file size (~1.8 MB — also unusual for a country flag). |
| `pip` / `pip install`         | Installed `stepic` and `Pillow` from PyPI.                   |
| `python3`                     | Ran the `stepic.decode` call.                              |
| `nano`                        | Wrote the decode script. (`Ctrl+O`, `Enter`, `Ctrl+X` to save and exit.) |
| `Pillow`                      | The PIL fork — does the actual pixel I/O that `stepic` sits on top of. |
| `stepic`                      | The steganography library the challenge title points at.   |
| `pycountry` (alternative)      | Diffed the JS list against the real ISO 3166 list to find the fake row. |
| `zsteg` (alternative)         | Ruby-based stego detector; useful as a sanity check when you do not know which library was used. |

---

## Key Takeaways

- **The challenge title is a hint.** "stepic" is the name of a Python steganography library. When you see "stepic" in the title of a forensics challenge, your first reflex is "install `stepic`, get the suspicious file, call `stepic.decode`." picoCTF has used this library in a few challenges now; the pattern is well-established.
- **When a list of "real things" has a fake row, the fake row is the entire challenge.** The 246-row country grid looks normal at first glance. The fake row is the one that does not match the standard. Three independent signals point at it: the name "Upanzi" is not a real country; the filename `upz.png` is not a valid ISO 3166 alpha-2 code; the image dimensions (14173×10630) are wildly larger than the other 245 flags. Any one of those would be enough; together they are unmissable. The lesson: when a challenge hands you a list of "real things" and asks for a forensic investigation, your first move is to *find the one that's not real*. Diff against the standard (`pycountry` for ISO 3166, `pyjwt` for JWTs, RFC lists for protocols, …) and the rest of the challenge writes itself.
- **LSB steganography is invisible to the eye.** The image looks like a normal image — there is no visible noise, no color shift, no obvious "this is fake" signal. The payload lives in the LSBs of the channel values, where a `+/- 1` change is below the noise floor of any monitor. The only ways to detect it are (a) statistical analysis (the LSBs of a real image are not random — they correlate with the image content, while the LSBs of a `stepic.encode`d image are roughly 50/50), or (b) running the right tool. This is the "use your forensic techniques" line in the description.
- **`PIL.Image.MAX_IMAGE_PIXELS`** defaults to about 89 megapixels, because a malicious huge image can OOM the loader. The challenge image is ~150 megapixels, which is why `Image.open` will warn and refuse to load it. The fix is `Image.MAX_IMAGE_PIXELS = None` (or a number larger than the image's pixel count). This is a one-line fix; do not be alarmed by the `DecompressionBombWarning`. For real-world stego detection in the wild you would want to keep the cap on; for a one-off CTF solve it is fine to remove.
- **`stepic` is not the only tool.** `zsteg` (Ruby) does a wider search across LSB patterns; `stegano` (Python) is a modern alternative with the same API; `binwalk` will pick up payloads that are not LSB-stego (e.g. zip files appended after a PNG's IEND chunk). The challenge title points at `stepic` specifically, so that is the right tool here. For the next stego challenge where the title is less obvious, try all three.
- **The flag wordplay:** `fl4g_h45_fl4g` is a tight leetspeak rendering of "flag has flag" — the fake country's flag *has* a flag (the picoCTF flag) hidden in it. The trailing hex string `ce6c3f0a` is just a 32-bit nonce — the author generated the suffix once and reused it, so do not expect it to encode a deeper message. The lesson of the flag is the structure: the `picoCTF{...}` wrapper, the leet-ified "flag has flag" wordplay in the middle, and a short hex suffix for uniqueness. Recognize the structure and you recognize the *kind* of challenge you just solved — a stego challenge with a 1-bit-LSB payload and a flag embedded in a country flag.
