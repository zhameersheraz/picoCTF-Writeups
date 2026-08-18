# Tab, Tab, Attack — picoCTF Writeup

**Challenge:** Tab, Tab, Attack  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Using tabcomplete in the Terminal will add years to your life, esp. when dealing with long rambling directory structures and filenames.

**Attachment:** `Addadshashanammu.zip`

---

## Hints

1. After `unzip`ing, this problem can be solved with 11 button-presses... (mostly `Tab`)...

---

## Background Knowledge (Read This First!)

### What is shell tab-completion?

Most modern shells (Bash, Zsh, Fish, even Windows `cmd.exe` and PowerShell) support **tab-completion**: hit the `Tab` key while typing a path or command, and the shell tries to complete what you started typing. If the prefix is unique, the shell fills in the rest. If it is ambiguous, the shell either fills in as far as it can and waits for another `Tab`, or beeps and lists the matches.

For deeply nested directories or long filenames, tab-completion is the difference between typing 80 characters per path and typing 2. It is also the difference between *getting* a path right and mistyping a vowel somewhere.

### The "11 button presses" hint

The hint tells you the entire challenge is solvable in 11 button presses, mostly `Tab`. Let us count:
- `Enter` x 2 (after `cd` and after `./fang-of-haynekhtnamet`)
- `Tab` x ~9 (one per directory level: 8 directories, then the file)
- `cd ` (3 keys)
- `./` (2 keys before the file)

That comes out to about 11 key presses, give or take. The point is: you should not be typing the directory names by hand. The challenge is teaching you a *muscle memory* skill.

### Why deep nesting is annoying without tab-completion

Inside `Addadshashanammu`, the flag binary lives at:

```
Addadshashanammu/Almurbalarammi/Ashalmimilkala/Assurnabitashpi/Maelkashishi/Onnissiralis/Ularradallaku/fang-of-haynekhtnamet
```

That is **8 levels** of unique-named directories. The names are 11-15 characters each, all capitalised, all looking similar enough that one mistype will silently put you in a non-existent path. Tab-completion turns this nightmare into a 1-second exercise.

---

## Solution

### Step 1: Unzip the attachment

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip Addadshashanammu.zip
Archive:  Addadshashanammu.zip
   creating: Addadshashanammu/Almurbalarammi/Ashalmimilkala/Assurnabitashpi/Maelkashishi/Onnissiralis/Ularradallaku/
  inflating: Addadshashanammu/Almurbalarammi/Ashalmimilkala/Assurnabitashpi/Maelkashishi/Onnissiralis/Ularradallaku/fang-of-haynekhtnamet
  inflating: Addadshashanammu/Almurbalarammi/Ashalmimilkala/Assurnabitashpi/Maelkashishi/Onnissiralis/Ularradallaku/fang-of-haynekhtnamet.c
```

Two files are dropped in the deepest directory: a compiled binary (`fang-of-haynekhtnamet`) and its C source (`.c`).

### Step 2: Use Tab-completion to descend

I typed `cd Addadshashanammu/` and then `Tab` through each subdirectory. The shell fills in the rest:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cd A<Tab>     # -> Addadshashanammu/
└─$ cd A<Tab>     # -> Addadshashanammu/Almurbalarammi/
└─$ cd A<Tab>     # -> Addadshashanammu/Almurbalarammi/Ashalmimilkala/
└─$ cd A<Tab>     # -> Addadshashanammu/.../Assurnabitashpi/
└─$ cd M<Tab>     # -> Addadshashanammu/.../Maelkashishi/
└─$ cd O<Tab>     # -> Addadshashanammu/.../Onnissiralis/
└─$ cd U<Tab>     # -> Addadshashanammu/.../Ularradallaku/
└─$ cd f<Tab>     # -> Addadshashanammu/.../fang-of-haynekhtnamet
└─$ cd ..
└─$ ./f<Tab>      # -> ./fang-of-haynekhtnamet
└─$ ./fang-of-haynekhtnamet
*ZAP!* picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}
```

The first letters of each path component are unique enough that a single `Tab` per level finishes the typing. I went back up one with `cd ..` so I could run the binary with the relative path.

### Step 3: Run the binary (or just read the source)

Two options, both give the flag:

```
┌──(zham㉿kali)-[/media/sf_downloads/.../Ularradallaku]
└─$ ./fang-of-haynekhtnamet
*ZAP!* picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}
```

Or, skip the run and just `cat` the source:

```
┌──(zham㉿kali)-[/media/sf_downloads/.../Ularradallaku]
└─$ cat f<Tab>      # -> cat fang-of-haynekhtnamet.c
#include <stdio.h>


int main(){
    printf("*ZAP!* picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}\n");
}
```

The `*ZAP!*` is just flavor text the author added to make the printout feel more dramatic.

**Flag:** `picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}`

---

## What Happened Internally (Timeline)

| Step | What I did | What happened |
|---|---|---|
| 1 | `unzip Addadshashanammu.zip` | Created the directory tree and dropped two files at the bottom. |
| 2 | Pressed `Tab` 8 times to descend | The shell autocompleted each unique-first-letter directory name. |
| 3 | `cd ..` then `./f<Tab>` | Got the relative path to the binary, autocompleted. |
| 4 | `<Enter>` to run the binary | It printed `*ZAP!*` and the flag, then exited. |

Total key presses for the descent: 8 (one `Tab` per directory) + `cd ..` (5) + `./f<Tab>` (4) + `<Enter>` x2 (2) = roughly 19 keys. The hint of 11 is a *floor* — the writer was probably counting only the `Tab` + `Enter` keys, not the typeable bits in between. Either way, it is way fewer than typing the path out.

---

## Alternative Solve Methods

### Method 1: Read the source file directly

You do not need to run the binary. The `.c` file has the flag in plain ASCII. Skip the descent entirely:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep -r picoCTF Addadshashanammu/
Addadshashanammu/Almurbalarammi/Ashalmimilkala/Assurnabitashpi/Maelkashishi/Onnissiralis/Ularradallaku/fang-of-haynekhtnamet.c:    printf("*ZAP!* picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}\n");
```

`grep -r` walks the entire directory tree, so it finds the flag in the source file without you ever having to type the path. This is the "I know about `grep`" path.

### Method 2: `find` + `xargs` `grep`

Same idea, more general:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ find Addadshashanammu -type f -exec grep -l picoCTF {} +
Addadshashanammu/.../fang-of-haynekhtnamet
Addadshashanammu/.../fang-of-haynekhtnamet.c
```

The `find ... -exec` form is more portable than `grep -r` (which is GNU-only).

### Method 3: `strings` the binary from anywhere

If you know the file is a binary that will contain ASCII strings, `strings` is your friend:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ find Addadshashanammu -type f -executable -exec strings {} \; | grep picoCTF
*ZAP!* picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}
```

Walks the tree, finds the executable, dumps the printable strings, filters to the flag.

### Method 4: Recursive `grep` from PowerShell

If you are on Windows:

```powershell
Get-ChildItem -Path C:\Users\ASUS\Downloads\Addadshashanammu -Recurse -File |
    Select-String -Pattern 'picoCTF' |
    Select-Object -ExpandProperty Line
```

`Select-String` is the PowerShell equivalent of `grep` and it walks the directory tree with `Get-ChildItem -Recurse`.

### Method 5: The intended (Tab) path

The intended path is to actually use Tab-completion to descend. That is the muscle-memory lesson. Pick this if you are doing the challenge to *learn* — once you have the flag, the other methods are also fair game.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `unzip` | Extract the attachment |
| Shell `Tab` completion | Descend into the nested directory tree without typing |
| `./fang-of-haynekhtnamet` | Run the binary to print the flag |
| `cat` (alt) | Read the source file directly |
| `grep -r` (alt) | Skip the descent entirely; find any file containing `picoCTF` |
| `find` + `strings` (alt) | Find and dump the binary's printable strings |
| `Get-ChildItem -Recurse` + `Select-String` (alt) | Windows-native recursive grep |

---

## Key Takeaways

- **Tab-completion is non-negotiable.** If you find yourself typing a long path in a CTF, stop and `Tab` instead. The hint is literally "mostly Tab" — the author wants you to build the habit.
- **Recursive search beats descent.** For "find the flag in this giant tree" challenges, `grep -r picoCTF .` is your first move, not `cd` and `ls`. Save the descent for when you actually need to run something.
- **Long file names with capital letters are a typing trap.** Names like `Ularradallaku` and `Onnissiralis` look similar at a glance. One wrong letter and you are in `/dev/null`. Tab-completion dodges the trap.
- **`find -type f` walks the tree, `find -type f -executable` walks the executables, `find -type f -name '*.c'` walks the source.** Combine with `-exec` or pipe into `xargs` for arbitrary processing.
- **Wordplay, as always:** the challenge name "Tab, Tab, Attack" is onomatopoeia for the sound of mashing the Tab key. The flag decodes from leetspeak as **level up, take a rest** — a video-game reference to advancing past a checkpoint and saving. The author staged this as a "skill check" (`Addadshashanammu` is fake Sumerian-style gibberish, with names like "Almurbalarammi" and "Ularradallaku" that read like lost incantations) and the flag itself is what you say when you have beaten the skill check and unlocked the next dungeon. The lesson: even the most annoying directory trees bow to `Tab`.
