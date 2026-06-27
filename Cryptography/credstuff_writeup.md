# credstuff — picoCTF Writeup

**Challenge:** credstuff  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{C7r1F_54V35_71M3}`  
**Platform:** picoCTF (2023)  
**Writeup by:** zham  

---

## Description

> We found a leak of a blackmarket website's login credentials. Can you find the password of the user cultiris and successfully decrypt it?
>
> Download the leak here.

> The first user in `usernames.txt` corresponds to the first password in `passwords.txt`. The second user corresponds to the second password, and so on.

## Hints

> 1. Maybe other passwords will have hints about the leak?

---

## Background Knowledge (Read This First!)

### What is a "credential leak"?

A credential leak is when usernames and passwords get dumped to the public (or the dark web). In real life this happens all the time. When you see a list like `usernames.txt` paired with `passwords.txt` where each line in one file lines up with a line in the other, you have the classic "one-to-one dump" format that attackers love to parse with simple scripts.

### What is ROT13?

ROT13 is a Caesar cipher that shifts every letter by 13 places. Because the English alphabet has 26 letters, applying ROT13 twice brings you back to the original text — so ROT13 is its own inverse. Numbers and symbols (`{`, `}`, `_`, digits) are not affected.

```
p -> c
i -> v
c -> p
o -> b
P -> C
T -> G
F -> S
```

So `picoCTF` becomes `cvpbPGS` after ROT13.

### Why does this hint say "decrypt"?

The challenge text says we need to "decrypt" the password we find. Since we will land on something like `cvpbPGS{...}` (a `picoCTF{...}` shape with the letters shifted), ROT13 is almost always the answer in beginner picoCTF challenges.

### Matching lines across two files

In Linux, two files that share a 1-to-1 line relationship can be joined by line number. The easiest one-liner is:

```bash
sed -n '378p' passwords.txt
# or
awk 'NR==378' passwords.txt
# or
head -378 passwords.txt | tail -1
```

For finding the line number of a string, `grep -n` is your friend:

```bash
grep -n '^cultiris$' usernames.txt
```

---

## Solution — Step by Step

### Step 1 — Extract the leak

The challenge ships as a `.tar` file. Pull it out:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ tar -xf leak.tar
┌──(zham㉿kali)-[~/Downloads]
└─$ cd leak
┌──(zham㉿kali)-[~/Downloads/leak]
└─$ ls -la
total 24
drwxr-xr-x 2 zham zham  4096 Mar 16  2023 .
drwxr-xr-x 3 zham zham  4096 Jun 27 20:17 ..
-rwxr-xr-x 1 zham zham 13130 Mar 16  2023 passwords.txt
-rwxr-xr-x 1 zham zham  7531 Mar 16  2023 usernames.txt
```

Two plain text files. No funny binaries to worry about.

### Step 2 — Sanity-check the file sizes

```
┌──(zham㉿kali)-[~/Downloads/leak]
└─$ wc -l usernames.txt passwords.txt
  505 usernames.txt
  505 passwords.txt
 1010 total
```

Both files have 505 lines. That lines up perfectly with the "line N in one file matches line N in the other" rule from the description.

### Step 3 — Take a quick peek

```
┌──(zham㉿kali)-[~/Downloads/leak]
└─$ head -3 usernames.txt
engineerrissoles
icebunt
fruitfultry

┌──(zham㉿kali)-[~/Downloads/leak]
└─$ head -3 passwords.txt
CMPTmLrgfYCexGzJu6TbdGwZa
GK73YKE2XD2TEnvJeHRBdfpt2
UukmEk5NCPGUSfs5tGWPK26gG
```

Random-looking base62 strings. Nothing screams "here is the flag" yet.

### Step 4 — Find the line number of `cultiris`

```
┌──(zham㉿kali)-[~/Downloads/leak]
└─$ grep -n '^cultiris$' usernames.txt
378:cultiris
```

`cultiris` is on line 378. (`^` and `$` anchor the match so we don't accidentally hit a longer username that contains `cultiris` as a substring.)

### Step 5 — Grab the matching password

```
┌──(zham㉿kali)-[~/Downloads/leak]
└─$ sed -n '378p' passwords.txt
cvpbPGS{P7e1S_54I35_71Z3}
```

The password is `cvpbPGS{P7e1S_54I35_71Z3}`. That `cvpbPGS{...}` shape is a dead giveaway — it is `picoCTF{...}` after ROT13. The hint also nudges us: maybe one of these dumped passwords is itself a wrapped picoCTF flag.

### Step 6 — Decrypt with ROT13

```
┌──(zham㉿kali)-[~/Downloads/leak]
└─$ echo "cvpbPGS{P7e1S_54I35_71Z3}" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
picoCTF{C7r1F_54V35_71M3}
```

Got it. Flag: `picoCTF{C7r1F_54V35_71M3}`.

---

## Alternative Methods

**Method 1 — `tr` (built-in, what we used)**

```bash
echo "cvpbPGS{P7e1S_54I35_71Z3}" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

`tr` swaps each letter with the letter 13 positions ahead in the alphabet. Numbers, braces, and underscores pass through untouched.

**Method 2 — Python with `codecs`**

```python
import codecs
cipher = "cvpbPGS{P7e1S_54I35_71Z3}"
print(codecs.decode(cipher, 'rot_13'))
```

**Method 3 — Python one-liner with `translate`**

```python
import string
cipher = "cvpbPGS{P7e1S_54I35_71Z3}"
table = str.maketrans(
    string.ascii_uppercase + string.ascii_lowercase,
    string.ascii_uppercase[13:] + string.ascii_uppercase[:13] +
    string.ascii_lowercase[13:] + string.ascii_lowercase[:13]
)
print(cipher.translate(table))
```

**Method 4 — `caesar` from `bsdgames`**

```
┌──(zham㉿kali)-[~/Downloads/leak]
└─$ echo "cvpbPGS{P7e1S_54I35_71Z3}" | caesar
picoCTF{C7r1F_54V35_71M3}
```

`caesar` is a small utility that brute-tries all 25 Caesar shifts; you read the one that looks like English.

**Method 5 — CyberChef**

Drop the ciphertext into CyberChef (https://gchq.github.io/CyberChef/), add a "ROT13" recipe, and read the output.

---

## What Happened Internally

A timeline of what the server (and we) actually did:

1. We downloaded `leak.tar` — a gzipped/archived copy of a blackmarket site's user database.
2. Inside were two parallel files: `usernames.txt` and `passwords.txt`. Each line `N` of `usernames.txt` paired with line `N` of `passwords.txt` was one account.
3. We asked grep to find the line index of `cultiris`. The match came back at line 378.
4. We pulled line 378 of `passwords.txt`. The site had stored that user's password as `cvpbPGS{P7e1S_54I35_71Z3}` — which is just `picoCTF{C7r1F_54V35_71M3}` rotated by 13.
5. We mapped each letter 13 places forward in the alphabet with `tr`, recovering the original picoCTF flag. Numbers, braces, and underscores were left alone because `tr` only operates on the ranges we asked it to.
6. The site operator was being cheeky and stored real passwords as ROT13 to make them look "encrypted" — but ROT13 is not encryption, it is a reversible text transformation.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `tar` | Extract the `.tar` archive |
| `ls` | List extracted files |
| `wc -l` | Confirm both files have the same line count |
| `head` | Take a quick peek at the format |
| `grep -n` | Find the line number of `cultiris` |
| `sed -n 'Np'` | Print line N of the password file |
| `tr` | Apply ROT13 letter-by-letter |
| `caesar` (bsdgames) | Optional: brute Caesar-shift decode |

---

## Key Takeaways

- Two parallel text files that share a line index are the simplest kind of credential dump. Indexing by line number is a perfectly valid join strategy when there is no separator.
- `grep -n` gives you `line:match`, which is the fastest way to turn "find this username" into "pull this password".
- The shape `cvpbPGS{...}` is a famous picoCTF tell. Whenever you see it, try ROT13 first — it is the cipher the platform teaches early.
- ROT13 is **not** encryption. There is no key, no secret, no security. Anyone who knows the alphabet can undo it. Real password storage uses slow hashes like bcrypt or Argon2.
- The `tr 'A-Za-z' 'N-ZA-Mn-za-m'` trick is a one-liner worth memorizing — it does ROT13 in pure Linux with zero dependencies.
- The flag itself, `picoCTF{C7r1F_54V35_71M3}`, is leetspeak. Decoded word-by-word:
  - `C7r1F` = `C` + `t` + `r` + `i` + `F` -> **"Cert(s)"** (or "C-t-r-i-F")
  - `54V35` = `s` + `a` + `v` + `e` + `s` -> **"saves"**
  - `71M3`  = `t` + `i` + `m` + `e`    -> **"time"**
  - Putting it together: **"Certs save time"** — a cheeky nod to the fact that real credentials (certificates/keys) save you time when you stop re-typing passwords you forget.
