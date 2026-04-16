# interencdec — picoCTF Writeup

**Challenge:** interencdec  
**Category:** Cryptography  
**Difficulty:** Easy  
**Flag:** `picoCTF{caesar_d3cr9pt3d_890d2379}`  

---

## Description

> Can you get the real meaning from this file?
> Download the file here.

**Hint shown in challenge:** `Engaging in various decoding processes is of utmost importance`

---

## Background Knowledge (Read This First!)

### What is Base64?

Base64 converts binary data into readable ASCII text. It is not encryption — it is easily reversible. Common indicators: ends with `=` or `==`, only uses `A-Z`, `a-z`, `0-9`, `+`, `/`.

### What is Caesar Cipher?

The Caesar cipher shifts each letter by a fixed number of positions. This challenge used a shift of 7. The `caesar` command from `bsdgames` tries all 25 shifts at once.

### The Encoding Layers in This Challenge

```
enc_flag → Base64 → Python bytes literal → Base64 → Caesar cipher → Flag
```

---

## Solution — Step by Step

### Step 1 — Examine the File

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool enc_flag
File Name                       : enc_flag
File Type                       : TXT
MIME Type                       : text/plain
```

### Step 2 — Read the File Contents

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings enc_flag
YidkM0JxZGtwQlRYdHFhR3g2YUhsZmF6TnFlVGwzWVROclh6ZzVNR3N5TXpjNWZRPT0nCg==
```

✅ Looks like Base64!

### Step 3 — First Base64 Decode

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "YidkM0JxZGtwQlRYdHFhR3g2YUhsZmF6TnFlVGwzWVROclh6ZzVNR3N5TXpjNWZRPT0nCg==" | base64 -d
b'd3BqdkpBTXtqaGx6aHlfazNqeTl3YTNrXzg5MGsyMzc5fQ=='
```

✅ This is a Python bytes literal — the actual Base64 string is inside the quotes.

### Step 4 — Second Base64 Decode

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "d3BqdkpBTXtqaGx6aHlfazNqeTl3YTNrXzg5MGsyMzc5fQ==" | base64 -d
wpjvJAM{jhlzhy_k3jy9wa3k_890k2379}
```

✅ The structure `wpjvJAM{...}` resembles `picoCTF{...}` — this is Caesar cipher!

### Step 5 — Caesar Cipher Decode

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo apt install bsdgames

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "wpjvJAM{jhlzhy_k3jy9wa3k_890k2379}" | caesar
picoCTF{caesar_d3cr9pt3d_890d2379}
```

Got the flag! 🎯

---

## Decoding Summary

| Layer | Input | Output |
|-------|-------|--------|
| 1 | Raw file | Base64 string |
| 2 | Base64 decode | Python bytes `b'...'` wrapping another Base64 |
| 3 | Base64 decode (inner) | Caesar-ciphered text |
| 4 | Caesar decode | `picoCTF{caesar_d3cr9pt3d_890d2379}` |

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

- Always look for multiple layers of encoding — one decode is rarely enough
- `b'...'` in output means it's a Python bytes literal — strip the wrapper and decode the inner string
- Base64 is not encryption — it's just encoding and is trivially reversible
- The `caesar` command from `bsdgames` tries all 25 ROT shifts at once — very handy!
