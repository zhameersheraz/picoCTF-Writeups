# repetitions — picoCTF Writeup

**Challenge:** repetitions  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{base64_n3st3d_dic0d!n8_d0wnl04d3d_3f81f7be}`  

---

## Description

> Can you make sense of this file?
> Download the file here.

**Hint 1:** `Multiple decoding is always good.`  
**Tag:** `base64`

---

## Background Knowledge (Read This First!)

### What is Base64?

**Base64** is an encoding scheme that converts binary data into a string of printable characters. It uses the characters `A-Z`, `a-z`, `0-9`, `+`, and `/`, and always pads the end with `=` or `==` to make the length a multiple of 4.

It is immediately recognizable by:
- Characters only from `A-Za-z0-9+/=`
- Often ends with `=` or `==`
- Length is always a multiple of 4

Example: `picoCTF` → `cGljb0NURg==`

### What does "nested" Base64 mean?

Normally Base64 is decoded **once** to get the original data. **Nested** (or repeated) Base64 means the data has been Base64-encoded **multiple times in a row**. To recover the original, you must decode it the same number of times — each decoding peels back one layer.

Think of it like an onion: each `base64 -d` removes one layer until you reach the flag at the center.

### The Hint Confirms It

> "Multiple decoding is always good."

This directly tells you that one decode is not enough — you need to keep going until the output stops looking like Base64.

### ⚠️ Important Note

Make sure the `enc_flag` file is in your current working directory before running any commands. Navigate to the folder where you downloaded it first:

```bash
cd /media/sf_downloads
```

---

## The File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat enc_flag
VmpGU1EyRXlUWGxTYmxKVVYwZFNWbGxyV21GV1JteDBUbFpPYWxKdFVsaFpWVlUxWVZaS1ZWWnVh
RmRXZWtab1dWWmtSMk5yTlZWWApiVVpUVm10d1VWZFdVa2RpYlZaWFZtNVdVZ3BpU0VKeldWUkNk
...
```

The trailing `=` and character set immediately confirm this is Base64. But decoding it once just gives more Base64 — the hint says to keep going.

---

## Solution — Step by Step

### Step 1 — Try one decode first

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ base64 -d enc_flag
VjFSQ2EyTXlSblJUV0dSVllrWmFWRmx0TlZOalJtUlhZVVU1YVZKVVZuaFdWekZoWVZkR2NrNVVX...
```

Still Base64 — we need to go deeper.

### Step 2 — Keep piping `base64 -d` until the flag appears

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat enc_flag | base64 -d | base64 -d | base64 -d | base64 -d | base64 -d | base64 -d
picoCTF{base64_n3st3d_dic0d!n8_d0wnl04d3d_3f81f7be}
```

Six pipes of `base64 -d` peels all six layers and reveals the flag. ✅ Got the flag! 🎯

---

## Why Six Times?

Each `base64 -d` removes exactly one layer of encoding. The file was encoded 6 times, so it needs to be decoded 6 times. Here is what each layer produces:

| Layer | Output starts with | Still Base64? |
|-------|-------------------|---------------|
| Original file | `VmpGU1EyRXlUWGxT...` | ✅ Yes |
| After decode 1 | `VjFSQ2EyTXlSblJU...` | ✅ Yes |
| After decode 2 | `V1RCa2MyRnRTWGRV...` | ✅ Yes |
| After decode 3 | `WTBkc2FtSXdUbFZT...` | ✅ Yes |
| After decode 4 | `Y0dsamIwTlVSbnRp...` | ✅ Yes |
| After decode 5 | `cGljb0NURnt...` | ✅ Yes |
| After decode 6 | `picoCTF{base64_n3st3d...}` | ❌ Flag! |

---

## Alternative Method 1 — Python loop (auto-detects layers)

If you don't know how many layers there are, use a Python loop that keeps decoding until it fails:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
import base64
data = open('enc_flag').read().strip()
step = 0
while True:
    try:
        data = base64.b64decode(data).decode().strip()
        step += 1
    except:
        print(f'Flag after {step} decodings:')
        print(data)
        break
"
Flag after 6 decodings:
picoCTF{base64_n3st3d_dic0d!n8_d0wnl04d3d_3f81f7be}
```

This works for any number of layers — no counting needed.

---

## Alternative Method 2 — CyberChef (browser GUI)

1. Open **CyberChef** → https://gchq.github.io/CyberChef/
2. Paste the file contents into the Input box
3. Add the **"From Base64"** recipe six times
4. Or use the **Magic** recipe — it auto-detects and applies repeated Base64 decoding automatically
5. The Output box shows the flag

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `cat` | Read the encoded file | ⭐ Easy |
| `base64 -d` | Decode one Base64 layer | ⭐ Easy |
| Pipe `\|` | Chain multiple decodings in one command | ⭐ Easy |
| Python `base64` loop (optional) | Auto-detect and decode all layers | ⭐ Easy |
| CyberChef Magic (optional) | Decode all layers in browser GUI | ⭐ Easy |

---

## Key Takeaways

- **When Base64 decodes to more Base64, keep going** — the hint "multiple decoding is always good" is the direct giveaway
- **Piping `| base64 -d` repeatedly** is the fastest one-liner approach in the terminal
- **A Python loop** is cleaner when you don't know the number of layers — it handles any depth automatically
- **Recognize Base64 on sight** — trailing `=`, characters only from `A-Za-z0-9+/`, length divisible by 4
- The flag says it all: `base64_n3st3d_dic0d!n8` → "Base64 nested decoding" — exactly what the challenge required
