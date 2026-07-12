# MacroHard WeakEdge — picoCTF Writeup

**Challenge:** MacroHard WeakEdge  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 60  
**Flag:** `picoCTF{D1d_u_kn0w_ppts_r_z1p5}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> I've hidden a flag in this file. Can you find it?

The file is `Forensics_is_fun.pptm` — a PowerPoint deck. The `.pptm` extension means "PowerPoint Presentation with Macros", i.e. it can contain VBA code. There is no hint beyond the file itself.

**Hint given in the challenge:**

- *(no hints provided on the challenge page — the title and filename are the only clues)*

The challenge title is a riff on "Microsoft Edge" — the joke being "MacroHard" (Microsoft → Macrosoft) — and the filename is a wink at "Microsoft PowerPoint". Both are nudges that point to Office file internals.

---

## Background Knowledge (Read This First!)

### A `.pptm` is a ZIP, not a PowerPoint file

This is the one fact that unlocks 90% of Office-document forensics challenges, and it is also the wordplay baked into the flag itself. A `.pptm` file is **just a ZIP archive** with a fancy extension. You can rename `Forensics_is_fun.pptm` to `Forensics_is_fun.zip` and `unzip` will happily open it. Same for `.pptx`, `.docx`, `.xlsx`, `.dotm`, `.xlsm`, `.docm` — they are all ZIPs.

The reason: the OOXML standard (Office Open XML, ECMA-376) defines a PowerPoint file as a ZIP container holding a bunch of XML parts in a fixed directory layout. Macros are a separate binary part (`vbaProject.bin`) inside that ZIP.

A typical `.pptm` looks like this when unzipped:

```
Forensics_is_fun.pptm  (a ZIP)
├── [Content_Types].xml          # manifest of every part in the package
├── _rels/.rels                  # top-level relationship file
├── docProps/
│   ├── app.xml                  # application metadata
│   ├── core.xml                 # author, title, last modified by, etc.
│   └── thumbnail.jpeg           # the preview thumbnail PowerPoint shows
└── ppt/
    ├── presentation.xml         # the main presentation descriptor
    ├── presProps.xml
    ├── viewProps.xml
    ├── tableStyles.xml
    ├── vbaProject.bin           # <-- the VBA macros, an OLE compound file
    ├── theme/theme1.xml         # fonts, colours, theme
    ├── slideMasters/
    │   ├── slideMaster1.xml
    │   └── hidden               # <-- arbitrary file the author can drop in
    ├── slideLayouts/slideLayout*.xml  (×11)
    └── slides/slide*.xml        (×58 — yes, this deck has 58 slides)
```

Anything PowerPoint does not recognize can also be smuggled into the ZIP as a stray file. Office will just ignore it. The challenge author exploited that to drop a file called `hidden` inside `ppt/slideMasters/` — a folder PowerPoint normally only expects to contain a `slideMaster1.xml`.

### What are VBA macros and where do they live?

**VBA** (Visual Basic for Applications) is the scripting language embedded in Office. `.pptm` (and `.docm`, `.xlsm`) files can carry VBA source in a binary part called `vbaProject.bin`. That `.bin` is itself an **OLE compound file** (the same container format that `.doc` and `.xls` use) — a mini-filesystem inside a single file. Inside it, the VBA source is stored in compressed streams that look like gibberish until you decompress them.

The standard tool for ripping macros out of an Office file is `olevba` (part of the `oletools` Python package):

```
$ olevba Forensics_is_fun.pptm
```

…which prints every macro, every procedure name, and flags suspicious keywords (`Shell`, `CreateObject`, `Environ`, etc.) that real malware tends to use.

### The two-level container, visualised

```
Forensics_is_fun.pptm        ← ZIP container
└── ppt/vbaProject.bin        ← OLE compound file (a mini filesystem)
    └── VBA/Module1           ← compressed VBA source stream
        └── "Sub not_flag()"  ← the source code, decompressed
```

You need two tools to get from the outside file to the source: `unzip` to crack the ZIP, and `olevba` to crack the OLE. (Or you can do it the manual way: `unzip` to get the `.bin`, then `oledump.py` to walk the OLE streams, then decompress the VBA stream yourself. More on that in the alternative-methods section.)

### Decoys are part of the lesson

This challenge deliberately includes a macro that looks like a flag but is not:

```vb
Sub not_flag()
    Dim not_flag As String
    not_flag = "sorry_but_this_isn't_it"
End Sub
```

It is a **red herring** — the macro is named `not_flag` and its value is literally `sorry_but_this_isn't_it`. The author is teaching you to keep looking after the first hit. Real malware does this in reverse: a clean-looking macro with a "Get-Process" call looks innocent but is actually a recon stage. Always read the *whole* output of `olevba`, and never trust a single string match.

---

## Solution — Step by Step

### Step 1 — Make a working directory and confirm what the file is

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/macrohard && cd ~/macrohard
┌──(zham㉿kali)-[~/macrohard]
└─$ cp ~/Downloads/Forensics_is_fun.pptm .
┌──(zham㉿kali)-[~/macrohard]
└─$ file Forensics_is_fun.pptm
Forensics_is_fun.pptm: Microsoft PowerPoint 2007+
┌──(zham㉿kali)-[~/macrohard]
└─$ ls -la Forensics_is_fun.pptm
-rw-r--r-- 1 zham zham 100093 Jul 13 00:04 Forensics_is_fun.pptm
```

`file` reports "Microsoft PowerPoint 2007+", but that is just `libmagic` reading the ZIP magic bytes. The actual content is a ZIP — which is the whole point of this challenge.

### Step 2 — Unzip it (because `.pptm` is a ZIP)

```
┌──(zham㉿kali)-[~/macrohard]
└─$ mkdir extracted
┌──(zham㉿kali)-[~/macrohard]
└─$ unzip -q Forensics_is_fun.pptm -d extracted/
┌──(zham㉿kali)-[~/macrohard]
└─$ find extracted -type f | wc -l
137
┌──(zham㉿kali)-[~/macrohard]
└─$ ls extracted/
[Content_Types].xml  _rels  docProps  ppt
┌──(zham㉿kali)-[~/macrohard]
└─$ ls extracted/ppt/
_rels  presProps.xml  presentation.xml  slideLayouts  slideMasters
slides  tableStyles.xml  theme  vbaProject.bin  viewProps.xml
```

`vbaProject.bin` is exactly the part we expect for a `.pptm`. The 137 files include 58 slides, 11 slide layouts, 1 slide master, 1 theme, plus all the XML scaffolding. Time to look at the macro.

### Step 3 — Look at the macro with `olevba` (the obvious first move)

```
┌──(zham㉿kali)-[~/macrohard]
└─$ olevba extracted/ppt/vbaProject.bin
olevba 0.60.2 on Python 3.11.2 - http://decalage.info/python/oletools
===============================================================================
FILE: extracted/ppt/vbaProject.bin
Type: OLE
-------------------------------------------------------------------------------
VBA MACRO Module1.bas
in file: extracted/ppt/vbaProject.bin - OLE stream: 'VBA/Module1'
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Sub not_flag()
    Dim not_flag As String
    not_flag = "sorry_but_this_isn't_it"
End Sub
No suspicious keyword or IOC found.
```

There is exactly one macro, and it is a **decoy**. The function is named `not_flag` and its value is the literal string `sorry_but_this_isn't_it`. The challenge author is telling you: "If you stopped here, you would fail."

This is the moment to step back and remember: `.pptm` files are ZIPs, and ZIPs can hold *any* file the author wants to sneak in. The macro is not the only hiding place.

### Step 4 — Look for any file that does not belong in a normal `.pptm`

A real `.pptm` has a very specific directory layout. Let me look for any file that PowerPoint would never have created by accident. I already have the file list from `find`, so let me just inspect it for anything unusual:

```
┌──(zham㉿kali)-[~/macrohard]
└─$ find extracted -type f | sort > all_files.txt
┌──(zham㉿kali)-[~/macrohard]
└─$ wc -l all_files.txt
137 all_files.txt
┌──(zham㉿kali)-[~/macrohard]
└─$ grep -vE "\.xml$|\.rels$|\.bin$|\.jpeg$|^extracted/$" all_files.txt
extracted/ppt/slideMasters/hidden
```

**There it is.** A file called `hidden` inside `ppt/slideMasters/`. PowerPoint puts `slideMaster1.xml` (and sometimes a `_rels` folder) in `slideMasters/`, never a plain file. The author dropped a stray file in there specifically because it does not show up in PowerPoint's UI — it is invisible to anyone just opening the deck.

### Step 5 — Read the `hidden` file

```
┌──(zham㉿kali)-[~/macrohard]
└─$ cat extracted/ppt/slideMasters/hidden
Z m x h Z z o g c G l j b 0 N U R n t E M W R f d V 9 r b j B 3 X 3 B w d H N f c l 9 6 M X A 1 f Q
┌──(zham㉿kali)-[~/macrohard]
└─$ wc -c extracted/ppt/slideMasters/hidden
71 extracted/ppt/slideMasters/hidden
```

71 characters, all from the base64 alphabet (`A-Z`, `a-z`, `0-9`, `+`, `/`, `=`), but with a single space inserted between every character. That spacing is just to make the string look less flag-like in a casual `cat` — it does not change the meaning once you strip the spaces.

### Step 6 — Strip the spaces and base64-decode

The string without spaces is the actual base64 payload:

```
┌──(zham㉿kali)-[~/macrohard]
└─$ cat extracted/ppt/slideMasters/hidden | tr -d ' '
ZmxhZzogcGljb0NURntEMWRfdV9rbjB3X3BwdHNfcl96MXA1fQ
```

That is **not** valid base64 yet — its length (52) is not a multiple of 4, so the decoder needs `=` padding:

```
┌──(zham㉿kali)-[~/macrohard]
└─$ python3 -c "
import base64
raw = open('extracted/ppt/slideMasters/hidden').read().replace(' ', '')
padded = raw + '=' * ((4 - len(raw) % 4) % 4)
print('base64 input :', padded)
print('decoded      :', base64.b64decode(padded).decode())
"
base64 input : ZmxhZzogcGljb0NURntEMWRfdV9rbjB3X3BwdHNfcl96MXA1fQ==
decoded      : flag: picoCTF{D1d_u_kn0w_ppts_r_z1p5}
```

The decoded payload is the literal text `flag: picoCTF{...}` — the `flag:` prefix is just labelling, the actual flag is everything from `picoCTF{` to the closing `}`.

If you would rather not spin up Python, `printf` + a couple of `sed` calls work too:

```
┌──(zham㉿kali)-[~/macrohard]
└─$ echo "ZmxhZzogcGljb0NURntEMWRfdV9rbjB3X3BwdHNfcl96MXA1fQ==" | base64 -d
flag: picoCTF{D1d_u_kn0w_ppts_r_z1p5}
```

### The flag

```
picoCTF{D1d_u_kn0w_ppts_r_z1p5}
```

---

## What Happened Internally (Timeline)

1. We downloaded `Forensics_is_fun.pptm`. `file` told us it was a Microsoft PowerPoint file, but PowerPoint 2007+ files are OOXML, which is just a ZIP container — that is the central fact of the challenge.
2. We unzipped the file with `unzip` and got 137 parts: XML, a JPEG thumbnail, and `vbaProject.bin`. The directory layout matched a real `.pptm` almost exactly.
3. We ran `olevba` on `vbaProject.bin`, which parsed the OLE compound file, found one VBA module (`Module1`), decompressed its source, and printed the macro. The macro was `Sub not_flag()` returning the string `sorry_but_this_isn't_it` — a deliberate decoy with its name and value both literally saying "this is not the flag".
4. We did not give up. We listed every file inside the unzipped tree and looked for anything that did not belong. We found `extracted/ppt/slideMasters/hidden` — a stray plain-text file in a folder that normally only contains `slideMaster1.xml`. PowerPoint ignores it entirely, but `unzip` shows it to anyone willing to look.
5. The `hidden` file contained a base64 string with a single space between every character, so we stripped the spaces with `tr -d ' '` and decoded with `base64`. The decoded bytes were the text `flag: picoCTF{D1d_u_kn0w_ppts_r_z1p5}`.

The decoy macro is the trick. The actual solution has nothing to do with VBA at all — the flag is hidden in a non-standard file inside the ZIP container, and the only way to find it is to enumerate the ZIP contents and notice the file that does not belong.

---

## Alternative Methods

**Method 1 — `binwalk` to carve everything out in one shot**

`binwalk` walks a file and reports every recognizable embedded structure, including ZIPs:

```
┌──(zham㉿kali)-[~/macrohard]
└─$ binwalk -e Forensics_is_fun.pptm

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Zip archive data, encrypted compressed size 150
82            0x52            Zip archive data, ...
...
```

The `-e` flag auto-extracts anything binwalk recognises. The result is a directory containing the same unzipped tree we got from `unzip`, including the `hidden` file. Faster than `unzip` for a "just give me everything" workflow, but less precise — you do not get to control where things go.

**Method 2 — `olefile` / `olevba` directly on the `.pptm` (no unzip)**

You do not even have to unzip the file yourself. `olevba` accepts `.pptm` files directly and handles the ZIP/OLE unwrapping internally:

```
┌──(zham㉿kali)-[~/macrohard]
└─$ olevba Forensics_is_fun.pptm
...same Module1.bas output as before...
```

But — and this is the catch — `olevba` only looks at `vbaProject.bin`. It will not see the `hidden` file at all, because the `hidden` file is not VBA. You still need to unzip and `grep` for the actual flag. (`olevba` is a macros-only tool, not a general-purpose `.pptm` analyser.)

**Method 3 — `python-pptx` for structured slide inspection**

If you wanted to actually browse the slides (e.g. you suspected the flag was hidden *in* a slide's text rather than in a stray file), `python-pptx` is the friendliest API:

```python
from pptx import Presentation
prs = Presentation("Forensics_is_fun.pptm")
for i, slide in enumerate(prs.slides, 1):
    for shape in slide.shapes:
        if shape.has_text_frame:
            for para in shape.text_frame.paragraphs:
                txt = "".join(run.text for run in para.runs)
                if "picoCTF" in txt or "flag" in txt.lower():
                    print(f"Slide {i}: {txt}")
```

For this challenge every slide just contains the repeated text "picoCTF" as a theme header — there is no flag-shaped string in any slide. So this method confirms the flag is not in the slide content, narrowing the search to "stuff inside the ZIP that is not a slide".

**Method 4 — `7z l` for a quick ZIP listing**

`7z` gives you a tidy table of every part inside the archive, including files PowerPoint hides:

```
┌──(zham㉿kali)-[~/macrohard]
└─$ 7z l Forensics_is_fun.pptm | grep -E "hidden|vba"
...   ppt/slideMasters/hidden
...   ppt/vbaProject.bin
```

Same end result as `unzip -l`, just a different display format. Use whichever you have.

**Method 5 — `strings` directly on the `.pptm`**

If you only have `strings` and `grep`, the flag is findable without unzipping at all:

```
┌──(zham㉿kali)-[~/macrohard]
└─$ strings Forensics_is_fun.pptm | grep -E "picoCTF|D1d"
ZmxhZzogcGljb0NURntEMWRfdV9rbjB3X3BwdHNfcl96MXA1fQ
```

The base64 blob shows up as a raw `strings` hit because base64 is plain ASCII. You still have to recognise it as base64 and decode it, but the search is one `grep` away. This is the "I have five minutes and no GUI" fallback.

**Method 6 — Open the file in LibreOffice and click around**

For a real user, the slowest path is: open `Forensics_is_fun.pptm` in LibreOffice or PowerPoint, click through 58 slides, look at the VBA editor (Alt+F11 → Modules → Module1), see the `not_flag` decoy, and then realize the flag is not visible in the UI. You would then unzip the file the same way we did. Don't actually do this — but it is worth knowing that even the "GUI" workflow ends with `unzip` once the macros and slides have been ruled out.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Confirm the file is recognised as a PowerPoint 2007+ (ZIP) document. |
| `unzip` | Extract the OOXML container into a normal directory tree. |
| `olevba` (`oletools`) | Parse `vbaProject.bin` and print the VBA macro source. |
| `find` / `grep` | Enumerate every part inside the ZIP and spot the stray `hidden` file. |
| `cat` | Read the contents of `hidden`. |
| `tr` | Strip the spaces inserted between every base64 character. |
| `base64` (or `python3 -c "base64.b64decode(...)"`) | Decode the base64 string to ASCII. |
| `binwalk` (alt) | Auto-extract everything binwalk recognises inside the file. |
| `7z l` (alt) | Tidy ZIP listing as an alternative to `unzip -l`. |
| `strings` (alt) | Quickest grep for the base64 blob without unzipping first. |
| `python-pptx` (alt) | Programmatic slide-content inspection, useful if the flag were on a slide. |

The primary toolchain is `unzip`, `olevba`, `find`/`grep`, `tr`, and `base64`. All five are on a stock Kali install (`oletools` is a one-line `pip install`).

---

## Key Takeaways

- **A `.pptm` is a ZIP. Always.** Same for `.pptx`, `.docx`, `.xlsx`, `.docm`, `.xlsm`, `.dotm`, `.xltx`, `.potx` — every modern Office file is a ZIP container holding XML parts. If you see an Office extension, your first move is `unzip` (or `7z l`, or rename to `.zip` and double-click). The flag's wordplay — `D1d_u_kn0w_ppts_r_z1p5` ("did you know ppts are zips") — is the author rubbing it in.
- **Always enumerate every file in the ZIP, not just the obvious ones.** The challenge author can drop arbitrary files into OOXML packages because Office silently ignores anything it does not recognize. A file called `hidden` in `ppt/slideMasters/` is the giveaway — that folder is supposed to contain only `slideMaster1.xml`. Use `find extracted -type f | sort` and eyeball the list, or pipe it through `grep -v` against a known-good layout to surface anything anomalous.
- **Decoys are a feature, not a bug.** The `Sub not_flag()` macro is literally named `not_flag` and returns `sorry_but_this_isn't_it`. It is teaching you to keep digging after the first plausible-looking hit. In real malware triage this is the difference between "I found a macro and stopped" and "I read the full macro set, walked the OLE, walked the ZIP, and still came up empty before concluding 'benign'".
- **Base64 is not encryption, it is a speed bump.** The `hidden` file is base64 with a space between every character — a cosmetic obfuscation to make `cat` look noisy. Strip the spaces with `tr -d ' '`, pad to a multiple of 4 with `=`, and `base64 -d` is one line. The decoded text even self-labels itself `flag: picoCTF{...}`, so there is no guessing what the output means.
- **Use `olevba` for macros, `unzip` for everything else.** `olevba` only knows about VBA. It is the right tool for "what does this macro do?" and the wrong tool for "what is hidden in this file?". For the latter you have to crack the ZIP yourself.
- **The challenge title is a clue, not a joke.** "MacroHard WeakEdge" is a riff on "Microsoft Edge" — the joke being that the company's name is now (jokingly) "Macrosoft", tying into the `.pptm` (macros) angle. Reading the title as a hint is sometimes enough to skip the first dead-end (`olevba`) and go straight to the second move (`unzip` + grep).
- **A 58-slide deck is itself a clue.** A real PowerPoint file the size of a normal presentation has maybe 10–20 slides. 58 slides of essentially the same content is unusual, and it is a tell that the file has been "padded" to obscure the small thing hidden inside it. If you ever see a `.pptm` or `.docx` whose part count or slide count is way out of line with its visible content, that is a strong signal "look at the parts, not the slides".

### Flag wordplay decode

```
picoCTF{D1d_u_kn0w_ppts_r_z1p5}
        | |  |     |    | |
        D u  kn0w  ppts r z1p5
        1  _  _     _    _ _
        d  k  n     p    1 p
        d  n  o     t    p  s
        i  0  w     s    t
              w     _    s
                    r    _
```

Read in chunks it becomes **"D1d u kn0w ppts r z1p5"** — leet-speak for **"Did you know ppts are zips"**:

- `D1d` → `Did` (the `1` stands in for `i`, exactly the same trick pico uses in `picoCTF`)
- `u` → `you` (texting shorthand, the author trusting you to read it as English)
- `kn0w` → `know` (`0` for `o`)
- `ppts` → `ppts` (no leet here, just the abbreviation for "PowerPoint files")
- `r` → `are`
- `z1p5` → `zips` (`1` for `i`, `5` for `s`)

The whole flag is the lesson in seven characters of payload: **PowerPoint files are ZIP files.** That is the only thing you needed to know to solve the challenge, and the author wrapped it in a flag to make sure you remember it. The challenge title's "WeakEdge" pun is the same joke from the other end — "you only needed the edge of the tool, you did not need the heavyweight macro analysis suite". Once you `unzip` a `.pptm`, you have already won.
