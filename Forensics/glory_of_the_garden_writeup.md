# Glory of the Garden - picoCTF Writeup

**Challenge:** Glory of the Garden  
**Category:** Forensics  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{more_than_m33ts_the_3y339cbe6dc}`

---

## Description

This file contains more than it seems.

Get the flag from garden.jpg.

---

## Hints

1. What is a hex editor?

---

## Solution

### Step 1: Download the File

I downloaded the file `garden.jpg` from the challenge.

### Step 2: Check the File Type

First, I verified it was actually an image:
```bash
file garden.jpg
```

**Output:**
```
garden.jpg: JPEG image data, JFIF standard 1.01
```

It's a valid JPEG image file.

### Step 3: Use strings Command

I used the `strings` command to extract all readable text from the file:
```bash
strings garden.jpg
```

**Output:**
```
JFIF
XICC_PROFILE
HLino
mntrRGB XYZ 
acspMSFT
IEC sRGB
-HP  
cprt
3desc
...
(lots of strings)
...
Here is a flag: picoCTF{more_than_m33ts_the_3y339cbe6dc}
```

### Step 4: Find the Flag

At the very bottom of the strings output, I found:
```
Here is a flag: picoCTF{more_than_m33ts_the_3y339cbe6dc}
```

---

## Why This Works

* Image files (JPEG, PNG, etc.) are binary files
* Text can be hidden inside image files without affecting how they display
* The **strings** command extracts all printable ASCII strings from any file
* Hidden messages embedded in images can be found this way
* This technique is common in **steganography** (hiding data inside other files)

---

## What is the strings Command?

The `strings` command finds and prints sequences of printable characters in a file:

**Basic usage:**
```bash
strings filename
```

**Useful options:**
```bash
# Show strings of minimum length 10
strings -n 10 garden.jpg

# Search for specific text
strings garden.jpg | grep "picoCTF"

# Save output to file
strings garden.jpg > output.txt
```

---

## Alternative Method: Hex Editor

As the hint suggests, you could also use a hex editor:

1. Open `garden.jpg` in a hex editor (like `hexedit`, `ghex`, or online at https://hexed.it/)
2. Scroll to the bottom of the file
3. Look for ASCII text near the end
4. Find the flag: `Here is a flag: picoCTF{more_than_m33ts_the_3y339cbe6dc}`

**Using hexedit:**
```bash
hexedit garden.jpg
# Press Ctrl+E to go to the end
# Look for readable ASCII text
```

---

## Terminal Commands
```bash
# Navigate to downloads folder
cd /media/sf_downloads

# Check file type
file garden.jpg

# Extract strings and view all
strings garden.jpg

# Extract strings and search for flag
strings garden.jpg | grep "picoCTF"

# Extract strings and save to file
strings garden.jpg > strings_output.txt

# View the last few lines (where flag is)
strings garden.jpg | tail -20
```

---

## Flag

`picoCTF{more_than_m33ts_the_3y339cbe6dc}`

---

## Tools Used

* **strings** - Extract printable strings from binary files
* **file** - Check file type
* **Hex editor** (optional) - View raw binary data

---

## Key Takeaways

* Image files can contain hidden text data
* The `strings` command is essential for forensics challenges
* Always check files for embedded text, even if they appear to be just images
* Steganography often hides data at the end of files
* "More than meets the eye" challenges usually involve hidden data in images
* Always try basic commands (strings, file, exiftool) before complex tools
* The flag name references the phrase "more than meets the eye" (also a Transformers reference)
