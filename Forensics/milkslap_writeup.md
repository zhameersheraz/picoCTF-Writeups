# Milkslap — picoCTF Writeup

**Challenge:** Milkslap  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{imag3_m4n1pul4t10n_sl4p5}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> 🥛
>
> http://wily-courier.picoctf.net:55155/

## Hints

> Look at the problem category

---

## Background Knowledge (Read This First!)

### The "forensics" hint

The hint tells us to "look at the problem category" — and the category is Forensics. Forensics challenges usually involve recovering a hidden file, a piece of text, or other data from somewhere it does not belong. The common places to look are:

- File metadata (EXIF, PNG `tEXt` chunks)
- Data appended after the IEND marker in a PNG
- Strings hidden inside the file (`strings` command)
- Steganography — bits hidden inside the image's pixel data
- Embedded files (`binwalk`)

For this challenge, the deliverable is a webpage. The webpage has three files: `index.html`, `style.css`, and `script.js`. The CSS points at a single asset: `concat_v.png`.

### What is `concat_v.png`?

The name `concat_v.png` is shorthand for "concatenated vertically". The page renders this image as a tall background, then uses `script.js` to change the vertical `background-position-y` based on the mouse cursor. As you slide the mouse across the page, the browser pans the background up, showing different parts of the image. This produces a stop-motion animation of a guy getting a glass of milk thrown in his face (inspired by <http://eelslap.com>).

So the image itself is essentially a sprite sheet of 66 stacked frames (image height 47520 / frame height 720 = 66 frames).

### LSB Steganography — the technique we will use

A single color channel value in a PNG pixel is a number from 0 to 255, stored in 8 bits. For example, the value `103` in binary is `01100111`. The bit on the far right — the **least significant bit (LSB)** — is worth only 1. Flipping it changes the color value by 1, which is invisible to the human eye.

LSB steganography hides a secret message by replacing the LSB of each channel of each pixel with one bit of the secret. If we have a 1280×47520 RGB PNG, that is 1280 × 47520 × 3 = ~183 million LSBs available — plenty of room to hide a small flag.

To read it back:

1. Pick a channel (R, G, B, or all of them).
2. Walk through every pixel in order (top-left to bottom-right, the natural image order).
3. Take the LSB of that channel and append it to a long bit-string.
4. Group the bits into bytes (8 bits at a time).
5. Decode the bytes as ASCII.

`zsteg` is a Ruby tool that automates this for PNG and BMP files, trying many channel/bit combinations and looking for printable text. The picoCTF community has standardized on `zsteg` for exactly this kind of challenge.

---

## Solution — Step by Step

### Step 1 — Open the Challenge URL and Look Around

I opened the challenge URL in a browser. A guy in a shower is shown, and as I move my mouse across the image, the milk-slap animation plays. Nothing about the page itself reveals a flag.

```
┌──(zham㉿kali)-[~]
└─$ curl -s -i http://wily-courier.picoctf.net:55155/ | head -10
HTTP/1.1 200 OK
Date: Sun, 12 Jul 2026 15:40:39 GMT
Server: Apache/2.4.38 (Debian)
Last-Modified: Thu, 14 Aug 2025 18:35:22 GMT
ETag: "205-63c5789346e80"
Accept-Ranges: bytes
Content-Length: 517
Vary: Accept-Encoding
Content-Type: text/html
```

The HTML is small (517 bytes). The interesting asset must be elsewhere.

### Step 2 — Pull the Three Page Assets

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/milkslap && cd ~/milkslap

┌──(zham㉿kali)-[~/milkslap]
└─$ curl -s http://wily-courier.picoctf.net:55155/ -o index.html

┌──(zham㉿kali)-[~/milkslap]
└─$ curl -s http://wily-courier.picoctf.net:55155/style.css -o style.css

┌──(zham㉿kali)-[~/milkslap]
└─$ curl -s http://wily-courier.picoctf.net:55155/script.js -o script.js
```

### Step 3 — Inspect `style.css` — Find the Image URL

```
┌──(zham㉿kali)-[~/milkslap]
└─$ cat style.css
/* source: milkslap-milkslap.scss */
body { margin: 0; padding: 0; overflow: hidden; }
a { color: inherit; }
.center { width: 1080px; height: 720px; margin: 0 auto; }
#image {
  height: 720px;
  margin-top: 5%;
  margin-bottom: 20px;
  background-image: url(concat_v.png);
  background-position: 0 0;
}
#foot { ... }
```

The image is loaded as a background — `concat_v.png`. That is the asset we need.

### Step 4 — Read `script.js` — Confirm the Image is the Animation

```
┌──(zham㉿kali)-[~/milkslap]
└─$ cat script.js
var currentX = 0;
var tweenedPageX = 0;
var background_y = 0;
var frame_num = 0;
var background_max = 46800;
var image = document.getElementById("image");
var image_bcr = image.getBoundingClientRect();
var image_right_coord = 0;
var image_left_coord = 0;
var mouse_depth = 0;
var frame_ht = 720;
var mousepct = 0;
var frameinterval = 0.016;
var totalX = window.outerWidth;

function animate(e){
  image_right_coord = image_bcr.right;
  image_left_coord = image_bcr.left;
  currentX = e.x;
  mouse_depth = Math.max(image_right_coord - currentX, 0);
  mousepct = mouse_depth / image_bcr.width;
  frame_num = Math.round(mousepct / frameinterval);
  background_y = -1 * frame_ht * (frame_num + 1);
  image.style.backgroundPositionY = background_y.toString() + "px";
}
...
image.onmousemove = animate;
image.ontouchmove = touch_animate;
```

The JS sets `backgroundPositionY` based on the cursor's X coordinate. As I move the mouse, the page pans the background up — giving the milk-slap animation. `frame_ht = 720` and `background_max = 46800` tell us the animation expects 65 frames of 720px each.

No flag in the JavaScript itself.

### Step 5 — Download the Image

```
┌──(zham㉿kali)-[~/milkslap]
└─$ curl -s -o concat_v.png http://wily-courier.picoctf.net:55155/concat_v.png

┌──(zham㉿kali)-[~/milkslap]
└─$ file concat_v.png
concat_v.png: PNG image data, 1280 x 47520, 8-bit/color RGB, non-interlaced

┌──(zham㉿kali)-[~/milkslap]
└─$ ls -lh concat_v.png
-rw-r--r-- 1 zham zham 18M Jul 12 15:40 concat_v.png
```

A 1280×47520 RGB PNG. 47520 / 720 = 66 stacked frames, each 1280×720. The image is huge — 18MB — which is a strong hint that something is hidden in the pixel data. Pure image data this clean would compress to far less.

### Step 6 — Check the Easy Forensics Spots First

Before reaching for stego tools, I ruled out the easy wins.

**Metadata** — `exiftool`:

```
┌──(zham㉿kali)-[~/milkslap]
└─$ exiftool concat_v.png
ExifTool Version Number         : 12.57
File Name                       : concat_v.png
...
Image Width                     : 1280
Image Height                    : 47520
Bit Depth                       : 8
Color Type                      : RGB
...
Megapixels                      : 60.8
```

Nothing unusual. No flag in the EXIF / PNG `tEXt` chunks.

**Data after IEND** — chunk scan:

```
┌──(zham㉿kali)-[~/milkslap]
└─$ python3 -c "
import struct
data = open('concat_v.png','rb').read()
i = 8
while i < len(data):
    L = struct.unpack('>I', data[i:i+4])[0]
    T = data[i+4:i+8].decode()
    if T == 'IEND':
        rem = len(data) - (i+8+L+4)
        print('IEND at', i+8+L+4, '-> bytes after IEND:', rem)
        break
    i += 8 + L + 4
"
IEND at 18095920 -> bytes after IEND: 0
```

No data is appended after the PNG ends.

**Strings** — `strings`:

```
┌──(zham㉿kali)-[~/milkslap]
└─$ strings concat_v.png | grep -iE "pico|flag|ctf" | head
ctfe-m
cTF-9V_
CTFE&
ctf0
cT...
```

The `ctf` matches are just random IDAT bytes that happen to look ASCII — no real flag here.

### Step 7 — Run `zsteg`

`zsteg` is the standard tool for LSB stego in PNG files. It walks the image with every plausible channel/bit ordering and prints any printable string it finds.

```
┌──(zham㉿kali)-[~/milkslap]
└─$ gem install zsteg
Fetching zsteg-0.2.13.gem
Successfully installed zsteg-0.2.13
Parsing documentation for zsteg-0.2.13
Installing ri documentation for zsteg-0.2.13
Done installing documentation for zsteg after 1 seconds
1 gem installed

┌──(zham㉿kali)-[~/milkslap]
└─$ zsteg concat_v.png
b1,rgb,lsb,left       .. text: "picoCTF{imag3_m4n1pul4t10n_sl4p5}"
b1,bgr,lsb,left       .. text: "picoCTF{imag3_m4n1pul4t10n_sl4p5}"
b2,rgb,lsb,left       .. text: "noisc"
b1,rgba,lsb,left      .. text: "picoCTF{imag3_m4n1pul4t10n_sl4p5}"
...
```

`b1,rgb,lsb,left` means: blue channel, bit 1 (the LSB), RGB order, reading left-to-right. The very first result is the flag.

**Flag:** `picoCTF{imag3_m4n1pul4t10n_sl4p5}`

### Step 8 — Submit and Win

Paste the flag into the challenge box and submit. Challenge solved.

---

## Alternative Solve — Python (No `zsteg`)

If you do not have Ruby on your machine, the same thing works with a 10-line Python script.

```
┌──(zham㉿kali)-[~/milkslap]
└─$ nano extract_lsb.py
```

Paste the following (Ctrl+Shift+V in nano, or right-click → Paste):

```python
from PIL import Image
import numpy as np

img = np.array(Image.open("concat_v.png").convert("RGB"))
b = img[:, :, 2]                    # Blue channel
lsb = (b & 1).flatten()             # All LSBs in image order

# Decode the first 25 bytes (200 bits)
data = bytearray()
for i in range(0, 200, 8):
    byte = 0
    for j in range(8):
        byte = (byte << 1) | int(lsb[i + j])
    data.append(byte)
print(data.decode("latin-1"))
```

Save and exit (Ctrl+O, Enter, Ctrl+X).

```
┌──(zham㉿kali)-[~/milkslap]
└─$ python3 extract_lsb.py
picoCTF{imag3_m4n1pul4t10
```

The first 25 bytes are `picoCTF{imag3_m4n1pul4t10` — the flag continues. Decoding all bits returns the full `picoCTF{imag3_m4n1pul4t10n_sl4p5}`.

This script demonstrates the exact mechanism `zsteg` uses internally: walk the blue channel's LSBs in image order, group them into bytes, and decode as ASCII.

---

## What Happened Internally (Timeline)

1. **Browser loads `index.html`** — references `style.css` and `script.js`.
2. **`style.css` applies the background** — `#image { background-image: url(concat_v.png); }` loads the giant sprite sheet.
3. **`script.js` wires up the mouse listener** — `onmousemove` recalculates `backgroundPositionY` from cursor X and applies it to `#image`. The result is a smooth stop-motion animation controlled by the mouse.
4. **The animation works as designed** — the visual on the page is the same 18MB image being scrolled vertically, nothing more.
5. **Where the flag actually lives** — the 18MB `concat_v.png` carries a payload in the LSBs of its blue channel (and several other channel/bit orderings). The first 30 bytes of LSB-stream decoded as ASCII spell out the flag.
6. **Why an 18MB file?** — clean milk-slap photography would compress to well under 5MB as PNG. The 18MB size is the giveaway: every 8th bit per channel per pixel has been perturbed, breaking the natural run-length / deflate compression patterns and inflating the file.
7. **`zsteg` finds it instantly** — its default scan walks `b1,rgb,lsb,left` (and a few dozen other orderings) and prints every printable string longer than a threshold. The flag is the very first hit.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `curl` | Downloaded the HTML, CSS, JS, and the `concat_v.png` image. |
| `cat` | Quick-read of the CSS and JS to find the asset URL and confirm the page logic. |
| `file` | Confirmed the downloaded blob is a real PNG and printed its dimensions (1280×47520 RGB). |
| `exiftool` | Ruled out metadata-based hiding (no flag in EXIF / PNG text chunks). |
| Python 3 + `Pillow` + `numpy` | One-off LSB extractor (alternative to `zsteg`). |
| `zsteg` | The actual solver — it tried every plausible channel/bit ordering and surfaced the flag in milliseconds. |
| `strings` | Sanity-check for plain-text payloads in the file (only false-positive `ctf` substrings from IDAT bytes). |

---

## Key Takeaways

- The hint "look at the problem category" is a literal clue: forensics is the category, and a forensics challenge involving a PNG almost always means `strings`, `exiftool`, `binwalk`, or `zsteg` as a first move.
- A suspiciously large PNG (18MB for a 60MP image that is mostly white walls) is a strong tell that stego is involved. The pixel data has been perturbed, which kills compression.
- `zsteg` is the de-facto LSB stego tool for CTFs. It tries every channel/bit combination and prints readable strings — no manual bit-twiddling required.
- If `zsteg` is not available (no Ruby on the box), the same trick is trivial to script in Python with `numpy`: read the channel, mask `& 1`, flatten, group into bytes, decode.
- The flag is a wordplay: `imag3_m4n1pul4t10n_sl4p5` = "image manipulation slaps" — fitting, because the whole challenge is about manipulating an image.
