# Fresh Java — picoCTF Writeup

**Challenge:** Fresh Java  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{700l1ng_r3qu1r3d_738cac89}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Can you get the flag?
>
> Reverse engineer this `Java program`.

## Hints

> 1. Use a decompiler for Java!

---

## Background Knowledge

The whole challenge rides on one idea: a `.java` source file and a `.class` bytecode file describe the same program. Java compiles `.java` → `.class`, and a Java decompiler runs the process in reverse — `.class` → readable `.java`. Knowing that, here is the rest of the vocabulary.

**1. `.java` vs `.class`.**
A `.java` file is human-readable source. After `javac KeygenMe.java` you get `KeygenMe.class`, which contains JVM bytecode (instructions like `aload`, `invokevirtual`, `bipush`). Bytecode is not source, but it is also not binary like an ELF — it is structured, named, and rich in metadata. A decompiler (`cfr`, `procyon`, `jadx`, `vineflower`) reads that structure and produces source-level Java.

**2. Why `javap` is the first stop.**
`javap` ships with the JDK and prints the bytecode disassembly of a class file with no extra tooling. It is enough to read the literal characters and offsets that the program checks. When the source is "just a list of `if (s.charAt(i) != 'X')` checks" — as it is here — `javap` is usually all you need.

**3. `bipush` vs `iconst_N`.**
The JVM encodes small integer constants with single-byte instructions:
- `iconst_0` ... `iconst_5` push `0` ... `5` directly.
- `bipush N` pushes a one-byte signed integer (`-128` ... `127`).

You will see both in this class because the indexes `0-5` use `iconst_*` (the JVM is allowed to use a smaller instruction) and the indexes `7-33` use `bipush`. The expected characters (which are mostly `'p'` (112), `'i'` (105), `'7'` (55) etc.) all fit in a signed byte, so they are always `bipush`. That is why the first few `charAt` calls look different from the rest.

**4. The "keygen" pattern.**
The class is named `KeygenMe`. Its `main` reads a line of input, checks `length == 34`, then checks every character at a specific index against a literal `'X'`. If every check passes it prints "Valid key". The flag *is* the key — there is no second layer. So a decompiler turns the challenge from "guess 34 characters" into "read 34 character literals."

**5. CFR is the decompiler that "just works."**
CFR (Class File Reader) is a standalone `.jar` — no install, no project, just `java -jar cfr.jar Some.class`. It handles modern Java bytecode (Java 8, 11, 17, ...) and produces clean output that compiles back to the same `.class`. It is the decompiler the hint is pointing at.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The downloaded file is `FreshJava.class`.

### 1. Move the file into a working directory and check what it is

```
┌──(zham㉿kali)-[~/fresh-java]
└─$ mkdir -p ~/fresh-java && cd ~/fresh-java

┌──(zham㉿kali)-[~/fresh-java]
└─$ cp ~/Downloads/FreshJava.class .

┌──(zham㉿kali)-[~/fresh-java]
└─$ file FreshJava.class
FreshJava.class: compiled Java class data, version 55.0 (Java SE 11)
```

`version 55.0` is Java 11. Two immediate takeaways: the original source was compiled with `javac --release 11` (or older), and CFR will be able to read it (CFR supports everything from very old to current Java).

### 2. Peek at the bytecode with `javap` (optional but useful)

`javap` ships with the JDK and is the lightest way to confirm what the class does *without* installing a decompiler yet:

```
┌──(zham㉿kali)-[~/fresh-java]
└─$ javap -p -c FreshJava.class | head -30
```

```
Compiled from "KeygenMe.java"
public class KeygenMe {
  public KeygenMe();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class java/util/Scanner
       3: dup
       4: getstatic     #3                  // Field java/lang/System.in:Ljava/io/InputStream;
       ...
      28: bipush        34
      30: if_icmpeq     42
      ...
      43: bipush        33
      45: invokevirtual #11                 // Method java/lang/String.charAt:(I)C
      48: bipush        125
      50: if_icmpeq     62
```

The bytecode already says a lot:

- The class is internally named `KeygenMe` (the filename `FreshJava.class` is just the picoCTF download wrapper).
- `main` reads a string with `Scanner`, then runs a chain of `string.charAt(N)` comparisons.
- The very first one (`bipush 34`) is the length check: input must be 34 characters.
- The next ones compare `charAt(33)` to `bipush 125` = `'}'` — the closing brace.

### 3. Install CFR (the decompiler the hint is pointing at)

```
┌──(zham㉿kali)-[~/fresh-java]
└─$ wget -q https://github.com/leibnitz27/cfr/releases/download/0.152/cfr-0.152.jar

┌──(zham㉿kali)-[~/fresh-java]
└─$ ls -la cfr-0.152.jar
-rw-r--r-- 1 zham zham 2162315 Dec 11  2021 cfr-0.152.jar
```

### 4. Decompile `FreshJava.class`

```
┌──(zham㉿kali)-[~/fresh-java]
└─$ java -jar cfr-0.152.jar FreshJava.class > KeygenMe.java 2>&1

┌──(zham㉿kali)-[~/fresh-java]
└─$ cat KeygenMe.java
```

```java
/*
 * Decompiled with CFR 0.152.
 */
import java.util.Scanner;

public class KeygenMe {
    public static void main(String[] stringArray) {
        Scanner scanner = new Scanner(System.in);
        System.out.println("Enter key:");
        String string = scanner.nextLine();
        if (string.length() != 34) {
            System.out.println("Invalid key");
            return;
        }
        if (string.charAt(33) != '}') { ... }
        if (string.charAt(32) != '9') { ... }
        if (string.charAt(31) != '8') { ... }
        ...
        if (string.charAt(0) != 'p') { ... }
        System.out.println("Valid key");
    }
}
```

The decompiled source is a literal list of "is character at index N equal to literal X?" checks. The flag is just the concatenation of those literals in order.

### 5. Extract the flag from the decompiled source

A 34-character flag built from 34 character literals is trivial to harvest. I used a one-liner that pulls every `charAt(N) != 'X'` literal and sorts by `N`:

```
┌──(zham㉿kali)-[~/fresh-java]
└─$ grep -oE "charAt\([0-9]+\) != '.'" KeygenMe.java \
      | python3 -c "
import sys, re
flag = {}
for line in sys.stdin:
    m = re.match(r\"charAt\((\d+)\) != '(.+)'\", line.strip())
    if m:
        flag[int(m.group(1))] = m.group(2)
print(''.join(flag[i] for i in range(max(flag)+1)))
"
picoCTF{700l1ng_r3qu1r3d_738cac89}
```

The flag is `picoCTF{700l1ng_r3qu1r3d_738cac89}`.

### 6. Confirm by running the original class with the recovered flag

The class file is named `FreshJava.class` but it contains a public class called `KeygenMe`. Java requires the file name to match the public class, so I copy it to the right name before running:

```
┌──(zham㉿kali)-[~/fresh-java]
└─$ cp FreshJava.class KeygenMe.class

┌──(zham㉿kali)-[~/fresh-java]
└─$ echo "picoCTF{700l1ng_r3qu1r3d_738cac89}" | java -cp . KeygenMe
Enter key:
Valid key
```

`Valid key` — the program accepts the flag we recovered.

---

## Alternative Solves

**A. Skip the decompiler — read the bytecode directly with `javap`.**
The whole program is 34 sequential `charAt` comparisons and every expected character is a `bipush` or `iconst_*` literal in the bytecode. A small script that pulls every `bipush V` immediately following `charAt` is enough — no decompiler needed:

```
┌──(zham㉿kali)-[~/fresh-java]
└─$ javap -p -c FreshJava.class > bytecode.txt

┌──(zham㉿kali)-[~/fresh-java]
└─$ python3 << 'PY'
import re
text = open('bytecode.txt').read()
ICON = {f'iconst_{i}': i for i in range(6)}
flag = {}
for m in re.finditer(r'bipush\s+(\d+)\s*\n\s+\d+: invokevirtual.*charAt.*\(I\)C\s*\n\s+\d+: (bipush|iconst_\d)\s+(\d+)', text):
    idx = int(m.group(1))
    op, val = m.group(2), int(m.group(3))
    if op.startswith('iconst'):
        val = ICON[op]
    flag[idx] = chr(val)
print(''.join(flag[i] for i in range(max(flag)+1)))
PY
picoCTF{700l1ng_r3qu1r3d_738cac89}
```

Useful when you do not want to download CFR (e.g. on an offline CTF box).

**B. Use a different decompiler.**
If you have `procyon`, `jadx`, or `vineflower` available, any of them produces essentially the same source. Example with `jadx`:

```
┌──(zham㉿kali)-[~/fresh-java]
└─$ jadx -d decompiled FreshJava.class
```

`decompiled/sources/KeygenMe.java` will contain the same chain of `charAt` checks.

**C. Brute-force each character independently.**
Each character is checked in isolation — `if (s.charAt(i) != X) { invalid; return; }` — so the program tells you "Invalid key" at the *first* mismatch. A 34-step brute forcer recovers the flag one character at a time in well under a second:

```
┌──(zham㉿kali)-[~/fresh-java]
└─$ python3 << 'PY'
import subprocess, string
flag = ['?'] * 34
for i in range(34):
    for c in string.printable.strip():
        attempt = ''.join(flag[:i]) + c + '?' * (33 - i)
        out = subprocess.run(
            ['java', '-cp', '.', 'KeygenMe'],
            input=attempt, capture_output=True, text=True).stdout
        if 'Valid key' in out or 'Invalid key' not in out:
            flag[i] = c
            print(f'pos {i}: {c}  -> so far: {"".join(flag)}')
            break
print('FLAG:', ''.join(flag))
PY
```

Same answer, but you never read any source. This works because the program is a textbook character-by-character oracle.

---

## What Happened Internally

1. The challenge downloads a single file `FreshJava.class` — JVM bytecode for a public class whose real name is `KeygenMe` (the `Compiled from "KeygenMe.java"` line in the constant pool tells us the original source filename).
2. `file` reports `version 55.0 (Java SE 11)` — the bytecode is Java 11 era. CFR 0.152, released 2021, supports Java 17 features and reads Java 11 trivially.
3. `javap -p -c FreshJava.class` walks the constant pool, lists the two methods (`<init>` and `main`), and disassembles every bytecode instruction with its operand. The output is structured but verbose: each `charAt` check is roughly five lines.
4. `java -jar cfr-0.152.jar FreshJava.class` runs CFR's bytecode → AST → Java printer pipeline. CFR parses the constant pool, rebuilds the local-variable table and the stack, recognises the `Scanner`/`String.charAt`/literal-compare idiom, and emits a clean `KeygenMe.java` source that compiles back to a near-identical class.
5. The decompiled source is a chain of `if (s.charAt(N) != 'X') { System.out.println("Invalid key"); return; }` blocks. Each `N` runs from `33` down to `0`, and each `'X'` is the corresponding character of the flag.
6. A simple Python loop harvests `(N, 'X')` pairs from the decompiled source (or directly from the `javap` output), sorts by `N`, and joins them into the 34-character flag.
7. Copying `FreshJava.class` to `KeygenMe.class` and running `java KeygenMe` with the recovered flag as stdin makes the program print `Valid key`, confirming the answer.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `file`           | Confirmed the artifact is `compiled Java class data` (v55).    |
| `default-jdk-headless` | Installed `java` and `javap`.                          |
| `javap -p -c`    | Quick bytecode peek before installing a decompiler.            |
| `wget`           | Downloaded the standalone CFR jar.                             |
| `cfr-0.152.jar`  | The Java decompiler the hint points at.                        |
| `grep` + `python3` | Harvested the 34 character literals from the decompiled source. |
| `python3`        | Reconstructed the flag from `(index, char)` pairs.             |
| `java` (run)     | Confirmed the flag by feeding it into `KeygenMe`.              |

---

## Key Takeaways

- `.class` files are not "binary" in the scary sense. They are structured, symbol-rich, and reversible with a decompiler. The first reflex on any Java challenge should be `javap` for a quick read, then CFR / procyon / jadx for the real source.
- A "keygen" class that reads a line and checks every character separately is the *worst* kind of protection — the entire secret lives in the bytecode as 34 readable character literals. There is no math to invert, no key to derive; just read the source.
- The class name in the file (`FreshJava.class`) is *not* the class name inside (`KeygenMe`). The constant pool tells you the truth (`Compiled from "KeygenMe.java"`). If you ever try to run a `.class` and get `ClassNotFoundException`, the filename is the first thing to check.
- Decompiler output is good enough to recompile — if the decompiled source fails to build, you are either missing a dependency or your tool is broken. CFR's output is one of the cleanest for this reason.
- The flag's leet: `700l1ng_r3qu1r3d` reads as **tooling required** (`7→t`, `0→o`, `1→i`, `3→e`). The hint literally says "use a decompiler" — and the flag's wordplay confirms it: this challenge is solvable only if you have the *right tooling* on hand.
