# YaraRules0x100 — picoCTF Writeup

**Challenge:** YaraRules0x100  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{yara_rul35_r0ckzzz_714786c1}`  
**Platform:** picoCTF 2025  
**Writeup by:** zham  

---

## Description

> Dear Threat Intelligence Analyst,
> Quick heads up - we stumbled upon a shady executable file on one of our employee's Windows PCs. Good news: the employee didn't take the bait and flagged it to our InfoSec crew.
> Seems like this file sneaked past our Intrusion Detection Systems, indicating a fresh threat with no matching signatures in our database.
> Can you dive into this file and whip up some YARA rules? We need to make sure we catch this thing if it pops up again.
> Thanks a bunch!
>
> The suspicious file can be downloaded here. Unzip the archive with the password `picoctf`
> Once you have created the YARA rule/signature, submit your rule file as follows:
> `socat -t60 - TCP:standard-pizzas.picoctf.net:65056 < sample.txt`

**Hint 1:** `The test cases will attempt to match your rule with various variations of this suspicious file, including a packed version, an unpacked version, slight modifications to the file while retaining functionality, etc.`  
**Hint 2:** `Since this is a Windows executable file, some strings within this binary can be "wide" strings. Try declaring your string variables something like $str = "Some Text" wide ascii wherever necessary.`  
**Hint 3:** `Your rule should also not generate any false positives (or false negatives). Refine your rule to perfection! One YARA rule file can have multiple rules! Maybe define one rule for Packed binary and another rule for Unpacked binary in the same rule file?`

**Downloads:** `suspicious.zip`

---

## Background Knowledge (Read This First!)

### What is YARA?

**YARA** is a tool used by malware researchers to identify and classify malware samples. You write rules that describe patterns — strings, byte sequences, or conditions — and YARA checks if a file matches those patterns.

A basic YARA rule looks like this:

```yara
rule ExampleRule {
    meta:
        description = "Detects something suspicious"
    strings:
        $s1 = "suspicious string" wide ascii
    condition:
        $s1
}
```

- `strings` — defines what to look for (text, hex bytes, regex)
- `condition` — logic that must be true for the rule to match
- `wide ascii` — checks for both normal ASCII and UTF-16 wide strings (common in Windows binaries)
- `uint16(0) == 0x5A4D` — checks the first 2 bytes of the file equal `MZ`, confirming it is a Windows PE executable

### What is UPX packing?

**UPX** (Ultimate Packer for eXecutables) is a common tool used to compress executables. A UPX-packed binary:
- Has very few readable strings — the real content is compressed inside
- Contains the section names `UPX0`, `UPX1`, and the magic bytes `UPX!`
- Decompresses itself at runtime

For YARA, a packed binary needs its own rule because the strings from the unpacked version are not visible — they are compressed. The file must be unpacked first to write rules for the unpacked version.

### What is a false negative vs false positive?

- **False negative** — your rule fails to match a file it should match (missed detection)
- **False positive** — your rule matches a file it should not match (wrong detection)

Both are bad in real threat intelligence. The challenge requires zero false negatives and zero false positives across 64 test cases.

---

## Extracting the File

### Step 1 — Unzip with the password

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip -P picoctf suspicious.zip
Archive:  suspicious.zip
  inflating: suspicious.exe
```

### Step 2 — Check what kind of file it is

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file suspicious.exe
suspicious.exe: PE32 executable for MS Windows 6.00 (GUI), Intel i386, UPX compressed, 3 sections
```

Key findings:
- Windows PE32 executable (32-bit)
- **UPX compressed** — packed, strings are hidden
- 3 sections — typical for UPX (`UPX0`, `UPX1`, `.rsrc`)

### Step 3 — Check strings on the packed binary

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings suspicious.exe | head -20
!This program cannot be run in DOS mode.
xRich
UPX0
UPX1
.rsrc
4.21
UPX!
```

Very few readable strings — confirms packing. The `UPX0`, `UPX1`, and `UPX!` markers are the packed binary's signature.

---

## Unpacking the Binary

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ upx -d suspicious.exe -o suspicious_unpacked.exe
        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
     40960 <-     26624   65.00%    win32/pe     suspicious_unpacked.exe

Unpacked 1 file.
```

Now run strings on the unpacked version:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings -e l suspicious_unpacked.exe
NtQueryInformationProcess
SeDebugPrivilege
[FATAL ERROR] CreateToolhelp32Snapshot failed.
[FATAL ERROR] OpenProcessToken failed.
[FATAL ERROR] AdjustTokenPrivileges failed.
[FATAL ERROR] Failed to create the Mutex.
No debugger was present. Exiting successfully.
Debugger process terminated successfully.
This is a fake malware. It means no harm.
picoCTF
SUSPICIOUS
...
```

And Win32 API imports:

```
NtQueryInformationProcess
DebugActiveProcessStop
IsDebuggerPresent
CreateToolhelp32Snapshot
CreateMutexW
OpenProcessToken
AdjustTokenPrivileges
LookupPrivilegeValueW
```

---

## Analyzing the Binary

From the strings and imports, the binary clearly:

| Behavior | Evidence |
|---|---|
| Anti-debugging | `NtQueryInformationProcess`, `IsDebuggerPresent`, `DebugActiveProcessStop` |
| Privilege escalation | `SeDebugPrivilege`, `AdjustTokenPrivileges`, `LookupPrivilegeValueW` |
| Process enumeration | `CreateToolhelp32Snapshot`, `Process32FirstW`, `Process32NextW` |
| Mutex creation | `CreateMutexW` — checks if already running |

These Win32 API imports are behavioral signatures — they describe **what the program does**, not just what strings it contains. They survive minor modifications to the binary, making them ideal YARA signatures.

---

## Writing the YARA Rules

### Step 1 — Create the rule file

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano suspicious.yar
```

### Step 2 — Write two rules: one for packed, one for unpacked

```yara
rule Suspicious_Packed {
    meta:
        description = "Detects UPX-packed version of suspicious.exe"
        author = "zham"
    strings:
        $upx0 = "UPX0"
        $upx1 = "UPX1"
        $upx_magic = "UPX!"
    condition:
        uint16(0) == 0x5A4D and
        all of ($upx*)
}

rule Suspicious_Unpacked {
    meta:
        description = "Detects unpacked version of suspicious.exe"
        author = "zham"
    strings:
        $anti_debug1 = "NtQueryInformationProcess" wide ascii
        $anti_debug2 = "DebugActiveProcessStop" wide ascii
        $anti_debug3 = "IsDebuggerPresent" wide ascii
        $priv_esc    = "SeDebugPrivilege" wide ascii
        $proc_enum   = "CreateToolhelp32Snapshot" wide ascii
        $mutex       = "CreateMutexW" wide ascii
    condition:
        uint16(0) == 0x5A4D and
        $priv_esc and
        $proc_enum and
        $mutex and
        2 of ($anti_debug*)
}
```

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

---

## Submitting the Rules

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ socat -t60 - TCP:standard-pizzas.picoctf.net:65056 < suspicious.yar
:::::
Status: Passed
Congrats! Here is your flag: picoCTF{yara_rul35_r0ckzzz_714786c1}
:::::
```

---

## Failed First Attempt

The first submission included this string in the unpacked rule:

```yara
$unique = "This is a fake malware. It means no harm." wide ascii
```

Result:

```
Status: Failed
False Negatives Check: Testcase failed. Your rule generated a false negative.
Stats: 63 testcase(s) passed. 1 failed.
```

The hint says test cases include *"slight modifications to the file while retaining functionality"* — that unique string was changed in one test variant, causing a false negative. The fix was removing it entirely and relying only on Win32 API imports, which are tied to the binary's **behavior** rather than its text content and survive modifications.

---

## What Happened Internally

```
Timeline:
1. suspicious.exe is downloaded — UPX packed, very few readable strings
2. upx -d unpacks it to suspicious_unpacked.exe — full strings now visible
3. strings -e l reveals Win32 API imports and wide strings
4. Two YARA rules written: one matching UPX section names (packed), one matching API imports (unpacked)
5. First attempt fails — $unique string too fragile, modified in one test case
6. Rule updated to use only behavioral API imports — passes all 64 test cases
7. Flag returned
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `unzip -P` | Extract password-protected zip | Easy |
| `file` | Identify file type and confirm UPX packing | Easy |
| `strings` | Extract readable strings from binary | Easy |
| `strings -e l` | Extract wide (UTF-16) strings from Windows binary | Easy |
| `upx -d` | Unpack UPX-compressed executable | Easy |
| `nano` | Write the YARA rule file | Easy |
| `socat` | Submit the rule file to the challenge server | Easy |

---

## Key Takeaways

- **UPX-packed binaries need their own YARA rule** — strings are compressed and invisible until unpacked; match on UPX section markers instead
- **API imports are better YARA signatures than strings** — behavioral indicators like `SeDebugPrivilege` and `CreateToolhelp32Snapshot` survive minor file modifications; hardcoded strings often do not
- **`wide ascii` is required for Windows binaries** — many strings in PE files are stored as UTF-16 wide strings; omitting `wide` causes false negatives
- **`uint16(0) == 0x5A4D` scopes rules to PE files** — the `MZ` header check prevents the rule from matching non-executable files and reduces false positives
- **One YARA file can contain multiple rules** — the packed and unpacked variants needed separate rules in the same file, each covering a different form of the same threat
- The flag `yara_rul35_r0ckzzz` is "yara rules rockzzz" — the challenge is literally just about writing good YARA rules
