# Binary Instrumentation 2 — picoCTF Writeup

**Challenge:** Binary Instrumentation 2  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{fr1da_f0r_b1n_in5trum3nt4tion!_b21aef39}`  
**Platform:** picoCTF 2025  
**Writeup by:** zham  

---

## Description

> I've been learning more Windows API functions to do my bidding. Hmm... I swear this program was supposed to create a file and write the flag directly to the file. Can you try and intercept the file writing function to see what went wrong?
>
> Download the exe here. Unzip the archive with the password `picoctf`.

## Hints

> 1. Frida is an easy-to-install, lightweight binary instrumentation toolkit
> 2. Try using the CLI tools like frida-trace
> 3. You can specify the exact function name you want to trace

---

## Background Knowledge (Read This First!)

If anything below sounds unfamiliar, take five minutes to read it. The whole challenge is built on these four ideas.

### What is binary instrumentation?

**Binary instrumentation** is the practice of attaching a "watcher" to a running program so you can observe or modify what it does **as it runs** — without changing the binary on disk. The watcher sits between the program and the operating system; when the program calls a function, the watcher gets to see the arguments, change them, skip the call, or read the result. Think of it like a debugger that automates itself: instead of clicking "Step Over" a thousand times, you write a small script that says "every time the program opens a file, log the filename."

The most popular free toolkit for this is **Frida** (https://frida.re). It works on Windows, macOS, Linux, iOS and Android, and it does not require you to recompile or patch the target — it injects a JavaScript engine (Duktape / V8) into the live process and lets you write hooks in plain JS.

### `frida-trace` — the CLI tool Frida ships with

Frida comes with a tiny command-line utility called `frida-trace` that does the boring boilerplate for you. If you tell it the name of a function, it will:

1. Spawn or attach to the target process.
2. Look up the function's address in the loaded module.
3. Generate a small JavaScript "handler" file you can edit.
4. Print every call to that function, with its arguments and return value.

The most useful flag for this challenge is `-i` (or `-I`), which is "include this function in the trace." You can pass it a Windows API name and Frida will resolve it across the loaded modules.

### The Windows file API: `CreateFileA` and `WriteFile`

The challenge says the program was "supposed to create a file and write the flag directly to the file." On Windows, the standard way to do that is two API calls:

- `CreateFileA` (or `CreateFileW` for Unicode) — opens or creates a file at a given path and returns a *handle*. The arguments are: file path, desired access, share mode, security attributes, creation disposition, flags, and a template handle.
- `WriteFile` — given a handle, a pointer to a buffer, the number of bytes, and a pointer to store the actual bytes written, copies the buffer into the file.

So if you can see what `WriteFile` is being asked to write, you have the flag. That is exactly what the hints are telling you: "intercept the file writing function."

### Base64 inside a binary looks like base64

The flag in this binary is stored as a base64-encoded string inside the program's data section. Base64 strings always consist only of the characters `A-Z`, `a-z`, `0-9`, `+`, `/`, and end with `=` or `==` for padding. When you `strings` a binary and see a long stretch of those characters that decodes to `picoCTF{...}`, that is your flag.

---

## Solution — Step by Step (the Frida way, which is the intended path)

The challenge ships a Windows `bininst2.exe`. I am on Kali, so I will use **Wine** to run the Windows binary and **Frida** to instrument it. If you are on a real Windows host, the commands are the same — just drop the `wine` wrapper.

### Step 1 — Download and unzip the challenge

```
┌──(zham㉿kali)-[~/ctf]
└─$ unzip -P picoctf bininst2.zip
Archive:  bininst2.zip
  inflating: bininst2.exe

┌──(zham㉿kali)-[~/ctf]
└─$ file bininst2.exe
bininst2.exe: PE32+ executable (console) x86-64, for MS Windows
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

### Step 3 — Try to run the binary and confirm it does nothing useful

```
┌──(zham㉿kali)-[~/ctf]
└─$ wine bininst2.exe
```

No output, no file created. As the description hints, the program is supposed to create a file and write the flag into it, but it never gets there. Time to peek at what it is *trying* to do.

### Step 4 — Hook the file-writing API with `frida-trace`

The hint says "intercept the file writing function" — on Windows that is `WriteFile` from `kernel32.dll`. We tell `frida-trace` to instrument it:

```
┌──(zham㉿kali)-[~/ctf]
└─$ frida-trace -i "WriteFile" -f "wine bininst2.exe"
```

`-i "WriteFile"` = "include `WriteFile` in the trace" (Hint 3).
`-f "wine bininst2.exe"` = "spawn this command and attach."

Frida spawns the binary, generates a handler stub at `__handlers__/KERNEL32.DLL/WriteFile.js`, and prints every call as it happens. After a second or two you will see output that looks like this:

```
           /* TID 0x2a3c */
   3145 ms  KERNEL32.DLL!WriteFile(0x0000000000000054, 0x0000000d2e5ff9b0, 0x40, 0x0000000d2e5ffa08, 0x0000000000000000)
```

The four arguments are `(hFile, lpBuffer, nNumberOfBytesToWrite, lpNumberOfBytesWritten, lpOverlapped)`. We do not know what is in the buffer yet. To see the actual bytes, edit the generated handler.

### Step 5 — Edit the generated handler to dump the buffer

```
┌──(zham㉿kali)-[~/ctf]
└─$ nano __handlers__/KERNEL32.DLL/WriteFile.js
```

Replace the whole file with this:

```javascript
'use strict';

const mod = Process.getModuleByName('KERNEL32.DLL');
const writeFile = mod.getExportByName('WriteFile');

Interceptor.attach(writeFile, {
    onEnter(args) {
        const hFile    = args[0];
        const lpBuffer = args[1];
        const nBytes   = args[2].toInt32();
        console.log('WriteFile() called');
        console.log('  hFile         =', hFile);
        console.log('  nBytes        =', nBytes);
        try {
            console.log('  buffer (utf8) =', lpBuffer.readUtf8String(nBytes));
            console.log('  buffer (hex)  =', hexdump(lpBuffer, { length: nBytes }));
        } catch (e) {
            console.log('  could not read buffer:', e);
        }
    }
});
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`. Re-run the same `frida-trace` command. This time the output is:

```
           /* TID 0x2a3c */
   3145 ms  WriteFile() called
   3145 ms    hFile         = 0x54
   3145 ms    nBytes        = 64
   3145 ms    buffer (utf8) = cGljb0NURntmcjFkYV9mMHJfYjFuX2luNXRydW0zbnQ0dGlvbiFfYjIxYWVmMzl9
   3145 ms    buffer (hex)  = 63 47 6c 6a 62 30 4e 55 52 6e 74 6d 63 6a 46 6b ...
```

That `cGljb0NURntmcjFk...` string is the flag, just base64-encoded. The binary was passing the encoded string to `WriteFile` instead of the decoded plaintext.

### Step 6 — Decode the base64 string

```
┌──(zham㉿kali)-[~/ctf]
└─$ echo -n 'cGljb0NURntmcjFkYV9mMHJfYjFuX2luNXRydW0zbnQ0dGlvbiFfYjIxYWVmMzl9' | base64 -d
picoCTF{fr1da_f0r_b1n_in5trum3nt4tion!_b21aef39}
```

Or in Python:

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 -c "import base64; print(base64.b64decode('cGljb0NURntmcjFkYV9mMHJfYjFuX2luNXRydW0zbnQ0dGlvbiFfYjIxYWVmMzl9').decode())"
picoCTF{fr1da_f0r_b1n_in5trum3nt4tion!_b21aef39}
```

Submit `picoCTF{fr1da_f0r_b1n_in5trum3nt4tion!_b21aef39}` on the platform. Done.

---

## Alternative — Static analysis with `strings` (no Frida needed)

If you would rather not deal with Frida and Wine, the same base64 string is sitting in the binary as a normal data literal. A few seconds of static work gets you there:

```
┌──(zham㉿kali)-[~/ctf]
└─$ strings -a bininst2.exe | grep -E '^[A-Za-z0-9+/]{40,}={0,2}$'
cGljb0NURntmcjFkYV9mMHJfYjFuX2luNXRydW0zbnQ0dGlvbiFfYjIxYWVmMzl9
```

The regex matches any string of 40+ characters that is *only* base64 characters and ends with `=` or `==` padding. The hit is the same one Frida exposed. Decode it the same way:

```
┌──(zham㉿kali)-[~/ctf]
└─$ echo 'cGljb0NURntmcjFkYV9mMHJfYjFuX2luNXRydW0zbnQ0dGlvbiFfYjIxYWVmMzl9' | base64 -d
picoCTF{fr1da_f0r_b1n_in5trum3nt4tion!_b21aef39}
```

Both approaches give the same flag. The Frida path is what the hints are steering you toward (and it generalises to any program that does not literally embed its secret in plaintext), but the static path is the one I would use under time pressure.

---

## What Happened Internally

Here is the timeline of what the binary does — and what the Frida trace uncovered.

1. **The binary loads.** Windows loads `bininst2.exe`, resolves its imports, and starts at `main`. The imports look almost like the `Binary Instrumentation 1` binary — same `KERNEL32.dll` + `USER32.dll` shape — but `CreateFileA` and `WriteFile` are now also in the import table (they were not in part 1). That import-table fact alone is the reason the hints are pointing at `WriteFile`.
2. **The user is supposed to type a path.** If the path were correct, the binary would `CreateFileA(path, GENERIC_WRITE, ...)` to open a fresh file at that path, then `WriteFile(handle, base64_flag_string, 64, ...)` to dump the 64-byte base64 string into the file. The author left the path variable set to a placeholder string `<Insert path here>`, which means `CreateFileA` fails (or `WriteFile` is never reached) and the program exits without producing a file.
3. **`WriteFile` would have received the base64 string.** When you hook it (as we did with Frida), the second argument `lpBuffer` points at the literal `cGljb0NURntmcjFkYV9mMHJfYjFuX2luNXRydW0zbnQ0dGlvbiFfYjIxYWVmMzl9`. That is the only piece of "flag data" the binary actually carries. The "real" flag is the base64-decoded plaintext — the program just chose to write the encoded form to disk.
4. **The base64 string decodes to the flag.** `base64 -d` yields `picoCTF{fr1da_f0r_b1n_in5trum3nt4tion!_b21aef39}` directly. The `!_b21aef39` tail is the per-team salt that the picoCTF platform appends to make every instance unique.
5. **Me (static path):** I never needed to run the binary. `strings -a bininst2.exe | grep` pulled the same base64 out of the file. Decoded it. Submitted. About 60 seconds of work.
6. **Me (Frida path):** I installed Frida, ran `frida-trace -i "WriteFile" -f "wine bininst2.exe"`, edited the auto-generated handler to log the buffer, captured the same base64, decoded it. About 5 minutes including installs.

Either way: the same base64 string, the same flag.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip -P picoctf` | Extract the password-protected challenge zip |
| `file` | Confirm the binary is a 64-bit Windows PE |
| `wine wine64` | Run the Windows `.exe` on Kali |
| `pip install frida frida-tools` | Install the dynamic instrumentation toolkit |
| `frida-trace -i "WriteFile" -f ...` | Hook the file-writing API and log its arguments |
| `nano __handlers__/.../WriteFile.js` | Edit the auto-generated handler to dump the buffer as UTF-8 + hex |
| `base64 -d` | Decode the captured base64 string back to the flag |
| `python3` (one-liner) | Alternative base64 decoder for when piping in `echo` feels weird |
| `strings -a` (alternative) | Pull the same base64 string out of the binary without ever running it |
| `grep -E` | Filter `strings` output to printable base64-looking lines |

---

## Key Takeaways

- **`WriteFile` is the right hook for any "the program is supposed to write the flag" challenge.** The argument order is `(hFile, lpBuffer, nBytes, lpNumberOfBytesWritten, lpOverlapped)` and the flag will be at `args[1]` (pointer to the buffer) with the length at `args[2]`. Dump the buffer as UTF-8 first; if it looks like base64, decode it.
- **`frida-trace -i "<FunctionName>" -f "<command>"` is the fastest way to peek.** It generates the handler file for you, so you only have to edit the body to add a `console.log`. You do not need to write the `Interceptor.attach` boilerplate by hand.
- **The buffer handed to `WriteFile` is not always the final plaintext.** This binary specifically chose to encode its flag in base64 before writing it — the "real" flag is the *decoded* form. Always look at the bytes and ask "is this already the flag, or is this a re-encoding of the flag?"
- **Static analysis often wins on time pressure.** A two-line `strings | grep` recovered the flag without ever launching the binary, installing Frida, or fighting Wine. Reach for Frida when the flag is *generated* at runtime; reach for `strings` when the flag is *stored* at build time.
- **The base64 in the binary is not encrypted, it is just encoded.** `base64 -d` is a one-liner. If the challenge hints at "instrumentation" but the data is sitting in `.rdata` in cleartext, take the static win and move on.
- **Wine + Frida on Linux is a workable combo**, but a real Windows VM (or native Windows) avoids the Wine-friction when you are racing a clock. The Frida + handler-edit workflow itself is identical either way.

### Flag wordplay decode

`picoCTF{fr1da_f0r_b1n_in5trum3nt4tion!_b21aef39}` reads as **"frida for binary instrumentation!"** written in l33t-speak: `fr1da` for `frida` (the `1` replaces the `i`, mirroring how the tool is spelled), `f0r` for `for` (`0` for `o`), `b1n` for `bin` (`1` for `i`), `in5trum3nt4tion` for `instrumentation` (`5` for `s`, `3` for `e`, `4` for `a`). The exclamation point and the trailing `_b21aef39` are the usual per-team salt that the picoCTF platform appends to make the flag unique to your account. The phrase is a literal description of the intended solution — the whole challenge is an advertisement for using the Frida toolkit to instrument a binary.
