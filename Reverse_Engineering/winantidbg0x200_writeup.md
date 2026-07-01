# WinAntiDbg0x200 — picoCTF Writeup

**Challenge:** WinAntiDbg0x200  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{0x200_debug_f0r_Win_e6b68f6e}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> If you have solved WinAntiDbg0x100, you'll discover something new in this one. Debug the executable and find the flag!
>
> This challenge executable is a Windows console application, and you can start by running it using Command Prompt on Windows.
>
> This executable requires admin privileges. You might want to start Command Prompt or your debugger using the 'Run as administrator' option.

## Hints

> Hints will be displayed to the Debug console. Good luck!

---

## Background Knowledge (Read This First!)

If you have already read the **WinAntiDbg0x100** writeup, you can skip straight to the solution. This challenge uses the same overall idea but adds one extra twist. If not, the short version is: there is a Windows PE32 binary, it uses anti-debug tricks, and the actual flag is hidden in `config.bin` next to the exe. You reverse the binary, re-implement the algorithm in Python, and recover the flag without ever running the .exe on Windows.

### What is new in 0x200?

In 0x100 the binary printed the flag itself. In 0x200 the binary uses a **parent / child** trick that is common in real anti-debug research:

- The binary spawns **itself** as a child process.
- The **child** calls `DebugActiveProcess(parent_pid)` to attach to the parent as a debugger.
- The **parent** does the actual flag work and waits for the child to exit.

Why bother? Because in this setup, when the parent calls `IsDebuggerPresent` it returns **0** — the parent has no debugger, only the child does. So the parent's anti-debug check passes. Meanwhile the child is busy sitting in a debug loop and never prints anything user-visible. The whole dance exists to make `IsDebuggerPresent` lie to the parent.

### Useful APIs in 0x200

- `CreateMutexW` plus `GetLastError` returning `ERROR_ALREADY_EXISTS` (`0xB7` = 183) — the classic way to detect that you are the second instance.
- `CreateProcessA` — spawn the child, inheriting handles.
- `DebugActiveProcess(pid)` — make this process the debugger of another pid.
- `WaitForSingleObject` / `GetExitCodeProcess` / `CloseHandle` — basic process plumbing.
- `AdjustTokenPrivileges` / `OpenProcessToken` / `LookupPrivilegeValueW` — turn on `SeDebugPrivilege`, which is required for `DebugActiveProcess` to work. That is why the challenge description says it needs admin.

### The transform and XOR (same idea as 0x100)

The parent's `main` repeatedly calls a transform function on the encoded string from `config.bin`, then XORs the configured XOR key with the now-mangled string to get the flag. The transform is the same bit-mixed Caesar-style loop as in 0x100 — it just runs a different number of times.

---

## Files Provided

- `WinAntiDbg0x200.exe` — the Windows console application
- `config.bin` — the data file the exe reads from its own folder

Both come inside a password-protected ZIP. The password is `picoctf`.

---

## Solution — Step by Step

I solved this entirely from Kali by statically reversing the Windows binary. No Windows VM, no debugger required. The trick is that the binary applies the same transform/XOR combo as in 0x100, just with a different number of rounds — and we can figure out that count by reading the disassembly carefully.

### Step 1 — Unzip the challenge archive

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x200]
└─$ unzip -P picoctf WinAntiDbg0x200.zip
Archive:  WinAntiDbg0x200.zip
  inflating: WinAntiDbg0x200.exe
 extracting: config.bin
```

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x200]
└─$ ls -la
-rwxr-xr-x 1 zham zham 16384 Feb 24  2024 WinAntiDbg0x200.exe
-rw-r--r-- 1 zham zham    80 Apr  2  2024 config.bin
```

### Step 2 — Quick file analysis

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x200]
└─$ file WinAntiDbg0x200.exe
WinAntiDbg0x200.exe: PE32 executable (console) Intel 80386, for MS Windows, 5 sections
```

Same as before — 32-bit Windows PE. Now the imports include the parent / child APIs:

```
KERNEL32.dll:
  GetLastError, WaitForSingleObject, CreateMutexW, GetCurrentProcessId,
  GetExitCodeProcess, CloseHandle, GetModuleFileNameA, GetModuleFileNameW,
  GetModuleHandleW, GetProcAddress, MultiByteToWideChar,
  DebugActiveProcess, OutputDebugStringW, CreateProcessA, IsDebuggerPresent, ...

ADVAPI32.dll:
  AdjustTokenPrivileges, OpenProcessToken, LookupPrivilegeValueW
```

The `ADVAPI32.dll` trio plus `DebugActiveProcess` is the dead giveaway for the parent / child trick. `CreateMutexW` plus `GetLastError == 0xB7` is the "am I the first or second instance?" check.

`strings` confirms the rest of the story:

```
WinAntiDbg0x200
SeDebugPrivilege
### [ERROR] Unable to create the child process. Assuming a debugger messed with it.
### Error reading the 'config.bin' file... Challenge aborted.
### Level 2: Why did the parent process get a promotion at work? Because it had a "fork-tastic" child process that excelled in multitasking!
### Oops! The debugger was detected. Try to bypass this check to get the flag!
### Good job! Here's your flag:
```

The "fork-tastic" pun is the official hint that this is the parent / child trick.

### Step 3 — Inspect `config.bin`

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x200]
└─$ od -A x -t x1z -v config.bin
000000 25 00 00 00 71 78 68 6e 71 71 73 78 68 6b 69 72  >%...qxhnqqsxhkir<
000010 73 77 73 64 72 72 67 74 76 71 71 79 77 79 66 61  >swsdrrgtvqqywyfa<
000020 62 6d 6f 69 70 76 66 66 71 00 01 06 1a 19 2b 2d  >bmoipvffq.....+-<
000030 27 0c 49 0b 43 41 51 29 16 11 0b 0f 08 2c 02 40  >'.I.CAQ).....,.@<
000040 02 30 32 11 0b 2e 04 55 07 46 5f 02 58 00 04 00  >.02....U.F_.X...<
```

Same format as in 0x100:

- Bytes `0..3`: `0x25` = **37**, the length.
- Bytes `4..40`: encoded string `qxhnqqsxhkirswsdrrgtvqqywyfabmoipvffq` (37 chars).
- Byte `41`: null separator.
- Bytes `42..78`: 37-byte XOR key: `01 06 1a 19 2b 2d 27 0c 49 0b 43 41 51 29 16 11 0b 0f 08 2c 02 40 02 30 32 11 0b 2e 04 55 07 46 5f 02`.
- Byte `79`: trailing null.

### Step 4 — Reverse the binary with `pefile` and `capstone`

I disassembled the `.text` section. Two facts from the disassembly matter most for getting the flag:

1. **The transform function is at `0x401090`.** It is the same bit-mixed Caesar loop from 0x100, but its parameter tells us how many outer rounds to run. It always operates on the global encoded string pointer at `0x40509C` and uses the global length at `0x4050A4`.
2. **The parent process calls `0x401090` a total of 17 times before XOR.** Specifically:

   - In the parent's main, right after reading `config.bin` and printing the Level 2 hint, it calls `0x401090(3)`.
   - Then it calls `0x4011D0`, the "spawn child, wait, check exit code" helper.
   - Inside `0x4011D0` it calls `0x401090(5)` **before** `CreateProcessA`, then `0x401090(4)` after `WaitForSingleObject`, then `0x401090(4)` again after closing handles.
   - Back in main it calls `0x401090(1)` right before the XOR.

   That is `3 + 5 + 4 + 4 + 1 = 17` rounds.

   The child process does no transforms — it only runs `DebugActiveProcess` and `exit`.

3. **The XOR step is `0x401180`.** Same as 0x100: it XORs the XOR-key buffer (`0x4050A0`) with the encoded string (`0x40509C`), leaving the flag bytes in `0x4050A0`.

So my plan was simple: re-implement `0x401090` in Python, run it 17 times on the encoded string, XOR with the key from `config.bin`, and print the result.

### Step 5 — Re-implement the algorithm in Python

I wrote a small script that does exactly that.

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x200]
└─$ nano solve.py
```

Paste this inside `nano`:

```python
import struct
import sys


def transform(buf, iterations):
    """Re-implementation of the function at 0x401090 in WinAntiDbg0x200.exe."""
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
    encoded = bytearray(data[4 : 4 + length])
    xor_key = bytearray(data[4 + length + 1 : 4 + length + 1 + length])

    # Parent calls the transform function 3 + 5 + 4 + 4 + 1 = 17 times.
    transformed = transform(encoded, 17)

    flag = bytes(a ^ b for a, b in zip(xor_key, transformed)).decode()
    print(flag)


if __name__ == "__main__":
    main()
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x200]
└─$ python3 solve.py config.bin
picoCTF{0x200_debug_f0r_Win_e6b68f6e}
```

Got the flag.

If you want to double-check that 17 is really the right round count, try other values — `13`, `15`, `16`, `17`, `18` all give different garbage, only `17` gives a clean `picoCTF{...}` string.

---

## Alternative Solve — The "Intended" Windows Way

If you have a Windows VM with x32dbg and an admin shell, the intended path looks like this:

1. Open `cmd` as Administrator, `cd` to the folder with `WinAntiDbg0x200.exe` and `config.bin`.
2. Open `WinAntiDbg0x200.exe` in x32dbg and run it.
3. The parent process spawns a child. In x32dbg attach to the child, not the parent (or the other way around, depending on what you want to inspect).
4. The Level 2 hint *"fork-tastic child process"* will be visible in the debug console once the parent reaches the right point.
5. The parent has two anti-debug checks: the `IsDebuggerPresent` call inside `0x4011D0` (around `0x401828`) and the second one in main (around `0x401828` in the parent itself). Patch the `je` after each `IsDebuggerPresent` call to `jmp` so the parent never sees itself as being debugged.
6. Hit run. The flag is printed to the debug console after `### Good job! Here's your flag:`.

The flag is the same: `picoCTF{0x200_debug_f0r_Win_e6b68f6e}`.

### Alternative Solve — Brute Force the Round Count

If you do not want to count rounds by hand, the algorithm is so fast that you can just brute force the round count from 0 to 30 and pick the first printable result that starts with `picoCTF{`:

```python
import struct

with open("config.bin", "rb") as f:
    data = f.read()

length = struct.unpack("<I", data[:4])[0]
encoded = bytearray(data[4 : 4 + length])
xor_key = bytearray(data[4 + length + 1 : 4 + length + 1 + length])


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


for n in range(30):
    transformed = transform(encoded, n)
    candidate = bytes(a ^ b for a, b in zip(xor_key, transformed))
    try:
        text = candidate.decode("utf-8")
        if text.startswith("picoCTF{"):
            print(n, text)
    except UnicodeDecodeError:
        pass
```

Run it:

```
┌──(zham㉿kali)-[~/picoCTF/WinAntiDbg0x200]
└─$ python3 brute.py
17 picoCTF{0x200_debug_f0r_Win_e6b68f6e}
```

Same flag, found in well under a second. This trick is useful whenever you can guess the format of the plaintext (and `picoCTF{...}` is always printable ASCII).

---

## What Happened Internally — Step-by-Step Timeline

Here is the full story of what the binary does, in order, when run normally on Windows as admin:

1. **Entry (`0x401C34`)** — standard CRT startup, then jump to `main`.
2. **`main` (`0x40166F`)** — saves the stack frame and creates a mutex named `"WinAntiDbg0x200"` via `CreateMutexW(NULL, FALSE, "WinAntiDbg0x200")`.
3. **`GetLastError` branch** — `main` checks `GetLastError`. If the value is `0xB7` (`ERROR_ALREADY_EXISTS`), it means the mutex was already taken, so we are the **second instance** (the child). Otherwise we are the **first instance** (the parent).
4. **Child branch** — the child calls `atoi(argv[1])` to read its parent's PID, then calls `DebugActiveProcess(pid)`. If it succeeds it exits with code `0`, otherwise with `0xBEEF` (48879). It does **no** transforms.
5. **Parent branch, privilege setup** — the parent calls `OpenProcessToken`, `LookupPrivilegeValueW("SeDebugPrivilege", ...)` and `AdjustTokenPrivileges` to enable `SeDebugPrivilege`, which is required for `DebugActiveProcess`. If any step fails, it prints a `[ERROR] ... admin ...` message and exits.
6. **Print banner** — the parent prints the welcome banner and the `"To start the challenge..."` message.
7. **Print setup hint** — the parent prints another `OutputDebugStringW` banner.
8. **`0x401400`** — prints a helper wide string stored at `0x405000` (some flavor text / hint).
9. **`0x401450` — read `config.bin`** — same job as in 0x100: `GetModuleFileNameA` + `strrchr('\\')` + `sprintf("%s\\config.bin", dir)` + `malloc(200)` + `fopen` + `fread(4 bytes length)` + `fread(200 bytes)`. Sets `0x40509C` = encoded string pointer, `0x4050A0` = XOR key pointer, `0x4050A4` = length, `0x405098` = malloc'd buffer.
10. **`0x401090(3)` — three rounds of transform** on the encoded string.
11. **Print Level 2 hint** — `OutputDebugStringW` with the *"fork-tastic"* joke. This is the line you see in the debug console.
12. **`0x4011D0` — child-spawn helper**:
    1. `0x401090(5)` — five more rounds of transform.
    2. Build `STARTUPINFO` and `PROCESS_INFORMATION` for the child.
    3. `CreateProcessA` with the same exe path and `argv[1]` = parent PID. The child will use that to call `DebugActiveProcess`.
    4. `WaitForSingleObject(child_handle, INFINITE)` — block until the child exits.
    5. `0x401090(4)` — four more rounds of transform.
    6. `GetExitCodeProcess(child_handle, &code)` — read the child's exit code.
    7. `CloseHandle` twice.
    8. `0x401090(4)` — four more rounds of transform.
    9. If exit code was `0xBEEF` (48879), set success flag.
13. **Anti-debug #2** — `call IsDebuggerPresent`. If it returns 1 (it shouldn't, because the parent has no debugger, only the child does), print the *"Oops! The debugger was detected"* message and exit.
14. **`0x401090(1)` — one final round** of transform.
15. **`0x401180` — XOR** the encoded string with the XOR key, leaving the flag bytes inside the XOR key buffer at `0x4050A0`.
16. **`0x401000` — UTF-8 to UTF-16** conversion of the flag, via `MultiByteToWideChar(65001, ...)` twice.
17. **Print the flag** — five `OutputDebugStringW` calls in a row: `### Good job! Here's your flag:` + `### ~~~ ` + flag + newline + warning.
18. **Cleanup** — `free` the buffer, return.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip` | Extract the password-protected challenge archive |
| `file` | Confirm the `.exe` is a 32-bit Windows PE binary |
| `strings` / `strings -el` | Find the banner, level hint, mutex name, and wide-string messages |
| `od` | Hex-dump `config.bin` to see its raw structure |
| `pefile` (Python) | Parse PE headers and list the imported APIs (spot `DebugActiveProcess`, `CreateProcessA`, `CreateMutexW`, `AdjustTokenPrivileges`) |
| `capstone` (Python) | Disassemble `.text` and trace the control flow to count how many times the transform runs |
| `python3` | Re-implement the transform and XOR, recover the flag offline |

---

## Key Takeaways

- The **parent / child anti-debug trick** is everywhere in real malware. The child attaches to the parent via `DebugActiveProcess` so the parent's `IsDebuggerPresent` returns 0, fooling casual checks. The `SeDebugPrivilege` + `AdjustTokenPrivileges` dance is what makes this possible without a driver.
- `CreateMutexW` + `GetLastError == ERROR_ALREADY_EXISTS` is the standard "am I first or second instance?" trick. The second instance knows it is the child because the first one already grabbed the mutex.
- **Counting calls matters.** The transform in 0x100 ran 18 times. In 0x200 it runs 17 times. Disassemble, find every call site of the transform function, and add up the arguments. Getting this number wrong is the most common mistake in these challenges.
- The `OutputDebugStringW` "hint" mechanism is the same as in 0x100 — only visible from inside a debugger.
- Brute-forcing the round count is a perfectly valid strategy when you know the plaintext starts with `picoCTF{`. The transform is cheap, the search space is tiny, and you do not even need to read the disassembly carefully.
- **Flag wordplay decode**: `picoCTF{0x200_debug_f0r_Win_e6b68f6e}` reads as *"0x200 debug for Win"*. The `0x200` matches the challenge name suffix (`WinAntiDbg0x200`), the suffix `e6b68f6e` is a random 32-bit hex tag used to make the flag unique per challenge instance, and the rest is the human-readable hint about the parent / child trick.

