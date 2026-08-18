# Wave a flag — picoCTF Writeup

**Challenge:** Wave a flag  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{b1scu1ts_4nd_gr4vy_ac5832c}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Can you invoke help flags for a tool or binary? This program has extraordinarily helpful information...

**Attachment:** `warm` (ELF 64-bit shared library, 19 KB)

---

## Hints

1. This program will only work in the webshell or another Linux computer.
2. To get the file accessible in your shell, enter the following in the Terminal prompt: `$ wget <URL here>`, where the url can be found in the details section.
3. Run this program by entering the following in the Terminal prompt: `$ ./warm`, but you'll first have to make it executable with `$ chmod +x warm`
4. `-h` and `--help` are the most common arguments to give to programs to get more information from them!
5. Not every program implements help features like `-h` and `--help`.

---

## Background Knowledge (Read This First!)

### What is `strings`?

`strings` is a standard Unix tool that scans a file (binary or text) and prints every printable **run of characters** it finds. By default it looks for sequences of 4+ printable bytes, terminated by a null or newline. It is one of the first things you should run on any binary you don't recognize — most compiled programs leak useful info as embedded ASCII: error messages, file paths, function names, URLs, and yes, the occasional flag.

### Why does the flag end up in the binary at all?

A flag inside a binary is a **string literal**. The compiler stores it in the `.rodata` section of the ELF file, and `strings` reads that section straight out. When the program runs and decides to print the flag (e.g. when you pass `-h`), it just calls `puts()` on the same literal. So even if you never run the program, the flag is sitting there in the file the whole time.

### ELF file basics

The first 4 bytes of any ELF file are the magic number `7F 45 4C 46` (i.e. `\x7fELF`). `exiftool`, `file`, and `xxd` will all happily report this. Knowing the file type up front tells you what you're dealing with — a binary, an image, a script — before you start poking at it.

---

## Solution

### Step 1: Pull the file and check what it is

I downloaded `warm` from the challenge page and dropped it into `/media/sf_downloads` so my Kali VM could see it (the shared folder between Kali and the Windows host).

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool warm
ExifTool Version Number         : 13.50
File Name                       : warm
Directory                       : .
File Size                       : 19 kB
File Modification Date/Time     : 2026:08:18 09:07:15-04:00
File Type                       : ELF shared library
File Type Extension             : so
MIME Type                       : application/octet-stream
CPU Architecture                : 64 bit
CPU Byte Order                  : Little endian
Object File Type                : Shared object file
```

`exiftool` told me this is a 64-bit ELF shared object. So it is a Linux binary, not a script.

### Step 2: Try running it (it will fail — and that is fine)

The challenge hints say to make it executable and run it. I tried:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ chmod +x warm

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ warm
Command 'warm' not found, did you mean:
  command 'swarm' from deb swarm
  command 'warp' from deb libghc-wai-app-static-dev
  command 'warg' from deb clonalorigin
```

It is not in `$PATH`, so the shell cannot find it. Two options here:
- run it as `./warm` so the shell looks in the current directory, or
- skip running it entirely and just look at the binary.

I went with the second option because I did not actually need to execute the program to find the flag.

### Step 3: `cat` the file to see what is inside

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat warm
@$#@@@\x80\x82... (mostly gibberish, with one readable line near the end)
Oh, help? I actually don't do much, but I do have this flag here: picoCTF{b1scu1ts_4nd_gr4vy_ac5832c}
I don't know what '%s' means! I do know what -h means though!
```

The terminal renders only the printable parts of the binary, so most of it is unreadable. But right in the middle I can already see the line `Oh, help? ... picoCTF{b1scu1ts_4nd_gr4vy_ac5832c}`. That is the flag.

But `cat` is messy and I want a cleaner way. So I used `grep`:

### Step 4: Use `grep` to confirm the flag is in the binary

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep picoCTF warm
grep: warm: binary file matches
```

`grep` detects that `warm` is a binary file and refuses to print the matching line — it just tells me a match exists. The fix is to either use `grep -a` (treat binary as text) or use `strings` first.

### Step 5: Use `strings` to extract the flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings warm | grep picoCTF
Oh, help? I actually don't do much, but I do have this flag here: picoCTF{b1scu1ts_4nd_gr4vy_ac5832c}
```

`strings` pulls out all the printable ASCII runs of length 4 or more, and `grep picoCTF` filters to just the line that contains the flag.

**Flag:** `picoCTF{b1scu1ts_4nd_gr4vy_ac5832c}`

---

## What Happened Internally (Timeline)

| Step | What I did | What the binary did |
|---|---|---|
| 1 | `exiftool warm` | Confirmed the file is a 64-bit ELF shared library. |
| 2 | `chmod +x warm` | Made the file executable. |
| 3 | `warm` (no `./`) | Bash looked in `$PATH`, did not find it, errored. |
| 4 | `cat warm` | Terminal printed the printable parts, including the flag line. |
| 5 | `grep picoCTF warm` | grep refused to print the match because warm is binary. |
| 6 | `strings warm \| grep picoCTF` | strings extracted all printable runs, grep filtered to the flag line. |

---

## Alternative Solve Methods

### Method 1: Run the binary with `-h` as the hint suggests

If I had simply run `./warm -h`, the program would have printed the same flag string. The hints push you toward this path, and it is a valid solution — but it is a bit slower than the strings approach because the file is a 19 KB binary, and the `strings` route works without ever executing the binary.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ./warm -h
Oh, help? I actually don't do much, but I do have this flag here: picoCTF{b1scu1ts_4nd_gr4vy_ac5832c}
```

### Method 2: `grep -a` treats the binary as text

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep -a picoCTF warm
Oh, help? I actually don't do much, but I do have this flag here: picoCTF{b1scu1ts_4nd_gr4vy_ac5832c}
```

`-a` (or `--text`) tells grep to process the file as if it were text, skipping the binary detection. Same result, fewer pipes.

### Method 3: Hex-dump the .rodata section directly

If you want to be fancy, the flag is in the `.rodata` section of the ELF. You can read just that section with `objdump`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ objdump -s -j .rodata warm
```

You will see the flag bytes laid out in a hex table. This is overkill for a General Skills challenge but useful when the flag is encoded or split across multiple literals.

### Method 4: `xxd` or `hexdump` to eyeball the raw bytes

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ xxd warm | grep -i pico
```

This works if the flag appears in the binary as raw ASCII bytes (which it does here). Less clean than `strings`, but it gets the job done.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `exiftool` | Identify the file type (ELF shared library) |
| `chmod` | Make the binary executable |
| `cat` | Quick eyeball of the file contents |
| `grep` | Search for the flag in the file |
| `strings` | Extract printable ASCII runs from the binary |
| `objdump` (alt) | Dump the .rodata section directly |
| Kali Linux shell | Run all of the above |

---

## Key Takeaways

- **`strings` is your first move on any unknown binary.** It finds human-readable sequences without executing anything. Add it to muscle memory next to `file`, `xxd`, and `objdump`.
- **`grep` refuses to print matches in binary files by default.** Use `grep -a` or pipe through `strings` first. This trips up beginners constantly.
- **Executables leak information.** Anything baked into the source as a string literal — error messages, debug prints, embedded flags — is in the binary forever. Stripping symbols and removing debug info helps, but `strings` will still find user-facing text.
- **Hint 5 is the real lesson:** not every program implements `-h`. `strings` works on all of them.
- **Wordplay, as always:** the challenge is literally called "Wave a flag" and the binary is called `warm`. The flag itself decodes from leetspeak as **biscuits and gravy** — a classic Southern US comfort breakfast, and the kind of thing you eat while you wave your state's flag at a parade. The author slipped a pun in three times: the challenge name, the binary name, and the flag content. Spotting all three is the "wave" you are supposed to catch.
