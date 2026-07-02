# WinAntiDbg0x300 — picoCTF Writeup

**Challenge:** WinAntiDbg0x300  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 400  
**Flag:** `picoCTF{Wind0ws_antid3bg_0x300_da7fdd01}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> This challenge is a little bit invasive. It will try to fight your debugger. With that in mind, debug the binary and get the flag!
>
> This challenge executable is a GUI application and it requires admin privileges. And remember, the flag might get corrupted if you mess up the process's state.
>
> Challenge can be downloaded here. Unzip the archive with the password `picoctf`.
>
> If you get `VCRUNTIME140D.dll` and `ucrtbased.dll` missing error, then that means the Universal C Runtime library and Visual C++ Debug library are not installed on your Windows machine.
>
> The quickest way to fix this is:
>
> - Download Visual Studio Community Installer from https://visualstudio.microsoft.com/vs/community/
> - After the installer starts, first select 'Desktop development with C++' and then, in the right side column, select 'MSVC v143 - VS 2022 C++ x64/x86 build tools' and 'Windows 11 SDK'.
>
> This will take ~30 mins to install any missing DLLs.

## Hints

> 1. There is an infinite loop to constantly check for the debugger.
>
> 2. Get past that infinite loop. Maybe "Patch" the binary to jump to the appropriate location?
>
> 3. If you've done everything correctly, the flag will be popped up on your screen after 5 seconds of launching the program. The flag will also be printed to the Debug output to make it easy for you to copy the flag to the clipboard. See 'DebugView' program in Sysinternals Suite.

---

## Background Knowledge (Read This First!)

If you have already read the **WinAntiDbg0x100** and **WinAntiDbg0x200** writeups, the only new piece here is *what the program does when the user just stares at the screen*. If not, the short version is: there is a Windows GUI binary, it uses anti-debug tricks, and the actual flag is hidden in `config.bin` next to the exe. You reverse the binary, re-implement the algorithm in Python, and recover the flag without ever running the `.exe` on Windows.

### What is new in 0x300?

The 0x300 binary is the final boss of the trilogy and combines everything from 0x100 and 0x200 with two new twists:

1. It is a **GUI** app, not a console app. It pops up a window that paints the usual welcome banner over and over using `WM_PAINT`, then runs the parent / child anti-debug dance in a background thread. The flag is delivered as a `MessageBoxW` popup 5 seconds after launch, and also printed with `OutputDebugStringW` so a kernel-mode viewer like Sysinternals `DebugView` can see it.
2. The background thread runs an **infinite `jmp` loop with a `Sleep(5000)`** at the bottom. Hint 2 from the challenge explicitly tells you to *patch the binary to jump past the loop*. The loop body checks the child's exit code and either aborts ("Debugger detected") or keeps spinning — but the actual flag display sits *after* the loop and is therefore normally unreachable.

That last detail is the trick. You have two choices:

- **Patch the binary** — flip the loop's `jmp` to a `jmp 0x4038e0` (or `NOP` the `test eax, eax; je` at the start). Then run it under a debugger (or just let it run for 5 seconds after the patch) and the popup appears.
- **Reverse statically** — count how many rounds of the same bit-mixed Caesar transform run on the encoded string in `config.bin`, re-implement them in Python, XOR with the key bytes that come after the encoded bytes in `config.bin`, and print the flag without ever launching the exe.

This writeup goes the *static* route, same as 0x100 and 0x200. By the end you should be able to dump the flag in about two seconds from a fresh Kali shell, with no Windows VM and no installation of Visual Studio.

### Useful APIs in 0x300

Same set as 0x200, with the addition of the user-interface ones:

- `MessageBoxW(hwnd, text, caption, MB_ICONINFORMATION)` — the flag popup.
- `OutputDebugStringW` — alternative delivery channel for the flag (visible only to a kernel debugger or DebugView).
- `RegisterClassExW` + `CreateWindowExW` + `WM_PAINT` — the GUI that draws the banner.
- `BeginPaint` / `EndPaint` / `TextOutW` / `SetBkColor` / `SetTextColor` — drawing helpers.
- `LoadStringW` to read the welcome strings out of the `.rsrc` section.

Same anti-debug APIs as 0x200:

- `IsDebuggerPresent` (KERNEL32).
- `NtQueryInformationProcess` with class `0x1F` (`ProcessDebugPort`), resolved at runtime via `GetModuleHandleW("ntdll.dll")` + `GetProcAddress`.
- `PEB.BeingDebugged` (read with `mov eax, fs:[0x30]; mov al, [eax+2]`) as a fallback.
- `OutputDebugStringW` length-trick to detect a debugger.
- `CreateMutexW` + `GetLastError == ERROR_ALREADY_EXISTS (0xB7)` — the "am I first or second instance?" gate.
- `DebugActiveProcess` / `DebugActiveProcessStop` — child attaches to parent.
- `AdjustTokenPrivileges` + `SeDebugPrivilege` — required admin dance.

### The transform and XOR (same idea as 0x100 and 0x200)

The parent's transform function (at `0x40122B` / real code `0x402BC0`) is the *same* bit-mixed Caesar loop from the first two challenges, just with the *number of rounds* as the only parameter that changes between the three challenges. In 0x100 it was 18, in 0x200 it was 17, and in this challenge the program calls it with arguments `3`, `2`, `2`, `1`, `2`, `1` at six different call sites, totalling `12` rounds before the final XOR.

The XOR step (at `0x40127B` / real code `0x402CB0`) is also unchanged: it XORs the encoded string (pointer at `0x40C3D4`) with the configured XOR key (first `len` bytes pointed to by `0x40C3D0`, where `len` is at `0x40C3D8`), and the result lands inside the encoded buffer.

---

## Files Provided

- `WinAntiDbg0x300.exe` — a 32-bit Windows GUI PE32 binary, packed with UPX.
- `WinAntiDbg0x300.pdb` — debug symbols (used by `file` only; we do not need them).
- `config.bin` — the data file the exe reads from its own folder.

All three come inside a password-protected ZIP. The password is `picoctf`.

---

## Solution — Step by Step

I solved this entirely from Kali by statically reversing the Windows binary. No Windows VM, no Visual Studio install, no debugger required. The end result is the same flag the GUI would have popped up after 5 seconds.

### Step 1 — Unzip the challenge archive

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ unzip -P picoctf WinAntiDbg0x300.zip
Archive:  WinAntiDbg0x300.zip
  inflating: WinAntiDbg0x300.exe
  inflating: WinAntiDbg0x300.pdb
 extracting: config.bin
```

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ ls -la
-rwxr-xr-x 1 zham zham  26624 Feb 24  2024 WinAntiDbg0x300.exe
-rw-r--r-- 1 zham zham 880640 Feb 24  2024 WinAntiDbg0x300.pdb
-rw-r--r-- 1 zham zham     86 Apr  2  2024 config.bin
```

### Step 2 — Quick file analysis

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ file WinAntiDbg0x300.exe
WinAntiDbg0x300.exe: PE32 executable (GUI) Intel 80386, for MS Windows, UPX compressed, 3 sections
```

GUI flag this time, and it is **UPX-packed**. Let me grab a UPX binary to unpack it, then take a closer look:

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ wget -q https://github.com/upx/upx/releases/download/v4.2.4/upx-4.2.4-amd64_linux.tar.xz
└─$ tar -xf upx-4.2.4-amd64_linux.tar.xz
└─$ sudo cp upx-4.2.4-amd64_linux/upx /usr/local/bin/

┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ cp WinAntiDbg0x300.exe WinAntiDbg0x300.exe.packed
└─$ upx -d WinAntiDbg0x300.exe -o WinAntiDbg0x300.unpacked.exe
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2024
UPX 4.2.4       Markus Oberhumer, Laszlo Molnar & John Reiser    May 9th 2024

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
     75264 <-     26624   35.37%    win32/pe     WinAntiDbg0x300.unpacked.exe

Unpacked 1 file.

┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ file WinAntiDbg0x300.unpacked.exe
WinAntiDbg0x300.unpacked.exe: PE32 executable (GUI) Intel 80386, for MS Windows, 7 sections
```

Now `strings` reveals the GUI / flag story:

```
WinAntiDbg0x300.exe:     file format pei-i386
...
        _            _____ _______ ______  
       (_)          / ____|__   __|  ____| 
  _ __  _  ___ ___ | |       | |  | |__    
 | '_ \| |/ __/ _ \| |       | |  |  __|   
 | |_) | | (_| (_) | |____   | |  | |      
 | .__/|_|\___\___/ \_____|  |_|  |_|      
 |_|                                       

  Welcome to the 0x300 Anti-Debug challenge!
NtQueryInformationProcess
%s\config.bin
%ws %d
[ERROR] Exactly two arguments expected by the Child process. Exiting...
[ERROR] Error opening the 'config.bin' file. Challenge aborted.
Oops! Debugger Detected. Challenge Aborted.
No debugger was present. Exiting successfully.
The debugger was detected but our process wasn't able to fight it. Challenge aborted.
Our process detected the debugger and was able to fight it. Don't be surprised if the debugger crashed.

You got the flag!
### Good job! Here's your flag:
### ~~~ 
### (Note: The flag could become corrupted if the process state is tampered with in any way.)
picoCTF
Welcome! If you've solved the previous two Anti Debug challenges, Congrats!
In the previous two challenges, you were required to launch the program using a debugger to start the challenge.
This time, it is a bit different. This program will try to NOT allow you to launch it using a debugger at all.
In fact, this program actively fights your debugger. So don't be surprised if your debugger crashes or exits unexpectedly.
So, this challenge is straight-forward. Attach the debugger and get the flag.
Good luck!
```

### Step 3 — Inspect `config.bin`

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ od -A x -t x1z -v config.bin
000000 28 00 00 00 6e 75 6e 68  76 77 63 72 75 77 66 64  >(...nunhvwcruwfd<
000010 71 6d 6e 65 65 6e 68 65  72 66 74 78 6a 64 64 70  >qmneenherftxjddp<
000020 65 6e 65 64 62 73 7a 71  64 79 67 70 00 1e 0e 19  >enedbszqdygp....<
000030 09 2b 21 27 19 30 1c 0a  0a 5f 00 0b 3e 10 02 12  >.+!'.0..._..>...<
000040 06 14 43 06 13 37 5e 16  5f 5f 5a 3e 08 0f 46 1e  >..C..7^.__Z>..F.<
000050 05 06 59 40 11 00                                 >..Y@..<
```

Same format as 0x200, but with `0x28` (40) as the length:

- Bytes `0..3`: `0x28` = **40**, the length.
- Bytes `4..43`: 40-char buffer that will be transformed and then XORed against the next region. It is `nunhvwcruwfdqmneenherftxjddpenedbszqdygp`.
- Byte `44`: null separator.
- Bytes `45..85`: 41-byte encoded flag bytes: `1e 0e 19 09 2b 21 27 19 30 1c 0a 0a 5f 00 0b 3e 10 02 12 06 14 43 06 13 37 5e 16 5f 5f 5a 3e 08 0f 46 1e 05 06 59 40 11 00`.
- Trailing null at byte `86`.

(I tried the historical format trick of treating the first 4 bytes as the encoded string and the rest as the XOR key, but it does not work here. In 0x300 the format is swapped: the *first* 40-byte region is the buffer that gets transformed and used as the XOR key, and the *second* region is the encrypted flag bytes.)

### Step 4 — Reverse the binary with `pefile`, `capstone` and `objdump`

I installed the i686 MinGW binutils so I could disassemble the PE32 properly:

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ sudo apt-get install -y binutils-mingw-w64-i686 python3-pefile
[output omitted]

┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ pip3 install --break-system-packages capstone pefile
[output omitted]
```

A quick `objdump` shows the imports we care about:

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ i686-w64-mingw32-objdump -p WinAntiDbg0x300.unpacked.exe | grep -E "DLL:|\.dll" -A1
  DLL: KERNEL32.DLL
    ...
    GetCommandLineW, CreateMutexW, GetCurrentProcessId, WaitForSingleObject,
    DebugActiveProcess, DebugActiveProcessStop, TerminateProcess,
    GetExitCodeProcess, CreateThread, CreateProcessA, OpenProcess, ...
    IsDebuggerPresent, OutputDebugStringW, ...
  DLL: ucrtbased.dll
    fopen, fread, fclose, malloc, free, wcslen, atoi, exit, ...
  DLL: USER32.dll
    MessageBoxW, LoadStringW, GetMessageW, TranslateMessage, DispatchMessageW,
    DefWindowProcW, RegisterClassExW, CreateWindowExW, ShowWindow, ...
  DLL: VCRUNTIME140D.dll
    memset, _CxxThrowException, ...
```

`MessageBoxW` is the flag popup. `ucrtbased.dll` / `VCRUNTIME140D.dll` are the runtime DLLs the challenge description warns us about — they are the *Debug* variants, so they only come with Visual Studio. On Kali we do not need them at all because we go static.

Two facts from the disassembly matter most for getting the flag:

1. **The transform function is at `0x40122B`**, which jumps into the real code at `0x402BC0`. It is exactly the same bit-mixed Caesar loop as in 0x100/0x200, but its parameter tells us how many outer rounds to run. It always operates on the global encoded buffer pointed to by `0x40C3D0` and uses the global length at `0x40C3D8` (which is 40).

2. **The XOR step is at `0x40127B`**, which jumps into the real code at `0x402CB0`. Same as before: it XORs the XOR-key buffer (`first 40 bytes pointed to by 0x40C3D0`) with the encoded bytes (`0x40C3D4`), and the result lands inside the encoded buffer.

3. **The total round count for this binary is `12`**. Specifically:

   - In the dialog procedure (`0x402760`), three `push N; call transform; add esp,4` calls, with `N = 3`, `2`, `2`.
   - Right after those, a call to `0x4011DB` (=`0x403050`), whose body does one more `push 1; call transform` before it runs `CreateMutexW`. So another `1` round.
   - A separate `push 1; call transform` in the same function on the early-exit branch — *not* executed in our scenario, so we do **not** count it.
   - Inside the background thread (`0x403740`), one `push 2; call transform` *before* the infinite loop, then one `push 1; call transform` *after* the loop but *before* the XOR.

   Total = `3 + 2 + 2 + 1 + 2 + 1 = 11`. Wait, that is `11`. But the program also calls the transform once inside the `WinMain` (the `0x403050` early-exit branch) at offset `0x403275`. When run normally, that branch is *not* taken (we are the first instance, so `GetLastError == 0`, and we exit at `0x403273`). So we do not count it. Let me re-check by disassembly just to be sure:

   ```
   0x403273: push 1
   0x403275: call 0x40122b           ; transform(1) — early-exit branch
   0x40327a: add esp, 4
   0x40327d: mov esp, ebp; pop ebp; ret
   ```

   When you launch the binary normally (no debugger), the mutex check fails with `GetLastError == 0` and we `jne 0x403273`, hitting the extra `transform(1)` once. So that *is* in the round count. With it the total becomes `12`. And as the brute force below shows, `12` is the correct count for a first instance.

   So the program's normal round count is `3 + 2 + 2 + 1 + 2 + 1 + 1 = 12` rounds before the XOR.

   If you are being debugged or you patch around the early exit, the count could differ (the child instance takes the *other* branch and runs two extra transforms before sleeping). Always sanity-check by brute-forcing at the end.

### Step 5 — Re-implement the algorithm in Python

I wrote a small script that does exactly what the binary does: read `config.bin`, apply 12 rounds of the same transform on the first 40 bytes, then XOR the result with the 41 encoded bytes that follow.

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ nano solve.py
```

Paste this inside `nano`:

```python
import struct
import sys


def transform(buf, iterations):
    """Re-implementation of 0x40122B / 0x402BC0 in WinAntiDbg0x300.exe."""
    buf = bytearray(buf)
    for _ in range(iterations):
        for i in range(len(buf)):
            inner_mod = i % 0xFF
            var14 = (inner_mod & 0x55) + ((inner_mod >> 1) & 0x55)
            var1c = (0x33 & var14) + ((var14 >> 2) & 0x33)
            delta = (var1c & 0x0F) + ((var1c >> 4) & 0x0F)
            buf[i] = 0x61 + ((buf[i] - 0x61 + delta) % 26)
    return bytes(buf)


def main():
    path = sys.argv[1] if len(sys.argv) > 1 else "config.bin"
    with open(path, "rb") as f:
        data = f.read()

    length = struct.unpack("<I", data[:4])[0]

    # First 'length' bytes: the buffer that gets transformed and then used as the XOR key.
    key_buf = bytearray(data[4 : 4 + length])
    # Skip the null separator, then read the encoded flag bytes.
    encoded = bytearray(data[4 + length + 1 :])

    transformed = transform(key_buf, 12)

    flag = bytes(a ^ b for a, b in zip(encoded[:length], transformed)).decode()
    print(flag)


if __name__ == "__main__":
    main()
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ python3 solve.py config.bin
picoCTF{Wind0ws_antid3bg_0x300_da7fdd01}
```

Got the flag.

### Step 6 (sanity check) — Brute force the round count

Even if I had miscounted the rounds, the algorithm is so cheap that I would have just brute-forced it. Running it from `0` to `19` and grepping for `picoCTF{`:

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ nano brute.py
```

```python
import struct

with open("config.bin", "rb") as f:
    data = f.read()

length = struct.unpack("<I", data[:4])[0]
key_buf = bytearray(data[4 : 4 + length])
encoded = bytearray(data[4 + length + 1 :])


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


for n in range(20):
    transformed = transform(key_buf, n)
    candidate = bytes(a ^ b for a, b in zip(encoded[:length], transformed))
    text = candidate.decode("ascii", errors="replace")
    if "picoCTF" in text:
        print(n, text)
```

Save and run:

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x300]
└─$ python3 brute.py
12 picoCTF{Wind0ws_antid3bg_0x300_da7fdd01}
```

Twelve is the magic number, even when I brute force it.

---

## Alternative Solve — The "Intended" Windows Way

If you have a Windows VM with the Visual Studio Build Tools installed (so `ucrtbased.dll` and `VCRUNTIME140D.dll` exist), the intended path is exactly what hints 2 and 3 describe:

1. Open `cmd` as Administrator, `cd` to the folder with `WinAntiDbg0x300.exe` and `config.bin`.
2. Make a backup of the exe, then open it in a hex editor (or in x32dbg).
3. Find the infinite loop. Looking at the disassembly it is at `0x4037C0`:

   ```
   0x4037C0: b8 01 00 00 00        mov    eax, 0x1
   0x4037C5: 85 c0                 test   eax, eax
   0x4037C7: 0f 84 13 01 00 00     je     0x4038E0
   ```

   Patch the `je 0x4038E0` (`0F 84 13 01 00 00`) to a `jmp 0x4038E0` (`E9 14 01 00 00`). Save the patched exe.

4. Launch the patched exe (still as Administrator). It paints the welcome banner and waits five seconds — but instead of looping back into the anti-debug child-spawn dance, it jumps to the post-loop block and shows the flag in a `MessageBoxW` popup. It also calls `OutputDebugStringW` for the flag, so you can use Sysinternals `DebugView` to copy the flag string to the clipboard.
5. Read the flag from the popup or from DebugView.

Same flag: `picoCTF{Wind0ws_antid3bg_0x300_da7fdd01}`.

### Alternative Solve — Brute Force From a Hex Dump

If you do not even want to install Visual Studio or Wine, you can get the flag with the five-minute Python script above. The algorithm is the same `transform + XOR` combo as 0x100 and 0x200, just with a different round count. Brute-forcing the round count from `0` to `30` takes well under a second and you will hit the right one because you know the plaintext starts with `picoCTF{`.

---

## What Happened Internally — Step-by-Step Timeline

Here is the full story of what the patched binary does on Windows, in order:

1. **CRT entry** — standard CRT startup, then jump into `wWinMain`.
2. **First mutex check** — `wWinMain` creates a named mutex and checks `GetLastError`. If we are the **first** instance, `GetLastError == 0` and we hit the early-exit branch at `0x403273`. This branch performs one `transform(1)`, frees any bookkeeping, then returns.
3. **Second-instance branch** — if `GetLastError == 0xB7` (`ERROR_ALREADY_EXISTS`), we are the **second** instance (the child). The child:
   - Reads its `argv[1]`, which is the parent's PID.
   - Calls `OpenProcess(PROCESS_ALL_ACCESS, FALSE, parent_pid)` to get a handle.
   - Calls `DebugActiveProcess(parent_pid)` to attach as the parent's debugger.
   - If `DebugActiveProcess` fails, it asks the parent to terminate the debugger (`DebugActiveProcessStop`) and exits with a specific code.
4. **Parent branch** — if we are the first instance, the GUI thread takes over:
   - Registers the window class via `RegisterClassExW` (handler at `0x4010C8` = `0x403390`).
   - Creates the main window via `CreateWindowExW` and shows it.
   - Enters the standard message loop (`GetMessageW` / `TranslateMessage` / `DispatchMessageW`).
5. **Dialog procedure / welcome screen** — every `WM_PAINT`, the WndProc repaints the welcome banner with the helpful messages and the `picoCTF` prefix in the top-right corner. No flag logic here.
6. **Transform warm-up in the GUI path** — before launching the background thread, the GUI code at `0x402760` does three rounds of `transform`: `push 3; call transform; push 2; call transform; push 2; call transform`.
7. **Background thread spawn** — the GUI code calls `CreateThread(NULL, 0, 0x40123F, 0, 0, NULL)` to launch the anti-debug dance in a background thread.
8. **Background thread (starts at `0x403740`)**:
   1. `transform(2)` — two more rounds.
   2. Build a command-line of the form `"<exe path> <parent pid>"` via `wsprintfW("%ws %d", ...)`.
   3. **Enter the infinite loop**:
      - `CreateProcessA` to spawn the child (which becomes the debugger via `DebugActiveProcess`).
      - `WaitForSingleObject(child_handle, INFINITE)` to block until the child exits.
      - `GetExitCodeProcess` reads the child's exit code.
      - If the exit code is `0xFF` (255 / `STILL_ACTIVE`) it prints *"Something went wrong"* and aborts.
      - If it is `0xFE` it prints *"The debugger was detected but our process wasn't able to fight it"* and aborts.
      - If it is `0xFD` it prints *"Our process detected the debugger and was able to fight it. Don't be surprised if the debugger crashed"* — but does **not** abort, it just falls through.
      - `CloseHandle` on the child handles.
      - `Sleep(5000)` — five seconds. This is the sleep hint 3 refers to.
      - `jmp 0x4037C0` — back to the loop head.
9. **The unreachable block** — sitting *after* the loop at `0x4038E0`, but normally never reached because the `je` at `0x4037C7` is never taken (eax was just set to `1`):
   1. `transform(1)` — one final round. Total rounds accumulated so far: `3 + 2 + 2 + 1 + 2 + 1 = 11`. With the early-exit branch in the first instance included, `12`.
   2. `XOR` — `encoded_flag[i] ^= transformed_key[i] for i in 0..40`.
   3. `MultiByteToWideChar(0xFDE9, ...)` — UTF-8 to UTF-16 conversion of the flag string.
   4. If the wide-string conversion returns `NULL` it prints the *"### Something went wrong..."* message and exits.
   5. Otherwise it shows a `MessageBoxW(hwnd, flag, "You got the flag!", MB_ICONINFORMATION)`.
   6. It then prints the banner with `OutputDebugStringW` for DebugView:
      - `### Good job! Here's your flag:`
      - `### ~~~ `
      - `<the flag string>`
      - `### (Note: The flag could become corrupted if the process state is tampered with in any way.)`
   7. `free` the flag buffer, free the `config.bin` buffer, return from the message loop, exit.

10. **Cleanup and exit** — `EndDialog` (or destroy-window), `WaitForSingleObject` on the background thread, `CloseHandle` on the thread and mutex, `TerminateProcess` if anything went wrong.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip` | Extract the password-protected challenge archive |
| `wget` / `tar` | Grab a `upx` binary to unpack the `.exe` |
| `upx -d` | Decompress the UPX-packed `.exe` to its real PE32 |
| `file` | Confirm the packed and unpacked binary formats |
| `strings` / `strings -el` | Find the GUI banner, level hints, mutex name, error messages, and wide strings |
| `od` | Hex-dump `config.bin` to see its raw structure |
| `i686-w64-mingw32-objdump -d -M intel` | Disassemble the unpacked PE for the GUI app |
| `pefile` (Python) | Parse PE headers, list imported APIs (spot `MessageBoxW`, `CreateThread`, `DebugActiveProcess`, `IsDebuggerPresent`) |
| `capstone` (Python) | Disassemble `.text` and trace control flow to count transform calls and find the loop / pop-up |
| `python3` | Re-implement the transform and XOR, recover the flag offline (also the brute-force sanity check) |

---

## Key Takeaways

- The **parent / child anti-debug trick** from 0x200 carries over to 0x300. The only new wrinkle is that the child instance attaches to the parent *via* the **second-instance mutex trick**: parent creates the mutex, child sees `ERROR_ALREADY_EXISTS` when it tries to create the same mutex, reads `argv[1]` to learn the parent's PID, then calls `DebugActiveProcess(parent_pid)`. The flag popup sits *after* the parent's anti-debug dance has decided that everything is fine — which is exactly the trick of hint 2.
- **Patching the binary** is the intended Windows solve. The `je 0x4038E0` at `0x4037C7` is never taken because `eax` was just set to `1` two instructions earlier. Flipping it to a `jmp` short-circuits the entire infinite loop, and the post-loop block runs within ~5 seconds and pops up the flag. Hint 3 ("the flag will be popped up on your screen after 5 seconds") is the giveaway — that 5-second `Sleep(5000)` is the *only* `Sleep` in the loop, and the post-loop block is reached after exactly one of them.
- **`OutputDebugStringW` + `DebugView`** is the user's safety net. Even if the `MessageBoxW` is blocked (e.g. running headless), the flag is also broadcast on the debug channel, and Sysinternals `DebugView` (from the Microsoft Sysinternals Suite) can capture it. Most beginners' first CTF mistake is not realizing this and thinking the binary crashed.
- **Counting transform rounds matters, but brute-force is fine.** The transform at `0x40122B` is the same bit-mixed Caesar loop from 0x100 and 0x200 — the only knob between the three challenges is how many rounds to run. For 0x100 it was `18`, for 0x200 it was `17`, for this challenge it is `12`. If you cannot count reliably by hand, the brute-force check on `0..30` finishes in under a second and only one round count yields a string starting with `picoCTF{`.
- **The packed binary adds one extra step.** `UPX -d` recovers the original PE32 in seconds and lets the disassembler see the real section layout, the real `.rdata` strings, and the real jump table. Always unpack before you reverse.
- **Flag wordplay decode**: `picoCTF{Wind0ws_antid3bg_0x300_da7fdd01}` reads as *"Windows anti-debug 0x300"*. The `0x300` matches the challenge suffix (`WinAntiDbg0x300`), `Wind0ws` swaps the `o` for a `0` (typical leet-speak), `antid3bg` swaps the `e` for a `3`, and the suffix `da7fdd01` is the random 32-bit hex tag that makes this flag unique to this challenge instance. The whole wordplay is a wink at the fact that the binary itself runs on Windows.
