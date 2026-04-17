# StegoRSA — picoCTF Writeup

**Challenge:** StegoRSA  
**Category:** Cryptography  
**Difficulty:** Easy  
**Flag:** `picoCTF{rs4_k3y_1n_1mg_6fdc126c}`  

---

## Description

> A message has been encrypted using RSA. The public key is gone... but someone might have been careless with the private key. Can you recover it and decrypt the message?
> Download the **flag** and **image**.

**Hint 1:** `Metadata can tell you more than you expect.`

**Attachments:** `flag.enc`, `image.jpg`

---

## Background Knowledge (Read This First!)

### What is RSA?

**RSA** (Rivest–Shamir–Adleman) is an asymmetric encryption algorithm. It uses a **key pair**:
- A **public key** — used to encrypt data (can be shared freely)
- A **private key** — used to decrypt data (must be kept secret)

If you have the private key, you can decrypt anything encrypted with the matching public key.

### What is Steganography?

**Steganography** is the practice of hiding data **inside other files** — not encrypting it, but concealing its very existence. Common carriers include images, audio files, and documents.

In this challenge, the private key is hidden inside `image.jpg` — not as a visual watermark, but tucked inside the file's **metadata**.

### What is EXIF Metadata?

Every JPEG image carries **EXIF metadata** — invisible data embedded in the file header that stores information like camera model, GPS coordinates, timestamps, and arbitrary comment fields. The `Comment` field in particular has no enforced format and can hold any data — including a hex-encoded RSA private key.

### What is `openssl pkeyutl`?

`openssl pkeyutl` is the OpenSSL command for performing raw **asymmetric key operations** — including RSA decryption. Given a private key file and an encrypted binary, it recovers the original plaintext.

### ⚠️ Two Important Notes

**Note 1 — Tools needed**
```
sudo apt-get install exiftool openssl
```
Both are typically pre-installed on Kali Linux.

**Note 2 — The hex comment is long**
The hex string in the image comment encodes a full RSA private key in PEM format. Do not try to copy it manually — use a one-liner to extract and decode it automatically.

---

## Solution — Step by Step

### Step 1 — Inspect the image metadata

Run `exiftool` on `image.jpg` to check all embedded metadata:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool image.jpg
```

Most fields are unremarkable, but the `Comment` field contains a massive hex string:

```
Comment : 2d2d2d2d2d424547494e2050524956415445204b45592d2d2d2d2d0a...
```

The hint said "Metadata can tell you more than you expect" — this is exactly it.

### Step 2 — Decode the hex to recover the private key

The hex decodes to a standard PEM-formatted RSA private key. Extract it in one command:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool -Comment image.jpg | awk '{print $NF}' | python3 -c \
"import sys; print(bytes.fromhex(sys.stdin.read().strip()).decode())" > private_key.pem
```

**What this does:**
- `exiftool -Comment image.jpg` — extracts only the Comment field
- `awk '{print $NF}'` — strips the `Comment :` label, leaving just the hex string
- `python3 -c "..."` — converts hex to bytes and decodes as UTF-8 text
- `> private_key.pem` — saves the result as a PEM key file

If you open `private_key.pem` you'll see:
```
-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQDFX2mh8uDgQSTg
...
-----END PRIVATE KEY-----
```

### Step 3 — Decrypt the flag

Now use the recovered private key to decrypt `flag.enc`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ openssl pkeyutl -decrypt -inkey private_key.pem -in flag.enc -out flag.txt
```

### Step 4 — Read the flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat flag.txt
picoCTF{rs4_k3y_1n_1mg_6fdc126c}
```

✅ Got the flag! 🎯

---

## Alternative Methods

### Alternative 1 — Decode hex with xxd instead of Python

If you prefer shell-only tools:

```bash
exiftool -Comment image.jpg | awk '{print $NF}' | xxd -r -p > private_key.pem
```

`xxd -r -p` reads raw hex from stdin and outputs binary/text — cleaner and avoids Python entirely.

### Alternative 2 — Use CyberChef (GUI / Browser)

If you're more comfortable with a graphical tool:

1. Open **CyberChef** → https://gchq.github.io/CyberChef/
2. Paste the hex string from the Comment field into the Input box
3. Add the **"From Hex"** recipe
4. The output pane shows the full PEM private key — copy and save it as `private_key.pem`
5. Then run the `openssl` decryption command as normal

This is great for beginners who want to visually confirm each conversion step.

### Alternative 3 — Python full solve script

You can automate the entire process end-to-end in a single Python script:

```python
import subprocess

# Step 1: Extract the hex comment from image.jpg
result = subprocess.run(
    ['exiftool', '-Comment', '-b', 'image.jpg'],
    capture_output=True, text=True
)
hex_str = result.stdout.strip()

# Step 2: Decode hex → PEM private key
pem_key = bytes.fromhex(hex_str).decode()
with open('private_key.pem', 'w') as f:
    f.write(pem_key)

# Step 3: Decrypt flag.enc using openssl
subprocess.run([
    'openssl', 'pkeyutl', '-decrypt',
    '-inkey', 'private_key.pem',
    '-in', 'flag.enc',
    '-out', 'flag.txt'
])

# Step 4: Print the flag
with open('flag.txt', 'r') as f:
    print(f.read())
```

Run it with: `python3 solve.py`

---

## Why Was the Key in the Image?

RSA private keys must never be stored carelessly. In this challenge, someone embedded the private key directly into an image's EXIF Comment field — probably thinking it was obscure enough to go unnoticed. This is a classic **security through obscurity** mistake.

The challenge name gives it away: **Stego** (steganography) + **RSA** (the encryption). The flag even encodes the lesson: `rs4_k3y_1n_1mg` → "RSA key in image."

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `exiftool` | Read EXIF metadata from image.jpg | ⭐ Easy |
| `python3` / `xxd` | Decode hex string to PEM key | ⭐ Easy |
| `openssl pkeyutl` | RSA decryption using private key | ⭐⭐ Medium |
| CyberChef (optional) | GUI hex decoder for visual learners | ⭐ Easy |

---

## Key Takeaways

- **Always inspect EXIF metadata** — the Comment field can hide anything, including cryptographic keys
- **Steganography ≠ encryption** — the data is hidden, not scrambled; once found, it's fully readable
- **RSA private keys must never be embedded in files** — even obscure locations like image metadata are discoverable with standard forensic tools
- **`openssl pkeyutl`** is the go-to for raw RSA operations when no padding scheme is specified
- If a CTF challenge combines two concepts in its name (StegoRSA), look for **both** in the files — the stego hides the crypto
