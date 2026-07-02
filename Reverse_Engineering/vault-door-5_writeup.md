# vault-door-5 — picoCTF Writeup

**Challenge:** vault-door-5  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Flag:** `picoCTF{c0nv3rt1ng_fr0m_ba5e_64_be9f10a4}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> In the last challenge, you mastered octal (base 8), decimal (base 10), and hexadecimal (base 16) numbers, but this vault door uses a different change of base as well as URL encoding!
> The source code for this vault is here: VaultDoor5.java

**Hint 1:** You may find an encoder/decoder tool helpful, such as https://encoding.tools/  
**Hint 2:** Read the wikipedia articles on URL encoding and base 64 encoding to understand how they work and what the results look like.

---

## Background Knowledge (Read This First!)

### What does `checkPassword()` do?

This time the password is hidden behind **two layers of encoding**. Look at the pipeline inside `checkPassword`:

```java
String urlEncoded    = urlEncode(password.getBytes());
String base64Encoded = base64Encode(urlEncoded.getBytes());
String expected      = "JTYzJTMw...JTM0";
return base64Encoded.equals(expected);
```

```
password  ──urlEncode──►  urlEncoded  ──base64Encode──►  base64Encoded
```

`base64Encoded` is compared against a hardcoded expected string. If they match, access is granted.

We don't know the password, but we have:
- the **expected base64 output**, and
- the **exact two functions** that produced it.

To recover the password we just run the pipeline **backwards**:

```
expected  ──base64Decode──►  urlEncoded  ──urlDecode──►  password
```

### What is URL encoding (percent encoding)?

URL encoding turns bytes that aren't safe in URLs into a `%XX` form, where `XX` is the byte's value in hex.

| Character | Byte (hex) | URL-encoded |
|---|---|---|
| `c`  | `0x63` | `%63` |
| `_`  | `0x5F` | `%5f` |
| `7`  | `0x37` | `%37` |
| space | `0x20` | `%20` |

So the 4-character string `c_7 ` would URL-encode to the 12-character string `%63%5f%37%20`. Every input byte becomes **3 output characters** — the password roughly triples in size before base64 is applied.

### What is base64?

Base64 represents binary data using 64 printable ASCII characters: `A-Z`, `a-z`, `0-9`, `+`, `/` (with `=` for padding). Every **3 input bytes** become **4 output characters**. The output is safe to paste anywhere text is allowed.

The expected string in this challenge is 128 base64 characters, which means it encodes 96 raw bytes — exactly the size of `c0nv3rt1ng_fr0m_ba5e_64_be9f10a4` (32 bytes) URL-encoded into 96 bytes.

### Reversing the pipeline

Encoding is a one-way function that you can always undo if you know the algorithm. The trick is to apply each decoder in **reverse order**:

```
Forward :  password  --urlEncode-->  urlEncoded  --base64Encode-->  base64
Reverse :  password  <--urlDecode--  urlEncoded  <--base64Decode--  base64
```

Two Python one-liners do the whole job.

---

## Reading the Source Code

The interesting part of `VaultDoor5.java`:

```java
public boolean checkPassword(String password) {
    String urlEncoded = urlEncode(password.getBytes());
    String base64Encoded = base64Encode(urlEncoded.getBytes());
    String expected = "JTYzJTMwJTZlJTc2JTMzJTcyJTc0JTMxJTZlJTY3JTVm"
                    + "JTY2JTcyJTMwJTZkJTVmJTYyJTYxJTM1JTY1JTVmJTM2"
                    + "JTM0JTVmJTYyJTY1JTM5JTY2JTMxJTMwJTYxJTM0";
    return base64Encoded.equals(expected);
}
```

The two helper functions just wrap `String.format("%%%2x", b)` (percent-encoding) and Java's built-in `Base64.getEncoder()`.

---

## Solution — Step by Step

### Step 1 — Decode by hand, just to see what's happening

Take the first 8 base64 characters of the expected string and decode them:

| Step | Value |
|---|---|
| First 8 chars of base64 | `JTYzJTMw` |
| base64-decode | bytes `25 36 33 25 33 30` → ASCII `% 6 3 % 3 0` |
| url-decode | `c`, `0` |

So the password starts with `c0`. That lines up with the leet for `converting`. Same trick works for any prefix, but doing all 32 characters by hand is tedious — let Python handle it.

### Step 2 — Write the inverse solver

`nano solve_vd5.py`, paste this, save with `Ctrl+O`, `Enter`, `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-5]
└─$ nano solve_vd5.py
```

```python
#!/usr/bin/env python3
import base64
import re
import urllib.parse

with open("VaultDoor5.java") as f:
    src = f.read()

# Pull the multi-line expected string out of the source.
m = re.search(
    r'"([A-Za-z0-9+/=]+)"\s*\+\s*"([A-Za-z0-9+/=]+)"\s*\+\s*"([A-Za-z0-9+/=]+)"',
    src,
)
expected = m.group(1) + m.group(2) + m.group(3)

# 1) base64-decode -> urlEncoded ASCII string
url_encoded = base64.b64decode(expected).decode("ascii")
print(f"[+] base64-decoded (url-encoded): {url_encoded}")

# 2) url-decode -> original password
password = urllib.parse.unquote(url_encoded)
print(f"[+] Recovered password: {password}")
print(f"[+] Flag              : picoCTF{{{password}}}")
```

### Step 3 — Run it

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-5]
└─$ python3 solve_vd5.py
[+] base64-decoded (url-encoded): %63%30%6e%76%33%72%74%31%6e%67%5f%66%72%30%6d%5f%62%61%35%65%5f%36%34%5f%62%65%39%66%31%30%61%34
[+] Recovered password: c0nv3rt1ng_fr0m_ba5e_64_be9f10a4
[+] Flag              : picoCTF{c0nv3rt1ng_fr0m_ba5e_64_be9f10a4}
```

That intermediate `%63%30%6e%76%33%72...` line is gold for learning — you can see the URL-encoded form before it gets decoded. Each `%XX` triplet is one byte of the password.

### Step 4 — Verify by re-running the original pipeline

I never trust a recovered flag blindly, so I re-implement the forward pipeline (urlEncode → base64Encode) on the candidate and check it matches `expected`:

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-5]
└─$ python3 verify_vd5.py
rebuilt base64 : JTYzJTMwJTZlJTc2JTMzJTcyJTc0JTMxJTZlJTY3JTVmJTY2JTcyJTMwJTZkJTVmJTYyJTYxJTM1JTY1JTVmJTM2JTM0JTVmJTYyJTY1JTM5JTY2JTMxJTMwJTYxJTM0
expected       : JTYzJTMwJTZlJTc2JTMzJTcyJTc0JTMxJTZlJTY3JTVmJTY2JTcyJTMwJTZkJTVmJTYyJTYxJTM1JTY1JTVmJTM2JTM0JTVmJTYyJTY1JTM5JTY2JTMxJTMwJTYxJTM0
checkPassword -> Access granted.
flag: picoCTF{c0nv3rt1ng_fr0m_ba5e_64_be9f10a4}
```

"Access granted." matches the Java program output. The flag is good.

---

## Alternative Method — CyberChef / encoding.tools

If you'd rather not touch Python, the hints literally point at a browser-based tool. Paste the `expected` string into https://encoding.tools/ (or CyberChef), apply **base64 decode** then **URL decode**, and you get the password in two clicks.

```
Input :  JTYzJTMwJTZlJTc2JTMzJTcyJTc0JTMxJTZlJTY3JTVmJTY2JTcyJTMwJTZkJTVm...

  ──base64 decode──►  %63%30%6e%76%33%72%74%31%6e%67%5f%66%72%30%6d%5f...

  ──url decode──►     c0nv3rt1ng_fr0m_ba5e_64_be9f10a4
```

The order matters: you must **base64-decode first**, then **URL-decode**. The forward pipeline applied them in the opposite order (`urlEncode` first, then `base64Encode`), so the reverse order is `base64Decode` first, then `urlDecode`.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Reading the Java source | Spot the `urlEncode → base64Encode` pipeline and the `expected` constant | Easy |
| Mental base64 decode (first 8 chars) | See what `%XX` triplets look like before automating | Easy |
| `python3` + `base64` + `urllib.parse` | Two one-liners do the whole reversal | Easy |
| Forward verification | Re-run the original `urlEncode → base64Encode` pipeline on the candidate | Easy |
| CyberChef / encoding.tools (optional) | Browser-based two-click solve | Easy |
| `javac` / `java` (optional) | Compile `VaultDoor5.java` and type the recovered password for a 1:1 Java check | Easy |

---

## Key Takeaways

- **Encoding is not encryption.** Both URL encoding and base64 are fully reversible — no key, no secret. They're just notations. Anyone who has the encoded form can recover the original.
- **Reverse the order of operations.** Forward was `urlEncode → base64Encode`, so reverse is `base64Decode → urlDecode`. Mixing the order produces garbage.
- **`String.format("%%%2x", b)` is just percent-encoding.** The literal `%` consumes one `%` in the format string, then `%2x` formats `b` as two-hex-digit lowercase. Each input byte becomes exactly 3 output characters.
- **`base64.b64decode` and `urllib.parse.unquote` are the Python equivalents** of Java's `Base64.getDecoder()` and a manual percent-decoder. Both ship with the standard library — no installs needed.
- **The pipeline doubles in size, then grows by 33%.** URL-encoding takes the 32-byte password to 96 bytes; base64 then takes 96 bytes to 128 characters. The expected string length (128) is a quick sanity check that you decoded the right thing.
- **Always verify the recovered flag.** Re-running the forward pipeline on the candidate catches transcription typos in the expected string (a one-character swap there would silently break the check).
- **Wordplay decode.** The flag reads as a sentence once you translate the leet:

  | Token | Reads as |
  |---|---|
  | `c0nv3rt1ng` | **converting** (`0`→`o`, `3`→`e`, `1`→`i`) |
  | `fr0m` | **from** (`0`→`o`) |
  | `ba5e_64` | **base 64** (`5`→`s`) |
  | `be9f10a4` | a deliberate hex-looking tail; not a word, just a fingerprint |

  Decoded message: *"converting from base 64"*. The challenge is literally telling you what the puzzle is — the password is base64-encoded (and URL-encoded on top of that), and your job is to undo both layers. The hex tail `be9f10a4` is picoCTF's signature random suffix.
