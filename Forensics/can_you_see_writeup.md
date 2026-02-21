# CanYouSee - picoCTF Writeup

**Challenge:** CanYouSee  
**Category:** Forensics  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{ME74D47A_HIDD3N_a6df8db8}`

---

## Description

How about some hide and seek?

Download this file here.

---

## Hints

1. How can you view the information about the picture?

---

## Solution

### Step 1: Download the File

I downloaded the file `unknown.zip` from the challenge.

### Step 2: Check the ZIP File Metadata

I used `exiftool` to check the ZIP file's metadata:
```bash
cd /media/sf_downloads
exiftool unknown.zip
```

**Output:**
```
File Name                       : unknown.zip
File Type                       : ZIP
Zip File Name                   : ukn_reality.jpg
```

The ZIP contains a file named `ukn_reality.jpg`.

### Step 3: Extract the ZIP File

I extracted the contents:
```bash
unzip unknown.zip
```

**Output:**
```
Archive:  unknown.zip
replace ukn_reality.jpg? [y]es, [n]o, [A]ll, [N]one, [r]ename: y
  inflating: ukn_reality.jpg
```

This extracted `ukn_reality.jpg`.

### Step 4: Check the Image Metadata

I used `exiftool` to examine the JPEG's metadata:
```bash
exiftool ukn_reality.jpg
```

**Key finding:**
```
Attribution URL                 : cGljb0NURntNRTc0RDQ3QV9ISUREM05fYTZkZjhkYjh9Cg==
```

Found Base64-encoded data in the "Attribution URL" field!

### Step 5: Decode the Base64 String

I decoded the Base64 string:
```bash
echo "cGljb0NURntNRTc0RDQ3QV9ISUREM05fYTZkZjhkYjh9Cg==" | base64 -d
```

**Output:**
```
picoCTF{ME74D47A_HIDD3N_a6df8db8}
```

---

## Why This Works

* **EXIF metadata** stores information about images (camera settings, location, copyright, etc.)
* Tools can add custom metadata fields to images
* In this challenge, the flag was hidden in a custom "Attribution URL" field
* The flag was Base64-encoded to avoid being immediately visible
* **exiftool** is the standard tool for reading and writing image metadata
* Metadata is often overlooked but can contain sensitive information

---

## What is EXIF Metadata?

**EXIF (Exchangeable Image File Format)** stores metadata in images:

**Common EXIF fields:**
- Camera make and model
- Date/time taken
- GPS coordinates
- ISO, aperture, shutter speed
- Copyright information
- Custom fields (like "Attribution URL" in this challenge)

**Security concerns:**
- Can reveal location where photo was taken
- Can identify camera/device used
- May contain sensitive personal information
- Always strip metadata before sharing sensitive images

---

## What is exiftool?

`exiftool` is a powerful metadata reader/writer for images and other files:

**Basic usage:**
```bash
# View all metadata
exiftool image.jpg

# View specific field
exiftool -AttributionURL image.jpg

# Remove all metadata
exiftool -all= image.jpg

# Add custom metadata
exiftool -AttributionURL="https://example.com" image.jpg

# Search for text in metadata
exiftool -a -G1 image.jpg | grep "picoCTF"
```

---

## Terminal Commands
```bash
# Navigate to downloads
cd /media/sf_downloads

# Check ZIP file metadata
exiftool unknown.zip

# Extract ZIP file
unzip unknown.zip

# Check image metadata (MAIN SOLUTION)
exiftool ukn_reality.jpg

# Look for the Attribution URL field with Base64
exiftool ukn_reality.jpg | grep "Attribution"

# Decode the Base64 string
echo "cGljb0NURntNRTc0RDQ3QV9ISUREM05fYTZkZjhkYjh9Cg==" | base64 -d
```

---

## Alternative Methods

### Method 1: Search All Metadata
```bash
# View all metadata and search for picoCTF
exiftool ukn_reality.jpg | grep -i "pico"
```

### Method 2: Check for Base64 Patterns
```bash
# Look for Base64-like strings in metadata
exiftool ukn_reality.jpg | grep "=="
```

### Method 3: Use strings Command
```bash
# Extract all strings from the file
strings ukn_reality.jpg | grep "picoCTF"
```

---

## Flag

`picoCTF{ME74D47A_HIDD3N_a6df8db8}`

---

## Tools Used

* **exiftool** - Read and write metadata in files
* **unzip** - Extract ZIP archives
* **base64** - Decode Base64-encoded data

---

## Key Takeaways

* Always check image metadata with `exiftool` in forensics challenges
* Metadata can contain hidden information or clues
* The hint "How can you view the information about the picture?" points directly to metadata
* Base64 encoding is commonly used to encode data in metadata fields
* Custom metadata fields (like "Attribution URL") can store arbitrary data
* Real-world privacy concern: Images often contain more information than visible in the picture itself
* Always strip metadata (using `exiftool -all=`) before sharing sensitive photos
* The challenge name "CanYouSee" hints that you need to "see" hidden metadata

