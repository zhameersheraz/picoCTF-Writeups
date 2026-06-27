# It is my Birthday 2 — picoCTF Writeup

**Challenge:** It is my Birthday 2  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 170  
**Flag:** `picoCTF{h4ppy_b1rthd4y_2_m3_a3023453}`  
**Platform:** picoCTF  
**Writeup by:** zham  

---

## Description

> My birthday is coming up again, but I want to have a very exclusive party for only the best cryptologists. See if you can solve my challenge, upload 2 valid PDFs that are different but have the same SHA1 hash. They should both have the same 1000 bytes at the end as the original invite.
>
> [http://wily-courier.picoctf.net:58299/invite.pdf](http://wily-courier.picoctf.net:58299/invite.pdf)

## Hints

> 1. This isn't REALLY a birthday attack problem
> 2. https://shattered.io/
> 3. The PDFs cannot be the same
> 4. The PDFs must be valid
> 5. The last 1000 bytes of each PDF must match the last 1000 bytes of the original

---

## Background Knowledge (Read This First!)

### Hash collisions, in plain English

A cryptographic hash like SHA-1 takes any file and produces a fixed-length "fingerprint" — for SHA-1 that fingerprint is 40 hex characters. In a perfect world, no two different files should produce the same fingerprint. That property is called *collision resistance*.

In practice, real hash functions get weaker over time as researchers find clever ways to break them. SHA-1 has been considered "broken" since 2017, when Google and CWI Amsterdam published the first practical SHA-1 collision — two real PDFs that produce **the exact same SHA-1 hash** even though the files themselves are byte-for-byte different.

That is the entire premise of this challenge. We are not running a birthday attack (the original "It is my Birthday" challenge did). We just need to grab a pre-built SHA-1 collision pair and submit it.

### The famous SHAttered attack

The SHAttered attack is the name of the 2017 research paper from Google/CWI. The researchers produced two PDF files, `shattered-1.pdf` and `shattered-2.pdf`, that:

- have **different content** (the color in a background image differs, blue vs red)
- produce the **same SHA-1 hash**: `38762cf7f55934b34d179ae6a4c80cadccbb7f0a`

These files are still hosted at https://shattered.io/ as a public proof of concept. You can grab them with `curl` or your browser. They are 422,435 bytes each.

The collision works because SHAttered found two different message blocks that, when run through SHA-1, produce identical internal states after the chosen-prefix stage. The rest of the file is identical between the two PDFs — only the carefully-crafted first blocks differ.

### Why appending 1000 bytes is safe

PDFs are forgiving. A PDF parser starts at the top of the file, follows the cross-reference table (`xref`), and stops at the first `%%EOF` marker it finds. Anything after that marker is ignored — comments, junk, even extra data.

That is why we can take `shattered-1.pdf`, paste 1000 arbitrary bytes on the end, and the file is still a perfectly valid PDF. The original SHAttered PDFs already contain their own valid `%%EOF` somewhere around byte 421,000. We are not modifying that. We are just appending.

### Why we need the last 1000 bytes from the original invite

The challenge server reads the original `invite.pdf` and grabs its last 1000 bytes. It then checks that the last 1000 bytes of **each** uploaded PDF match those bytes exactly. This is the server's way of saying "your PDFs need to look like they came from the same family as the invite". Since PDFs ignore post-`%%EOF` data, we can simply take those 1000 bytes from the invite and paste them onto the end of each shattered PDF.

The last 1000 bytes of a PDF almost always contain the trailer dictionary:

```
trailer
<<
/Size 18
/Info 17 0 R
/Root 1 0 R
/ID [<...> <...>]
>>
startxref
2247248
%%EOF
```

That is the structure of a normal PDF closing block. Copy-pasting it is harmless.

### Tools overview

| Tool | What it does for us |
|------|--------------------|
| `curl` | Download the SHAttered PDFs and (if reachable) the original invite |
| `tail` | Extract the last 1000 bytes of the invite |
| `cat` | Glue the tail back onto each shattered PDF |
| `sha1sum` | Confirm the collision still holds after the append |
| `curl -F` | Multipart-form upload to the challenge server |

---

## Solution — Step by Step

### Step 1 — Set up a working folder

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/birthday2 && cd ~/birthday2

┌──(zham㉿kali)-[~/birthday2]
└─$ pwd
/home/zham/birthday2
```

### Step 2 — Grab the original invite.pdf

The challenge description links to `invite.pdf` on the instance server. On Kali we just download it with `curl`.

```
┌──(zham㉿kali)-[~/birthday2]
└─$ curl -s -o invite.pdf http://wily-courier.picoctf.net:58299/invite.pdf

┌──(zham㉿kali)-[~/birthday2]
└─$ ls -la invite.pdf
-rw-r--r-- 1 zham zham 2247830 Jun 27 14:30 invite.pdf

┌──(zham㉿kali)-[~/birthday2]
└─$ file invite.pdf
invite.pdf: PDF document, version 1.4, 1 pages
```

Quick sanity check on its SHA-1 (it will not collide with anything else, this is just so we know we have the right file):

```
┌──(zham㉿kali)-[~/birthday2]
└─$ sha1sum invite.pdf
4506f9e2d5555971fad2aeb84ce88aeac3172d91  invite.pdf
```

Note: if the instance link is dead or you are working offline, the original invite.pdf from picoCTF 2021 is mirrored at https://github.com/HHousen/PicoCTF-2021/raw/master/Cryptography/It%20is%20my%20Birthday%202/invite.pdf — it has the same SHA-1 and the same last 1000 bytes, so the rest of the solution still works.

### Step 3 — Pull the SHAttered collision pair from shattered.io

```
┌──(zham㉿kali)-[~/birthday2]
└─$ curl -s -o shattered-1.pdf https://shattered.io/static/shattered-1.pdf

┌──(zham㉿kali)-[~/birthday2]
└─$ curl -s -o shattered-2.pdf https://shattered.io/static/shattered-2.pdf

┌──(zham㉿kali)-[~/birthday2]
└─$ ls -la shattered-*.pdf
-rw-r--r-- 1 zham zham 422435 Jun 27 14:28 shattered-1.pdf
-rw-r--r-- 1 zham zham 422435 Jun 27 14:28 shattered-2.pdf
```

Confirm the magic: same SHA-1, different bytes.

```
┌──(zham㉿kali)-[~/birthday2]
└─$ sha1sum shattered-1.pdf shattered-2.pdf
38762cf7f55934b34d179ae6a4c80cadccbb7f0a  shattered-1.pdf
38762cf7f55934b34d179ae6a4c80cadccbb7f0a  shattered-2.pdf
```

Same hash on two different files. That is the whole point of SHAttered.

```
┌──(zham㉿kali)-[~/birthday2]
└─$ cmp -l shattered-1.pdf shattered-2.pdf | head
   193  142 101
   194  142 101
   195  142 101
   ...
```

`cmp -l` shows byte offsets where the files differ. They start near the top of the file and then diverge again later — that is the SHAttered near-collision block structure.

### Step 4 — Extract the last 1000 bytes of the invite

```
┌──(zham㉿kali)-[~/birthday2]
└─$ tail -c 1000 invite.pdf > invite-tail.bin

┌──(zham㉿kali)-[~/birthday2]
└─$ ls -la invite-tail.bin
-rw-r--r-- 1 zham zham 1000 Jun 27 14:30 invite-tail.bin
```

Take a peek to confirm it really is the trailer:

```
┌──(zham㉿kali)-[~/birthday2]
└─$ tail -c 200 invite-tail.bin
Size 18
/Info 17 0 R
/Root 1 0 R
/ID [<4edcb5dadd39b3a68450d3313e623fc7864e179af9f46eed1bd0a3d3567beea5> <4edcb5dadd39b3a68450d3313e623fc7864e179af9f46eed1bd0a3d3567beea5>]
>>
startxref
2247248
%%EOF
```

That is a normal PDF trailer.

### Step 5 — Glue the tail onto both shattered PDFs

Because the SHAttered PDFs already contain a valid `%%EOF` somewhere inside them, anything we append after that point is invisible to a PDF parser. So we can just concatenate.

```
┌──(zham㉿kali)-[~/birthday2]
└─$ cat shattered-1.pdf invite-tail.bin > solve-1.pdf

┌──(zham㉿kali)-[~/birthday2]
└─$ cat shattered-2.pdf invite-tail.bin > solve-2.pdf

┌──(zham㉿kali)-[~/birthday2]
└─$ ls -la solve-*.pdf
-rw-r--r-- 1 zham zham 423435 Jun 27 14:30 solve-1.pdf
-rw-r--r-- 1 zham zham 423435 Jun 27 14:30 solve-2.pdf
```

The collision **must** still hold — appending the same 1000 bytes to both files does not break anything. Verify:

```
┌──(zham㉿kali)-[~/birthday2]
└─$ sha1sum solve-1.pdf solve-2.pdf
133f99dc4ac356ad2b2b611f82d51fda3451d352  solve-1.pdf
133f99dc4ac356ad2b2b611f82d51fda3451d352  solve-2.pdf
```

Identical. The two files are still distinct (`diff` shows the same offsets as before) but they hash to the same SHA-1.

### Step 6 — Upload both PDFs to the challenge server

The web form accepts `multipart/form-data` with two fields, `file1` and `file2`. We replicate that with `curl -F`.

```
┌──(zham㉿kali)-[~/birthday2]
└─$ curl -s -X POST http://wily-courier.picoctf.net:58299/upload \
    -F "file1=@solve-1.pdf" \
    -F "file2=@solve-2.pdf" \
    -i | head
HTTP/1.1 302 FOUND
Server: Werkzeug/2.2.2 Python/3.8.20
Date: Sat, 27 Jun 2026 14:30:26 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 189
Location: /
Vary: Cookie
Set-Cookie: session=eyJfZmxhc2hlcyI6W3siIHQiOlsic3VjY2VzcyIsIkZsYWc6IHBpY29DVEZ7aDRwcHlfYjFydGhkNHlfMl9tM19hMzAyMzQ1M31cbiJdfV19...; HttpOnly; Path=/
```

The 302 redirect is a Flask "flash message" pattern. The flag is baked into the session cookie. Decode the cookie with a one-liner:

```
┌──(zham㉿kali)-[~/birthday2]
└─$ python3 -c "
import base64, json
cookie = 'eyJfZmxhc2hlcyI6W3siIHQiOlsic3VjY2VzcyIsIkZsYWc6IHBpY29DVEZ7aDRwcHlfYjFydGhkNHlfMl9tM19hMzAyMzQ1M31cbiJdfV19'
cookie += '=' * (-len(cookie) % 4)
print(base64.urlsafe_b64decode(cookie).decode())
"
{"_flashes":[{" t":["success","Flag: picoCTF{h4ppy_b1rthd4y_2_m3_a3023453}\n"]}]}
```

Flag: **`picoCTF{h4ppy_b1rthd4y_2_m3_a3023453}`**

If you would rather see it in your browser, just open the upload page in Kali's Firefox and select `solve-1.pdf` and `solve-2.pdf` manually — the success banner shows the same flag.

---

## Alternative Methods

**Method 1 — Python one-liner instead of `cat`**

If you prefer to keep everything in Python:

```python
with open("solve-1.pdf", "wb") as out:
    out.write(open("shattered-1.pdf","rb").read() + open("invite-tail.bin","rb").read())

with open("solve-2.pdf", "wb") as out:
    out.write(open("shattered-2.pdf","rb").read() + open("invite-tail.bin","rb").read())
```

**Method 2 — Do not extract `tail.bin` first, just shell-pipe**

```
┌──(zham㉿kali)-[~/birthday2]
└─$ cat shattered-1.pdf <(tail -c 1000 invite.pdf) > solve-1.pdf

┌──(zham㉿kali)-[~/birthday2]
└─$ cat shattered-2.pdf <(tail -c 1000 invite.pdf) > solve-2.pdf
```

Bash process substitution `<(...)` saves you a file but does the same thing.

**Method 3 — Replace the trailer instead of appending**

A more aggressive variant: open each shattered PDF, find its `startxref` line, throw away everything from there to EOF, and replace it with the corresponding trailer block from the invite. The result is a PDF that opens in any reader **without** post-EOF garbage. The trade-off is that you have to copy not just the trailer but also the `xref` table to match, which is finicky. The append-after-EOF trick above is way simpler and the server is happy with it.

**Method 4 — GUI submission**

You can also do this entirely from Firefox:

1. Open the challenge URL in Firefox.
2. Click the **Choose File** button next to "file1" and pick `solve-1.pdf`.
3. Click the **Choose File** button next to "file2" and pick `solve-2.pdf`.
4. Click **Upload**.
5. The success message on the page is the flag.

Same result, no terminal needed. Useful when you want to double-check what the server actually rendered.

**Method 5 — Use the SHAmbles SHA-1 collisions instead of SHAttered**

Since 2020, the `SHA-mbles` paper produced additional SHA-1 collisions (`shambles-1.pdf`, `shambles-2.pdf`) at a different prefix. They work identically for this challenge — same idea, different magic bytes. Stick with SHAttered because the files are smaller and the download URL is stable.

---

## What Happened Internally

A walkthrough of the full chain of events:

1. We opened the challenge page and saw two constraints: upload two PDFs that share a SHA-1 hash, and the last 1000 bytes of each must match the last 1000 bytes of the original `invite.pdf`.
2. Hint 1 told us this is **not** a birthday attack — no brute-forcing is needed. The server is just checking equality on byte slices.
3. Hint 2 pointed us straight to https://shattered.io/, the public proof-of-concept for the first practical SHA-1 collision. We downloaded both PDFs.
4. Both SHAttered PDFs share SHA-1 `38762cf7...` but differ byte-for-byte at the start of the file. Their first ~320 bytes are the magic collision block, after which the rest is essentially the same PDF body.
5. We pulled the last 1000 bytes of `invite.pdf` with `tail -c 1000`. That slice contains the PDF trailer dictionary, the `startxref` offset, and the `%%EOF` marker — standard PDF boilerplate.
6. We concatenated the trailer onto both shattered PDFs. Because each shattered PDF already has its own `%%EOF` somewhere inside the file, the appended bytes live *after* the parser's stopping point and are ignored when the PDF is rendered.
7. SHA-1 is computed over the entire file including the appended tail. Since the same 1000 bytes were appended to both files, the collision still holds: `133f99dc...`.
8. We uploaded the pair to the challenge server. The server did five checks:
   - both files start with the `%PDF-` magic (yes, shattered PDFs do)
   - both files end with `%%EOF` (yes, we just appended one)
   - the SHA-1 of both files matches (yes)
   - the two files are not byte-identical (yes, shattered-1 vs shattered-2)
   - the last 1000 bytes of each match the last 1000 bytes of the invite (yes, we pasted them in)
9. On all five checks passing, the server returned a success flash containing the flag.
10. We decoded the Flask session cookie to read the flag directly from the response — no need to follow the redirect back to `/`.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `mkdir` / `cd` | Create an isolated working directory |
| `curl` | Download the invite, the two shattered PDFs, and POST the multipart upload |
| `file` | Confirm we have a real PDF and not HTML |
| `sha1sum` | Verify the SHAttered collision still holds after the append |
| `cmp -l` | Show byte offsets where the two shattered PDFs differ (curiosity) |
| `tail -c 1000` | Slice off the last 1000 bytes of the invite |
| `cat` | Append the tail to each shattered PDF |
| `ls -la` | Inspect file sizes before and after the append |
| Python `base64` | Decode the Flask session cookie to read the flag from the redirect |
| Firefox (optional) | GUI upload if you do not want to type the curl command |

---

## Key Takeaways

- **SHA-1 is dead for collision resistance.** SHAttered (2017) and SHA-mbles (2020) both produced real public collisions. Any system that relies on SHA-1 to detect "different file" needs to migrate to SHA-256 or SHA-3. Git, for example, now actively detects SHAttered-shaped collisions on push.
- **You almost never need to *generate* a collision yourself.** For CTFs, the public SHAttered pair is usually enough. Generating a fresh collision needs ~6,500 CPU-years of compute or ~$110k of GPU time — way outside a CTF budget.
- **PDFs are forgiving about trailing data.** The cross-reference table and `%%EOF` marker define the document; anything past `%%EOF` is ignored. This is why appending arbitrary bytes to a valid PDF keeps it valid.
- **The hint "this isn't REALLY a birthday attack problem" is the giveaway.** A real birthday attack against SHA-1 (brute-forcing ~2^80 hashes) is infeasible on a CTF laptop. The challenge author is telling you to go grab the pre-built collision instead.
- **Flask flash messages leak through cookies.** When a server returns 302 with `Set-Cookie: session=...`, the flash content (including success messages and flags) is base64-encoded into the cookie. You can decode it without ever visiting the redirected page.
- **`cmp -l` is a great way to eyeball a collision.** It prints every byte offset where two files differ, which makes it obvious that SHAttered's collision lives in a small prefix near the start of the file.
- **Mirror your inputs.** Challenge servers go down, instance URLs change, and original download links rot. Saving the SHAttered PDFs and the invite to a local folder means you can re-run the solve offline if needed.
- **Flag wordplay decode:**
  - `h4ppy` -> **"happy"** (`4` for `a`)
  - `b1rthd4y` -> **"birthday"** (`1` for `i`, `4` for `a`)
  - `2` -> **"to"** (literal "2", as in "birthday 2")
  - `m3` -> **"me"** (`3` for `e`)
  - `a3023453` -> hex-ish tail; reads as "a3" + "023453", most naturally parsed as a random-looking nonce padding the flag (no clean English meaning, just opaque characters to make the flag unique per instance)
  - Putting it together: **"happy birthday to me, a3023453"** — the second birthday challenge in the series, paying off the sequel premise from "It is my Birthday".
