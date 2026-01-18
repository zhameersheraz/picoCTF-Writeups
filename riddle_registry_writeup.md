Riddle Registry - picoCTF Writeup
Challenge: Riddle Registry
Category: Forensics
Difficulty: Easy
Points: (not specified)
Flag: picoCTF{puzzl3d_m3tadata_f0und!_3578739a}

Description
This challenge provides a PDF file that appears to contain only garbled nonsense. However, the description hints that there's a hidden treasure within the file's metadata. The goal is to extract and decode the hidden flag from the PDF's metadata.

Approach
The challenge name "Riddle Registry" and the mention of metadata immediately suggested that the flag might be hidden in the PDF's file properties rather than in the visible content. PDF files can store metadata like author name, creation date, and custom fields - perfect places to hide information.

Solution
Step 1: Download the PDF
I downloaded confidential.pdf from the challenge link provided.
Step 2: Extract Metadata
Instead of just opening the PDF to read it, I used exiftool to examine all the hidden metadata embedded in the file. This tool reveals information that's not visible when you simply view the PDF.
bashexiftool confidential.pdf
```

**Key Output:**
```
Author: cGljb0NURntwdXp6bDNkX20zdGFkYXRhX2YwdW5kIV8zNTc4NzM5YX0=
The Author field immediately stood out. It didn't look like a normal author name. It looked like encoded data.
Step 3: Identify the Encoding
I recognized this as Base64 encoding for several reasons:

It ends with = (a Base64 padding character)
It only contains characters from the Base64 alphabet: A-Z, a-z, 0-9, +, /, and =
No spaces, special symbols, or unusual characters

Base64 is commonly used in CTFs to hide text data because it's reversible and looks obfuscated to the untrained eye.
Step 4: Decode Base64
Once I confirmed it was Base64, I decoded it using the built-in base64 command in Linux:
bashecho cGljb0NURntwdXp6bDNkX20zdGFkYXRhX2YwdW5kIV8zNTc4NzM5YX0= | base64 -d
```

**Result:**
```
picoCTF{puzzl3d_m3tadata_f0und!_3578739a}
The flag was successfully revealed!

Why This Works
PDF files store metadata separately from the visible content. This metadata is often overlooked by casual users but can be easily extracted with forensics tools like exiftool. In this challenge, the flag was Base64-encoded and hidden in the Author field to make it less obvious.
Key lesson: Always check file metadata in forensics challenges. It's a common hiding spot for flags.

Common Encoding Types Reference
Understanding different encoding types helps you identify what you're dealing with:
Encoding TypeCharacteristicsExampleDecode CommandBase64A-Z, a-z, 0-9, +, /, =cGljb0NURg==echo "..." | base64 -dHexadecimal0-9, a-f (or A-F)70696e6f435446echo "..." | xxd -r -pMD5 Hash32 hex characters5f4dcc3b5aa765d61d8327deb882cf99(cannot decode - one-way hash)BinaryOnly 0 and 101110000 01101001Custom conversionROT13Letters shifted 13 positionscvpbPGSecho "..." | tr 'A-Za-z' 'N-ZA-Mn-za-m'URL EncodingContains %XX codespico%20CTFecho "..." | urldecode

Flag
picoCTF{puzzl3d_m3tadata_f0und!_3578739a}

Tools Used

exiftool - Extracts metadata from files (images, PDFs, documents)
base64 - Decodes Base64-encoded strings


Key Takeaways

Always check metadata first in forensics challenges. Use tools like exiftool, strings, or pdfinfo
Learn to recognize common encodings like Base64, Hex, and ROT13 by their patterns
PDF files can hide data in metadata fields that aren't visible when you just open the file
Forensics is about looking beyond the obvious. What you see isn't always all there is

