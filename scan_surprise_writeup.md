# Scan Surprise - picoCTF Writeup

**Challenge:** Scan Surprise  
**Category:** Forensics  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{p33k_@_b00_0194a007}`

---

## Description

I've gotten bored of handing out flags as text. Wouldn't it be cool if they were an image instead?

You can download the challenge files here: challenge.zip

The same files are accessible via SSH here:
```
ssh -p 59076 ctf-player@atlas.picoctf.net
```

Using the password `84b12bae`. Accept the fingerprint with `yes`, and `ls` once connected to begin. Remember, in a shell, passwords are hidden!

---

## Hints

1. Mobile phones have included native QR code scanners in their cameras since version 8 (Oreo) and iOS 11

---

## Solution

### Step 1: Download and Extract the File

I downloaded `challenge.zip` and examined it:
```bash
cd /media/sf_downloads
exiftool challenge.zip
```

Then extracted it:
```bash
unzip challenge.zip
```

This extracted `flag.png` to the path `home/ctf-player/drop-in/flag.png`.

### Step 2: Examine the PNG File

I checked the image metadata:
```bash
exiftool flag.png
```

**Output:**
```
File Type                       : PNG
Image Width                     : 99
Image Height                    : 99
```

It's a small 99x99 pixel PNG image. The hint mentions QR codes, so this is likely a QR code!

### Step 3: Connect to the SSH Server

The challenge provides SSH access where tools are already available. I connected:
```bash
ssh -p 59076 ctf-player@atlas.picoctf.net
```

**Password:** `84b12bae`

When prompted about the fingerprint, I typed `yes` and pressed Enter.

### Step 4: Scan the QR Code

Once connected, I navigated to the directory and used `zbarimg` to scan the QR code:
```bash
cd drop-in
zbarimg flag.png
```

**Output:**
```
QR-Code:picoCTF{p33k_@_b00_0194a007}
scanned 1 barcode symbols from 1 images in 0 seconds
```

Got the flag! 🎯

---

## Why This Works

* **QR Codes** (Quick Response codes) are two-dimensional barcodes that store data
* They can be scanned by cameras or specialized software
* **zbarimg** is a command-line tool that reads barcodes and QR codes from image files
* The challenge encoded the flag as a QR code instead of plain text
* By scanning the QR code, we decode it back to the original flag text

---

## What is a QR Code?

**QR Code (Quick Response Code)** is a 2D barcode that can store various types of data:

**Common uses:**
- Website URLs
- Contact information
- WiFi credentials
- Payment information
- Text messages (like our flag!)

**Structure:**
- Square grid of black and white pixels
- Contains position markers (three large squares in corners)
- Can store up to ~4,000 alphanumeric characters
- Has built-in error correction

---

## What is zbarimg?

`zbarimg` is a command-line barcode and QR code scanner:

**Installation:**
```bash
# On Debian/Ubuntu/Kali
sudo apt install zbar-tools

# On macOS
brew install zbar
```

**Basic usage:**
```bash
# Scan a QR code image
zbarimg qrcode.png

# Scan with raw output (no extra text)
zbarimg --raw qrcode.png

# Scan multiple images
zbarimg image1.png image2.png image3.png
```

---

## Alternative Methods

### Method 1: Use Your Phone Camera
1. Open the flag.png image on your computer
2. Point your phone camera at it
3. Most modern phones automatically detect and scan QR codes
4. The flag will appear as a notification

### Method 2: Use an Online QR Reader
1. Go to https://zxing.org/w/decode
2. Upload flag.png
3. The website will decode and display the flag

### Method 3: Install zbar Locally
```bash
# Install zbar-tools on Kali
sudo apt install zbar-tools

# Scan the QR code
zbarimg flag.png
```

---

## Terminal Commands
```bash
# Navigate to downloads
cd /media/sf_downloads

# Check ZIP metadata
exiftool challenge.zip

# Extract ZIP
unzip challenge.zip

# Check PNG metadata
exiftool flag.png

# Try strings (won't show QR code data)
strings flag.png

# Connect to SSH server (MAIN SOLUTION)
ssh -p 59076 ctf-player@atlas.picoctf.net
# Password: 84b12bae

# Once connected, navigate and scan
cd drop-in
zbarimg flag.png
```

---

## Flag

`picoCTF{p33k_@_b00_0194a007}`

---

## Tools Used

* **ssh** - Connect to remote server
* **zbarimg** - Scan QR codes and barcodes from images
* **exiftool** - Check file metadata
* **unzip** - Extract ZIP archives

---

## Key Takeaways

* QR codes can encode text, URLs, and other data in a scannable format
* Modern phones have built-in QR code scanners in their camera apps
* `zbarimg` is a powerful command-line tool for scanning QR codes
* The challenge provides SSH access with pre-installed tools for convenience
* QR codes are commonly used in CTFs to present flags in a non-text format
* The flag "peek_a_boo" is a play on words - you need to "peek" at the QR code
* Always check the challenge environment - SSH access may have tools you don't have locally
