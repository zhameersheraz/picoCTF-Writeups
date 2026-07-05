# Safe Opener — picoCTF Writeup

**Challenge:** Safe Opener  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{pl3as3_l3t_m3_1nt0_th3_saf3}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Can you open this safe?
>
> I forgot the key to my safe but this `program` is supposed to help me with retrieving the lost key. Can you help me unlock my safe?
>
> Put the password you recover into the picoCTF flag format like:
>
> `picoCTF{password}`

## Hints

**Hint 1:** (None shown — the source is small enough to read end-to-end.)

---

## Background Knowledge

A few concepts before diving in.

**1. Java source vs Java bytecode.**
A `.java` file is human-readable source. After `javac`, it becomes a `.class` file — JVM bytecode that you can decompile with `cfr`, `procyon`, or `jadx` if you ever lose the source. Either way, the logic is the same and the strings inside the class file are easy to recover with `strings`.

**2. Base64 encoding.**
Base64 turns arbitrary bytes into a 64-character alphabet (`A-Z`, `a-z`, `0-9`, `+`, `/`) plus `=` padding. It is **not encryption** — anyone can reverse it with `base64 -d` or `atob()`. Whenever you see a string of mixed-case letters, digits, `+`, `/`, and trailing `=`, it is almost certainly Base64 and instantly decodable.

**3. Why "encode then compare" is a weak check.**
The program reads your password, **encodes it** to Base64, and compares the encoded result to a hard-coded Base64 string. That makes the source look busy, but the secret is sitting in the source as a literal Base64 blob. Decoding it gives the original password directly.

**4. `BufferedReader.readLine()` for input.**
`keyboard.readLine()` reads a full line of text. So whatever you type into the prompt is the raw password — the program is what turns it into Base64. Knowing this lets you skip the encoding step entirely if you already have the encoded string.

---

## Step-by-step Solution

I worked this out on my Kali VM (VirtualBox). The downloaded challenge file is `SafeOpener.java`.

### 1. Move the file into a working directory and read it

```
┌──(zham㉿kali)-[~/safe-opener]
└─$ mkdir -p ~/safe-opener && cd ~/safe-opener

┌──(zham㉿kali)-[~/safe-opener]
└─$ cp ~/Downloads/SafeOpener.java .

┌──(zham㉿kali)-[~/safe-opener]
└─$ cat SafeOpener.java
```

The relevant piece is `openSafe()`:

```java
public static boolean openSafe(String password) {
    String encodedkey = "cGwzYXMzX2wzdF9tM18xbnQwX3RoM19zYWYz";

    if (password.equals(encodedkey)) {
        System.out.println("Sesame open");
        return true;
    }
    else {
        System.out.println("Password is incorrect\n");
        return false;
    }
}
```

The program takes user input, base64-encodes it with `Base64.getEncoder().encodeToString(key.getBytes())`, and compares the result to the literal `encodedkey`. Whatever base64-decodes to that literal is the password.

### 2. Decode the Base64 literal with `base64 -d`

```
┌──(zham㉿kali)-[~/safe-opener]
└─$ echo "cGwzYXMzX2wzdF9tM18xbnQwX3RoM19zYWYz" | base64 -d
pl3as3_l3t_m3_1nt0_th3_saf3
```

The decoded password is `pl3as3_l3t_m3_1nt0_th3_saf3`.

### 3. Wrap it in the picoCTF flag format

```
┌──(zham㉿kali)-[~/safe-opener]
└─$ echo -n "picoCTF{" && echo "cGwzYXMzX2wzdF9tM18xbnQwX3RoM19zYWYz" | base64 -d && echo "}"
picoCTF{pl3as3_l3t_m3_1nt0_th3_saf3}
```

Flag: `picoCTF{pl3as3_l3t_m3_1nt0_th3_saf3}`.

### 4. Verify by actually running the program

To make sure the password is right, I compiled the source and ran the binary, feeding the decoded password on stdin:

```
┌──(zham㉿kali)-[~/safe-opener]
└─$ javac SafeOpener.java

┌──(zham㉿kali)-[~/safe-opener]
└─$ echo "pl3as3_l3t_m3_1nt0_th3_saf3" | java SafeOpener
Enter password for the safe: cGwzYXMzX2wzdF9tM18xbnQwX3RoM19zYWYz
Sesame open
```

The program echoes back the base64 form, prints `Sesame open`, and exits. Password verified.

---

## Alternative Solves

**A. Decoding with Python instead of `base64`.**
If you live in Python more than the shell, the same one-liner works:

```
┌──(zham㉿kali)-[~/safe-opener]
└─$ python3 -c "import base64; print(base64.b64decode('cGwzYXMzX2wzdF9tM18xbnQwX3RoM19zYWYz').decode())"
pl3as3_l3t_m3_1nt0_th3_saf3
```

**B. Decompiling from `.class` if you only have the bytecode.**
If the challenge ever ships only `SafeOpener.class`, you can recover the same string with `strings`:

```
┌──(zham㉿kali)-[~/safe-opener]
└─$ strings SafeOpener.class | grep -E "^[A-Za-z0-9+/=]{20,}$"
cGwzYXMzX2wzdF9tM18xbnQwX3RoM19zYWYz
```

Pipe that through `base64 -d` and you have the password without ever running the program.

**C. Skipping the comparison with a Java patch.**
If for some reason you wanted to brute-force or modify behavior, you could recompile `SafeOpener` with the `openSafe` body hard-coded to `return true;`. But for this challenge the static decode is so much faster that patching is overkill.

---

## What Happened Internally

1. `SafeOpener.main` set up a `BufferedReader` over `System.in` and a Base64 `Encoder`.
2. It looped up to three times, prompting for a password.
3. Each iteration called `encoder.encodeToString(key.getBytes())` — converting the typed string to UTF-8 bytes, then Base64.
4. The encoded string was passed to `openSafe(password)`.
5. `openSafe` compared the encoded password (byte-for-byte) against the literal `"cGwzYXMzX2wzdF9tM18xbnQwX3RoM19zYWYz"`.
6. If the comparison matched, the function printed `Sesame open` and returned `true`, breaking the loop. Otherwise, it printed `Password is incorrect`, returned `false`, and decremented the remaining attempts counter.

The "trick" of the challenge is just that the secret is stored in Base64 in plain sight. Decoding the literal gives you the password directly — you never need to interact with the program.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `cat`            | Quick read of `SafeOpener.java`.                               |
| `base64 -d`      | Decoded the hard-coded Base64 literal in one command.          |
| `python3`        | Optional — alternative way to decode the same string.          |
| `javac`          | Compiled `SafeOpener.java` into a runnable `.class`.           |
| `java`           | Ran the compiled `SafeOpener` to confirm the password.         |
| `echo + pipe`    | Fed the recovered password into `java SafeOpener` non-interactively. |
| `strings` (alt)  | Mentioned for the `.class`-only variant.                       |

---

## Key Takeaways

- Base64 is encoding, not encryption. Any Base64-looking string in a source file is reversible with `base64 -d` (or `atob()` in a browser console).
- A "checks the encoded form" pattern is a common lightweight obfuscation trick — but the secret is the **decoded** value, not the encoded literal. Always try decoding first.
- When a program reads input and applies a transformation before comparing, the comparison value in the source is what you should reverse the transformation on, not what you should type.
- The flag's leet: `pl3as3_l3t_m3_1nt0_th3_saf3` reads as **please let me into the safe** (3→e, 1→i, 0→o). So the wordplay is exactly the safe's plea — "please let me into the safe."
