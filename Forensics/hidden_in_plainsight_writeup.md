# Hidden in Plainsight — picoCTF Writeup

**Challenge:** Hidden in Plainsight  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{h1dd3n_1n_1m4g3_1c55ccd0}`  

---

## Description

> This challenge provides a JPG image file with a hidden payload tucked inside. The task is to discover and extract the hidden flag from the image file.

---

## Background Knowledge (Read This First!)

### What is Steghide?

`steghide` is a steganography tool that can hide files **inside** images without visibly altering them. It requires a password to extract the hidden data.

### What is EXIF Metadata?

EXIF metadata stores extra information about an image file (camera settings, GPS, copyright, custom fields). Tools like `exiftool` can read and write this metadata — and attackers can hide hints or passwords inside it.

### What is Base64?

Base64 converts binary data into readable ASCII text. It is not encryption — it is easily reversible. Common indicator: string ends with `=` or `==`.

---

## Solution — Step by Step

### Step 1 — Download the Image

I downloaded `img.jpg` from the challenge link provided.

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

### Step 2 — Extract Metadata

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool img.jpg
...
Comment: c3RlZ2hpZGU6Y0VGNmVuZHZjbVE9
...
```

The Comment field contained a suspicious Base64-encoded string.

### Step 3 — Decode the First Base64 Layer

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo c3RlZ2hpZGU6Y0VGNmVuZHZjbVE9 | base64 -d
steghide:cEF6endvcmQ=
```

This revealed that `steghide` was used, and another Base64 string was the password.

### Step 4 — Decode the Password

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo cEF6endvcmQ= | base64 -d
pAzzword
```

✅ Password found: `pAzzword`

### Step 5 — Extract Hidden Data with Steghide

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ steghide --extract -p pAzzword -sf img.jpg
wrote extracted data to "flag.txt".
```

### Step 6 — Read the Flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat flag.txt
picoCTF{h1dd3n_1n_1m4g3_1c55ccd0}
```

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `exiftool` | Extract image metadata |
| `base64` | Decode Base64 strings (used twice) |
| `steghide` | Extract hidden data from images |
| `cat` | Read file contents |

---

## Key Takeaways

- Always check image metadata with `exiftool` — it often contains hints or hidden data
- Steganography is common in forensics challenges — learn tools like `steghide`, `stegsolve`, and `binwalk`
- Base64 encoding can be layered multiple times — decode iteratively
- The metadata comment field is a common place to hide passwords or hints
- Image files can look completely normal while containing hidden files inside
