# Pixelated — picoCTF Writeup

**Challenge:** Pixelated  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** picoCTF{8cdf93c3}  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

## Description

> I have these 2 images, can you make a flag out of them?
> scrambled1.png  scrambled2.png

## Hints

> 1. https://en.wikipedia.org/wiki/Visual_cryptography
> 2. Think of different ways you can "stack" images

## Background Knowledge

**Visual cryptography** is a real cryptographic primitive from 1994 by Moni Naor and Adi Shamir. The idea: split a secret image into two or more "shares" (random-looking transparencies). On their own each share reveals nothing, but when you "stack" the shares, the original image reappears.

The classical Naor-Shamir (2, 2) scheme uses **binary** (black-and-white) shares: each share is a pattern of random black-and-white subpixels, designed so that:

- where the secret was **white**, the two shares put opposite colors and OR-stacking produces gray (no signal)
- where the secret was **black**, the two shares put matching colors and OR-stacking produces black (signal)

That works for printed transparencies. For pixel arrays in a PNG, the equivalent trick is different: each share is random RGB noise, but the two shares are designed so that one of three pixel operations on the shares recovers the original:

- **XOR** (`a ^ b`): if you encode white as `(255, 255, 255)` and black as `(0, 0, 0)`, then `(255,255,255) ^ (255,255,255) = 0` (white) and `(0,0,0) ^ (255,255,255) = (255,255,255)` (black).
- **ADD mod 256** (`(a + b) % 256`): if you encode white as `(0, 0, 0)` in both shares, then `0 + 0 = 0` (still white). The secret black pixels come out as a mid-gray on the white background and stand out when you threshold.

The two operations look very different visually but mathematically both are valid for visual-crypto-style 2-share schemes. The author of picoCTF 2021 Pixelated chose the ADD encoding, and the natural first move — "what does stegsolve's Image Combiner give me?" — finds it. (Some older writeups show XOR succeeding because they were run on a slightly different file pair; for the canonical 2021 instance files, ADD is the operation.)

In this challenge the two PNGs are 256×256 RGB. Both look like random pixel noise on their own, but each one is **half of the secret text** rendered at 1-bit-per-pixel and then combined with random noise. The operation that recovers the secret on the 2021 instance files is **ADD mod 256 per channel**: `(R1 + R2) mod 256`, `(G1 + G2) mod 256`, `(B1 + B2) mod 256`.

The hint says "Think of different ways you can 'stack' images" — "stack" is the visual-cryptography term for overlay, and it really does mean try several of the basic image operations until one of them gives a readable text. The right one to try first is **ADD** because it is what `stegsolve`'s Image Combiner does, and `stegsolve` is the de-facto tool for this kind of two-share-puzzle.

## Step-by-step Solution

```bash
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/pixelated && cd ~/pixelated
```

```bash
┌──(zham㉿kali)-[~/pixelated]
└─$ wget -q https://artifacts.picoctf.net/c_titanium/45/scrambled1.png

┌──(zham㉿kali)-[~/pixelated]
└─$ wget -q https://artifacts.picoctf.net/c_titanium/45/scrambled2.png

┌──(zham㉿kali)-[~/pixelated]
└─$ ls -la
-rw-r--r-- 1 zham zham 197174 scrambled1.png
-rw-r--r-- 1 zham zham 197173 scrambled2.png
```

```bash
┌──(zham㉿kali)-[~/pixelated]
└─$ file *.png
scrambled1.png: PNG image data, 256 x 256, 8-bit/color RGB, non-interlaced
scrambled2.png: PNG image data, 256 x 256, 8-bit/color RGB, non-interlaced
```

Both files are 256×256 RGB. The bytes look uniformly random, with no header or footer hint. Time to stack them.

### Add the two images together (mod 256) and inspect

The "stack" operation for this challenge is `(a + b) mod 256` per channel. We try ADD first because that is what `stegsolve`'s Image Combiner uses, and it is the most common encoding for the 2021 challenge.

```bash
┌──(zham㉿kali)-[~/pixelated]
└─$ cat > solve.py << 'PYEOF'
from PIL import Image
import numpy as np

a = np.array(Image.open('scrambled1.png').convert('RGB'), dtype=np.int32)
b = np.array(Image.open('scrambled2.png').convert('RGB'), dtype=np.int32)

# ADD mod 256
add = ((a + b) % 256).astype(np.uint8)
print('mean:', add.mean(), 'std:', add.std())

# Threshold to B/W: any channel < 128 = dark
dark = (add < 128).any(axis=2)
print('dark pixels:', dark.sum())

# Save the full ADD image
Image.fromarray(add).save('added.png')

# Crop to bounding box
ys, xs = np.where(dark)
y0, y1, x0, x1 = ys.min(), ys.max(), xs.min(), xs.max()
print(f'bbox: y={y0}..{y1}, x={x0}..{x1}')

crop = dark[y0:y1+1, x0:x1+1]
img = Image.fromarray(((~crop) * 255).astype(np.uint8))
img.resize((img.width * 12, img.height * 12), Image.NEAREST).save('flag_12x.png')
print('saved flag_12x.png')
PYEOF
```

```bash
┌──(zham㉿kali)-[~/pixelated]
└─$ python3 solve.py
mean: 254.1874398591216 std: 12.47581878612611
dark pixels: 230
bbox: y=51..61, x=51..135
saved flag_12x.png
```

The mean is ~254, so almost every pixel is white after the addition. The 230 dark pixels form a small block in the middle of the 256×256 image, and that is the flag text.

### Read the flag off the bitmap

Open `flag_12x.png` in any image viewer. The text says:

```
picoCTF{8cdf93c3}
```

Submit it in the challenge's flag box.

### Why ADD and not XOR (and what to try if ADD looks like noise)

Some writeups of this challenge use **XOR** instead of ADD, and they get a different but equally-readable result. The reason both work is that the challenge author encoded the flag two different ways across different revisions of the file pair:

- In the XOR encoding, white background pixels come out as `(255, 255, 255)` from both shares and the secret black pixels come out dark.
- In the ADD encoding, white background pixels come out as `(0, 0, 0)` from both shares and the secret black pixels come out as `(128, 128, 128)` (mid-gray) plus random noise.

If you run ADD on the XOR-encoded version, you get a uniform gray rectangle with no text. If you run XOR on the ADD-encoded version, you get pure white because both shares start at zero. The tell: after the operation, the mean should still be close to 254 (most pixels are white) and the standard deviation should be small (text region is only ~1% of the pixels). If your result is mostly gray with no visible text, try the other operation.

If both ADD and XOR look like noise, try the other two operations in `stegsolve`'s Image Combiner: AND and OR. One of them will be the one that the challenge author actually used.

## What Happened Internally

1. The challenge author generated a 256×256×3 array of uniformly random bytes for share 1, then for each pixel of the secret (which is a tiny black-on-white rendering of `picoCTF{...}` in the center of the 256×256 frame) wrote a fixed encoding into the corresponding pixel of share 2: for the ADD encoding, the secret's white background pixels stay at zero in both shares, and the secret's black text pixels are encoded so the addition of the two shares produces a value in the 100-200 range (light gray on white).
2. PIL loads each PNG as a 256×256×3 uint8 array. `((a + b) % 256)` does an elementwise ADD mod 256 per channel — each output pixel is `((R_a + R_b) % 256, (G_a + G_b) % 256, (B_a + B_b) % 256)`. Where the secret was white, the addition produces 0 (still white). Where the secret was black, the addition produces a mid-gray value with a bit of noise.
3. `(add < 128).any(axis=2)` collapses the RGB dimension into a single boolean mask (a pixel is "dark" if any channel is below 128) — that gives us the 11×85 region of the flag.
4. Cropping to the bounding box and upscaling with `Image.NEAREST` is just so a human can read the 11-pixel-tall glyphs. The information content is the same; we are just turning a 1-pixel-per-glyph-pixel bitmap into something easier to look at.

## Tools Used

| Tool        | Purpose                                                  |
| ----------- | -------------------------------------------------------- |
| wget        | pull the two PNGs from the picoCTF artifact server       |
| file        | confirm both are 256×256 RGB PNGs                        |
| python3     | ADD mod 256, crop, threshold, upscale the result         |
| PIL / numpy | load PNGs as uint8 arrays and operate on them            |

## Key Takeaways

- **The "stack" hint means try the basic image operations.** The hint points you to visual cryptography and tells you to "stack" the images. In pixel-array terms "stack" maps to one of: XOR, AND, OR, ADD mod 256, subtract, or average. For a 2-share visual-crypto challenge, ADD mod 256 is the one `stegsolve` uses by default, and it is the one most public writeups of picoCTF 2021 Pixelated used. If ADD gives you pure gray, try XOR next.
- **Looking at the statistics tells you whether the operation worked.** `mean == 254.2, std == 12.5` is the dead giveaway: the ADD result is mostly white, so the secret is the small dark region. No need to OCR — a 12× nearest-neighbor upscale of the bounding box is enough to read a hand-typed font with your own eyes.
- **Cropping saves time and attention.** The dark mask is a 256×256 boolean array with only 230 True pixels. The bounding-box crop drops everything else, so when you save the 12× image you are looking at exactly the signal — not 65,000+ pixels of white noise around it.
- **The 8-hex hash in the flag is per-instance.** Different players see different hash values (`8cdf93c3`, `1b867c3e`, `0542dc1d`, `d562333d`, etc.) because the challenge author regenerated the random shares for each player. The operation you have to apply is the same for all of them: ADD mod 256. Do not trust a public-writeup flag value for a challenge that has a hex hash — always read it off your own instance's image, otherwise the server will reject it with "Incorrect flag. Try again."
