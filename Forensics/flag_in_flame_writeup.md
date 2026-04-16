# Flag in Flame — picoCTF Writeup

**Challenge:** Flag in Flame  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{forensics_analysis_is_amazing_b9ac4cb9}`  

---

## Description

> The SOC team discovered a suspiciously large log file after a recent breach. When they opened it, they found an enormous block of encoded text instead of typical logs. Could there be something hidden within? Your mission is to inspect the resulting file and reveal the real purpose of it.
> Download the encoded data here: Logs Data.

---

## Background Knowledge (Read This First!)

### What is Base64?

Base64 converts binary data into readable ASCII text. A suspiciously large Base64 blob in a log file is a classic sign of data being hidden or exfiltrated.

### What is OCR?

OCR (Optical Character Recognition) extracts text from images. It's useful when a flag is displayed visually in an image and you want to avoid manual transcription errors with similar-looking characters (0 vs O, 5 vs S, etc.).

---

## Solution — Step by Step

### Step 1 — Decode the Base64 Log File

The file contained a large Base64-encoded string. I decoded it:

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ base64 -d logs.txt > output
```

### Step 2 — Identify the File Type

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file output
output: PNG image data, 896 x 1152, 8-bit/color RGB, non-interlaced
```

✅ The decoded data was a PNG image!

### Step 3 — Rename and View the Image

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ mv output image.png

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ xdg-open image.png
```

The image showed a hacker figure surrounded by computer screens, with a long **hexadecimal string** at the bottom.

### Step 4 — Check for Hidden Data

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool image.png
# No flag found in metadata

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings image.png | grep -i pico
# No results
```

The flag was only in the visible hex string displayed in the image.

### Step 5 — Use OCR to Extract the Hex String

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo apt install tesseract-ocr

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ tesseract image.png output

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat output.txt
7069636F4354467B666F72656E736963735F616E616C797369735F69735F616D617A696E675F62396163346362397D
```

### Step 6 — Decode the Hex String

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "7069636F4354467B666F72656E736963735F616E616C797369735F69735F616D617A696E675F62396163346362397D" | xxd -r -p
picoCTF{forensics_analysis_is_amazing_b9ac4cb9}
```

Got the flag! 🎯

---

## What Happened Step by Step

1. **Base64 Encoding** — The log file contained Base64-encoded data that decoded into a PNG image
2. **Visual Steganography** — The flag was hidden in plain sight as a hexadecimal string displayed in the image
3. **OCR** — Used to extract the hex string without manual transcription errors

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `base64` | Decode Base64 encoded data |
| `file` | Identify file types |
| `exiftool` | Extract metadata from files |
| `tesseract-ocr` | OCR tool to extract text from images |
| `xxd` | Convert hexadecimal to binary/text |

---

## Key Takeaways

- Always decode Base64 data first when dealing with encoded logs or suspicious files
- Use OCR tools instead of manual transcription to avoid errors when reading hex from images
- Check file types after decoding — Base64 can hide any file type, not just text
- Visual steganography can hide flags in plain sight — sometimes the answer is literally displayed in the image
