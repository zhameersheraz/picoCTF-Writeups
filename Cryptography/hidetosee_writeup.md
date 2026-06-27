# HideToSee - picoCTF Writeup

**Challenge:** HideToSee  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{atbash_crack_05b2a65a}`  
**Platform:** picoCTF (2023)  
**Writeup by:** zham  

---

## Description

> How about some hide and seek heh?
>
> Look at this image [here].

## Hints

> 1. Download the image and try to extract it.

---

## Background Knowledge

This challenge layers two classic techniques: steganography (hiding a message inside an image file) and a classical cipher (Atbash). Let us unpack both.

**Steganography vs. cryptography.** Cryptography hides the *meaning* of a message (you can see the ciphertext but cannot read it without the key). Steganography hides the *existence* of a message (you cannot even tell there is a hidden file). picoCTF's `HideToSee` uses steganography to smuggle a file inside a JPEG, then applies a classical cipher to the file contents.

**JPEG steganography with `steghide`.** `steghide` is a popular tool that embeds data inside JPEG and BMP files. It works by tweaking the image's DCT (discrete cosine transform) coefficients in ways that are visually imperceptible. The data can be:
- Encrypted (so even if you extract it, you still need a passphrase).
- Compressed.
- Stored as a named file with a size hint.

The `steghide info` command tells you what is hidden without extracting it. If a passphrase was set, you need to know it to extract.

**Why JPEG?** PNGs use lossless compression, which preserves every bit exactly — that makes them great for LSB steganography (hiding data in the least-significant bits of pixel values). JPEGs use lossy compression, which destroys any LSB pattern; instead, you need a tool like `steghide` that operates on the DCT coefficients, not the raw pixels.

**The Atbash cipher.** Atbash is one of the oldest known ciphers (originating in Hebrew around 500 BC). For each letter, you substitute its reverse-alphabet counterpart:
- `a` ↔ `z`, `b` ↔ `y`, `c` ↔ `x`, ... `m` ↔ `n`.
- Case is preserved. Digits and symbols pass through unchanged.

It is its own inverse — applying Atbash twice gets you back to the original. Mathematically: for letter with alphabet index `i` (0-25), Atbash maps it to index `(25 - i)`.

**Putting it together.** The challenge image has `encrypted.txt` embedded via `steghide` (no passphrase — empty string). The contents of that file are the flag with Atbash applied. Reverse the Atbash to recover the flag.

---

## Solution

### Step 1 — Inspect the file

```bash
┌──(zham㉿kali)-[~/picoCTF/HideToSee]
└─$ mkdir hidetosee && cd hidetosee
┌──(zham㉿kali)-[~/picoCTF/HideToSee]
└─$ cp ~/Downloads/atbash.jpg image.jpg
┌──(zham㉿kali)-[~/picoCTF/HideToSee]
└─$ file image.jpg
image.jpg: JPEG image data, JFIF standard 1.01, ..., 465x455
```

A plain JPEG. No obvious metadata trick. Time to check for stego.

### Step 2 — Probe with steghide

If `steghide` is not installed yet, grab it:

```bash
┌──(zham㉿kali)-[~/picoCTF/HideToSee]
└─$ sudo apt update && sudo apt install -y steghide
```

Now probe the file. picoCTF authors often set an empty passphrase for steghide challenges, so I try `""` first:

```bash
┌──(zham㉿kali)-[~/picoCTF/HideToSee]
└─$ steghide info image.jpg -p ""
"image.jpg":
  format: jpeg
  capacity: 2.4 KB
  embedded file "encrypted.txt":
    size: 31.0 Byte
    encrypted: rijndael-128, cbc
    compressed: yes
```

There is an embedded file called `encrypted.txt` — 31 bytes, AES-128-CBC encrypted, but it is hiding *inside* the JPEG. Extract it:

```bash
┌──(zham㉿kali)-[~/picoCTF/HideToSee]
└─$ steghide extract -sf image.jpg -p ""
wrote extracted data to "encrypted.txt".
┌──(zham㉿kali)-[~/picoCTF/HideToSee]
└─$ cat encrypted.txt
krxlXGU{zgyzhs_xizxp_05y2z65z}
```

The contents look like a flag with the letters shifted. The clue from the challenge image (an Atbash cipher wheel) tells us which cipher was used.

### Step 3 — Reverse the Atbash cipher

Atbash is its own inverse: applying it once to the ciphertext gives us the plaintext.

```bash
┌──(zham㉿kali)-[~/picoCTF/HideToSee]
└─$ nano solve.py
```

```python
#!/usr/bin/env python3
ct = open('encrypted.txt').read().strip()

def atbash(c):
    if 'a' <= c <= 'z':
        return chr(ord('z') - (ord(c) - ord('a')))
    if 'A' <= c <= 'Z':
        return chr(ord('Z') - (ord(c) - ord('A')))
    return c  # digits, _, {, } pass through

print(''.join(atbash(c) for c in ct))
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

Run it:

```bash
┌──(zham㉿kali)-[~/picoCTF/HideToSee]
└─$ python3 solve.py
picoCTF{atbash_crack_05b2a65a}
```

Done.

---

## Alternative Solve Methods

### Alternative 1 — `stegseek` (much faster brute-force)

If you do not know whether the passphrase is empty, `stegseek` can brute-force it against a wordlist in seconds. Install it via the [stegseek GitHub releases](https://github.com/RickdeJager/stegseek) (Debian-derived: `apt install stegseek`):

```bash
┌──(zham㉿kali)-[~/picoCTF/HideToSee]
└─$ stegseek image.jpg
StegSeek version 0.6
...
[+] Found passphrase: ""
[+] Extracted to: image.jpg.out
┌──(zham㉿kali)-[~/picoCTF/HideToSee]
└─$ cat image.jpg.out
krxlXGU{zgyzhs_xizxp_05y2z65z}
```

Then apply Atbash as above. `stegseek` is preferred over `steghide` for CTF work because it can crack passphrases automatically.

### Alternative 2 — Online Atbash decoder

Skip the Python step entirely. After extracting `encrypted.txt`, paste the contents into any online Atbash decoder (CyberChef has a built-in one under Operations > Encrypt/Decrypt > Atbash). It will output the flag in one click.

### Alternative 3 — CyberChef recipe (entire pipeline)

Drag-and-drop the JPEG into CyberChef, then chain these operations:

1. **Extract LSB stego from JPEG** — actually CyberChef does not have a steghide recipe built-in. Use `steghide` first to extract `encrypted.txt`.
2. **Add the Atbash operation** to the recipe.
3. **Input:** paste `krxlXGU{zgyzhs_xizxp_05y2z65z}`.
4. **Output:** `picoCTF{atbash_crack_05b2a65a}`.

### Alternative 4 — Python with `stegano` library (for non-JPEG)

If the challenge had used a PNG instead, `stegano` is a Python library that hides data in LSBs. For this challenge, you cannot use `stegano` because the file is JPEG — `steghide` is the right tool.

---

## What Happened Internally

A short timeline of the decode:

1. **Identify the file** — `file image.jpg` reports a standard JPEG. No trailing data after the `FF D9` EOF marker, no EXIF secrets, no comment segments. So the flag is not in the JPEG metadata.
2. **Probe for stego** — `steghide info` reveals an embedded `encrypted.txt` (31 bytes, AES-128-CBC encrypted inside the JPEG's DCT coefficients). The empty passphrase works.
3. **Extract** — `steghide extract` writes `encrypted.txt` to disk. Contents: `krxlXGU{zgyzhs_xizxp_05y2z65z}`.
4. **Recognize Atbash** — the `p` → `k` shift is exactly what Atbash does for the alphabetic position 15 → 10 (mirror in a 26-letter alphabet). The challenge image (Atbash cipher wheel) is the hint.
5. **Apply Atbash** — for each character, map `a-z` ↔ `z-a` and `A-Z` ↔ `Z-A`. Digits and symbols pass through.
6. **Read the result** — `picoCTF{atbash_crack_05y2z65z}` — "atbash crack" with the random suffix.

Note that the encrypted file was itself AES-encrypted by `steghide`, but because the passphrase was empty, the encryption added no real protection. `steghide info` revealed both the embedded file's name and its size without needing the passphrase — only the *contents* require it.

---

## Tools Used

| Tool             | Purpose                                                                  |
|------------------|--------------------------------------------------------------------------|
| `file`           | Confirm the file is a JPEG.                                              |
| `steghide info`  | Detect whether anything is hidden in the JPEG.                           |
| `steghide extract` | Pull the embedded `encrypted.txt` out of the JPEG.                     |
| `python3`        | Apply the Atbash substitution.                                           |
| `nano`           | Edit `solve.py`.                                                         |
| (optional) `stegseek` | Faster brute-force extraction when the passphrase is unknown.        |
| (optional) CyberChef | GUI option for the Atbash step.                                       |

---

## Key Takeaways

- **Check the metadata, then the body, then the stego.** When metadata is clean and strings are empty, the flag is almost certainly hidden inside the file structure — not before or after it. `steghide info` is your one-shot check.
- **`steghide` works on JPEG, not PNG.** Different formats need different tools. For JPEG, use `steghide` or `stegseek`. For PNG/BMP, try `zsteg`, `stegano`, or `stegoveritas`.
- **Empty passphrase is the default.** picoCTF rarely sets a real passphrase on steghide challenges. Try `""` first; if it fails, fall back to wordlist brute-force with `stegseek` against `rockyou.txt`.
- **Read the visual hint.** The challenge image is an *Atbash cipher wheel* — that is the challenge author telling you which cipher to apply after extraction. Do not skip step 1 of looking at what the challenge image actually depicts.
- **Atbash is its own inverse.** No need to write a custom "decode" — the same function encrypts and decrypts. That is why the cipher wheel in the image is symmetric (each letter pairs with its mirror on the opposite side).

Flag decoded: `picoCTF{atbash_crack_05b2a65a}` — *atbash crack*, exactly what we just did: crack an Atbash-ciphered payload pulled out of a JPEG by steghide.
