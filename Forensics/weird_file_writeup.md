# Weird File — picoCTF Writeup

**Challenge:** Weird File  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 20  
**Flag:** `picoCTF{m4cr0s_r_d4ng3r0us}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> What could go wrong if we let Word documents run programs? (aka "in-the-clear").

The file is `weird.docm` — a Word document with macros enabled (the `m` in `.docm` stands for "macros"). The challenge title is "Weird File" and the description tells you up front that the interesting thing is what programs the document can run, and that the payload is "in-the-clear" — meaning the malicious code is stored as plain, readable text rather than encrypted or scrambled.

**Hints given in the challenge:**

- `https://www.youtube.com/watch?v=Y7ljJnLGqTQ` — a short video walking through the same kind of macro extraction the challenge wants you to do.

The hint video is a nudge toward "open the file in a macro-aware tool and look at what it actually executes". The phrase "in-the-clear" is a second hint: the macro is not obfuscated, encrypted, or packed. You will be able to read the source verbatim.

---

## Background Knowledge (Read This First!)

### A `.docm` is a ZIP, not a Word file

This is the single most important fact for this challenge, and the one that unlocks 90% of Office-document forensics. A `.docm` file is **just a ZIP archive** with a fancy extension. The `m` at the end means "macros enabled", but the container is the same OOXML format as `.docx`, `.pptx`, `.xlsx`, `.xlsm`, `.dotm`, etc.

You can prove this immediately:

```
$ file weird.docm
weird.docm: Microsoft Word 2007+
$ unzip -l weird.docm
```

`file` reports "Microsoft Word 2007+" because `libmagic` only sees the ZIP magic bytes at the start. The actual content is a ZIP holding a directory of XML parts plus a binary part called `vbaProject.bin` that contains the macros. If you ever forget this, just `unzip` the file and you will see the full layout:

```
weird.docm                 ← a ZIP container
├── [Content_Types].xml
├── _rels/.rels
├── docProps/
│   ├── app.xml
│   └── core.xml
├── customXml/
├── word/
│   ├── document.xml       ← the visible body of the doc
│   ├── vbaProject.bin     ← the macro container (OLE compound file)
│   ├── vbaData.xml        ← macro metadata (names, encryption flag)
│   ├── settings.xml
│   ├── styles.xml
│   ├── theme/theme1.xml
│   ├── fontTable.xml
│   └── webSettings.xml
└── word/_rels/
    ├── document.xml.rels
    └── vbaProject.bin.rels
```

The two parts we care about for this challenge are `word/vbaProject.bin` (the actual macro bytecode + compressed source) and `word/vbaData.xml` (a small XML manifest that lists macro names and whether each one is encrypted).

### What are VBA macros and why do they exist?

**VBA** (Visual Basic for Applications) is a scripting language Microsoft has shipped with Office since the 1990s. It lets documents run code on open (`AutoOpen`, `Document_Open`), on close (`AutoClose`), or in response to user actions like clicking a button. Macros are how Office apps expose automation, custom form behaviour, add-in glue, and — historically — how malware has spread through email attachments and downloads.

In the OOXML world, macros live in a single part: `word/vbaProject.bin`. That `.bin` is itself an **OLE compound file** (a mini-filesystem inside one file — the same container format that legacy `.doc` and `.xls` use). Inside the OLE you will find streams like:

- `_VBA_PROJECT` — the project metadata
- `dir` — the compressed module directory
- `VBA/<ModuleName>` — one compressed stream per module, holding the source code

The source streams are stored using a lightweight VBA-specific compression (RLE-style with token + copy commands). `olevba` knows how to decompress them, so you almost never need to do it yourself.

### What "in-the-clear" means here

Word and Excel give authors a checkbox in the VBA project properties called "Lock project for viewing". When that box is ticked and a password is set, the VBA bytecode is still inside the file, but the source text is encrypted with a per-file key derived from the password. Tools like `olevba` will report the macro names but not the source.

When the box is **not** ticked, the source streams are stored unencrypted. That is what "in-the-clear" means in the challenge description: the author deliberately left the source readable, and you can just print it and read it. The fastest way to confirm this for any `.docm` is to look at `word/vbaData.xml`:

```xml
<wne:mcd wne:macroName="PROJECT.THISDOCUMENT.AUTOOPEN"
         wne:name="Project.ThisDocument.AutoOpen"
         wne:bEncrypt="00" wne:cmg="56"/>
```

The `wne:bEncrypt="00"` attribute is the giveaway. `00` = not encrypted, `01` = encrypted with a project password. So as soon as you see `bEncrypt="00"`, you already know the macro source is going to come out in plain text.

### The two-level container, visualised

```
weird.docm                  ← ZIP container
└── word/vbaProject.bin     ← OLE compound file (mini-filesystem)
    └── VBA/ThisDocument    ← compressed VBA source stream
        └── "Sub runpython()
             Ret_Val = Shell(\"python -c 'print(\\\"...\\\")'\")
             ...
            End Sub"
```

To get from the outside file to the readable source you have to crack two containers in sequence: `unzip` to get past the ZIP, then `olevba` to get past the OLE and the VBA compression. `olevba` actually handles both layers for you when you point it at the `.docm` directly, but it is good to know what it is doing under the hood.

### Why `python -c` is the smoking gun

A real-world `.docm` macro almost always uses one of a small set of native Windows APIs to spawn a process:

- `Shell` (used here) — a thin wrapper around `ShellExecuteEx`
- `CreateObject("WScript.Shell")` — same engine, COM-flavoured
- `WinExec` — older, deprecated
- `Shell` + PowerShell — extremely common in modern malware

`Shell "python -c '...'"` is unusual because most attackers reach for PowerShell, `cmd /c`, or `mshta`. But it is just as effective: a `.docm` is a ZIP, and a ZIP can hold arbitrary files, so the attacker can drop a `.py` next to the document and `Shell` straight into it. In this challenge the author has inlined the Python directly with `python -c` so the entire attack fits in one macro.

### Base64, briefly

The macro in this challenge does not run a long Python script. It runs a one-liner that prints a base64 string. Base64 is an encoding, not encryption — it is just a way to represent arbitrary bytes using only `A-Z`, `a-z`, `0-9`, `+`, `/`, and `=` for padding. The decoder is one line in any language:

```python
import base64
base64.b64decode("cGljb0NURnttNGNyMHNfcl9kNG5nM3IwdXN9").decode()
# -> 'picoCTF{m4cr0s_r_d4ng3r0us}'
```

The same thing on the command line is `echo "..." | base64 -d`. Once you have read enough CTF challenges, any string that looks like `XXXX...==` and contains no spaces is base64 until proven otherwise.

---

## Solution — Step by Step

### Step 1 — Make a working directory and inspect the file

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/weird && cd ~/weird
┌──(zham㉿kali)-[~/weird]
└─$ cp ~/Downloads/weird.docm .
┌──(zham㉿kali)-[~/weird]
└─$ ls -la weird.docm
-rw-r--r-- 1 zham zham 24212 Jul 13 17:28 weird.docm
┌──(zham㉿kali)-[~/weird]
└─$ file weird.docm
weird.docm: Microsoft Word 2007+
```

`file` reports "Microsoft Word 2007+", which is what we expect for a `.docm`. The size is small (24 KB) — the document is almost certainly just a couple of paragraphs of filler plus a macro.

### Step 2 — Unzip it and look at the parts

A `.docm` is a ZIP, so we unzip it the same way we would for any other OOXML file:

```
┌──(zham㉿kali)-[~/weird]
└─$ mkdir extracted
┌──(zham㉿kali)-[~/weird]
└─$ unzip -o weird.docm -d extracted/ > /dev/null
┌──(zham㉿kali)-[~/weird]
└─$ ls extracted/
[Content_Types].xml  _rels  customXml  docProps  word
┌──(zham㉿kali)-[~/weird]
└─$ ls extracted/word/
_rels  document.xml  fontTable.xml  settings.xml  styles.xml
theme  vbaData.xml  vbaProject.bin  webSettings.xml
```

Two parts jump out: `vbaProject.bin` (the macro container) and `vbaData.xml` (the macro manifest). Both are normal for a `.docm`; the question is what is inside them.

### Step 3 — Check `vbaData.xml` to see if the macros are encrypted

The challenge says "in-the-clear", so the macros should be unencrypted. We can confirm that without running any tools, just by reading the manifest:

```
┌──(zham㉿kali)-[~/weird]
└─$ grep -oE 'wne:bEncrypt="[01]"' extracted/word/vbaData.xml
wne:bEncrypt="00"
wne:bEncrypt="00"
wne:bEncrypt="00"
```

Three macros, all with `bEncrypt="00"`. None of them are encrypted. That tells us `olevba` (or any other VBA extractor) is going to print the source verbatim.

The three macro names from the same file:

```
┌──(zham㉿kali)-[~/weird]
└─$ grep -oE 'wne:macroName="[^"]+"' extracted/word/vbaData.xml
wne:macroName="PROJECT.THISDOCUMENT.AUTOOPEN"
wne:macroName="PROJECT.THISDOCUMENT.RUNPYTHON"
wne:macroName="PROJECT.THISDOCUMENT.SIGNATURE"
```

So we are looking for three macros: `AutoOpen`, `runpython`, and `Signature`. The name `runpython` is a strong hint that whatever the macro does, it does it by spawning Python.

### Step 4 — Run `olevba` on the macro container

`olevba` is the standard tool for extracting VBA from Office files. It comes with the `oletools` Python package and accepts `.docm` files directly:

```
┌──(zham㉿kali)-[~/weird]
└─$ olevba -c extracted/word/vbaProject.bin
olevba 0.60.2 on Python 3.11.2 - http://decalage.info/python/oletools
===============================================================================
FILE: extracted/word/vbaProject.bin
Type: OLE
-------------------------------------------------------------------------------
VBA MACRO ThisDocument.cls 
in file: extracted/word/vbaProject.bin - OLE stream: 'VBA/ThisDocument'
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
Sub AutoOpen()
    MsgBox "Macros can any program", 0, "Title"
    Signature

End Sub
 
 Sub Signature()
    Selection.TypeText Text:="some text"
    Selection.TypeParagraph
    
 End Sub
 
 Sub runpython()

Dim Ret_Val
Args = """" '"""
Ret_Val = Shell("python -c 'print(\"cGljb0NURnttNGNyMHNfcl9kNG5nM3IwdXN9\")'" & " " & Args, vbNormalFocus)
If Ret_Val = 0 Then
   MsgBox "Couldn't run python script!", vbOKOnly
End If
End Sub
+----------+--------------------+---------------------------------------------+
|Type      |Keyword             |Description                                  |
+----------+--------------------+---------------------------------------------+
|AutoExec  |AutoOpen            |Runs when the Word document is opened        |
|Suspicious|Shell               |May run an executable file or a system       |
|          |                    |command                                      |
|Suspicious|vbNormalFocus       |May run an executable file or a system       |
|          |                    |command                                      |
|Suspicious|run                 |May run an executable file or a system       |
|          |                    |command                                      |
+----------+--------------------+---------------------------------------------+
```

The macro `runpython` calls `Shell` with a single command string:

```
python -c 'print("cGljb0NURnttNGNyMHNfcl9kNG5nM3IwdXN9")'
```

Read that out loud: it runs `python`, passes `-c` (run the next string as code), and the code is a single `print("...")` of a base64-looking string. The string `cGljb0NURnttNGNyMHNfcl9kNG5nM3IwdXN9` is the entire payload.

The `AutoExec` and `Suspicious: Shell` lines at the bottom are `olevba`'s heuristic flags — they are not a flag, just a warning that the macro auto-runs and uses `Shell`. They are why `olevba` exists as a category of tool: to surface the exact behaviours an analyst needs to know about.

### Step 5 — Decode the base64 string

The string is 44 characters long, all from the base64 alphabet. Decode it with the standard tool:

```
┌──(zham㉿kali)-[~/weird]
└─$ echo "cGljb0NURnttNGNyMHNfcl9kNG5nM3IwdXN9" | base64 -d
picoCTF{m4cr0s_r_d4ng3r0us}
```

Or with Python, which does not need a trailing newline handled:

```
┌──(zham㉿kali)-[~/weird]
└─$ python3 -c "import base64; print(base64.b64decode('cGljb0NURnttNGNyMHNfcl9kNG5nM3IwdXN9').decode())"
picoCTF{m4cr0s_r_d4ng3r0us}
```

Both decode to the same thing.

### Step 6 — The flag

```
picoCTF{m4cr0s_r_d4ng3r0us}
```

Note that this is the same string the macro would have *printed* if you had opened the `.docm` in Word with macros enabled and let `AutoOpen` call `runpython` for you. The challenge is showing you exactly what would have happened on a real machine, then asking you to recover the result by hand.

---

## What Happened Internally (Timeline)

1. We downloaded `weird.docm`. `file` told us it was a Microsoft Word 2007+ file, but every Office 2007+ file is OOXML, which is just a ZIP container with XML parts. That is the central fact of the challenge.
2. We unzipped the file with `unzip -o weird.docm -d extracted/` and got 17 parts: XML, the visible document body (`document.xml`), the macro container (`vbaProject.bin`), and the macro manifest (`vbaData.xml`). The directory layout was a normal `.docm` — no obviously suspicious extra files, no obvious hiding spots outside the macros.
3. We opened `vbaData.xml` and read it. Three macros were listed (`AutoOpen`, `runpython`, `Signature`), and every one had `wne:bEncrypt="00"`. That confirmed the macros were stored unencrypted — exactly what "in-the-clear" in the description was telling us.
4. We ran `olevba -c extracted/word/vbaProject.bin`, which parsed the OLE compound file, walked its streams, decompressed the VBA source, and printed every macro in full. The `runpython` macro contained a `Shell` call invoking `python -c 'print("cGljb0NURnttNGNyMHNfcl9kNG5nM3IwdXN9")'`. The `AutoExec: AutoOpen` flag at the bottom confirmed that the macro chain would run on document open, calling `Signature` (which types some filler text) and `runpython` (which spawns Python and prints a string).
5. The string `cGljb0NURnttNGNyMHNfcl9kNG5nM3IwdXN9` is 44 characters of clean base64. We decoded it with `echo ... | base64 -d` and got the flag `picoCTF{m4cr0s_r_d4ng3r0us}`.

The chain of `AutoOpen` → `Signature` → `runpython` is the actual attack flow: open the doc, drop a visible "signature" line into the body, then quietly spawn a process that exfiltrates a payload. In the real world the `print(...)` would be replaced with a network call, a file drop, or a credential stealer — the technique is the same, the target is not.

---

## Alternative Methods

**Method 1 — Run `olevba` on the `.docm` directly (skip the unzip)**

`olevba` accepts `.docm` files as input and handles the ZIP/OLE unwrapping internally, so you can skip the `unzip` step entirely:

```
┌──(zham㉿kali)-[~/weird]
└─$ olevba -c weird.docm
olevba 0.60.2 on Python 3.11.2 - http://decalage.info/python/oletools
===============================================================================
FILE: weird.docm
Type: OpenXML
...same ThisDocument.cls output as before...
```

You get the exact same macro source, no unzipping required. This is the fastest path if you are confident the file is a `.docm` and you just want the macros. (For challenges where the flag is *not* in the macros — e.g. hidden in a stray file inside the ZIP — you still need to `unzip` separately.)

**Method 2 — `mraptor` for a one-line triage**

`mraptor` (MacroRaptor) is also part of `oletools`. It does not print the source, but it does score the file's "macro risk" by looking at which macros auto-execute and which APIs they use. It is the right tool for a "is this macro malicious?" first pass:

```
┌──(zham㉿kali)-[~/weird]
└─$ mraptor weird.docm
MacroRaptor 0.56.2 - http://decalage.info/python/oletools
This is work in progress, please report issues at https://github.com/decalage2/oletools/issues
----------+-----+----+--------------------------------------------------------
Result    |Flags|Type|File
----------+-----+----+--------------------------------------------------------
WARNING  For now, VBA stomping cannot be detected for files in memory
SUSPICIOUS|A-X  |OpX:|weird.docm
...
Exit code: 20 - SUSPICIOUS
```

The `A` flag means "AutoExec" (the macro runs on open), the `X` flag means "Execute" (it spawns a process). Together they raise the file to `SUSPICIOUS` and a non-zero exit code, which is exactly the kind of signal a SOC analyst would want from a quick scan. After `mraptor` flags the file, you reach for `olevba` to read the actual source — which is the step that gives you the flag in this challenge.

**Method 3 — `strings` directly on `vbaProject.bin` (no `olevba`)**

The macro source is stored compressed, but the *string literals* (the Python command, the base64 blob, the `print(...)` text) often appear in the compressed stream as readable ASCII. `strings` picks them up with no setup:

```
┌──(zham㉿kali)-[~/weird]
└─$ strings extracted/word/vbaProject.bin | grep -E "cGl|python"
Ret_Val = Shell("python -c 'print(\"cGljb0NURnttNGNyMHNfcl9kNG5nM3IwdXN9\")'" & " " & Args, vbNormalFocus)
"cGljb0N
cGljb0NURnttNGNyMHNfcl9kNG5nM3IwdXN9
```

You see the full `Shell(...)` line and the base64 string in one `strings` hit. The `-el` flag (`strings -el`) would print little-endian 16-bit strings, which catches some additional UTF-16 fragments:

```
┌──(zham㉿kali)-[~/weird]
└─$ strings -el extracted/word/vbaProject.bin | grep -iE "cGl|python"
python
Couldn't run python script!
```

The first line is the `python` interpreter name; the second is the error message in the `If Ret_Val = 0` branch. Either way, `strings` gets you most of the way there without ever installing `oletools`.

**Method 4 — Read `vbaData.xml` to enumerate macros, then read `vbaProject.bin` with `oledump`**

If you do not have `oletools` at all, you can still make progress using only tools that come with every distro:

```
┌──(zham㉿kali)-[~/weird]
└─$ xxd extracted/word/vbaProject.bin | head -5
00000000: d0cf 11e0 a1b1 1ae1 0000 0000 0000 0000  ................
00000010: 0000 0000 0000 0000 3e00 0300 feff 0900  ..........>.....
```

`d0cf 11e0` is the OLE compound file magic header, confirming `vbaProject.bin` is an OLE file. From there, `olevba`/`oledump` is the friendly path, but if you really wanted to, you could use `olefile` from Python to walk the streams yourself and decompress the `VBA/ThisDocument` stream by hand. For this challenge, just installing `oletools` is faster.

**Method 5 — Open the file in LibreOffice and click "Tools > Macros > Edit Macros"**

For a GUI user, the slowest path is: open `weird.docm` in LibreOffice (or Word with macros enabled), accept the macro security warning, and either let `AutoOpen` run or open the macro editor (`Tools > Macros > Edit Macros` in LibreOffice, `Alt+F11` in Word). You will see the same three macros in the same order. The author deliberately disabled most of the visible document content, so most of what you see in the GUI is filler — the actual payload is in the macro source.

I do not recommend this for malware (never enable macros on a file you do not trust), but for picoCTF challenges where you control the file, it is a valid learning workflow.

**Method 6 — CyberChef "From Base64" recipe**

If you prefer a GUI, copy the string `cGljb0NURnttNGNyMHNfcl9kNG5nM3IwdXN9` into CyberChef, drop a `From Base64` block, and you get the flag. CyberChef also has a "VBA Macro Extract" recipe that wraps `olevba` and is useful for batch processing many suspicious files.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Confirm the file is recognised as a Word 2007+ document. |
| `unzip` | Extract the OOXML container into a normal directory tree. |
| `grep` | Quick scan of `vbaData.xml` to read macro names and the encryption flag. |
| `olevba` (`oletools`) | Parse `vbaProject.bin`, decompress the VBA source, and print the macros. |
| `mraptor` (`oletools`) | One-line macro risk score — "is this file suspicious?". |
| `strings` (alt) | Pull ASCII fragments directly out of `vbaProject.bin` without `oletools`. |
| `base64` / `python3 -c "base64.b64decode(...)"` | Decode the base64 string embedded in the macro. |
| `xxd` (alt) | Confirm the OLE compound file magic header at the start of `vbaProject.bin`. |
| CyberChef (alt) | GUI "From Base64" recipe if you prefer a visual decoder. |
| LibreOffice / Word macro editor (alt) | GUI view of the same three macros; not recommended for untrusted files. |

The primary toolchain is `unzip`, `olevba`, and `base64`. All three are easy to set up on a stock Kali install: `oletools` is a one-line `pip install` on top of the Python 3 that Kali already ships, and `base64` and `unzip` are part of `coreutils` and the `unzip` package respectively.

---

## Key Takeaways

- **A `.docm` is a ZIP. Always.** Same for `.docx`, `.pptx`, `.xlsx`, `.pptm`, `.xlsm`, `.dotm`, `.xltx`, `.potx` — every modern Office file is a ZIP container holding XML parts plus optional binary parts. If you see an Office extension, your first move is `unzip` (or just hand the file to `olevba`, which will unzip for you). The challenge title "Weird File" is half a joke: it is "weird" only until you remember it is a ZIP.
- **Macros are the standard "weird file" hiding spot.** A `.docm` exists specifically so authors can ship code alongside a document. The challenge is teaching you the default reflex: if the extension is `*m` (or `*.doc`, `*.xls` in the legacy world), look at the macros. `olevba` is the right tool for that first pass.
- **`bEncrypt="00"` in `vbaData.xml` means "readable source".** Before you even run `olevba`, you can read the macro manifest and know whether the source is going to come out in plain text. This is the difference between a 5-second challenge and a 5-hour crypto challenge. Always check the manifest first.
- **A `Shell` call inside a macro is a red flag.** `olevba` flags it as `Suspicious: Shell` for a reason. In a real `.docm` from the internet, `Shell "anything"` is a "stop, read the source, decide if this is safe" moment. In a CTF it is a "print, find the base64, decode it" moment. The reflex is the same: read the command, then run it in your head, then act.
- **`python -c '...'` is a one-line code drop.** Authors use it when they want to inline a payload without shipping a separate `.py` file. Same idea as `powershell -enc <base64>` in real malware — the whole attack is one short string the macro can hold. In a CTF the inline string is almost always a `print()` of a base64 blob, and the flag is one `base64 -d` away.
- **`strings` is a perfectly good first pass.** Even before installing `oletools`, `strings weird.docm | grep -E "pico|base64|flag"` is a quick triage. The string literal inside a `Shell` call is usually stored as plain ASCII, even when the surrounding VBA is compressed. If `strings` finds it, you can skip the heavy tooling.
- **"In-the-clear" is a hint, not a boast.** The challenge author is telling you up front that the macro is unencrypted. If you see that phrase in a CTF description, the solution is *going* to be: open the file, read the macro, decode whatever it prints. Do not waste time looking for steganography, hidden streams, or packed payloads when the author has explicitly told you the answer is on the surface.
- **Document the difference between this and the other macro challenges.** A `.docm` with a readable macro is *easy* (this challenge, 20 points). A `.docm` with a locked project + obfuscated strings + a P-code dropper is *hard* (see the real-world macro malware samples on Malware-Traffic-Analysis or ANY.RUN). The skills transfer — `olevba` reads both — but the time-to-flag is wildly different. Knowing the difference lets you triage quickly.

### Flag wordplay decode

```
picoCTF{m4cr0s_r_d4ng3r0us}
        |  | |  |       | |
        m  4 c  r       0 u
        a  2 r  _       r s
        c  0 o  d       e
        r  s  s  a
        o  _  _  n
        s     s  g
                 3
                 r
                 0
                 u
                 s
```

Read in chunks it becomes **"m4cr0s r d4ng3r0us"** — leet-speak for **"macros are dangerous"**:

- `m4cr0s` → `macros` (`4` for `a`, `0` for `o`, exactly the same trick pico uses in `picoCTF`)
- `r` → `are` (texting shorthand)
- `d4ng3r0us` → `dangerous` (`4` for `a`, `3` for `e`, `0` for `o`)

The whole flag is the lesson in a single sentence: **macros are dangerous**. That is the only thing you needed to know to solve the challenge — open a `.docm`, look at the macro, decode whatever the macro tries to print, and remember that the exact same code path can drop real malware on a real machine if you let macros run by default. The challenge title "Weird File" and the description "What could go wrong if we let Word documents run programs?" are both pointing at the same punchline: a Word document should *not* be able to spawn `python`, and yet here we are, in 2026, still having to tell people not to enable macros.

