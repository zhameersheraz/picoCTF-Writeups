# Safe Opener 2 — picoCTF Writeup

**Challenge:** Safe Opener 2  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{SAf3_0p3n3rr_y0u_solv3d_it_198203f7}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> What can you do with this file?
>
> I forgot the key to my safe but this file is supposed to help me with retrieving the lost key. Can you help me unlock my safe?

## Hints

> 1. Download and try to decompile the file.

---

## Background Knowledge (Read This First!)

### What is a `.class` file?

When you write Java (or Kotlin, or Scala, or any other JVM language) and run `javac MyFile.java`, the compiler does not produce machine code. It produces **bytecode** — a special instruction set that the Java Virtual Machine (JVM) understands. That bytecode is dumped into one or more `.class` files, one per public class.

A `.class` file is a binary, but it is a *very* structured binary. Unlike a stripped C/C++ ELF where function names and string literals are usually mangled or thrown away, the JVM `.class` format is designed to retain rich metadata:

- The class name and the names of its superclass and interfaces
- The names, descriptors, and access flags of every field and method
- The full **constant pool** — a table at the top of the file that holds every string, number, class reference, method reference, etc. the class uses

Because string literals have to end up in the constant pool in plain UTF-8, every hard-coded string in the original source (URLs, error messages, passwords, flags) is sitting right there in the binary, just waiting to be read. This is the entire reason this challenge is solvable in 10 seconds.

### What is a "constant pool"?

Think of it as the class file's own little dictionary. Every time the Java source uses a literal — `"Sesame open"`, `42`, a class name, a method signature — that literal is assigned an entry in the constant pool, indexed by a small integer. The bytecode instructions later just say "load constant #24" or "call method referenced by #16", and the JVM looks the entry up.

This is wonderful for reverse engineering. Tools like `javap` (the JDK's built-in disassembler), `cfr`, `procyon`, `jd-gui`, and `jadx` can all rebuild a near-perfect copy of the original source from a `.class` file, complete with the original variable names.

### What is `strings` and why does it work here?

The classic Unix `strings` command walks through any binary file and prints every sequence of printable ASCII (or UTF-8) characters it can find. It is a dumb tool — it knows nothing about Java, the JVM, or the constant pool. But because the constant pool stores strings as plain UTF-8, the flag embedded in the source code is just plain bytes in the file. `strings` reads those bytes and prints them.

This is the same trick we used in the `timer` challenge on the APK's `classes3.dex` file. Java/Kotlin/Dalvik bytecode all leak their string literals the same way.

### What is Base64 (and why is it relevant)?

Base64 is an encoding (not encryption!) that turns arbitrary bytes into a text-safe ASCII string using a 64-character alphabet (`A-Z a-z 0-9 + /`). The Java app in this challenge Base64-encodes whatever password you type and then compares the encoded version against a hard-coded flag string. So even though the comparison in `openSafe` looks like "is the typed password equal to the flag?", the typed password gets encoded first — meaning the flag itself is what is being compared, in the clear, in the constant pool.

### Why is this insecure?

Because the `.class` file has to ship to the user (or run on their device), any secret the developer puts in the source ends up in the constant pool. There is no way to hide it. The same is true for Android's `.dex` files. This is why you should never hard-code API keys, passwords, or flags into a Java/Android app — anyone with `strings` (or a decompiler) can read them.

---

## Solution — Step by Step

### Step 1 — Confirm what we downloaded

```
┌──(zham㉿kali)-[~/ctf]
└─$ file SafeOpener.class
SafeOpener.class: compiled Java class data, version 52.0 (Java 1.8)
```

`file` tells us it is a compiled Java `.class` file targeting Java 1.8 (major version 52.0). That means we can use any modern JVM toolchain to look at it.

### Step 2 — Grep for the flag with `strings`

Because the flag is a string literal in the original Java source, it ends up as plain UTF-8 in the constant pool of the `.class` file. `strings` finds it in one shot:

```
┌──(zham㉿kali)-[~/ctf]
└─$ strings SafeOpener.class | grep picoCTF
,picoCTF{SAf3_0p3n3rr_y0u_solv3d_it_198203f7}
```

On screen: `picoCTF{SAf3_0p3n3rr_y0u_solv3d_it_198203f7}` (ignore the leading comma — `strings` grabbed an adjacent character from the constant pool, it is not part of the flag).

That is the whole challenge. Paste it into the picoCTF submission box.

---

## Alternative — Decompile with CFR (the "real" reverse-engineering way)

The hint says "decompile the file". `strings` is the lazy-but-fast move. To actually learn something about how the app works, we decompile the `.class` back into Java source. CFR is one of the best modern Java decompilers and lives in a single jar file.

### Step 1 — Grab CFR

```
┌──(zham㉿kali)-[~/ctf]
└─$ wget -q https://github.com/leibnitz27/cfr/releases/download/0.152/cfr-0.152.jar
```

If the GitHub release URL ever changes, search "cfr decompiler jar download" — the file lives at `leibnitz27/cfr` on GitHub.

### Step 2 — Make sure you have a JDK installed

You need a JDK (not just a JRE) because we will use `javap` too. On a fresh Kali box:

```
┌──(zham㉿kali)-[~/ctf]
└─$ sudo apt-get update && sudo apt-get install -y default-jdk-headless
```

If Java is already installed, that command just no-ops. Check with:

```
┌──(zham㉿kali)-[~/ctf]
└─$ javap -version
```

### Step 3 — Decompile

```
┌──(zham㉿kali)-[~/ctf]
└─$ java -jar cfr-0.152.jar SafeOpener.class
```

CFR prints the recovered Java source straight to stdout. Capture it:

```
┌──(zham㉿kali)-[~/ctf]
└─$ java -jar cfr-0.152.jar SafeOpener.class > SafeOpener.java
```

On screen (or in `SafeOpener.java`):

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.Base64;

public class SafeOpener {
    public static void main(String[] args) throws IOException {
        BufferedReader keyboard = new BufferedReader(new InputStreamReader(System.in));
        Base64.Encoder encoder = Base64.getEncoder();
        String encodedkey = "";
        String key = "";
        for (int i = 0; i < 3; ++i) {
            System.out.print("Enter password for the safe: ");
            key = keyboard.readLine();
            encodedkey = encoder.encodeToString(key.getBytes());
            System.out.println(encodedkey);
            boolean isOpen = SafeOpener.openSafe(encodedkey);
            if (isOpen) break;
            System.out.println("You have  " + (2 - i) + " attempt(s) left");
        }
    }

    public static boolean openSafe(String password) {
        String encodedkey = "picoCTF{SAf3_0p3n3rr_y0u_solv3d_it_198203f7}";
        if (password.equals(encodedkey)) {
            System.out.println("Sesame open");
            return true;
        }
        System.out.println("Password is incorrect\n");
        return false;
    }
}
```

The flag is sitting in `openSafe` as the literal `encodedkey`. Confirmed.

### Step 4 (optional) — Inspect the constant pool with `javap`

If you want to peek at the bytecode itself (and not just the decompiled source), `javap` from the JDK does it for free:

```
┌──(zham㉿kali)-[~/ctf]
└─$ javap -v SafeOpener.class | less
```

In the output, find the `Constant pool:` section:

```
   #1 = Methodref          #29.#67       // java/lang/Object."<init>":()V
   ...
  #24 = String             #93           // picoCTF{SAf3_0p3n3rr_y0u_solv3d_it_198203f7}
  #25 = Methodref          #82.#94       // java/lang/String.equals:(Ljava/lang/Object;)Z
  #26 = String             #95           // Sesame open
  #27 = String             #96           // Password is incorrect\n
```

Entry `#24` is the flag — same string, same place. The bytecode later loads it with `ldc #24` and compares it with `String.equals`. That is exactly what the decompiled Java `password.equals(encodedkey)` line is doing.

If you also want the actual bytecode instructions:

```
┌──(zham㉿kali)-[~/ctf]
└─$ javap -c -p SafeOpener.class
```

The relevant part of the `openSafe` method is:

```
public static boolean openSafe(java.lang.String);
  Code:
     0: ldc           #24                 // String picoCTF{SAf3_0p3n3rr_y0u_solv3d_it_198203f7}
     2: astore_1
     3: aload_0
     4: aload_1
     5: invokevirtual #25                 // Method java/lang/String.equals:(Ljava/lang/Object;)Z
     8: ifeq          21
    11: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
    14: ldc           #26                 // String Sesame open
    16: invokevirtual #15                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    19: iconst_1
    20: ireturn
    21: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
```

`ldc #24` is the instruction that loads constant pool entry #24 — which is our flag — onto the operand stack. From there it gets compared with the user's password via `String.equals`.

### Bonus — actually run the program if you want

The app expects you to type a password, Base64-encodes it, and compares it to the flag string. That means if we Base64-decode the flag and feed the result back in, the safe opens:

```
┌──(zham㉿kali)-[~/ctf]
└─$ echo "cGljb0NURntTQWYzXzBwM25ycl95MHVfc29sdjNkX2l0XzE5ODIwM2Y3fQ==" | java SafeOpener.class
Enter password for the safe: cGljb0NURntTQWYzXzBwM25ycl95MHVfc29sdjNkX2l0XzE5ODIwM2Y3fQ==
Sesame open
```

(Note: this is just for fun. The flag is the same whether you submit the encoded or decoded form, because the flag is the literal string, not anything the program computes.)

---

## What Happened Internally

A timeline of what the Java app and its bytecode are doing behind the curtain:

1. The developer wrote `SafeOpener.java` in Java. The source has two methods: `main` and `openSafe`. `main` reads a password from the keyboard, Base64-encodes it, and calls `openSafe`. `openSafe` compares the encoded password against the hard-coded string `picoCTF{SAf3_0p3n3rr_y0u_solv3d_it_198203f7}`.
2. The developer ran `javac SafeOpener.java`, which compiled the source into `SafeOpener.class`. The compiler emitted a JVM version-52 (Java 1.8) bytecode file. As part of compilation, the flag literal was added to the constant pool as entry #24, stored as plain UTF-8 bytes.
3. The constant pool is structured: strings start with a `CONSTANT_Utf8_info` tag byte (`0x01`), followed by a 2-byte length, followed by the UTF-8 bytes themselves. So the flag bytes `70 69 63 6F 43 54 46 7B ...` (the ASCII for `picoCTF{...}`) literally sit in the file at the offset the constant pool index points to.
4. When `openSafe` is called, the bytecode at offset `0` runs `ldc #24`. This instruction says "load the constant pool entry #24 onto the operand stack". The JVM finds entry #24 (the flag) and pushes a reference to the `String` object onto the stack.
5. The next instruction, `aload_1`, loads the password parameter. `invokevirtual #25` then calls `String.equals` on the password, passing the flag string as the argument. The result (boolean) is left on the stack.
6. `ifeq 21` checks the boolean: if it is false (password did not match), jump to offset 21 and print `"Password is incorrect\n"`. If it is true, fall through and print `"Sesame open"`, then return `true`.
7. We never had to actually run the program. We just read the constant pool directly with `strings` (or `javap -v`, or CFR) and pulled out the flag string. The whole runtime flow is a red herring.

The crucial lesson: nothing the developer does at the Java source level can hide a string literal from someone who reads the `.class` file. The string *has* to be in the constant pool for the program to work.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download `SafeOpener.class` from the picoCTF artifact server |
| `file` | Confirm the download is a compiled Java class file and see the Java version it targets (1.8) |
| `strings` | Dump every printable ASCII sequence from the `.class` file; one of them is the flag |
| `grep picoCTF` | Filter `strings` output down to the actual flag line |
| `apt-get install default-jdk-headless` | Pull in a JDK so we have `javap` available for the disassembly step |
| `javap -v` | Disassemble the `.class` and print the constant pool (where the flag lives) plus the raw bytecode |
| `javap -c -p` | Print the bytecode instructions of every method (including private ones) |
| `cfr` (Java decompiler) | Decompile the `.class` back into readable Java source so we can read what the program does |
| `Base64` (mental) | Recognize that the comparison in `openSafe` is happening against the already-encoded flag, not the typed password |

---

## Key Takeaways

- **Java class files leak string literals.** Anything the developer writes as a string literal in source code (`"password123"`, `"https://api.example.com/secret"`, `"picoCTF{...}"`) ends up as plain UTF-8 in the constant pool of the `.class` file. `strings <file>` will find it. There is no way around this — the JVM needs the literal in the constant pool to execute the bytecode.
- **`strings | grep <known-prefix>` is your fastest move on any binary that hides a flag.** If you know the flag starts with `picoCTF{` or `flag{`, just grep for it. This works on ELF, PE, DEX, class files, JARs, PDFs, basically everything.
- **Java is one of the easiest targets to reverse engineer.** Class names, method names, variable names, types, and string constants are all preserved in the `.class` file. CFR / procyon / jadx / jd-gui can rebuild near-perfect Java source from bytecode. This is a feature of the JVM design (for runtime introspection, debugging, reflection), not a bug, and it directly enables this category of CTF challenges.
- **`javap -v` is the lowest-level "official" way to look at a `.class`.** It ships with every JDK. No third-party download required. The `Constant pool:` section is where every string the class uses is listed.
- **Base64 is encoding, not encryption.** Base64-encoding a secret before comparing it does not hide the secret from anyone who reads the source or the constant pool — the encoded form is also a string literal sitting right next to the comparison. If `openSafe` had `String.equals(encodedkey)` where `encodedkey = "picoCTF{...}"`, then the flag is right there as a string regardless of how many Base64 round-trips it goes through.
- **Never put secrets in Java/Android source.** This is the real-world lesson. Production apps that need secrets should fetch them from a server at runtime (and authenticate that server), or use a secrets manager. Hard-coding is convenient for development but catastrophic for security.
- **Two-tier solve speed.** Most picoCTF challenges of this shape have a 10-second solve (`strings` + `grep`) and a 5-minute solve (proper decompilation + understanding the logic). The flag is the same in both cases. I usually do the fast path first to confirm I am on the right track, then do the slow path to actually learn the lesson the challenge is teaching.

### Flag wordplay decode

`picoCTF{SAf3_0p3n3rr_y0u_solv3d_it_198203f7}` decodes to **"safe opener you solved it"**, written in l33t-speak:

- `SAf3_0p3n3rr` -> "safe opener" (`3` for `e`, `0` for `o`)
- `y0u_solv3d_it` -> "you solved it" (`0` for `o`, `3` for `e`)
- `198203f7` -> the unique hex suffix to make this flag different from anyone else's copy of the challenge.

The whole flag is the developer congratulating us on cracking open the safe. The joke is that there was no cracking involved — the password was painted on the side of the safe in bright letters (the constant pool). That is exactly the lesson picoCTF wants you to learn in this challenge: client-side secrets are not secrets.
