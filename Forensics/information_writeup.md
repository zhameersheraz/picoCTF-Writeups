# information - picoCTF Writeup

**Challenge:** information  
**Category:** Forensics  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{the_m3tadata_1s_modified}`

---

## Description

Files can always be changed in a secret way. Can you find the flag?

**Download:** cat.jpg

---

## Hints

1. Look at the details of the file
2. Make sure to submit the flag as picoCTF{YOURFLAG}

---

## Solution

### Step 1: Download the File

I downloaded the file `cat.jpg` from the challenge page and saved it to `/media/sf_downloads`.

### Step 2: Examine the File with exiftool

The hint said to "look at the details of the file", so I used `exiftool` to examine the image metadata:
```bash
cd /media/sf_downloads
exiftool cat.jpg
```

**Output:**
```
ExifTool Version Number         : 13.36
File Name                       : cat.jpg
Directory                       : .
File Size                       : 878 kB
File Modification Date/Time     : 2026:02:22 11:40:20-05:00
File Access Date/Time           : 2026:02:23 09:26:36-05:00
File Inode Change Date/Time     : 2026:02:22 11:40:20-05:00
File Permissions                : -rwxrwx---
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.02
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Current IPTC Digest             : 7a78f3d9cfb1ce42ab5a3aa30573d617
Copyright Notice                : PicoCTF
Application Record Version      : 4
XMP Toolkit                     : Image::ExifTool 10.80
License                         : cGljb0NURnt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9
Rights                          : PicoCTF
Image Width                     : 2560
Image Height                    : 1598
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 2560x1598
Megapixels                      : 4.1
```

I noticed something suspicious in the **License** field:
```
License: cGljb0NURnt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9
```

This looks like **Base64 encoding**!

### Step 3: Decode the Base64 String

I decoded the Base64 string using the `base64` command:
```bash
echo "cGljb0NURnt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9" | base64 -d
```

**Output:**
```
picoCTF{the_m3tadata_1s_modified}
```

Got the flag! 🎯

---

## Why This Works

* **EXIF metadata** (Exchangeable Image File Format) stores information about images
* This metadata can include:
  - Camera settings (ISO, shutter speed, aperture)
  - Date and time the photo was taken
  - GPS coordinates
  - Copyright information
  - **Custom fields** (like License, where the flag was hidden)
* **Base64 encoding** is used to represent binary data in ASCII text format
* The flag was hidden in the "License" metadata field as Base64
* Tools like `exiftool` can read and modify this metadata
* This demonstrates how metadata can be used to **hide information in plain sight**

---

## What is EXIF Metadata?

**EXIF (Exchangeable Image File Format)** is metadata embedded in image files:

**Common EXIF Fields:**
- **Camera Info:** Make, Model, Lens
- **Settings:** ISO, Aperture, Shutter Speed, Flash
- **Date/Time:** When photo was taken
- **GPS:** Latitude, Longitude, Altitude
- **Copyright:** Author, License, Rights
- **Custom Fields:** Any additional information

**Why This Matters for Security:**
- Photos can leak sensitive information (location, time, device)
- Metadata can be modified to hide data
- Always check metadata in forensics investigations

---

## Alternative Methods

### Method 1: strings Command
```bash
strings cat.jpg | grep pico
```

This might find the Base64 string, but exiftool is more reliable.

### Method 2: Online EXIF Viewer

Upload the image to: https://exifdata.com or https://jimpl.com

### Method 3: Using identify (ImageMagick)
```bash
identify -verbose cat.jpg | grep -i license
```

### Method 4: Python Script
```python
from PIL import Image
from PIL.ExifTags import TAGS

img = Image.open('cat.jpg')
exif = img._getexif()

for tag_id, value in exif.items():
    tag = TAGS.get(tag_id, tag_id)
    print(f"{tag}: {value}")
```

---

## Terminal Commands
```bash
# Navigate to downloads folder
cd /media/sf_downloads

# View all metadata with exiftool
exiftool cat.jpg

# Extract just the License field
exiftool cat.jpg | grep License

# Decode the Base64 string
echo "cGljb0NURnt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9" | base64 -d

# One-liner to extract and decode
exiftool cat.jpg | grep License | cut -d: -f2 | xargs echo | base64 -d
```

---

## Flag

`picoCTF{the_m3tadata_1s_modified}`

---

## Tools Used

* **exiftool** - Read and write EXIF metadata
* **base64** - Encode/decode Base64 strings
* **strings** (alternative) - Extract printable strings from files

---

## Key Takeaways

* **Always check metadata** in forensics challenges
* **exiftool** is essential for examining image files
* **Base64 encoding** is commonly used to hide data in text fields
* Images can contain hidden information beyond what's visible
* Metadata reveals more than you think - be careful what you share online
* The flag name "the metadata is modified" confirms this was intentional data hiding
* This is a reminder to **strip metadata** from photos before sharing publicly
* Journalists, activists, and security professionals must be aware of metadata risks
