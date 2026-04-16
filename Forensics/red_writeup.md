# RED — picoCTF Writeup

**Challenge:** RED  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}`  

---

## Description

> RED, RED, RED, RED
> Download the image: red.png

**Hint shown in challenge:** `The picture seems pure, but is it though?`

---

## Background Knowledge (Read This First!)

### What is LSB Steganography?

**LSB (Least Significant Bit)** steganography hides data in the least significant bits of pixel values. Each pixel has color channels (Red, Green, Blue, Alpha). Changing the LSB of each value barely affects the image visually but can store hidden data.

Example:
- Original pixel: `11001100` (204)
- Modified LSB: `11001101` (205) — visually identical, hides 1 bit of data

### What is zsteg?

`zsteg` is a steganography detection tool specifically for PNG and BMP images. It automatically checks multiple LSB configurations to find hidden data.

---

## Solution — Step by Step

### Step 1 — Download and Check the File

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file red.png
red.png: PNG image data, 128 x 128, ...
```

### Step 2 — Check Metadata

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool red.png
...
Poem                            : Crimson heart, vibrant and bold...
```

Found a poem in the metadata, but no flag.

### Step 3 — Try strings

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings red.png
```

Showed the poem but no flag.

### Step 4 — Use zsteg for LSB Steganography

The hint says "the picture seems pure, but is it though?" — this points to hidden data inside the pixels.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ zsteg red.png
b1,rgba,lsb,xy      .. text: "cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==..."
```

Found Base64-encoded text hidden in the LSB of the RGBA channels!

### Step 5 — Decode the Base64 String

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==" | base64 -d
picoCTF{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}
```

Got the flag! 🎯

---

## zsteg Reference

```bash
# Install zsteg
gem install zsteg

# Analyze image for hidden data
zsteg image.png

# Analyze all channels
zsteg -a image.png

# Extract and save specific channel
zsteg -E "b1,rgba,lsb,xy" image.png > output.txt
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Check file type |
| `exiftool` | Check image metadata |
| `strings` | Extract printable strings |
| `zsteg` | Detect and extract LSB steganography |
| `base64` | Decode Base64-encoded data |

---

## Key Takeaways

- Images can hide data in their least significant bits without any visible changes
- Always use steganography tools like `zsteg` for PNG image forensics
- The hint "seems pure, but is it though?" directly points to hidden pixel data
- LSB steganography is one of the most common CTF techniques
- Always try multiple forensics tools: `exiftool`, `strings`, `zsteg`, `binwalk`
