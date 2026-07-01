# WinAntiDbg0x100 — picoCTF Writeup

**Challenge:** WinAntiDbg0x100  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{d3bug_f0r_th3_Win_0x100_cfbacfab}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> This challenge will introduce you to 'Anti-Debugging'. Malware developers don't like it when you attempt to debug their executable files because debugging these files reveals many of their secrets! That's why, they include a lot of code logic specifically designed to interfere with your debugging process.
>
> Now that you've understood the context, go ahead and debug this Windows executable!
>
> This challenge binary file is a Windows console application and you can start with running it using `cmd` on Windows.

## Hints

> Hints will be displayed to the Debug console. Good luck!

---

## Background Knowledge (Read This First!)

If any of these words sound new, read the short notes below. Otherwise, skip ahead to the solution.

### What is Anti-Debugging?

Anti-debugging is a set of tricks a program uses to detect or block debuggers like x64dbg, x32dbg, OllyDbg, or WinDbg. Malware authors use these tricks so analysts cannot easily step through the program and watch what it does. Common techniques include:

- `IsDebuggerPresent` — a Windows API that returns 1 if a debugger is attached to the current process.
- `NtQueryInformationProcess` — a lower-level Windows API that can ask the kernel for debug-related info (like whether a debug object handle exists, class `0x1F`).
- `PEB.BeingDebugged` — a flag inside the Process Environment Block that is set to 1 when a debugger is attached.
- Timing checks, exception tricks, and `OutputDebugString` abuse.

In CTF challenges, you often have to **patch** these checks (flip a jump, NOP out a call, or change a returned value) so the binary follows the path that reveals the flag.

### What is `OutputDebugStringW`?

It is a Windows API that sends a string to the debugger's debug console. The string does **not** appear on stdout — you only see it inside a debugger like x32dbg on the "Debug" tab. That is exactly what the hint is telling us: the helpful strings are hidden in the debug console.

### What is `PE32`?

`PE32` stands for **Portable Executable, 32-bit**. It is the file format used by Windows `.exe` files. We can still analyze it on Kali with tools like `pefile`, `capstone`, and `objdump` even though we cannot run the Windows binary directly.

### What is XOR Decryption?

XOR (`^` in most languages) is a bitwise operation that is its own inverse: `A ^ B ^ B = A`. If a string was encrypted by XORing each byte with a key, you can decrypt it by XORing with the same key. This challenge uses a more complex transformation plus an XOR.

---

## Files Provided

- `WinAntiDbg0x100.exe` — the Windows console application
- `config.bin` — a small data file that the exe reads from the same folder

Both come inside a password-protected ZIP. The password is `picoctf`.

---

## Solution — Step by Step

I solved this entirely from Kali by reversing the Windows binary statically. No Windows VM, no x32dbg required. Here is how I went from a 14 KB `.exe` to the flag.

### Step 1 — Unzip the challenge archive

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x100]
└─$ unzip -P picoctf WinAntiDbg0x100.zip
Archive:  WinAntiDbg0x100.zip
  inflating: WinAntiDbg0x100.exe
 extracting: config.bin
```

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x100]
└─$ ls -la
-rw-r--r-- 1 zham zham 14336 Feb  7  2024 WinAntiDbg0x100.exe
-rw-r--r-- 1 zham zham    88 Apr  2  2024 config.bin
```

### Step 2 — Quick file analysis

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x100]
└─$ file WinAntiDbg0x100.exe
WinAntiDbg0x100.exe: PE32 executable (console) Intel 80386, for MS Windows, 5 sections
```

So it is a 32-bit Windows console binary. Good to know for the disassembly later.

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x100]
└─$ strings WinAntiDbg0x100.exe | head -40
```

The `strings` output already gives a great roadmap. I can spot:

```
        _            _____ _______ ______
       (_)          / ____|__   __|  ____|
  _ __  _  ___ ___ | |       | |  | |__
 | '_ \| |/ __/ _ \| |       | |  |  __|
 | |_) | | (_| (_) | |____   | |  | |
 | .__/|_|\___\___/ \_____|  |_|  |_|
 |_|
  Welcome to the Anti-Debug challenge!
NtQueryInformationProcess
%s\config.bin
### To start the challenge, you'll need to first launch this program using a debugger!
```

So the binary:

1. Prints a welcome banner.
2. Tells you to launch it under a debugger.
3. Uses `NtQueryInformationProcess` (an anti-debug API).
4. Reads a file called `config.bin` from its own folder using the format string `%s\config.bin`.

### Step 3 — Inspect `config.bin`

This is where the real flag data lives. I dumped it with `od` (Kali has no `xxd` by default).

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x100]
└─$ od -A x -t x1z -v config.bin
000000 29 00 00 00 6f 6e 6e 76 65 7a 65 69 75 72 6a 71  >)...onnvezeiurjq<
000010 70 62 7a 78 6d 67 6c 68 63 61 6d 6a 77 75 63 63  >pbzxmglhcamjwucc<
000020 6e 77 77 6e 68 67 6c 76 63 61 63 71 71 00 1f 0f  >nwwnhglvcacqq...<
000030 05 09 34 3e 29 10 09 51 16 06 1d 3b 04 42 17 2e  >..4>)..Q...;.B..<
000040 02 02 5e 3c 38 0d 09 28 55 0f 41 41 41 25 19 17  >..^<8..(U.AAA%..<
000050 14 19 0e 05 04 09 1c 00                          >........<
```

The structure is:

- Bytes `0..3` (little-endian): `0x29` = **41**, the length.
- Bytes `4..44`: an encoded 41-character ASCII string `onnvezeiurjqpbzxmglhcamjwuccnwwnhglvcacqq`.
- Byte `45`: a null separator `0x00`.
- Bytes `46..86`: a 41-byte XOR key: `1f 0f 05 09 34 3e 29 10 09 51 16 06 1d 3b 04 42 17 2e 02 02 5e 3c 38 0d 09 28 55 0f 41 41 41 25 19 17 14 19 0e 05 04 09 1c`.
- Byte `87`: trailing null.

So the file is `[len][encoded_string]\0[xor_key]\0`. To get the flag we need to XOR the two halves together, but the exe applies a transformation to the encoded string first.

### Step 4 — Reverse the binary with `pefile` and `capstone`

The key imports confirm what `strings` hinted at:

```
KERNEL32.dll:
  GetModuleFileNameA, MultiByteToWideChar, OutputDebugStringW,
  GetProcAddress, GetModuleHandleW, IsDebuggerPresent,
  GetCurrentProcess, ...
```

The most interesting function lives at `0x401580`. I disassembled the `.text` section with capstone and traced the logic. In short, the main function does this:

```
call  anti_debug_check       ; returns 1 only if a debugger is attached
if (not detected):
    print banner + "use a debugger"
    exit
else:
    OutputDebugString  -> "Level 1: ... growing a 'patch' of roses!"  (hint)
    call  setup_function
    if (IsDebuggerPresent == 1):          ; anti-debug check #2
        print "Oops! The debugger was detected"
        exit
    transform(encoded_string, 7 times)   ; function at 0x401440 with arg=7
    transform(encoded_string, 11 times)  ; same function with arg=11
    XOR(xor_key, transformed_string)
    wide_print(result)
    print "Good job! Here's your flag: ~~~ " + result
```

The 0x401440 transformation works character-by-character on the encoded string. For every character position `i`, it computes a small bit-mixed value out of `i` (using the masks `0x55`, `0x33`, `0x0F`) and then writes:

```
new_char = 'a' + ((old_char - 'a' + low4(var1c) + high4(var1c)) % 26)
```

So every character stays inside `'a'..'z'` no matter how many times we apply it.

The Level 1 hint, *"growing a patch of roses"*, is a wordplay on **patch**ing the binary in a debugger like x32dbg (flip the `je` after `IsDebuggerPresent` to `jmp` or just `NOP` the call). If you patch that one jump, the rest of the code runs the transformation + XOR and prints the flag on the debug console.

I did not need to patch anything because I re-implemented the same algorithm in Python and ran it locally.

### Step 5 — Re-implement the algorithm in Python

I wrote a small script that:

1. Reads `config.bin`.
2. Applies the transformation 18 times (7 from the first call + 11 from the second call).
3. XORs the result with the 41-byte key.
4. Prints the flag.

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x100]
└─$ nano solve.py
```

Paste this inside `nano`:

```python
import struct

with open("config.bin", "rb") as f:
    data = f.read()

# 1. Parse config.bin: [u32 length][encoded string][0x00][xor key][0x00]
length = struct.unpack("<I", data[:4])[0]
encoded = bytearray(data[4 : 4 + length])
xor_key = bytearray(data[4 + length + 1 : 4 + length + 1 + length])


# 2. Re-implement the 0x401440 transformation from the disassembly
def transform(buf, iterations):
    buf = bytearray(buf)
    for _ in range(iterations):
        for i in range(len(buf)):
            inner_mod = i % 0xFF
            var14 = (inner_mod & 0x55) + ((inner_mod >> 1) & 0x55)
            var1c = (0x33 & var14) + ((var14 >> 2) & 0x33)
            delta = (var1c & 0x0F) + ((var1c >> 4) & 0x0F)
            buf[i] = 0x61 + ((buf[i] - 0x61 + delta) % 26)
    return bytes(buf)


# 3. Apply 7 + 11 = 18 iterations, then XOR with the key
transformed = transform(encoded, 7 + 11)
flag = bytes([a ^ b for a, b in zip(xor_key, transformed)]).decode()
print(flag)
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x100]
└─$ python3 solve.py
picoCTF{d3bug_f0r_th3_Win_0x100_cfbacfab}
```

Got the flag.

---

## Alternative Solve — Manual XOR with the Disassembled Logic

If you do not want to write the full transform function, here is the simplest possible one-liner version that matches what the binary prints. The idea is identical, just inline:

```python
import struct

data = open("config.bin", "rb").read()
n = struct.unpack("<I", data[:4])[0]
s = bytearray(data[4:4+n])
k = bytearray(data[4+n+1:4+n+1+n])

# 18 rounds of the bit-mixed Caesar-style transform
for _ in range(18):
    for i in range(n):
        v14 = (i & 0x55) + ((i >> 1) & 0x55)
        v1c = (0x33 & v14) + ((v14 >> 2) & 0x33)
        d = (v1c & 0xF) + ((v1c >> 4) & 0xF)
        s[i] = 0x61 + ((s[i] - 0x61 + d) % 26)

print(bytes(a ^ b for a, b in zip(k, s)).decode())
```

Same output: `picoCTF{d3bug_f0r_th3_Win_0x100_cfbacfab}`.

---

## Alternative Solve — The "Intended" Windows Way

If you have a Windows VM with x32dbg installed:

1. Launch `cmd`, navigate to the folder, and open `WinAntiDbg0x100.exe` in x32dbg.
2. Run once. Look at the **Debug** tab — you will see the Level 1 hint about "growing a patch of roses".
3. Find the `IsDebuggerPresent` call and the `je` right after it. Patch that `je` to `jmp` (or just `NOP` the `call` so `eax` stays 0). The binary will think no debugger is attached.
4. Step past the two `0x401440` calls (or just hit Run). The flag will be printed to the debug console after `### Good job! Here's your flag:`.

The flag is the same: `picoCTF{d3bug_f0r_th3_Win_0x100_cfbacfab}`.

---

## What Happened Internally — Step-by-Step Timeline

Here is the full story of what the binary does, in order:

1. **Entry (`0x401923`)** — standard CRT startup, then jump to `main`.
2. **`main` (`0x401580`)** — saves the stack frame and calls the anti-debug helper.
3. **`0x401130` anti-debug check** — loads `ntdll.dll`, resolves `NtQueryInformationProcess`, and calls it with class `0x1F` (`ProcessDebugObjectHandle`). If the call succeeds, a debug object exists, so a debugger is attached. It returns 1 in that case. Otherwise it falls back to checking `PEB.BeingDebugged` at `fs:[0x30] + 2` and returns 0.
4. **No-debug branch** — main prints the ASCII-art banner and the message *"To start the challenge, you'll need to first launch this program using a debugger!"*, then exits.
5. **Debug branch** — main calls `OutputDebugStringW` with two hidden strings (the hint you only see inside a debugger) and then calls `setup_function` (`0x401200`).
6. **`0x401200` reads `config.bin`** — it asks the OS for the exe's own path with `GetModuleFileNameA`, strips the filename down to the directory with `strrchr('\\')`, then `sprintf`s `<dir>\config.bin`. It `malloc`s 200 bytes, opens the file in `"rb"`, reads the 4-byte length, then reads up to 200 bytes. It splits the buffer at the null byte: the first half is the encoded string, the second half is the XOR key. Both pointers are saved in global variables `0x405404` (encoded) and `0x405408` (key). It also stashes the malloc'd pointer in `0x405410` so it can `free` it later.
7. **`IsDebuggerPresent` check** — main calls the API. If a real debugger is attached, this returns 1 and the binary prints *"Oops! The debugger was detected. Try to bypass this check to get the flag!"* and exits.
8. **First transformation pass** — main calls `0x401440(7)`. This runs the bit-mixed Caesar-style transform over the encoded string seven times.
9. **Second transformation pass** — main calls `0x401440(11)`, applying eleven more rounds. Total: 18 rounds.
10. **`0x401530` XOR step** — main calls the XOR helper with the encoded string as the key. It XORs every byte of the stored XOR-key buffer with the now-transformed encoded string, leaving the decrypted flag inside the original `malloc`ed buffer.
11. **`0x4013b0` UTF-8 to UTF-16 conversion** — main calls `MultiByteToWideChar` with code page `0xFDE9` (65001 = UTF-8) twice: once to ask for the size, then again with the freshly allocated wide buffer. It returns the wide pointer.
12. **Print the flag** — main sends the wide string to the debug console: `### Good job! Here's your flag:` + `### ~~~ ` + `flag` + newline + the warning about process tampering.
13. **Cleanup** — `free` the buffer from `0x405410` and return.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip` | Extract the password-protected challenge archive |
| `file` | Confirm the `.exe` is a 32-bit Windows PE binary |
| `strings` | Spot the welcome banner, `NtQueryInformationProcess`, and the `config.bin` path hint |
| `od` | Hex-dump `config.bin` to see its raw structure |
| `pefile` (Python) | Parse PE headers, list sections and imported APIs |
| `capstone` (Python) | Disassemble the `.text` section so I could read the x86 logic |
| `python3` | Re-implement the transformation and XOR, recover the flag offline |

---

## Key Takeaways

- Anti-debugging checks are everywhere in real malware. The two APIs you will see most often are `IsDebuggerPresent` and `NtQueryInformationProcess` with class `0x1F` (`ProcessDebugObjectHandle`).
- `OutputDebugStringW` is a classic way for a binary to whisper hints only to whoever has a debugger attached. Always check the debug console when reversing Windows targets.
- The "Level 1" hint *"growing a patch of roses"* is a clear hint to **patch** the `IsDebuggerPresent` check in x32dbg — flip the `je` after the call, or `NOP` the call itself.
- A PE binary does not need a Windows machine to reverse. `pefile` + `capstone` (or `objdump`, Ghidra, IDA, Binary Ninja) is enough to fully understand the logic.
- The `config.bin` layout is a tiny file format: `[u32 length][data][0x00][xor_key][0x00]`. Recognizing patterns like this saves a lot of time in RE challenges.
- Bit-mixed Caesar-style transforms that always keep characters in `'a'..'z'` are common in CTFs because they look scary in the disassembly but are trivial to re-implement in Python.
- Static analysis beats dynamic patching when you do not have a Windows VM handy — the algorithm is the algorithm no matter what OS you run it on.
- **Flag wordplay decode**: `picoCTF{d3bug_f0r_th3_Win_0x100_cfbacfab}` reads as *"debug for the Win 0x100"*. The `0x100` matches the challenge name suffix (`WinAntiDbg0x100`), and `cfbacfab` is just a random hex tag used to make the flag unique.
