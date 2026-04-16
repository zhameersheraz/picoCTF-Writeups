# CanYouSee — picoCTF Writeup

**Challenge:** CanYouSee  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{ME74D47A_HIDD3N_a6df8db8}`  

---

## Description

> How about some hide and seek?
> Download this file here.

**Hint shown in challenge:** `How can you view the information about the picture?`

---

## Background Knowledge (Read This First!)

### What is EXIF Metadata?

EXIF metadata stores information about image files — camera settings, GPS location, copyright, and custom fields. Tools like `exiftool` can read and write this metadata. Attackers (and CTF challenge makers!) can hide data inside custom metadata fields.

Common EXIF fields:
- Camera make and model
- Date/time taken
- GPS coordinates
- Copyright information
- Custom fields (like "Attribution URL" — where the flag was hidden here)

---

## Solution — Step by Step

### Step 1 — Check the ZIP File

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool unknown.zip
File Name                       : unknown.zip
File Type                       : ZIP
Zip File Name                   : ukn_reality.jpg
```

### Step 2 — Extract the ZIP File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip unknown.zip
Archive:  unknown.zip
  inflating: ukn_reality.jpg
```

### Step 3 — Check the Image Metadata

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool ukn_reality.jpg
...
Attribution URL                 : cGljb0NURntNRTc0RDQ3QV9ISUREM05fYTZkZjhkYjh9Cg==
...
```

Found Base64-encoded data in the "Attribution URL" field!

### Step 4 — Decode the Base64 String

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "cGljb0NURntNRTc0RDQ3QV9ISUREM05fYTZkZjhkYjh9Cg==" | base64 -d
picoCTF{ME74D47A_HIDD3N_a6df8db8}
```

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `exiftool` | Read image metadata |
| `unzip` | Extract ZIP archives |
| `base64 -d` | Decode Base64-encoded data |

---

## Key Takeaways

- Always check image metadata with `exiftool` in forensics challenges
- Custom metadata fields (like "Attribution URL") can store arbitrary hidden data
- Metadata can contain hidden information or clues that are completely invisible when opening the image normally
- Always strip metadata (`exiftool -all=`) before sharing sensitive photos
