# Redaction gone wrong — picoCTF Writeup

Challenge: Redaction gone wrong  
Category: Forensics  
Difficulty: Medium  
Points: 100  
Flag: `picoCTF{C4n_Y0u_S33_m3_fully}`  
Platform: picoCTF 2022  
Writeup by: zham  

---

## Description

> Now you DON'T see me.
>
> This report has some critical data in it, some of which have been redacted correctly, while some were not. Can you find an important key that was not redacted properly?

## Hints

1. How can you be sure of the redaction?

## Background Knowledge

A few concepts I had to think through before I cracked this one.

**A PDF** is not a flat image. It is a small program for a page: a list of drawing instructions (a content stream) plus a list of objects (text, fonts, images, rectangles, etc.). When you open a PDF, the viewer executes those instructions in order, "draws" the page from scratch, and shows you the result.

**Redaction is supposed to destroy data.** A correct redaction permanently removes the redacted text from the PDF so it can never be recovered. In a properly redacted PDF, the text object is gone, the bytes are gone, the content stream is gone. All that is left is a black rectangle on the page.

**"Painting over" text is the classic rookie mistake.** A lot of people "redact" by opening the PDF in Word, drawing a black box on top of the text, exporting it back to PDF, and calling it a day. The text is still perfectly intact in the content stream, it is just hidden under a black rectangle. Anyone who runs `pdftotext`, opens the file in a text editor, or copies the text can still read it. This is the vulnerability behind tons of real-world leaks (court documents, government reports, police body-cam footage, you name it).

**`pdftotext`** is a CLI tool from the `poppler-utils` package. It takes a PDF, executes the content stream just like a viewer would, but instead of rasterising the page it pulls out every text object and prints it. If text is still in the PDF, `pdftotext` will find it, even if a black rectangle is sitting on top of it.

**Why this matters for the challenge.** The challenge tells us "this report has some critical data in it, some of which have been redacted correctly, while some were not". That is a huge hint that at least one piece of "redacted" text is still recoverable. The "How can you be sure of the redaction?" hint is telling me to look past the black box, not just trust that the text is gone.

## Step-by-step Solution

### Step 1: Get the file and check it is really a PDF

I already had `report.pdf` from the challenge. First, confirm what it is:

```text
┌──(zham㉿kali)-[~/ctf/redaction]
└─$ file report.pdf
report.pdf: PDF document, version 1.7, 1 pages
```

One page, plain PDF 1.7. Nothing exotic.

### Step 2: Try the obvious — just extract the text

The fastest possible move on a "redaction" challenge is to dump the text and see what is there. `pdftotext` is my default tool for this.

```text
┌──(zham㉿kali)-[~/ctf/redaction]
└─$ pdftotext report.pdf -
Financial Report for ABC Labs, Kigali, Rwanda for the year 2021.
Breakdown - Just painted over in MS word.

Cost Benefit Analysis
Credit Debit
This is not the flag, keep looking
Expenses from the
picoCTF{C4n_Y0u_S33_m3_fully}
Redacted document.
```

I did not even have to do anything fancy. The "redacted" line is just sitting there in plain text:

```
picoCTF{C4n_Y0u_S33_m3_fully}
```

The text after it (`Redacted document.`) was probably the surrounding line that did get properly redacted in the source. The flag itself was hidden under a black box but never removed from the PDF.

### Step 3: Confirm with a second method

I never trust a flag on the first hit, so I ran `pdftotext -layout` to preserve the original placement. Same flag, in the same place:

```text
┌──(zham㉿kali)-[~/ctf/redaction]
└─$ pdftotext -layout report.pdf - | grep -i pico
picoCTF{C4n_Y0u_S33_m3_fully}
```

For paranoia, I also dumped the raw PDF and grepped for the flag string. If it is in the file, this confirms it. If it is only rendered as a glyph, this will miss it.

```text
┌──(zham㉿kali)-[~/ctf/redaction]
└─$ grep -aoE 'picoCTF\{[^}]+\}' report.pdf
picoCTF{C4n_Y0u_S33_m3_fully}
```

`grep` finds it as a literal ASCII string in the PDF body, which means the text was never deleted. The "redaction" is just a black rectangle on top of live text.

### Step 4: Submit

```text
picoCTF{C4n_Y0u_S33_m3_fully}
```

## Alternative / Sanity-check Methods

A few other ways to land the same answer, in case `pdftotext` is not on the box.

### Method A: copy the text out of any PDF viewer

Open `report.pdf` in any viewer (Evince, Okular, Adobe Reader, browser). Click and drag to select across the black box. In most viewers the black box is not part of the text layer, so the cursor slips right through and the underlying text gets selected. Ctrl+C, paste into a text file, done.

### Method B: zoom in and adjust contrast

If the "redaction" was a poorly-applied opaque layer (instead of a true redaction), sometimes the original text is still visible at high zoom, or by sliding the brightness/contrast in an image editor. Open the file in GIMP or another image editor that can import a PDF page, zoom to 400% or 800%, and slide contrast to maximum. I did not need this for the picoCTF challenge because the text extraction was clean, but I have used it on real leaked PDFs that were "redacted" with a low-opacity marker.

### Method C: use a different PDF parser

`pdf2txt.py` (from `pdfminer.six`) gives the same answer if `pdftotext` is missing:

```text
┌──(zham㉿kali)-[~/ctf/redaction]
└─$ pdf2txt.py report.pdf | grep -i pico
picoCTF{C4n_Y0u_S33_m3_fully}
```

### Method D: look at the raw PDF objects

If I wanted to be extra sure, I could open the PDF in a text editor and look for the text inside a `Tj` or `TJ` operator in the content stream. PDF stores text as PostScript-style string commands, e.g. `(picoCTF{C4n_Y0u_S33_m3_fully}) Tj`. The `grep` step above already proved it, so this is just for the curious:

```text
┌──(zham㉿kali)-[~/ctf/redaction]
└─$ grep -aoE '\([^)]*pico[^)]*\)' report.pdf
(picoCTF{C4n_Y0u_S33_m3_fully})
```

## What Happened Internally

A rough timeline of the original report and the "redaction" that was done to it:

- The author wrote a financial report for ABC Labs in Kigali, Rwanda for the year 2021. Real text, real layout.
- Two lines on the page contained sensitive info. One was the flag line (`Expenses from the picoCTF{C4n_Y0u_S33_m3_fully}`), the other was something else (probably a real credit card number, account number, or key in the original challenge premise).
- The author opened the PDF in MS Word and "redacted" the text by drawing a black box on top of each sensitive line, then exported the result back to PDF. This is what the description means by "Just painted over in MS word".
- When Word exported the PDF, it added a black rectangle (an `f` filled path or an image) on top of the page. It did not delete the text underneath. The text object, the font, and the position are all still in the content stream.
- `pdftotext` ignores the rectangle (it is not a text object) and reads the text underneath. Flag recovered.
- The text "This is not the flag, keep looking" right above the flag is a classic picoCTF red herring. It is there to make sure you do not just submit the first plausible-looking string. Real redactions would have hidden that line too, but in this challenge they left it visible on purpose.

## Tools Used

| Tool           | Why I used it                                                                              |
|----------------|--------------------------------------------------------------------------------------------|
| `file`         | Confirmed the file is a real PDF and not renamed, encrypted, or wrapped.                  |
| `pdftotext`    | Dumped the text layer in one command. Pulled the flag straight out from under the redaction. |
| `pdftotext -layout` | Preserved the original layout, sanity check that the text was in the right place.     |
| `grep`         | Quick proof that the flag string lives in the PDF as a literal, not as glyphs.            |
| `pdf2txt.py`   | Backup method, in case `pdftotext` is not installed (e.g. on a minimal container).         |

## Key Takeaways

- A "redaction" is only a redaction if the text is actually deleted. Anything that just covers the text (a black rectangle, a white box, a low-opacity marker, an image) is a leak waiting to happen.
- This is a real-world bug. The most common offender is converting a PDF to Word, drawing a box, and converting back. The original text is fully recoverable with `pdftotext`, copy-paste, or even just selecting through the box.
- For picoCTF and similar CTFs, the first move on any "redaction" challenge is `pdftotext <file> - | less`. If the text is still in the PDF, you will see it. If the text really is gone, you move on to other tricks (image-layer OCR, font subset inspection, etc.).
- Always sanity-check a flag with a second method. In this case I confirmed with `pdftotext -layout` and with `grep` on the raw PDF. Both agreed.
- And the wordplay: the flag is `picoCTF{C4n_Y0u_S33_m3_fully}`, which reads "Can you see me fully". The challenge title "Now you DON'T see me" and the description "Can you find an important key that was not redacted properly?" are both playing on the same joke, you cannot see the redacted text, but the PDF can, and so can you, with the right tool.
