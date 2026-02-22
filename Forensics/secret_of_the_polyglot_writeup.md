# Secret of the Polyglot - picoCTF Writeup

**Challenge:** Secret of the Polyglot  
**Category:** Forensics  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{f1u3n7_1n_pn9_&_pdf_53b741d6}`

---

## Description

The Network Operations Center (NOC) of your local institution picked up a suspicious file, they're getting conflicting information on what type of file it is. They've brought you in as an external expert to examine the file. Can you extract all the information from this strange file?

Download the suspicious file here.

---

## Hints

1. This problem can be solved by just opening the file in different ways

---

## Solution

### Step 1: Download the File

I downloaded the suspicious file from the challenge.

### Step 2: Open as PDF (Find Part 1/2)

I opened the file as a PDF and saw part of the flag:
```
1n_pn9_&_pdf_53b741d6}
```

**Part 1/2:** `1n_pn9_&_pdf_53b741d6}`

I noticed "pn9" in the flag, which looks like "png"! This is a hint that the file might also be a PNG.

### Step 3: Open as PNG (Find Part 2/2)

I opened the same file as a PNG image (or renamed it to `.png` and opened it), and saw the other part of the flag:
```
picoCTF{f1u3n7_
```

**Part 2/2:** `picoCTF{f1u3n7_`

### Step 4: Combine Both Parts

I combined both parts:
```
Part 2: picoCTF{f1u3n7_
Part 1: 1n_pn9_&_pdf_53b741d6}
```

**Complete Flag:** `picoCTF{f1u3n7_1n_pn9_&_pdf_53b741d6}`

The flag reads: "fluent in png and pdf"!

---

## Why This Works

* A **polyglot file** is a file that is valid in multiple file formats
* This file is both a valid PDF and a valid PNG at the same time
* PDF and PNG have different file structures, so different parts of the file are displayed depending on which format is used
* When opened as PDF, it shows one part of the flag
* When opened as PNG, it shows another part of the flag
* This works because:
  - PDF readers ignore PNG-specific data
  - PNG readers ignore PDF-specific data
  - The flag is split between both formats

---

## What is a Polyglot File?

A **polyglot** (meaning "many languages") in file formats is a single file that is valid in multiple formats:

**Examples:**
- A file that is both a PDF and a PNG (this challenge)
- A file that is both a ZIP and a JPEG
- A file that is both HTML and JavaScript

**How it works:**
1. Different file formats have different structures
2. Some formats ignore data they don't recognize
3. By carefully crafting a file, you can make it valid in multiple formats
4. Each format "sees" different parts of the file

**Security implications:**
- Can bypass file upload restrictions
- Can hide malicious content
- Used in real-world attacks

---

## How to Open Files in Different Formats

### Method 1: Change File Extension
```bash
# Rename to PDF
mv suspicious_file suspicious_file.pdf
# Open with PDF viewer

# Rename to PNG
mv suspicious_file suspicious_file.png
# Open with image viewer
```

### Method 2: Use Specific Programs
```bash
# Open as PDF
evince suspicious_file
# or
firefox suspicious_file

# Open as PNG
eog suspicious_file
# or
gimp suspicious_file
```

### Method 3: Check with file Command
```bash
# Check file type
file suspicious_file

# It might show both PDF and PNG signatures
```

---

## Terminal Commands
```bash
# Download file
cd /media/sf_downloads

# Check what the file thinks it is
file suspicious_file

# Open as PDF
evince suspicious_file.pdf
# OR
firefox suspicious_file.pdf

# Rename to PNG
cp suspicious_file suspicious_file.png

# Open as PNG
eog suspicious_file.png
# OR
gimp suspicious_file.png

# View in hex to see both file signatures
xxd suspicious_file | head -20
```

---

## File Signatures (Magic Bytes)

Different file types start with specific byte sequences:

| File Type | Magic Bytes (Hex) | ASCII |
|-----------|-------------------|-------|
| **PDF** | `25 50 44 46` | `%PDF` |
| **PNG** | `89 50 4E 47 0D 0A 1A 0A` | `‰PNG....` |
| **JPEG** | `FF D8 FF E0` | `ÿØÿà` |
| **ZIP** | `50 4B 03 04` | `PK..` |

A polyglot file may contain multiple magic byte signatures.

---

## Flag

`picoCTF{f1u3n7_1n_pn9_&_pdf_53b741d6}`

---

## Tools Used

* **PDF Viewer** (evince, Firefox, Adobe Reader) - View as PDF
* **Image Viewer** (eog, GIMP, built-in viewer) - View as PNG
* **file command** (optional) - Check file type

---

## Key Takeaways

* A polyglot file is valid in multiple file formats simultaneously
* Always try opening suspicious files in different formats
* The hint "pn9" in the flag suggests "png" 
* File extensions don't determine file type - the actual file structure does
* Polyglot files can bypass security checks and file upload restrictions
* This technique is used in real-world attacks and security research
* Always check file signatures (magic bytes) to verify actual file type
* The challenge name "Secret of the Polyglot" directly hints at the solution method
