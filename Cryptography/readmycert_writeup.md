# ReadMyCert - picoCTF Writeup

**Challenge:** ReadMyCert  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{read_mycert_41d1c74c}`  
**Platform:** picoCTF (2019)  
**Writeup by:** zham  

---

## Description

> How about we take you on an adventure on exploring certificate signing requests.
>
> Take a look at this CSR file [here].

## Hints

> 1. Download the certificate signing request and try to read it.

---

## Background Knowledge

Before we crack open the CSR, here is what we are looking at.

**What is a CSR?** A Certificate Signing Request is a file that someone generates when they want a Certificate Authority (CA) to issue them a TLS/SSL certificate. The CSR contains:
- The requester's identifying information (the *Subject*).
- The *public key* they want the CA to sign.
- A *signature* proving the requester owns the corresponding private key.

The CA never sees your private key — they only sign the public half. When you browse to an HTTPS website, the server's certificate (signed by a trusted CA) is what your browser uses to verify the site's identity.

**PEM format.** PEM stands for "Privacy-Enhanced Mail" and is just a way to encode binary data (here, ASN.1 DER) as base64 between header/footer lines:

```
-----BEGIN CERTIFICATE REQUEST-----
<base64 data>
-----END CERTIFICATE REQUEST-----
```

If you see those `BEGIN`/`END` markers, you can decode the contents with `openssl` or any ASN.1 parser.

**ASN.1 and Distinguished Names.** CSR data is structured using ASN.1 (Abstract Syntax Notation One). The *Subject* is a Distinguished Name (DN) made up of typed fields like:
- `CN` — Common Name (often the domain or, in this case, the flag).
- `O`  — Organization.
- `OU` — Organizational Unit.
- `name` — a custom OpenSSL attribute.

When `openssl req` prints the Subject, it lists each field and its value, exactly as it was set when the CSR was generated.

**Why the flag is in the CN.** picoCTF authors love to hide flags in plain sight. A CSR's CN is a free-form text field, so it is the perfect place to drop a flag string. Anyone can decode and read it — which is exactly the point of the challenge: get comfortable with `openssl req` and CSR inspection.

---

## Solution

### Step 1 — Inspect the file

```bash
┌──(zham㉿kali)-[~/picoCTF/ReadMyCert]
└─$ mkdir readmycert && cd readmycert
┌──(zham㉿kali)-[~/picoCTF/ReadMyCert]
└─$ cp ~/Downloads/readmycert.csr csr.pem
┌──(zham㉿kali)-[~/picoCTF/ReadMyCert]
└─$ file csr.pem
csr.pem: PEM certificate request
┌──(zham㉿kali)-[~/picoCTF/ReadMyCert]
└─$ head -1 csr.pem
-----BEGIN CERTIFICATE REQUEST-----
```

PEM CSR. `openssl req` is the right tool.

### Step 2 — Read the CSR with openssl

```bash
┌──(zham㉿kali)-[~/picoCTF/ReadMyCert]
└─$ openssl req -in csr.pem -noout -subject
subject=CN = picoCTF{read_mycert_41d1c74c}, name = ctfPlayer
```

There it is, sitting in the Common Name. The flag is wrapped right there.

### Step 3 — Confirm with the full text dump

To be sure I am not missing anything hidden deeper in the request, I dump the whole CSR:

```bash
┌──(zham㉿kali)-[~/picoCTF/ReadMyCert]
└─$ openssl req -in csr.pem -noout -text
```

The relevant lines from the output:

```
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: CN = picoCTF{read_mycert_41d1c74c}, name = ctfPlayer
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                ...
        Attributes:
            Requested Extensions:
                X509v3 Extended Key Usage:
                    TLS Web Client Authentication
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        ...
```

Only one mention of the flag string: in the Subject's `CN`. Nothing else hidden in the public key, the extensions, or the signature.

---

## Alternative Solve Methods

### Alternative 1 — `openssl asn1parse`

If you want to see the underlying ASN.1 structure rather than the parsed fields:

```bash
┌──(zham㉿kali)-[~/picoCTF/ReadMyCert]
└─$ openssl asn1parse -in csr.pem | head -25
    0:d=0  hl=4 l= 679 cons: SEQUENCE
    4:d=1  hl=4 l= 399 cons: SEQUENCE
    8:d=2  hl=2 l=   1 prim: INTEGER           :00
   11:d=2  hl=2 l=  60 cons: SEQUENCE
   13:d=3  hl=2 l=  38 cons: SET
   15:d=4  hl=2 l=  36 cons: SEQUENCE
   17:d=5  hl=2 l=   3 prim: OBJECT            :commonName
   22:d=5  hl=2 l=  29 prim: UTF8STRING        :picoCTF{read_mycert_41d1c74c}
   53:d=3  hl=2 l=  18 cons: SET
   55:d=4  hl=2 l=  16 cons: SEQUENCE
   57:d=5  hl=2 l=   3 prim: OBJECT            :name
   62:d=5  hl=2 l=   9 prim: UTF8STRING        :ctfPlayer
```

The `UTF8STRING :picoCTF{read_mycert_41d1c74c}` at offset 22 is the flag, sitting inside the SET/SEQUENCE structure of the Distinguished Name.

### Alternative 2 — Python with `cryptography` or `pyOpenSSL`

If you prefer scripting:

```bash
┌──(zham㉿kali)-[~/picoCTF/ReadMyCert]
└─$ pip install cryptography --quiet
┌──(zham㉿kali)-[~/picoCTF/ReadMyCert]
└─$ python3 -c "
from cryptography import x509
from cryptography.hazmat.backends import default_backend
from cryptography.x509 import load_pem_x509_csr

with open('csr.pem', 'rb') as f:
    csr = load_pem_x509_csr(f.read(), default_backend())

print('Subject:', csr.subject)
for attr in csr.subject:
    print(f'  {attr.oid._name} = {attr.value}')
"
Subject: <Name(CN=picoCTF{read_mycert_41d1c74c},name=ctfPlayer)>
  commonName = picoCTF{read_mycert_41d1c74c}
  name = ctfPlayer
```

### Alternative 3 — Browser via `file` then GUI

`file csr.pem` told us it is a "PEM certificate request". Any system with `openssl` installed (most Linux distros, macOS via Homebrew, Windows via WSL) can decode it. On a desktop, you can also paste the PEM block into an online CSR decoder to get the same view.

---

## What Happened Internally

A short timeline of the decode:

1. **Identify the file format** — `file csr.pem` reported "PEM certificate request". PEM + `-----BEGIN CERTIFICATE REQUEST-----` = CSR.
2. **Pick the right tool** — `openssl req` is purpose-built for parsing CSRs. The `-noout -subject` flags print only the Subject line, which is where identifying info lives.
3. **Read the Subject** — `CN = picoCTF{read_mycert_41d1c74c}`. Done.
4. **Cross-check** — `-text` and `asn1parse` confirm there is no second copy of the flag hiding in extensions, the public key, or the signature. Just one canonical location: the Common Name.

The "trick" is purely one of knowing the right tool and the right flag to use. There is no cryptographic attack involved — the CSR is meant to be readable by any CA that might sign it.

---

## Tools Used

| Tool             | Purpose                                                       |
|------------------|---------------------------------------------------------------|
| `file`           | Identify the file as a PEM CSR.                               |
| `openssl req`    | Parse and display the CSR's Subject, public key, extensions.   |
| `openssl asn1parse` | Inspect raw ASN.1 structure (alternative verification).    |
| (optional) `python3 -m cryptography` | Programmatic CSR parsing.                          |
| (optional) `head` | Peek at the PEM header to confirm the file type visually.    |

---

## Key Takeaways

- **`file` is your first move.** Whenever you get a binary-ish blob, `file` tells you what kind of object it is. "PEM certificate request" immediately narrows your tool set to OpenSSL's `req` family.
- **`openssl req -noout -subject` is the CSR cheat code.** It skips the verbose dump and prints just the Subject line, which is where flags almost always hide in CSR-based picoCTF challenges.
- **CN is freeform.** Unlike email addresses or domain names, a CSR's Common Name accepts any UTF-8 string. CTF authors exploit this to drop the flag in plain sight.
- **Cross-check with `-text` and `asn1parse`.** A quick "trust but verify" pass makes sure the flag isn't duplicated elsewhere or hidden under extensions.
- **Read the hint.** "Try to read it" — the entire challenge is just opening the file with the right tool. Don't overthink it.

Flag decoded: `picoCTF{read_mycert_41d1c74c}` — *read my cert*, the entire challenge summary in five words and a hex suffix.
