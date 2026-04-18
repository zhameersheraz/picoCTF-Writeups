# MultiCode — picoCTF Writeup

**Challenge:** MultiCode  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{nested_enc0ding_848a466b}`  

---

## Description

> We intercepted a suspiciously encoded message, but it's clearly hiding a flag. No encryption, just multiple layers of obfuscation. Can you peel back the layers and reveal the truth?

**Download:** `message.txt`  
**Hint 1:** `The flag has been wrapped in several layers of common encodings such as ROT13, URL encoding, Hex, and Base64. Can you figure out the order to peel them back?`

---

## Background Knowledge (Read This First!)

### What is encoding vs encryption?

This challenge is careful to say **"no encryption, just obfuscation."** That's an important distinction:

- **Encryption** uses a secret key — without the key, you cannot read the message
- **Encoding** is just a way of representing data in a different format — it has no key and is always fully reversible if you know the format used

Base64, Hex, URL encoding, and ROT13 are all **encodings** — they look scrambled but anyone who knows the format can decode them instantly.

### What is Base64?

**Base64** represents binary data using only 64 printable characters (A-Z, a-z, 0-9, +, /). It always ends with `=` or `==` as padding. It is immediately recognizable by this padding and its character set.

Example: `cGljb0NURg==` → `picoCTF`

### What is Hex encoding?

**Hex** (hexadecimal) represents each byte of data as two characters from `0-9` and `a-f`. A hex-encoded string will only ever contain those characters.

Example: `7069636f` → `pico`

### What is URL encoding?

**URL encoding** (also called percent encoding) replaces special characters with `%XX` where XX is their hex value. This is used in web addresses to safely transmit characters like `{`, `}`, spaces, etc.

Example: `%7B` → `{` and `%7D` → `}`

### What is ROT13?

**ROT13** shifts every letter 13 positions forward in the alphabet. Since the alphabet has 26 letters, applying ROT13 twice returns the original text. It's the simplest possible substitution cipher.

Example: `cvpbPGS` → `picoCTF`

### The Key Insight — Peeling Layers in the Right Order

The trick with multi-layer encoding challenges is finding the **correct order** to decode. The outermost layer must be removed first. A useful way to identify each layer:

| Clue in the data | Encoding |
|-----------------|----------|
| Ends with `=` or `==`, uses A-Z/a-z/0-9/+/ | Base64 |
| Only characters 0-9 and a-f | Hex |
| Contains `%XX` sequences | URL encoding |
| Letters only, looks like scrambled English | ROT13 |

---

## The Message

```
NjM3NjcwNjI1MDQ3NTMyNTM3NDI2MTcyNjY2NzcyNzE1ZjcyNjE3MDMwNzE3NjYxNzQ1ZjM4MzQzODZlMzQzNjM2NmYyNTM3NDQ=
```

The trailing `=` and the character set immediately identify this as **Base64** — that's our starting point.

---

## Solution — Step by Step

### Layer 1 — Base64 Decode

```bash
echo "NjM3NjcwNjI1MDQ3NTMyNTM3NDI2MTcyNjY2NzcyNzE1ZjcyNjE3MDMwNzE3NjYxNzQ1ZjM4MzQzODZlMzQzNjM2NmYyNTM3NDQ=" | base64 -d
```

Result:
```
637670625047532537426172666772715f72617030717661745f3834386e3436366f253744
```

This output only contains characters `0-9` and `a-f` — that's **Hex**.

### Layer 2 — Hex Decode

```bash
echo "637670625047532537426172666772715f72617030717661745f3834386e3436366f253744" | xxd -r -p
```

Result:
```
cvpbPGS%7Barfgrq_rap0qvat_848n466o%7D
```

Now we can see `%7B` and `%7D` — those are **URL encoded** curly braces `{` and `}`.

### Layer 3 — URL Decode

```bash
python3 -c "import urllib.parse; print(urllib.parse.unquote('cvpbPGS%7Barfgrq_rap0qvat_848n466o%7D'))"
```

Result:
```
cvpbPGS{arfgrq_rap0qvat_848n466o}
```

This looks like a flag shape (`xxx{...}`) but the letters are shifted. The `cvpbPGS` part is a giveaway — that's `picoCTF` in **ROT13**.

### Layer 4 — ROT13 Decode

```bash
echo "cvpbPGS{arfgrq_rap0qvat_848n466o}" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Result:
```
picoCTF{nested_enc0ding_848a466b}
```

✅ Got the flag! 🎯

---

## One-Shot Python Script

You can also decode all four layers in a single script:

```python
import base64, urllib.parse, codecs

msg = 'NjM3NjcwNjI1MDQ3NTMyNTM3NDI2MTcyNjY2NzcyNzE1ZjcyNjE3MDMwNzE3NjYxNzQ1ZjM4MzQzODZlMzQzNjM2NmYyNTM3NDQ='

l1 = base64.b64decode(msg).decode()         # Base64
l2 = bytes.fromhex(l1).decode()             # Hex
l3 = urllib.parse.unquote(l2)               # URL decode
l4 = codecs.decode(l3, 'rot_13')            # ROT13

print(l4)
# picoCTF{nested_enc0ding_848a466b}
```

---

## Summary of Layers

| Layer | Encoding | Input (truncated) | Output (truncated) |
|-------|----------|-------------------|-------------------|
| 1 | Base64 | `NjM3Njcw...NDQ=` | `63767062...3744` |
| 2 | Hex | `63767062...3744` | `cvpbPGS%7B...%7D` |
| 3 | URL | `cvpbPGS%7B...%7D` | `cvpbPGS{...}` |
| 4 | ROT13 | `cvpbPGS{arfgrq...}` | `picoCTF{nested...}` |

---

## Alternative Method — CyberChef

**CyberChef** (https://gchq.github.io/CyberChef/) can handle all four layers visually with no coding:

1. Paste the message into the Input box
2. Add these recipes in order:
   - **From Base64**
   - **From Hex**
   - **URL Decode**
   - **ROT13**
3. The Output box shows the flag immediately

CyberChef is the fastest tool for layered encoding challenges — it even has an **"Magic" recipe** that auto-detects encoding layers for you.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `base64 -d` | Decode the outermost Base64 layer | ⭐ Easy |
| `xxd -r -p` | Decode the Hex layer | ⭐ Easy |
| Python `urllib.parse.unquote` | Decode URL-encoded characters | ⭐ Easy |
| `tr` / Python `codecs.decode` | Apply ROT13 | ⭐ Easy |
| CyberChef (optional) | Decode all layers in a GUI | ⭐ Easy |

---

## Key Takeaways

- **Encoding is not encryption** — Base64, Hex, URL, and ROT13 provide zero security; they just make data look scrambled
- **Identify each layer by its visual signature** before trying to decode — trailing `=` means Base64, `%XX` means URL encoding, hex-only characters means Hex, shifted letters means ROT13
- **Order matters** — always peel from the outside in; decoding the wrong layer first produces garbage
- **CyberChef's Magic recipe** can auto-detect encoding layers — a huge time saver in CTFs
- The flag name says it all: `nested_enc0ding` — the password was hidden inside multiple nested layers of encoding
