# interencdec - picoCTF Writeup

**Challenge:** interencdec  
**Category:** Cryptography  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{caesar_d3cr9pt3d_890d2379}`

---

## Description

Can you get the real meaning from this file?

Download the file here.

---

## Hints

1. Engaging in various decoding processes is of utmost importance

---

## Solution

### Step 1: Examine the File

I navigated to the downloaded file and used `exiftool` to check the file type:

```bash
cd /media/sf_downloads
exiftool enc_flag
```

**Output:**
```
File Name                       : enc_flag
File Type                       : TXT
MIME Type                       : text/plain
```

✅ It's a plain text file.

### Step 2: Read the File Contents

I used `strings` to read the content of the file:

```bash
strings enc_flag
```

**Output:**
```
YidkM0JxZGtwQlRYdHFhR3g2YUhsZmF6TnFlVGwzWVROclh6ZzVNR3N5TXpjNWZRPT0nCg==
```

✅ This looks like **Base64** encoding!

### Step 3: First Base64 Decode

I decoded the Base64 string:

```bash
echo "YidkM0JxZGtwQlRYdHFhR3g2YUhsZmF6TnFlVGwzWVROclh6ZzVNR3N5TXpjNWZRPT0nCg==" | base64 -d
```

**Output:**
```
b'd3BqdkpBTXtqaGx6aHlfazNqeTl3YTNrXzg5MGsyMzc5fQ=='
```

✅ This is a **Python bytes literal** — the actual Base64 string is inside the quotes: `d3BqdkpBTXtqaGx6aHlfazNqeTl3YTNrXzg5MGsyMzc5fQ==`

### Step 4: Second Base64 Decode

I decoded the inner Base64 string (without the `b'...'` wrapper):

```bash
echo "d3BqdkpBTXtqaGx6aHlfazNqeTl3YTNrXzg5MGsyMzc5fQ==" | base64 -d
```

**Output:**
```
wpjvJAM{jhlzhy_k3jy9wa3k_890k2379}
```

✅ This looks like a **Caesar cipher** — the structure `wpjvJAM{...}` resembles `picoCTF{...}` shifted!

### Step 5: Caesar Cipher Decode

I installed `bsdgames` to get the `caesar` command:

```bash
sudo apt install bsdgames
```

Then I piped the ciphertext through `caesar` to brute-force all possible ROT shifts:

```bash
echo "wpjvJAM{jhlzhy_k3jy9wa3k_890k2379}" | caesar
```

**Output:**
```
picoCTF{caesar_d3cr9pt3d_890d2379}
```

Got the flag! 🎯

---

## Why This Works

### The Encoding Layers

This challenge used **multiple layers of encoding**, which had to be peeled off one by one:

```
enc_flag → Base64 → Python bytes literal → Base64 → Caesar cipher → Flag
```

| Layer | Input | Output |
|-------|-------|--------|
| Layer 1 | Raw file | Base64 string |
| Layer 2 | Base64 decode | Python bytes `b'...'` wrapping another Base64 |
| Layer 3 | Base64 decode (inner) | Caesar-ciphered text |
| Layer 4 | Caesar decode | `picoCTF{caesar_d3cr9pt3d_890d2379}` |

### What is Base64?

**Base64** is an encoding scheme that converts binary data into readable ASCII text. It is **not encryption** — it is easily reversible. Common indicators:
- Ends with `=` or `==` (padding)
- Only uses characters: `A-Z`, `a-z`, `0-9`, `+`, `/`

### What is Caesar Cipher?

The **Caesar cipher** is one of the oldest encryption techniques. It works by shifting each letter in the alphabet by a fixed number of positions.

**Example (ROT13 / shift by 13):**
```
p → c
i → v
c → p
o → b
```

In this challenge, the shift was **7** (e.g., `w` → `p`, `p` → `i`).

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `exiftool` | Check file type and metadata |
| `strings` | Read file contents |
| `base64 -d` | Decode Base64 strings |
| `caesar` (bsdgames) | Brute-force Caesar cipher shifts |

---

## Key Takeaways

- Always look for **multiple layers of encoding** — one decode is rarely enough
- `b'...'` in output means it's a **Python bytes literal** — strip the wrapper and decode the inner string
- **Base64 is not encryption** — it's just encoding and is trivially reversible
- The `caesar` command from `bsdgames` tries all 25 ROT shifts at once — very handy!
- The hint "engaging in various decoding processes" was a direct hint at the multiple layers
- Caesar cipher output still preserves the `picoCTF{...}` structure shape, which helps identify it

