# Tab, Tab, Attack — picoCTF Writeup

**Challenge:** Tab, Tab, Attack  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}`  

---

## Description

> Using tabcomplete in the Terminal will add years to your life, esp. when dealing with long rambling directory structures and filenames.
> Download: `Addadshashanammu.zip`

**Hint 1:** `After unzipping, this problem can be solved with 11 button-presses...(mostly Tab)...`

---

## Background Knowledge (Read This First!)

### What is Tab Completion?

**Tab completion** is one of the most useful features in any Linux terminal. When you start typing a filename or path and press **Tab**, the terminal automatically fills in the rest — as long as it's unambiguous. If there are multiple matches, pressing Tab twice shows all options.

For example:
```bash
cd Add[TAB]        → cd Addadshashanammu/
cd Addadshashanammu/Al[TAB]  → cd Addadshashanammu/Almurbalarammi/
```

Without tab completion, typing out folder names like `Ashalmimilkala` and `Assurnabitashpi` by hand would be a nightmare. With Tab, each folder needs just 2-3 characters before autocompleting.

### What is a compiled binary?

The deepest folder contains two files:
- `fang-of-haynekhtnamet.c` — the **source code** (human-readable C code)
- `fang-of-haynekhtnamet` — the **compiled binary** (executable program)

Running the binary with `./` executes it and prints the flag.

### The Folder Structure

```
Addadshashanammu/
└── Almurbalarammi/
    └── Ashalmimilkala/
        └── Assurnabitashpi/
            └── Maelkashishi/
                └── Onnissiralis/
                    └── Ularradallaku/
                        ├── fang-of-haynekhtnamet.c
                        └── fang-of-haynekhtnamet  ← run this
```

7 levels deep — each folder name is a long, complicated word. Tab completion makes this trivial.

---

## Solution — Step by Step

### Step 1 — Extract the zip

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip Addadshashanammu.zip
Archive:  Addadshashanammu.zip
   creating: Addadshashanammu/
   creating: Addadshashanammu/Almurbalarammi/
   creating: Addadshashanammu/Almurbalarammi/Ashalmimilkala/
   ...
 extracting: .../Ularradallaku/fang-of-haynekhtnamet.c
  inflating: .../Ularradallaku/fang-of-haynekhtnamet
```

### Step 2 — Navigate using Tab completion

Start typing the folder name and press **Tab** to autocomplete at each level:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cd Add[TAB]/Al[TAB]/As[TAB]/As[TAB]/M[TAB]/O[TAB]/U[TAB]
```

Each `[TAB]` autocompletes the folder name. After all 7 tabs you'll be inside:
```
Addadshashanammu/Almurbalarammi/Ashalmimilkala/Assurnabitashpi/Maelkashishi/Onnissiralis/Ularradallaku/
```

### Step 3 — Run the binary

```
┌──(zham㉿kali)-[/.../Ularradallaku]
└─$ ./fang-of-haynekhtnamet
*ZAP!* picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}
```

✅ Got the flag! 🎯

---

## Alternative Method — Use `find` to skip all the navigation

Instead of `cd`-ing through 7 folders, use `find` to locate and run the binary directly:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ find Addadshashanammu/ -type f -name "fang-of-haynekhtnamet" ! -name "*.c"
Addadshashanammu/Almurbalarammi/Ashalmimilkala/Assurnabitashpi/Maelkashishi/Onnissiralis/Ularradallaku/fang-of-haynekhtnamet

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ./Addadshashanammu/Almurbalarammi/Ashalmimilkala/Assurnabitashpi/Maelkashishi/Onnissiralis/Ularradallaku/fang-of-haynekhtnamet
*ZAP!* picoCTF{l3v3l_up!_t4k3_4_r35t!_fc588427}
```

Or combine both steps into one:

```bash
$(find Addadshashanammu/ -type f ! -name "*.c")
```

This finds the binary and runs it in one shot.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `unzip` | Extract the zip archive | ⭐ Easy |
| Tab completion | Navigate deep folder structures instantly | ⭐ Easy |
| `./binary` | Execute the compiled binary | ⭐ Easy |
| `find` (optional) | Locate and run the binary without navigating | ⭐ Easy |

---

## Key Takeaways

- **Tab completion is essential** — for long filenames and deep paths, pressing Tab saves massive amounts of typing and prevents typos
- **`./filename` runs a binary** in the current directory — the `./` tells the shell to look in the current folder rather than system PATH
- **`find -type f ! -name "*.c"`** finds executable files while excluding source code — useful when a folder has both `.c` and compiled versions
- The challenge name is the lesson: **Tab, Tab, Attack** — use Tab to blast through the folder maze
- The flag `l3v3l_up!_t4k3_4_r35t!` → "level up! take a rest!" — you earned it after all those folder names!
