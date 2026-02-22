# RED - picoCTF Writeup

**Challenge:** RED  
**Category:** Forensics  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}`

---

## Description

RED, RED, RED, RED

Download the image: red.png

---

## Hints

1. The picture seems pure, but is it though?

---

## Solution

### Step 1: Download and Check the File

I downloaded `red.png` and checked the file type:
```bash
cd /media/sf_downloads
file red.png
```

It's a valid PNG image file.

### Step 2: Check Metadata with exiftool

I used `exiftool` to check the image metadata:
```bash
exiftool red.png
```

**Output:**
```
ExifTool Version Number         : 13.36
File Name                       : red.png
...
Image Width                     : 128
Image Height                    : 128
Poem                            : Crimson heart, vibrant and bold,
                                  Hearts flutter at your sight.
                                  Evenings glow softly red,
                                  Cherries burst with sweet life.
                                  Kisses linger with your warmth.
                                  Love deep as merlot.
                                  Scarlet leaves falling softly,
                                  Bold in every stroke.
```

Found a poem in the metadata, but no flag.

### Step 3: Try strings Command

I tried extracting strings from the image:
```bash
strings red.png
```

This showed the poem but no flag.

### Step 4: Use zsteg for Steganography Analysis

Since the hint says "the picture seems pure, but is it though?", I used `zsteg` to check for hidden data in the image:
```bash
zsteg red.png
```

**Key finding:**
```
b1,rgba,lsb,xy      .. text: "cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==..."
```

This shows Base64-encoded text hidden in the **least significant bits (LSB)** of the RGBA channels!

### Step 5: Extract and Decode the Base64 String

I extracted the Base64 string and decoded it:
```bash
echo "cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==" | base64 -d
```

**Output:**
```
picoCTF{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}
```

---

## Why This Works

* **LSB Steganography** hides data in the least significant bits of pixel values
* Each pixel in an image has color values (Red, Green, Blue, Alpha)
* The least significant bit of each value can be changed without visibly affecting the image
* By modifying these bits, data can be hidden inside the image
* **zsteg** is a tool specifically designed to detect and extract such hidden data
* The flag was Base64-encoded and embedded in the LSB of the RGBA channels

---

## What is LSB Steganography?

**Least Significant Bit (LSB)** steganography is a technique where data is hidden in the least important bits of an image:

**Example:**
- Original pixel color: `11001100` (204 in decimal)
- Change LSB to hide data: `11001101` (205 in decimal)
- Visual difference: Almost imperceptible to human eyes

**How it works:**
1. Take a message and convert it to binary
2. Replace the LSB of each pixel with one bit from the message
3. The image looks the same but contains hidden data

---

## What is zsteg?

`zsteg` is a steganography detection tool for PNG and BMP images:

**Installation:**
```bash
gem install zsteg
```

**Basic usage:**
```bash
# Analyze image for hidden data
zsteg image.png

# Extract specific channel data
zsteg -a image.png

# Extract and save data
zsteg -E "b1,rgba,lsb,xy" image.png > output.txt
```

---

## Terminal Commands
```bash
# Navigate to downloads
cd /media/sf_downloads

# Check file type
file red.png

# Check metadata
exiftool red.png

# Try strings
strings red.png

# Analyze with zsteg (MAIN SOLUTION)
zsteg red.png

# Decode the Base64 string found
echo "cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==" | base64 -d
```

---

## Flag

`picoCTF{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}`

---

## Tools Used

* **exiftool** - Check image metadata
* **strings** - Extract printable strings
* **zsteg** - Detect and extract steganography in PNG/BMP files
* **base64** - Decode Base64-encoded data

---

## Key Takeaways

* Images can hide data in their least significant bits without visible changes
* Always use steganography tools like `zsteg` for image forensics
* The hint "seems pure, but is it though?" suggests hidden data (steganography)
* LSB steganography is a common CTF technique
* Base64 encoding is often used to encode hidden data
* `zsteg` automatically checks multiple steganography methods
* Always try multiple forensics tools: exiftool, strings, zsteg, binwalk, etc.
