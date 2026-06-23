# m1n10n'5_53cr37 — picoCTF Writeup

**Challenge:** m1n10n'5_53cr37   
**Category:** Reverse Engineering   
**Difficulty:** Medium    
**Points:** 300   
**Flag:** `picoCTF{1t_w4sn7_h4rd_unr4v3l1n9_th3_m0b1l3_c0d3}`   
**Platform:** picoMini by CMU-Africa (2025)   
**Writeup by:** zham   

---

## Description

> Get ready for a mischievous adventure with your favorite Minions! They've been up to their old tricks, and this time, they've hidden the flag in a devious way within the Android source code. Your task is to channel your inner Minion and dive into the disassembled or decompiled code. Watch out, because these little troublemakers have hidden the flag in multiple sneaky spots or maybe even pulled a fast one and concealed it in the same location!
>
> Put on your overalls, grab your magnifying glass, and get cracking. The Minions have left clues, and it's up to you to follow their trail and uncover the flag. Can you outwit these playful pranksters and find their secret? Let the Minion mischief begin!
>
> Find the android apk here **Minions Mobile Application** and try to get the flag.

## Hints

> 1. Do you know how to disassemble an apk file?
> 2. Any interesting source files?

---

## Background Knowledge (Read This First!)

### What is an APK?

An **APK (Android Package Kit)** is the file format Android uses to distribute and install apps. Think of it like a `.zip` file for Android. Under the hood, an APK is actually just a ZIP archive with a specific structure:

```
Minions.apk
├── AndroidManifest.xml    ← App info, permissions, components
├── classes.dex            ← Compiled Java/Kotlin code (Dalvik bytecode)
├── classes2.dex           ← More compiled code (apps can have multiple)
├── classes3.dex           ← Even more compiled code
├── res/                   ← Resources (images, layouts, strings)
│   ├── layout/            ← UI XML files
│   ├── values/
│   │   └── strings.xml    ← All user-facing strings
│   └── ...
├── resources.arsc         ← Compiled resource table
└── META-INF/              ← Signing info
```

### Why is the Flag Hidden in an APK?

When an app ships to users, the Java/Kotlin source code is compiled into **DEX (Dalvik Executable)** bytecode. This bytecode can be **decompiled** back into nearly-readable Java. Anything stored in the app — strings, secrets, URLs — can be recovered by an attacker with the right tools. This challenge shows exactly that: a "secret" was hidden in plain sight inside the resource files.

### Tools You'll See in This Writeup

- **`apktool`** — Disassembles an APK into readable XML and smali code (a human-readable form of DEX bytecode)
- **`unzip`** — A normal Linux tool that opens the APK because it's just a ZIP
- **`strings`** — Pulls printable text out of binary files (useful for a quick win)
- **`base32`** — A decoder for Base32-encoded text (uses the alphabet `A-Z` and digits `2-7`)

### Base32 vs Base64 — How to Tell the Difference

Both encodings turn binary data into text so it can be stored in plain string fields. They look similar at a glance:

- **Base64** uses `A-Z`, `a-z`, `0-9`, `+`, `/` and ends with `=` or `==`
- **Base32** uses only uppercase `A-Z` and digits `2-7`, ends with `=` (the value usually looks "all caps")

If you ever see a string that's all uppercase letters and ends in `=`, **try Base32 first**.

---

## Solution — Step by Step

### Step 1 — Download the APK

I downloaded `Minions.apk` from the picoCTF challenge page and moved it into my working directory.

```
┌──(zham㉿kali)-[~/picoCTF/m1n10n]
└─$ ls
Minions.apk
```

### Step 2 — Try a Quick `strings` Sweep

Before reaching for apktool, I always run `strings` first. It dumps every printable text blob out of the binary. Sometimes the flag just falls out.

```
┌──(zham㉿kali)-[~/picoCTF/m1n10n]
└─$ strings Minions.apk | grep -i "picoCTF"
```

Nothing matched. The flag is encoded, so it doesn't appear as the literal text `picoCTF{...}`. Good to know — that confirms the flag is hidden behind some kind of encoding.

### Step 3 — Open the APK Like a ZIP

The challenge hint asks: *"Do you know how to disassemble an apk file?"* The simplest first step is to remember that an APK **is a ZIP file**. `unzip` will do.

```
┌──(zham㉿kali)-[~/picoCTF/m1n10n]
└─$ unzip Minions.apk -d apk_unzipped
Archive:  Minions.apk
  inflating: apk_unzipped/AndroidManifest.xml
  inflating: apk_unzipped/classes.dex
  inflating: apk_unzipped/classes2.dex
  inflating: apk_unzipped/classes3.dex
  inflating: apk_unzipped/resources.arsc
   creating: apk_unzipped/META-INF/
  inflating: apk_unzipped/META-INF/MANIFEST.MF
  inflating: apk_unzipped/META-INF/CERT.RSA
  inflating: apk_unzipped/META-INF/CERT.SF
   creating: apk_unzipped/res/
   creating: apk_unzipped/res/drawable/
  inflating: apk_unzipped/res/drawable/ic_launcher.png
   creating: apk_unzipped/res/layout/
  inflating: apk_unzipped/res/layout/activity_main.xml
   creating: apk_unzipped/res/values/
  inflating: apk_unzipped/res/values/strings.xml
   creating: apk_unzipped/res/mipmap-hdpi-v4/
  inflating: apk_unzipped/res/mipmap-hdpi-v4/ic_launcher.png
  ...
```

### Step 4 — Hunt for the Hint in the Layout XML

The challenge description said *"Look into me — my Banana value is interesting"* is a string sitting in the layout. Let me check that:

```
┌──(zham㉿kali)-[~/picoCTF/m1n10n]
└─$ grep -i "banana" apk_unzipped/res/layout/activity_main.xml
```

Hmm, the layout file is in Android's binary XML format (not readable as text). Time for the bigger hammer.

### Step 5 — Use `apktool` to Disassemble Properly

`apktool` decodes the binary XML files into normal text and disassembles the DEX into smali. It's the standard tool for this job.

```
┌──(zham㉿kali)-[~/picoCTF/m1n10n]
└─$ apktool d Minions.apk -o apktool_out
I: Using Apktool 2.9.3
I: Loading resource table...
I: Decoding AndroidManifest.xml
I: Loading resource table...
I: Decoding resources & values...
I: Baksmaling classes.dex...
I: Baksmaling classes2.dex...
I: Baksmaling classes3.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```

### Step 6 — Find the Banana Value in `strings.xml`

Now the resource files are readable. Time to look at `strings.xml`, where Android stores every user-facing string.

```
┌──(zham㉿kali)-[~/picoCTF/m1n10n]
└─$ ls apktool_out/res/values/
colors.xml  dimens.xml  ic_launcher_background.xml  strings.xml  styles.xml

┌──(zham㉿kali)-[~/picoCTF/m1n10n]
└─$ cat apktool_out/res/values/strings.xml
```

**Output:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">Minions Mobile Application</string>
    <string name="Banana">OBUWG32DKRDHWMLUL53TI43OG5PWQNDSMRPXK3TSGR3DG3BRNY4V65DIGNPW2MDCGFWDGX3DGBSDG7I=</string>
</resources>
```

There it is — a string resource called `Banana` with a long uppercase value that ends in `=`. That screams **Base32** (not Base64 — Base64 would usually have a mix of cases and would end in `==`).

### Step 7 — Decode the Base32 Value

The challenge says the flag is hidden in the mobile code, and the description hints at the encoding. Let me decode it.

```
┌──(zham㉿kali)-[~/picoCTF/m1n10n]
└─$ echo "OBUWG32DKRDHWMLUL53TI43OG5PWQNDSMRPXK3TSGR3DG3BRNY4V65DIGNPW2MDCGFWDGX3DGBSDG7I=" | base32 -d
picoCTF{1t_w4sn7_h4rd_unr4v3l1n9_th3_m0b1l3_c0d3}
```

Got the flag.

```
picoCTF{1t_w4sn7_h4rd_unr4v3l1n9_th3_m0b1l3_c0d3}
```

---

## Alternative Method — Python Script

For those who prefer Python, here's the same decode using `base64.b32decode()`. I'll write a small script that pulls the Banana value straight out of `strings.xml` so the whole pipeline is repeatable.

### Step 1 — Create the Script

```
┌──(zham㉿kali)-[~/picoCTF/m1n10n]
└─$ nano decode_banana.py
```

Paste this:

```python
#!/usr/bin/env python3
"""
Pull the Banana string from strings.xml and Base32-decode it.
Usage: python3 decode_banana.py <path-to-strings.xml>
"""
import base64
import re
import sys
from pathlib import Path


def extract_banana(path: Path) -> str:
    text = path.read_text(encoding="utf-8")
    match = re.search(r'name="Banana">([^<]+)<', text)
    if not match:
        raise SystemExit(f"[!] No <string name=\"Banana\"> found in {path}")
    return match.group(1)


def main() -> None:
    if len(sys.argv) != 2:
        print("Usage: python3 decode_banana.py <strings.xml>", file=sys.stderr)
        sys.exit(1)

    xml_path = Path(sys.argv[1])
    banana = extract_banana(xml_path)
    print(f"[*] Raw Banana value : {banana}")

    # Base32 needs uppercase, no extra padding tricks
    padded = banana.upper().rstrip("=")
    padded += "=" * ((8 - len(padded) % 8) % 8)
    flag = base64.b32decode(padded).decode("utf-8")
    print(f"[+] Decoded flag     : {flag}")


if __name__ == "__main__":
    main()
```

Save with `Ctrl+O`, `Enter`, then `Ctrl+X`.

### Step 2 — Run the Script

```
┌──(zham㉿kali)-[~/picoCTF/m1n10n]
└─$ python3 decode_banana.py apktool_out/res/values/strings.xml
[*] Raw Banana value : OBUWG32DKRDHWMLUL53TI43OG5PWQNDSMRPXK3TSGR3DG3BRNY4V65DIGNPW2MDCGFWDGX3DGBSDG7I=
[+] Decoded flag     : picoCTF{1t_w4sn7_h4rd_unr4v3l1n9_th3_m0b1l3_c0d3}
```

Same flag, automated.

---

## What Happened Internally (Timeline)

1. The challenge served an APK called `Minions.apk`.
2. The APK contains `res/values/strings.xml`, which holds all user-visible strings.
3. The developer tucked the flag into a `<string name="Banana">` entry but **encoded** it in Base32 so a naive `strings` search wouldn't catch the literal `picoCTF{...}` text.
4. Disassembling the APK with `apktool` decoded the binary XML into a readable text file, exposing the Banana value.
5. Recognizing the value as Base32 (uppercase only, ends in `=`), I decoded it with `base32 -d` (or `base64.b32decode()` in Python).
6. The decoded bytes spelled out the flag: `picoCTF{1t_w4sn7_h4rd_unr4v3l1n9_th3_m0b1l3_c0d3}`.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip` | Opened the APK as a ZIP archive to peek at its contents |
| `strings` | Searched the raw APK for the literal flag text (came up empty, as expected) |
| `apktool` | Disassembled the APK and decoded the binary `strings.xml` into readable XML |
| `grep` | Filtered `strings.xml` for the `Banana` key |
| `base32` | Decoded the Banana value back into the flag |
| Python 3 + `base64` (alt) | Alternative scripted version that parses `strings.xml` and decodes in one go |

---

## Key Takeaways

- An APK is just a ZIP file. You can open it with `unzip`, but the manifest and resource XMLs are stored in a binary format that needs a tool like `apktool` or `jadx` to read properly.
- Always look at `res/values/strings.xml` first when reversing an Android app — that's where every user-facing string lives.
- Base32 vs Base64: an all-uppercase string ending in `=` is almost always Base32. Mix of upper/lowercase with `+` and `/` is Base64.
- A quick `strings <file> | grep picoCTF` is a great first sweep — if it doesn't hit, the flag is hidden behind encoding, encryption, or obfuscation.
- This challenge name itself is a hint: `M1n10n'5_53cr37` reads "Minion's Secret" in leet speak, and the flag echoes the same playful tone.

**Flag wordplay decode:** `1t_w4sn7_h4rd_unr4v3l1n9_th3_m0b1l3_c0d3` — *"it wasn't hard unraveling the mobile code"*. The challenge name (`M1n10n'5_53cr37` = "Minion's Secret") and the flag both lean into the same Minions theme.
