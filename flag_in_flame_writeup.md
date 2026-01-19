# Flag in Flames - picoCTF Writeup

**Challenge:** Flag in Flames  
**Category:** Forensics  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{forensics_analysis_is_amazing_b9ac4cb9}`

---

## Description

The SOC team discovered a suspiciously large log file after a recent breach. When they opened it, they found an enormous block of encoded text instead of typical logs. Could there be something hidden within? Your mission is to inspect the resulting file and reveal the real purpose of it. The team is relying on your skills to uncover any concealed information within this unusual log.

Download the encoded data here: Logs Data. Be prepared - the file is large, and examining it thoroughly is crucial.

---

## Approach

The challenge provides a file called `logs.txt` containing what appears to be encoded data. Based on the hint mentioning "base64" and the need to "decode the data and generate the image file," I suspected the file contained a Base64-encoded image with the flag hidden inside it.

My strategy was to:
1. Decode the Base64 data to reveal the hidden file
2. Identify the file type
3. Extract the flag from the decoded file

---

## Solution

### Step 1: Download the Log File

I downloaded `logs.txt` from the challenge link provided.

### Step 2: Decode Base64 Data

The file contained a large Base64-encoded string. I decoded it:

```bash
base64 -d logs.txt > output
```

### Step 3: Identify the File Type

I checked what type of file was created:

```bash
file output
```

**Output:**
```
output: PNG image data, 896 x 1152, 8-bit/color RGB, non-interlaced
```

Great! The decoded data was a PNG image.

### Step 4: Rename and View the Image

I renamed the file to have a proper extension and opened it:

```bash
mv output image.png
xdg-open image.png
```

The image showed a hacker figure surrounded by computer screens, with a long hexadecimal string at the bottom.

### Step 5: Initial Failed Attempts

I initially tried to manually transcribe the hex string from the image, which led to several incorrect attempts:

```bash
# Wrong attempts due to manual transcription errors
echo "7069636f4354467b666e72656e736963735f616e616c797369735f616d617a696e675f62396163346362397d" | xxd -r -p
# Result: picoCTF{fnrensics_analysis_amazing_b9ac4cb9} - WRONG

echo "7069636f4354467b663072656e736963735f616e616c797369735f616d617a696e675f62396163346362397d" | xxd -r -p
# Result: picoCTF{f0rensics_analysis_amazing_b9ac4cb9} - WRONG
```

These attempts failed because I was misreading characters in the hex string.

### Step 6: Check for Hidden Data

I tried forensics techniques to see if there was additional hidden data:

```bash
# Check metadata
exiftool image.png
# No flag found in metadata

# Search for strings containing "pico"
strings image.png | grep -i pico
# No results

# Look for the flag pattern
strings image.png | grep "picoCTF{"
# No results
```

This confirmed the flag was only in the visible hex string, not embedded elsewhere.

### Step 7: Use OCR to Extract the Hex String

Instead of manually transcribing, I used OCR (Optical Character Recognition) to accurately extract the hex string:

```bash
# Install tesseract OCR tool
sudo apt install tesseract-ocr

# Extract text from image
tesseract image.png output

# View the extracted text
cat output.txt
```

**OCR Output:**
```
7069636F4354467B666F72656E736963735F 616E616C797369735F69735F61 6D617A696E675F 62396 163346362397D
```

### Step 8: Clean and Decode the Hex String

The OCR extracted the hex with spaces. I removed the spaces and decoded it:

```bash
echo "7069636F4354467B666F72656E736963735F616E616C797369735F69735F616D617A696E675F62396163346362397D" | xxd -r -p
```

**Result:**
```
picoCTF{forensics_analysis_is_amazing_b9ac4cb9}
```

**Flag accepted!**

---

## Why This Works

1. **Base64 Encoding**: The log file contained Base64-encoded data that decoded into a PNG image
2. **Visual Steganography**: The flag was hidden in plain sight as a hexadecimal string displayed in the image
3. **OCR Advantage**: Using OCR tools prevented manual transcription errors that occur when reading similar-looking characters (0 vs O, 5 vs S, etc.)

The key difference between my failed attempts and the correct flag:
- **Wrong**: `C5DdgFrfrnsics` (manual transcription)
- **Correct**: `forensics_analysis_is` (OCR extraction)

---

## Common Encoding Types Reference

| Encoding Type | Characteristics | Example | Decode Command |
|--------------|----------------|---------|----------------|
| Base64 | A-Z, a-z, 0-9, +, /, = | `cGljb0NURg==` | `base64 -d file.txt` |
| Hexadecimal | 0-9, a-f (or A-F) | `70696e6f435446` | `xxd -r -p` or `echo "..." \| xxd -r -p` |
| Binary | Only 0 and 1 | `01110000` | Custom conversion |

---

## Flag

`picoCTF{forensics_analysis_is_amazing_b9ac4cb9}`

---

## Tools Used

* `base64` - Decode Base64 encoded data
* `file` - Identify file types
* `exiftool` - Extract metadata from files
* `strings` - Extract readable strings from binary files
* `tesseract-ocr` - OCR tool to extract text from images
* `xxd` - Convert hexadecimal to binary

---

## Key Takeaways

* **Always decode Base64 data first** when dealing with encoded logs or suspicious files
* **Use OCR tools** instead of manual transcription to avoid errors when reading hex strings from images
* **Check file types** after decoding to understand what you're working with
* **Visual steganography** can hide flags in plain sight - sometimes the answer is literally displayed in the image
* **Hexadecimal is case-insensitive** but character accuracy is critical
* **Forensics tools** like `strings`, `exiftool`, and `file` are essential for initial reconnaissance
* **Don't assume the first hex string you see is correct** - verify with multiple methods

---

## Common OCR Commands

```bash
# Install tesseract
sudo apt install tesseract-ocr

# Basic OCR extraction
tesseract image.png output_filename

# Extract specific patterns (like hex strings)
tesseract image.png stdout | grep -E '[0-9A-Fa-f]{60,}'
```
