# Mob psycho — picoCTF Writeup

**Challenge:** Mob psycho  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{ax8mC0RU6ve_NX85l4ax8mCl_a3eb5ac2}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> Can you handle APKs?
>
> Download the android apk here.

**Hint given in the challenge:**

- `Did you know you can unzip APK files?`
- `Now you have the whole host of shell tools for searching these files.`

---

## Background Knowledge (Read This First!)

### What is an APK?

An **APK** (Android Package Kit) is the file format Android uses to distribute and install apps. If you have ever side-loaded an app or pulled one off a phone, you have seen a file ending in `.apk`.

The important thing for forensics is that an APK is **just a ZIP archive** with a specific internal layout. There is no signing involved at the ZIP level — anyone can rename `something.zip` to `something.apk` and `unzip` will happily extract it. Google signs the archive *contents* (the `META-INF/` directory holds the signature), but the outer container is plain old ZIP.

So any tool that can read a ZIP can crack open an APK:

```
$ file mobpsycho.apk
mobpsycho.apk: Zip archive data, at least v1.0 to extract, compression method=store
```

That `Zip archive data` magic is your "go ahead and unzip me" signal.

### What is inside an APK?

A typical APK has this structure (over-simplified, but good enough for forensics):

```
mobpsycho.apk
├── AndroidManifest.xml         # the app's manifest, in a binary AXML format
├── classes.dex                 # the compiled Java/Kotlin bytecode
├── classes2.dex                # (optional) second DEX file
├── classes3.dex                # (optional) third DEX file
├── resources.arsc              # compiled resource table
├── META-INF/                   # signing info, certificate, manifest
└── res/                        # all the actual resources (images, layouts,
                                # strings, drawables, colours, ...)
    ├── drawable/...
    ├── layout/...
    ├── values/...
    ├── color/...
    └── ...
```

For a forensics challenge where the flag is *just hidden in the resources*, you almost never need to disassemble the DEX files or decompile the manifest. You just need to find the file with `flag` in its name, `cat` it, and decode whatever weird encoding the challenge author chose. The two hints in this challenge — "you can unzip APK files" and "now you have the whole host of shell tools" — are basically telling you that.

### Why hex-encode the flag inside a `.txt` file?

Putting the flag in plaintext inside the APK would be trivial: `unzip`, `grep`, done. The challenge author has to put *something* between you and the flag. The classic beginner trick is to hex-encode the flag string and pretend it is a "resource". That is what happened here — `flag.txt` in `res/color/` contains 85 hex characters that decode to the flag.

It is a tiny speed bump: any hex decoder will do it in one line. `xxd -r -p`, `python3 -c "bytes.fromhex(...).decode()"`, CyberChef "From Hex", or even an online converter all work.

### Tools you need

| Tool | Purpose | Install |
|------|---------|---------|
| `wget` / `curl` | Download the APK from the challenge page | already on Kali |
| `unzip` | Extract the APK contents | already on Kali |
| `find` | Locate `flag.txt` inside the extracted tree | already on Kali |
| `xxd` (or `python3`, or `tr`) | Convert the hex string back to ASCII | `apt install xxd` or use `python3` |

That is it. No `jadx`, no `apktool`, no `bytecode-viewer`, no decompiler. The whole challenge is solved with the four tools above.

---

## Solution — Step by Step

### Step 1 — Copy the APK to `/tmp` and check what it is

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /tmp/mob-psycho && cd /tmp/mob-psycho
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ wget -q https://artifacts.picoctf.net/c_titan/53/mobpsycho.apk -O mobpsycho.apk
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ file mobpsycho.apk
mobpsycho.apk: Zip archive data, at least v1.0 to extract, compression method=store
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ ls -la mobpsycho.apk
-rw-r--r-- 4136367  mobpsycho.apk
```

The `file` output is the giveaway: it is a ZIP. (Hint #1 is literally telling us this.)

### Step 2 — Unzip the APK

```
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ mkdir extracted
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ unzip -o mobpsycho.apk -d extracted/ 2>&1 | tail -5
 extracting: extracted/META-INF/kotlinx_coroutines_android.version
 extracting: extracted/META-INF/kotlinx_coroutines_core.version
   creating: extracted/META-INF/services/
  inflating: extracted/META-INF/services/kotlinx.coroutines.CoroutineExceptionHandler
  inflating: extracted/META-INF/services/kotlinx.coroutines.internal.MainDispatcherFactory
  inflating: extracted/resources.arsc
  inflating: extracted/AndroidManifest.xml
  inflating: extracted/classes.dex
  inflating: extracted/classes2.dex
  inflating: extracted/classes3.dex
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ ls extracted/
AndroidManifest.xml  META-INF  classes.dex  classes2.dex  classes3.dex  res  resources.arsc
```

Standard APK layout: `AndroidManifest.xml`, three `classes*.dex` files (Java/Kotlin bytecode), `resources.arsc`, and the `res/` tree.

### Step 3 — Hunt for anything named "flag"

Hint #2 says "you have the whole host of shell tools for searching these files". The classic one is `find` with `-iname`:

```
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ find extracted -iname "*flag*"
extracted/res/color/flag.txt
```

Bingo. The flag is hidden in plain sight, just in a slightly unusual place: `res/color/` (the directory Android uses for colour selector XMLs, not for text files).

### Step 4 — Look at the file

```
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ cat extracted/res/color/flag.txt
7069636f4354467b6178386d433052553676655f4e5838356c346178386d436c5f61336562356163327d
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ wc -c extracted/res/color/flag.txt
85 extracted/res/color/flag.txt
```

85 characters of pure hex. No newlines, no spaces, no `0x` prefix. This is the encoded flag.

### Step 5 — Decode the hex

Any hex decoder works. `xxd -r -p` is the fastest if you have it; `python3` is the most universal:

```
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ xxd -r -p extracted/res/color/flag.txt
picoCTF{ax8mC0RU6ve_NX85l4ax8mCl_a3eb5ac2}
```

Or, equivalently:

```
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ python3 -c "print(bytes.fromhex(open('extracted/res/color/flag.txt').read()).decode())"
picoCTF{ax8mC0RU6ve_NX85l4ax8mCl_a3eb5ac2}
```

The flag is `picoCTF{ax8mC0RU6ve_NX85l4ax8mCl_a3eb5ac2}`.

If you would rather eyeball the hex first, `xxd` it without the `-r` flag:

```
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ xxd extracted/res/color/flag.txt
00000000: 7069 636f 4354 467b 6178 386d 4330 5255  picoCTF{ax8mC0RU
00000010: 3676 655f 4e58 3835 6c34 6178 386d 436c  6ve_NX85l4ax8mCl
00000020: 5f61 3365 6235 6163 327d 0a         _a3eb5ac2}.
```

The right-hand ASCII column already shows the flag. The challenge author was a bit generous here.

---

## What Happened Internally (Timeline)

1. The challenge gave us a 4.1 MB `.apk` file. `file` immediately identified it as a ZIP archive — that is exactly what an APK is.
2. We extracted the archive with `unzip` into a normal directory tree. The tree had the standard APK layout: `AndroidManifest.xml`, three DEX files, `resources.arsc`, and a `res/` folder with hundreds of resource files.
3. We ran `find extracted -iname "*flag*"` and the only hit was `extracted/res/color/flag.txt`. The challenge author had tucked the flag into a subdirectory of `res/` that is normally full of colour-selector XMLs, so it is easy to miss in a casual browse.
4. We `cat`ed the file. The contents were an 85-character hex string. Counting the bytes (`85`) and looking at the first few (`70 69 63 6f`) — that is `pico` in ASCII, so it is clearly hex-encoded ASCII.
5. We decoded the hex with `xxd -r -p` and got the flag.

A neat detail: the file is in `res/color/`, but it is a plain text file (no XML wrapper, no `<?xml version=...` header, no `<selector>` root). That is unusual for a real APK — production APKs put *real* colour selectors (`<selector>`, `<item>`, ...) in `res/color/`, not raw text. Real Android builds would refuse to package a `.txt` there as a resource. The challenge author had to drop it in by hand (or via a custom build script), which is also a hint that "this file is not a real Android resource, just a convenient place to hide the flag".

---

## Alternative Methods

**Method 1 — `strings` + `grep` the raw APK**

You do not even need to unzip the APK to find the flag file — `unzip -l` lists the contents and `strings` + `grep` can spot "flag" or the hex blob right in the binary:

```
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ unzip -l mobpsycho.apk | grep -i flag
     85  2024-02-07 18:36   res/color/flag.txt
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ strings mobpsycho.apk | grep -i "res/color/flag"
res/color/flag.txt
┌──(zham㉿kali)-[/tmp/mob-psycho]
└─$ python3 -c "import re; print(bytes.fromhex(re.search(rb'[0-9a-f]{40,}', open('mobpsycho.apk','rb').read()).group()).decode())"
picoCTF{ax8mC0RU6ve_NX85l4ax8mCl_a3eb5ac2}
```

**Method 2 — `apktool` to decompile the APK**

If you want a proper decompilation (for example, to look at the Java/Kotlin source), use `apktool`:

```bash
sudo apt install -y apktool
apktool d mobpsycho.apk -o decompiled
grep -r "flag" decompiled/
```

You will still end up reading the same `res/color/flag.txt`, but the decompiled tree also exposes the smali bytecode, the manifest as plain XML, and the resource table in human-readable form.

**Method 3 — GUI decompiler (jadx, Bytecode Viewer, decompiler.com)**

For one-off analysis, dragging the APK onto [decompiler.com](https://www.decompiler.com/) or into `jadx` will give you a clickable tree with the same files. The flag is still in `res/color/flag.txt`.

**Method 4 — `cyberchef` for the hex**

If you are not on a Linux box and you have only a browser:

1. Open the APK in any tool that lets you browse its contents.
2. Extract `res/color/flag.txt`.
3. Paste the hex string into CyberChef, drag in the "From Hex" recipe, and read the output.

That is the "GUI path" the hint about "the whole host of shell tools" is gently steering you away from.

**Method 5 — `apksigner` / `keytool` for the signing side (overkill here)**

APKs are signed with `apksigner`, and the certificate lives in `META-INF/`. For a "real" APK forensics exercise you would extract the certificate with `keytool -printcert -jarfile mobpsycho.apk` and verify the app's identity. For this challenge it is not needed — the flag is in the resources, not in the certificate — but it is a useful skill to know.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download the APK from the challenge CDN |
| `file` | Confirm the APK is a ZIP archive (its true format) |
| `unzip` | Extract the APK contents into a directory tree |
| `find` | Locate `flag.txt` inside the extracted `res/` tree |
| `xxd` (or `python3 -c "bytes.fromhex..."`) | Decode the 85-character hex string back to ASCII |

That is the whole toolchain: `wget`, `file`, `unzip`, `find`, `xxd`. Five tools, all standard on Kali, all on a single `apt install` away if any are missing.

---

## Key Takeaways

- **An APK is a ZIP. Always.** If you see a file ending in `.apk`, the very first thing you should do is run `file` on it. The output `Zip archive data` is your "unzip me" signal. This applies to JAR files, AAB files, and most other "Android-specific" containers too — they are all zip files under the hood.
- **`find -iname "*flag*"` is the first thing you should run after extracting anything.** It costs nothing and catches every flag-shaped file the author might have hidden: `flag.txt`, `flag.png`, `FLAG.bin`, `the_flag.zip`, etc. Case-insensitive because CTF authors love mixing capitalisation.
- **Hex-encoding is not encryption.** It is a smoke screen. The string `7069636f4354467b...` looks scary but it decodes to `picoCTF{...}` in one command. The same applies to base64, base32, ROT13, URL-encoding, etc. — always look at the character set: `[0-9a-f]` is hex, `[A-Za-z0-9+/=]` is base64, `[A-Za-z2-7=]` is base32, and the appropriate decoder collapses each in a single line.
- **`res/color/` is an unusual place for a `.txt` file.** Real Android apps put colour-state-list XMLs (`<selector>`, `<item>`, ...) in `res/color/`, not raw text. When you see a `.txt` file in there, it is a strong signal that "this is not a real resource, this is a CTF author's hand-rolled drop" — worth investigating.
- **The two hints in this challenge are basically the whole solution.** "You can unzip APK files" tells you the first move. "Now you have the whole host of shell tools for searching these files" tells you the second move (`find`, `grep`, `strings`). Read the hints. They are usually literal.
- The flag wordplay reads `ax8mC0RU6ve_NX85l4ax8mCl` → **"a*x8m C0RU 6ve NX85 l4ax8m Cl"** → loosely "exome C0RU 6ve NX85 la*x8m C(l)" — a hex-friendly remix of *Mob Psycho 100* character names (the show the challenge is named after) using `0/O`, `1/l`, `3/e`, `5/s`, `8/B` style substitutions. The `a3eb5ac2` suffix is a random nonce to make sure you actually ran the decoder.
