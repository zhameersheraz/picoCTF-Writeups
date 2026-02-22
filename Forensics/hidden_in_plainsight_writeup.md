# Hidden in plainsight - picoCTF Writeup

**Challenge:** Hidden in plainsight  
**Category:** Forensics  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{h1dd3n_1n_1m4g3_1c55ccd0}`

---

## Description
This challenge provides a JPG image file with a hidden payload tucked inside.  
The task is to discover and extract the hidden flag from the image file.

---

## Approach
Image files can hide data using steganography techniques.  
The challenge title "Hidden in plainsight" suggested something was embedded in the image.  
I needed to check the image metadata first, then look for steganographic tools.

---

## Solution

### Step 1: Download the Image
I downloaded `img.jpg` from the challenge link provided.

### Step 2: Extract Metadata
I used `exiftool` to examine the image metadata:
```bash
exiftool img.jpg
```

**Key Output:**
```
Comment: c3RlZ2hpZGU6Y0VGNmVuZHZjbVE9
```

The Comment field contained a Base64-encoded string.

### Step 3: Decode the First Base64 Layer
I decoded the comment field:
```bash
echo c3RlZ2hpZGU6Y0VGNmVuZHZjbVE9 | base64 -d
```

**Result:**
```
steghide:cEF6endvcmQ=
```

This revealed that `steghide` was used, and another Base64 string was present (likely the password).

### Step 4: Decode the Password
I decoded the second Base64 string:
```bash
echo cEF6endvcmQ= | base64 -d
```

**Result:**
```
pAzzword
```

This was the steghide password!

### Step 5: Extract Hidden Data with Steghide
I used `steghide` to extract the hidden file from the image:
```bash
steghide --extract -p pAzzword -sf img.jpg
```

**Output:**
```
wrote extracted data to "flag.txt".
```

### Step 6: Read the Flag
I read the extracted file:
```bash
cat flag.txt
```

**Result:**
```
picoCTF{h1dd3n_1n_1m4g3_1c55ccd0}
```

---

## Why This Works
Steganography tools like `steghide` can hide files inside images without visibly altering them.  
The tool requires a password to extract the hidden data.  
In this challenge, both the steganography tool name and the password were Base64-encoded and hidden in the image's metadata.

---

## Steganography Tools Reference

| Tool | Purpose | Common Usage |
|------|---------|--------------|
| `steghide` | Hide/extract data in images | `steghide --extract -sf image.jpg -p password` |
| `stegsolve` | Analyze image layers and bits | GUI-based image analysis |
| `binwalk` | Find embedded files in images | `binwalk -e image.jpg` |
| `zsteg` | PNG/BMP steganography detection | `zsteg image.png --all` |
| `exiftool` | Read/write image metadata | `exiftool image.jpg` |

---

## Flag
`picoCTF{h1dd3n_1n_1m4g3_1c55ccd0}`

---

## Tools Used
* `exiftool` - Extract image metadata
* `base64` - Decode Base64 strings (used twice)
* `steghide` - Extract hidden data from images
* `cat` - Read file contents

---

## Key Takeaways
* Always check image metadata with `exiftool` - it often contains hints or hidden data.
* Steganography is common in forensics challenges - learn tools like `steghide`, `stegsolve`, and `binwalk`.
* Base64 encoding can be layered multiple times - decode iteratively.
* The metadata comment field is a common place to hide passwords or hints.
* Image files can look completely normal while containing hidden files inside.
