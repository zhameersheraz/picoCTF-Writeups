# Secret of the Polyglot — picoCTF Writeup

**Challenge:** Secret of the Polyglot  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{f1u3n7_1n_pn9_&_pdf_53b741d6}`  

---

## Description

> The Network Operations Center (NOC) of your local institution picked up a suspicious file, they're getting conflicting information on what type of file it is. They've brought you in as an external expert to examine the file. Can you extract all the information from this strange file?
> Download the suspicious file here.

**Hint shown in challenge:** `This problem can be solved by just opening the file in different ways`

---

## Background Knowledge (Read This First!)

### What is a Polyglot File?

A **polyglot** file (meaning "many languages") is a single file that is **valid in multiple file formats simultaneously**. Different file formats have different structures, and some formats ignore data they don't recognize. By carefully crafting a file, it can be made valid as both a PDF and a PNG at the same time.

### File Magic Bytes

| File Type | Magic Bytes (Hex) | ASCII |
|-----------|-------------------|-------|
| PDF | `25 50 44 46` | `%PDF` |
| PNG | `89 50 4E 47 0D 0A 1A 0A` | `‰PNG....` |
| JPEG | `FF D8 FF E0` | `ÿØÿà` |

---

## Solution — Step by Step

### Step 1 — Download and Check the File

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file suspicious_file
suspicious_file: PDF document, version 1.4
```

### Step 2 — Open as PDF (Find Part 1)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ evince suspicious_file
```

The PDF showed part of the flag:
```
1n_pn9_&_pdf_53b741d6}
```

I noticed "pn9" in the flag, which looks like "png" — hinting the file is also a PNG!

### Step 3 — Open as PNG (Find Part 2)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cp suspicious_file suspicious_file.png

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ eog suspicious_file.png
```

The image showed the other part of the flag:
```
picoCTF{f1u3n7_
```

### Step 4 — Combine Both Parts

```
Part from PNG: picoCTF{f1u3n7_
Part from PDF: 1n_pn9_&_pdf_53b741d6}

Complete Flag: picoCTF{f1u3n7_1n_pn9_&_pdf_53b741d6}
```

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Check file type |
| `evince` | Open as PDF |
| `eog` | Open as PNG image |
| `xxd` | Inspect magic bytes (optional) |

---

## Key Takeaways

- A polyglot file is valid in multiple file formats at the same time
- Always try opening suspicious files in different formats — not just what `file` says
- The hint "pn9" in the flag directly hints at "png" — read the flag carefully!
- File extensions do not determine file type — the actual file structure does
- The challenge name "Secret of the Polyglot" directly hints at the solution
