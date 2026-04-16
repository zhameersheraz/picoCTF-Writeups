# Scan Surprise — picoCTF Writeup

**Challenge:** Scan Surprise  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{p33k_@_b00_0194a007}`  

---

## Description

> I've gotten bored of handing out flags as text. Wouldn't it be cool if they were an image instead?
> Download the challenge files here: challenge.zip
> SSH access: `ssh -p 59076 ctf-player@atlas.picoctf.net`
> Password: `84b12bae`

**Hint shown in challenge:** `Mobile phones have included native QR code scanners in their cameras since version 8 (Oreo) and iOS 11`

---

## Background Knowledge (Read This First!)

### What is a QR Code?

A QR Code (Quick Response Code) is a two-dimensional barcode that can store text, URLs, contact info, and more. It has three large squares in the corners (position markers) and built-in error correction.

### What is zbarimg?

`zbarimg` is a command-line tool that reads barcodes and QR codes directly from image files — no phone needed.

---

## Solution — Step by Step

### Step 1 — Download and Extract the File

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool challenge.zip
...
Zip File Name                   : flag.png
...

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip challenge.zip
  inflating: home/ctf-player/drop-in/flag.png
```

### Step 2 — Examine the PNG File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool flag.png
...
Image Width                     : 99
Image Height                    : 99
...
```

It's a small 99×99 pixel PNG image — the hint mentions QR codes, so this is a QR code!

### Step 3 — Connect to the SSH Server

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ssh -p 59076 ctf-player@atlas.picoctf.net
# Password: 84b12bae
# Type: yes (to accept fingerprint)
```

### Step 4 — Scan the QR Code

```
┌──(ctf-player㉿atlas)-[~/drop-in]
└─$ zbarimg flag.png
QR-Code:picoCTF{p33k_@_b00_0194a007}
scanned 1 barcode symbols from 1 images in 0 seconds
```

Got the flag! 🎯

---

## Alternative Methods

**Method 1 — Use Your Phone Camera**
Open the flag.png image on your computer and point your phone camera at it — most modern phones detect QR codes automatically.

**Method 2 — Online QR Reader**
Go to https://zxing.org/w/decode and upload flag.png.

**Method 3 — Install zbar locally**
```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo apt install zbar-tools

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ zbarimg flag.png
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `ssh` | Connect to remote server |
| `zbarimg` | Scan QR codes from images |
| `exiftool` | Check file metadata |
| `unzip` | Extract ZIP archives |

---

## Key Takeaways

- QR codes can encode text, URLs, and other data in a scannable image format
- `zbarimg` is a powerful command-line tool for scanning QR codes without a phone
- The challenge SSH server has tools pre-installed — always check what's available there first
- Always check the challenge environment — SSH access may have tools you don't have locally
