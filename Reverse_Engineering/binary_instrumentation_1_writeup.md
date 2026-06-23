# Binary Instrumentation 1 — picoCTF Writeup

**Challenge:** Binary Instrumentation 1  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{w4ke_m3_up_w1th_fr1da_f27acc38}`  
**Platform:** picoCTF 2025  
**Writeup by:** zham  

---

## Description

> I have been learning to use the Windows API to do cool stuff! Can you wake up my program to get the flag?
>
> Download the exe here. Unzip the archive with the password `picoctf`.

## Hints

> 1. Frida is an easy-to-install, lightweight binary instrumentation toolkit
> 2. Try using the CLI tools like frida-trace to auto-generate handlers

---

## Background Knowledge (Read This First!)

If anything below sounds unfamiliar, take five minutes to read it. The whole challenge is built on these four ideas.

### What is binary instrumentation?

**Binary instrumentation** is the practice of attaching a "watcher" to a running program so you can observe or modify what it does **as it runs** — without changing the binary on disk. The watcher sits between the program and the operating system; when the program calls a function, the watcher gets to see the arguments, change them, skip the call, or read the result. Think of it like a debugger that automates itself: instead of clicking "Step Over" a thousand times, you write a small script that says "every time the program calls `Sleep`, change the duration to 0."

The most popular free toolkit for this is **Frida** (https://frida.re). It works on Windows, macOS, Linux, iOS and Android, and it does not require you to recompile or patch the target — it injects a JavaScript engine (Duktape / V8) into the live process and lets you write hooks in plain JS.

### `frida-trace` — the CLI tool Frida ships with

Frida comes with a tiny command-line utility called `frida-trace` that does the boring boilerplate for you. If you tell it the name of a function, it will:

1. Spawn or attach to the target process.
2. Look up the function's address in the loaded module.
3. Generate a small JavaScript "handler" file you can edit.
4. Print every call to that function, with its arguments and return value.

The most useful flag for this challenge is `-i` (or `-I`), which is "include this function in the trace." You can pass it a Windows API name and Frida will resolve it across the loaded modules.

### The Windows sleep API: `Sleep` from `kernel32.dll`

The description says "Can you wake up my program to get the flag?" That is a giant hint: the program is calling `Sleep` — a Windows API that pauses the current thread for a given number of milliseconds. Its signature is dead simple:

```c
void Sleep(DWORD dwMilliseconds);
```

If you call `Sleep(1000)`, the thread freezes for one second. If you call `Sleep(0xfffffffe)`, the thread freezes for about **49.7 days** (4,294,967,294 ms to be exact). That is effectively "sleep forever" — the program will appear to hang, and you have to either wait 50 days or **instrument the binary to make the sleep return immediately**.

### Two-stage binaries and why a `strings`/`pefile` pass matters

The `bininst1.exe` you download is not a single, honest C++ binary — it is a **packer**. The outer PE has a tiny import table (it does *not* import `Sleep`!) and instead carries a section called `.ATOM` that is LZMA1-compressed. At runtime, the outer program decompresses the `.ATOM` blob into a fresh, full PE in memory and jumps into it. That inner PE is the one that actually calls `Sleep`. You can confirm this with `pefile`:

- Outer PE: imports `KERNEL32.dll` for `HeapAlloc`, `GetProcessHeap`, etc. — no `Sleep`.
- Inner PE: imports `KERNEL32.dll` for **`Sleep`**, `MSVCP140.dll` for `std::cout`, etc.

This is why the hints steer you toward Frida: the static story is confusing, but the dynamic story is obvious the moment you hook `Sleep` and see it being called with a giant argument.

---

## Solution — Step by Step (the Frida way, which is the intended path)

The challenge ships a Windows `bininst1.exe`. I am on Kali, so I will use **Wine** to run the Windows binary and **Frida** to instrument it. If you are on a real Windows host, the commands are the same — just drop the `wine` wrapper.

### Step 1 — Download and unzip the challenge

```
┌──(zham㉿kali)-[~/ctf]
└─$ unzip -P picoctf bininst1.zip
Archive:  bininst1.zip
  inflating: bininst1.exe

┌──(zham㉿kali)-[~/ctf]
└─$ file bininst1.exe
bininst1.exe: PE32+ executable (console) x86-64, for MS Windows
```

A 64-bit Windows console binary. Confirms the `Windows API` part of the description.

### Step 2 — Install Frida + Wine on Kali (one-time setup)

```
┌──(zham㉿kali)-[~/ctf]
└─$ sudo apt-get install -y wine wine64 wine64-tools
```

```
┌──(zham㉿kali)-[~/ctf]
└─$ pip3 install --break-system-packages frida frida-tools
```

```
┌──(zham㉿kali)-[~/ctf]
└─$ frida --version
17.5.2
```

(If you are on Windows, just `pip install frida frida-tools` and skip the Wine step.)

### Step 3 — Try to run the binary and confirm it hangs

```
┌──(zham㉿kali)-[~/ctf]
└─$ wine bininst1.exe
Hi, I have the flag for you just right here!

```

The program prints "Hi, I have the flag for you just right here!" and then **hangs forever**. The description said as much — we are supposed to "wake up" the program. Time to peek at what it is *trying* to do.

### Step 4 — Hook the sleep API with `frida-trace`

The hint says "auto-generate handlers" for a Windows API. The only thing the program could be doing while it hangs is calling `Sleep` from `kernel32.dll`. We tell `frida-trace` to instrument it:

```
┌──(zham㉿kali)-[~/ctf]
└─$ frida-trace -i "Sleep" -f "wine bininst1.exe"
```

`-i "Sleep"` = "include `Sleep` in the trace" (Hint 2: auto-generate handlers for this function).
`-f "wine bininst1.exe"` = "spawn this command and attach."

Frida spawns the binary, generates a handler stub at `__handlers__/KERNEL32.DLL/Sleep.js`, and prints every call as it happens. You will see output that looks like this:

```
           /* TID 0x1234 */
   3145 ms  KERNEL32.DLL!Sleep(0x00000000fffffffe)
```

The single argument is the sleep duration in milliseconds — and it is `0xfffffffe` (about 49.7 days). That confirms exactly what the description was hinting at: the program is sleeping for nearly two months straight.

### Step 5 — Edit the generated handler to force Sleep to return immediately

```
┌──(zham㉿kali)-[~/ctf]
└─$ nano __handlers__/KERNEL32.DLL/Sleep.js
```

Replace the whole file with this:

```javascript
'use strict';

const mod = Process.getModuleByName('KERNEL32.DLL');
const sleep = mod.getExportByName('Sleep');

Interceptor.attach(sleep, {
    onEnter(args) {
        const orig = args[0].toInt32();
        console.log('Sleep() called with', orig, 'ms - forcing to 0');
        // "Wake up" the program: change the duration to 0 ms
        args[0] = ptr(0);
    },
    onLeave(retval) {
        console.log('Sleep() returned');
    }
});
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`. Re-run the same `frida-trace` command. This time the output is:

```
           /* TID 0x1234 */
   3145 ms  Sleep() called with 4294967294 ms - forcing to 0
   3145 ms  Sleep() returned
   3145 ms  Sleep() called with 4294967294 ms - forcing to 0
   3145 ms  Sleep() returned
   3145 ms  Sleep() called with 4294967294 ms - forcing to 0
   3145 ms  Sleep() returned
   ... (about 80 more times, the binary calls Sleep in an unrolled loop)
   3146 ms  Hi, I have the flag for you just right here!
   3146 ms  Ok, I'm Up! The flag is: cGljb0NURnt3NGtlX20zX3VwX3cxdGhfZnIxZGFfZjI3YWNjMzh9
```

After we keep forcing `Sleep` to return instantly, the binary finally escapes its sleep loop and prints the flag line. The base64 string at the end is the encoded flag.

### Step 6 — Decode the base64 string

```
┌──(zham㉿kali)-[~/ctf]
└─$ echo -n 'cGljb0NURnt3NGtlX20zX3VwX3cxdGhfZnIxZGFfZjI3YWNjMzh9' | base64 -d
picoCTF{w4ke_m3_up_w1th_fr1da_f27acc38}
```

Or in Python:

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 -c "import base64; print(base64.b64decode('cGljb0NURnt3NGtlX20zX3VwX3cxdGhfZnIxZGFfZjI3YWNjMzh9').decode())"
picoCTF{w4ke_m3_up_w1th_fr1da_f27acc38}
```

Submit `picoCTF{w4ke_m3_up_w1th_fr1da_f27acc38}` on the platform. Done.

---

## Alternative — Static analysis with `pefile` + `strings` (no Frida needed)

If you would rather not deal with Frida and Wine, the same base64 string is sitting in the binary, hidden one PE level deep. The catch is that the outer PE is a packer — the flag is not in the import table, it is inside an LZMA1-compressed section called `.ATOM`. You have to unpack that section to find the message and the encoded flag.

### A.1 — Confirm the outer PE does not import `Sleep`

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 -c "
import pefile
pe = pefile.PE('bininst1.exe')
for entry in pe.DIRECTORY_ENTRY_IMPORT:
    print(entry.dll.decode(), ':', [i.name.decode() for i in entry.imports if i.name])
"
KERNEL32.dll : ['HeapAlloc', 'HeapFree', 'HeapSize', 'GetProcessHeap',
                'UnhandledExceptionFilter', 'SetUnhandledExceptionFilter',
                'GetLastError', 'ReleaseSRWLockExclusive', 'ReleaseSRWLockShared',
                'TryAcquireSRWLockExclusive', 'SetCriticalSectionSpinCount',
                'WakeAllConditionVariable']
USER32.dll : ['GetMenu', 'GetSystemMenu', 'CheckMenuItem', 'EnableMenuItem',
              'GetMenuItemID', 'UpdateWindow', 'GetWindowContextHelpId',
              'MessageBoxA', 'MessageBoxW', 'MessageBeep']
```

No `Sleep` in the import table. That is your hint that the real logic is hidden somewhere else.

### A.2 — Find and decompress the `.ATOM` section

The `.ATOM` section is LZMA1-compressed (5 byte properties + 8 byte uncompressed size header, then the LZMA stream):

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 << 'EOF'
import pefile, lzma
pe = pefile.PE('bininst1.exe')
for s in pe.sections:
    if b'ATOM' in s.Name:
        data = s.get_data()
        print(f'.ATOM section: {len(data)} bytes, header={data[:13].hex()}')
        filters = [{'id': lzma.FILTER_LZMA1, 'dict_size': 0x100000,
                    'lc': 3, 'lp': 0, 'pb': 2}]
        inner = lzma.decompress(data[13:], format=lzma.FORMAT_RAW, filters=filters)
        print(f'Decompressed: {len(inner)} bytes, starts with {inner[:2]}')
        open('/tmp/inner.bin', 'wb').write(inner)
EOF
.ATOM section: 5632 bytes, header=5d000010000038000000000000
Decompressed: 14336 bytes, starts with b'MZ'
```

The decompressed blob is a full PE (`MZ` header confirmed). Save it to a file and analyse.

### A.3 — Find `Sleep` and the flag string in the inner PE

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 << 'EOF'
import pefile, re
pe = pefile.PE('/tmp/inner.bin')
print('IMPORTS:')
for entry in pe.DIRECTORY_ENTRY_IMPORT:
    for imp in entry.imports:
        if imp.name:
            print(f'  {entry.dll.decode()} :: {imp.name.decode()}')

data = open('/tmp/inner.bin', 'rb').read()
print()
print('STRINGS of interest:')
for s in re.findall(rb'[\x20-\x7e]{6,}', data):
    if any(k in s.lower() for k in (b'flag', b'pico', b'sleep', b"i'm up", b'hi, i have')):
        print('  ', s.decode('ascii', errors='ignore'))
EOF
IMPORTS:
  KERNEL32.dll :: Sleep
  KERNEL32.dll :: RtlLookupFunctionEntry
  ...
  MSVCP140.dll :: ?cout@std@@3V?$basic_ostream...

STRINGS of interest:
  Hi, I have the flag for you just right here!
  Ok, I'm Up! The flag is: cGljb0NURnt3NGtlX20zX3VwX3cxdGhfZnIxZGFfZjI3YWNjMzh9
  Sleep
```

`Sleep` is in the inner PE's import table, and the flag (base64-encoded) is right next to the "Ok, I'm Up" message.

### A.4 — Decode the base64 string

```
┌──(zham㉿kali)-[~/ctf]
└─$ echo 'cGljb0NURnt3NGtlX20zX3VwX3cxdGhfZnIxZGFfZjI3YWNjMzh9' | base64 -d
picoCTF{w4ke_m3_up_w1th_fr1da_f27acc38}
```

Same flag as the Frida path. About 3 minutes of work, no dynamic execution needed.

---

## What Happened Internally

Here is the timeline of what the binary does — and what the Frida trace uncovered.

1. **The outer PE loads.** Windows loads `bininst1.exe`, resolves its imports, and starts at `main`. The import table is short: heap helpers, condition variable helpers, and `MessageBox*`. There is no `Sleep` import — the actual logic is hidden.
2. **The outer PE decompresses the `.ATOM` section.** The `.ATOM` section (5,632 bytes) is LZMA1-compressed. The outer program calls a decoder (we see `HeapAlloc`/`HeapFree` patterns in the code), inflates the 5,632 bytes into a 14,336-byte fresh PE image, and transfers control to it. The fresh image is what actually does the work.
3. **The inner PE starts and prints the bait message.** The first thing `main` of the inner PE does is load a pointer to `std::cout` and stream the string `Hi, I have the flag for you just right here!` to the console. The flag is right there — you just have to keep the program alive long enough to see it.
4. **The inner PE enters an unrolled sleep loop.** Looking at the disassembly around `0x14000106d`:
   ```
   0x14000106d:  mov      ecx, 0xfffffffe   ; duration = ~49.7 days
   0x140001072:  call     qword ptr [rip + 0x1f88]   ; call Sleep
   0x140001078:  mov      ecx, 0xfffffffe
   0x14000107d:  call     qword ptr [rip + 0x1f7d]   ; call Sleep
   ... (86 identical pairs in a row, then the print statement)
   0x14000141f:  mov      rcx, qword ptr [rip + 0x1c6a]   ; load std::cout
   0x140001426:  lea      rdx, [rip + 0x1f33]             ; "Ok, I'm Up! The flag is: ..."
   0x14000142d:  call     0x140001450                     ; cout << rdx
   0x140001432:  mov      rcx, rax
   0x140001435:  lea      rdx, [rip + 0x1f4]              ; base64 flag string
   0x14000143c:  call     qword ptr [rip + 0x1c6e]         ; cout << rdx
   0x140001442:  xor      eax, eax
   0x140001444:  add      rsp, 0x28
   0x140001448:  ret
   ```
   The author unrolled the "loop" into 86 copies of the same `mov ecx, 0xfffffffe; call Sleep` pattern. A loop unroll is harmless on its own; the *unusual* part is that the program is *not* going to come out the other side naturally — the duration is `0xfffffffe` ms, i.e. ~49.7 days, which is the practical "I am not coming back" value.
5. **The description's "wake up" maps directly to Frida's `args[0] = 0`.** When the Frida handler replaces `args[0]` (the sleep duration) with `0` on every call, `Sleep` returns immediately. The unrolled sequence of 86 calls burns through in microseconds, control falls through to the `cout << "Ok, I'm Up!..."` line, and the flag is printed.
6. **The flag is base64-encoded, not encrypted.** The "real" flag is `picoCTF{w4ke_m3_up_w1th_fr1da_f27acc38}`. The `f27acc38` tail is the per-team salt that the picoCTF platform appends to make every instance unique to the user who downloaded the binary.
7. **Me (Frida path):** I installed Frida, ran `frida-trace -i "Sleep" -f "wine bininst1.exe"`, edited the auto-generated handler to force `args[0] = 0`, captured the base64 string from the program's output, decoded it. About 5 minutes including installs.
8. **Me (static path):** I never needed to run the binary. `pefile` told me the outer PE had no `Sleep` import, which made me dig into the `.ATOM` section. I LZMA-decompressed it into a fresh PE, ran `pefile` and `strings` on that, and pulled the base64 string out of the data section. Decoded it. About 3 minutes of work.

Either way: the same base64 string, the same flag.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip -P picoctf` | Extract the password-protected challenge zip |
| `file` | Confirm the binary is a 64-bit Windows PE |
| `wine wine64` | Run the Windows `.exe` on Kali |
| `pip install frida frida-tools` | Install the dynamic instrumentation toolkit |
| `frida-trace -i "Sleep" -f ...` | Hook the sleep API and observe its arguments |
| `nano __handlers__/KERNEL32.DLL/Sleep.js` | Edit the auto-generated handler to force the duration to 0 |
| `base64 -d` | Decode the captured base64 string back to the flag |
| `python3` (one-liner) | Alternative base64 decoder |
| `pefile` (Python, alternative) | Walk the PE import table; spot that the outer PE has no `Sleep` |
| `lzma` (Python, alternative) | Decompress the LZMA1-compressed `.ATOM` section to recover the inner PE |
| `strings` (alternative) | Pull the same base64 string out of the inner PE without ever running it |

---

## Key Takeaways

- **`frida-trace -i "<FunctionName>" -f "<command>"` is the fastest way to peek.** It generates the handler file for you, so you only have to edit the body to add a `console.log` and an `args[0] = ptr(0)` line. You do not need to write the `Interceptor.attach` boilerplate by hand.
- **For "wake the program up" challenges, hook `Sleep` and force `args[0] = 0`.** The Windows `Sleep` API takes a single `DWORD` argument (the duration in ms). Setting it to `0` makes the call return instantly without yielding. Combined with not calling `Sleep` at all, this is the canonical "make a hanging program keep going" trick.
- **A missing import in the import table is itself a clue.** The outer `bininst1.exe` does not import `Sleep` at all. That should make you suspicious: if a Windows program calls `Sleep`, the import *should* be there. Either the program never calls it, or the program is doing something unusual (loading the inner PE, calling via `GetProcAddress`, etc.). The static path is: notice the missing import -> look at the sections -> spot `.ATOM` -> decompress -> re-check imports.
- **The `.ATOM` section is the giveaway.** Custom section names that are not `.text` / `.rdata` / `.data` are the section names of packers, protectors, and runtime decompressors. The fact that this section is large (~5.6 KB) and not loadable as code (it has a non-standard name) is your hint to dig in. The first 5 bytes `5d 00 00 10 00` are a classic LZMA1 properties header.
- **Unrolled loops are an anti-analysis trick, not a feature.** The author wrote 86 copies of `mov ecx, 0xfffffffe; call Sleep` instead of a tight 86-iteration loop. This bloats the binary and makes a casual reader (or a script) less likely to notice the pattern. Recognise it for what it is: a deliberate "I want this to be annoying to read" choice.
- **`0xfffffffe` is a famous near-`INFINITE` value for `Sleep`.** `0xFFFFFFFF` (the actual `INFINITE` constant on Windows) would mean "sleep forever, do not return under any signal." `0xFFFFFFFE` is one less — it is still ~49.7 days, but it lets the program be interrupted by signals. The author picked `0xFFFFFFFE` so the program can be killed cleanly during testing, but it is functionally indistinguishable from "forever" from a user perspective.
- **Frida's `args` array in `onEnter` is mutable.** If you change `args[0]` to `ptr(0)` before the real function runs, the original `Sleep` sees `0` instead of `0xFFFFFFFE`. This is the entire mechanism for "rewriting arguments" in Frida — you are not patching the function, you are patching the stack/registers the function will read.
- **The base64 in the binary is not encrypted, it is just encoded.** `base64 -d` is a one-liner. If the challenge hints at "instrumentation" but the data is sitting in `.rdata` / `.data` in cleartext, take the static win and move on.
- **Wine + Frida on Linux is a workable combo**, but a real Windows VM (or native Windows) avoids the Wine-friction when you are racing a clock. The Frida + handler-edit workflow itself is identical either way.

### Flag wordplay decode

`picoCTF{w4ke_m3_up_w1th_fr1da_f27acc38}` reads as **"wake me up with frida"** written in l33t-speak: `w4ke` for `wake` (`4` for `a`), `m3` for `me` (`3` for `e`), `up` is left as-is, `w1th` for `with` (`1` for `i`), `fr1da` for `frida` (`1` for `i`). The trailing `_f27acc38` is the usual per-team salt that the picoCTF platform appends to make the flag unique to your account. The phrase is a literal description of the intended solution: hook `Sleep` with Frida, force the duration to 0, and the program "wakes up" long enough to print the flag. The challenge title is also a riff on the same idea: "binary instrumentation" is the technique, "wake me up" is the specific application.
