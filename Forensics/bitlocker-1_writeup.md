# Bitlocker 1 — picoCTF Writeup

**Challenge:** Bitlocker 1  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{us3_b3tt3r_p4ssw0rd5_pl5!_3242adb1}`  
**Platform:** picoCTF (picoGym Exclusive)  
**Writeup by:** zham  

---

## Description

> Jacky is not very knowledgable about the best security passwords and used a simple password to encrypt their BitLocker drive. See if you can break through the encryption!
>
> Download the disk image here.

**Hint given in the challenge:**

- `Hash cracking`

---

## Background Knowledge (Read This First!)

### What is BitLocker?

BitLocker is a full-disk encryption feature built into Windows. It encrypts an entire NTFS volume using AES (128-bit or 256-bit). The encryption keys are protected by one or more **key protectors** stored in the FVE metadata block inside the volume. Common protectors are:

| Type | Code | Meaning |
|------|------|---------|
| Clear key | `0x0000` | No encryption (test only) |
| TPM | `0x0100` | Trusted Platform Module releases the key |
| Startup key | `0x0200` | USB key file |
| TPM + PIN | `0x0500` | TPM plus user PIN |
| Recovery password | `0x0800` | The 48-digit recovery key |
| **User password** | `0x2000` | The plain-text password |

This challenge uses the **user password** protector (`0x2000`) because Jacky "used a simple password" — so we can crack it.

### The `-FVE-FS-` signature

A BitLocker-encrypted volume does **not** have a normal MBR partition table. The very first bytes of the volume are the BitLocker header, and the `file` command recognises it by the OEM ID `-FVE-FS-`:

```
$ file disk.dd
disk.dd: DOS/MBR boot sector, ... OEM-ID "-FVE-FS-", ...
```

If you see `-FVE-FS-` it is BitLocker. You cannot mount the raw `.dd` directly — you must decrypt it first.

### The cracking pipeline

BitLocker passwords are slow to attack because of the PBKDF2 stretch key with **1,048,576 iterations**. But the hint "hash cracking" plus "simple password" tells us the password is in `rockyou.txt`, so it will crack fast even on CPU.

```
.bitlocker image
     │
     ▼
bitlocker2john   →   $bitlocker$0$…  (John-the-Ripper hash format)
     │
     ▼
hashcat -m 22100 + rockyou.txt
     │
     ▼
password (e.g. "jacqueline")
     │
     ▼
dislocker-file -u<password>   →   decrypted NTFS volume file
     │
     ▼
fls / icat   →   flag.txt
```

### Tools you need

| Tool | Purpose | Install |
|------|---------|---------|
| `bitlocker2john` | Extract the cracking hash from a `.dd` image | grab the Python script from john-jumbo |
| `john` or `hashcat` | Crack the password hash | `apt install john` or `apt install hashcat` |
| `dislocker` | Decrypt a BitLocker volume to a raw NTFS file | `apt install dislocker` |
| `fls` / `icat` | Read files from the decrypted NTFS without mounting | `apt install sleuthkit` |
| `rockyou.txt` | Standard wordlist | download from brannondorsey/naive-hashcat |

---

## Solution — Step by Step

### Step 1 — Copy the image to `/tmp`

BitLocker images are big. Always work in `/tmp`:

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /tmp/bitlocker && cd /tmp/bitlocker
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ cp ~/Downloads/disk.dd .
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ file disk.dd
disk.dd: DOS/MBR boot sector, ... OEM-ID "-FVE-FS-", ...
```

The `-FVE-FS-` OEM ID confirms this is a BitLocker volume, not a normal NTFS or FAT volume.

### Step 2 — Try `mmls` (it will be empty)

```
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ mmls disk.dd
(no output — no MBR partition table)
```

This is expected. The image starts directly with the BitLocker header, not a partitioned disk. Skip straight to extracting the hash.

### Step 3 — Extract the hash with `bitlocker2john`

The Debian `john` package does not ship `bitlocker2john`. Grab the Python version from the official john-jumbo repo:

```
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ curl -sL https://raw.githubusercontent.com/openwall/john/bleeding-jumbo/run/bitlocker2john.py -o bitlocker2john.py
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ python3 bitlocker2john.py disk.dd -o 0 > disk.hash 2> extract.log
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ grep -E '^\$bitlocker\$[01]\$' disk.hash > hash_pw.txt
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ cat hash_pw.txt
$bitlocker$0$16$cb4809fe9628471a411f8380e0f668db$1048576$12$d04d9c58eed6da010a000000$60$68156e51e53f0a01c076a32ba2b2999afffce8530fbe5d84b4c19ac71f6c79375b87d40c2d871ed2b7b5559d71ba31b6779c6f41412fd6869442d66d
$bitlocker$1$16$cb4809fe9628471a411f8380e0f668db$1048576$12$d04d9c58eed6da010a000000$60$68156e51e53f0a01c076a32ba2b2999afffce8530fbe5d84b4c19ac71f6c79375b87d40c2d871ed2b7b5559d71ba31b6779c6f41412fd6869442d66d
```

There are 4 hashes total — two are password-protected (`$bitlocker$0$` and `$bitlocker$1$`, the old and new hash algorithms) and two are recovery-password-protected (`$bitlocker$2$` and `$bitlocker$3$`). We only need the password ones because that is what Jacky actually used.

### Step 4 — Get `rockyou.txt`

```
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ curl -sL https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt -o rockyou.txt
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ wc -l rockyou.txt
14344391 rockyou.txt
```

14 million candidate passwords.

### Step 5 — Crack with `hashcat`

BitLocker password hashes are hashcat mode **22100**:

```
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ hashcat -m 22100 hash_pw.txt rockyou.txt --force -O --potfile-path=hc.pot
```

A few seconds later:

```
$bitlocker$0$16$cb4809fe9628471a411f8380e0f668db$1048576$12$d04d9c58eed6da010a000000$60$68156e51e53f0a01c076a32ba2b2999afffce8530fbe5d84b4c19ac71f6c79375b87d40c2d871ed2b7b5559d71ba31b6779c6f41412fd6869442d66d:jacqueline
```

Password is **`jacqueline`**.

Confirm with `--show`:

```
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ hashcat -m 22100 hash_pw.txt --show
...:jacqueline
```

### Step 6 — Decrypt the volume with `dislocker-file`

Important syntax: the password flag is `-u<password>` with **no space**, and the trailing path is the **output file** (not directory):

```
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ dislocker-file -V disk.dd -ujacqueline -O0 -- /tmp/bitlocker/dislocker.bin
```

Verify the decrypted output is a real NTFS volume:

```
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ file dislocker.bin
dislocker.bin: DOS/MBR boot sector, ... OEM-ID "NTFS    ", ...
```

### Step 7 — Read the NTFS volume with the Sleuth Kit

You do **not** need to mount the NTFS — `fls` and `icat` read it directly. `mmls` will not work either (the decrypted image has no partition table), so start with `fls`:

```
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ fls -r dislocker.bin | grep -iE "flag|pico|jacky"
r/r 38-128-1: flag.txt
```

One file, inode `38`. Extract it:

```
┌──(zham㉿kali)-[/tmp/bitlocker]
└─$ icat dislocker.bin 38
picoCTF{us3_b3tt3r_p4ssw0rd5_pl5!_3242adb1}
```

Got the flag.

---

## What Happened Internally (Timeline)

1. The challenge gave a 100 MB `.dd` image that started with `-FVE-FS-` — the BitLocker signature, not an MBR. So `mmls` returned nothing and we went straight to hash extraction.
2. `bitlocker2john` parsed the FVE metadata block (a JSON-like structure at a fixed offset in the volume) and found four key protectors. Two were password-protected (`0x2000`), two were recovery-password-protected (`0x0800`).
3. `hashcat -m 22100` ran each candidate from `rockyou.txt` through the BitLocker PBKDF2 stretch (1,048,576 iterations) and compared it to the stored hash. The password `jacqueline` matched in seconds because it is in the top 1000 most common passwords.
4. `dislocker-file` used that password to decrypt the Volume Master Key (VMK), which in turn decrypted the Full Volume Encryption Key (FVEK). It wrote the decrypted NTFS volume to `dislocker.bin`.
5. The Sleuth Kit's `fls` walked the decrypted NTFS $MFT and found one suspicious file: `/flag.txt`. `icat` read its data runs from the volume and printed the flag.

---

## Alternative Methods

**Method 1 — Crack with John the Ripper jumbo**

If you have john-jumbo installed (not the Debian basic john), the flow is the same:

```bash
john --format=bitlocker --wordlist=rockyou.txt disk.hash
john --format=bitlocker --show disk.hash
```

**Method 2 — Use `dislocker` (FUSE) and mount the NTFS**

Instead of producing a raw file with `dislocker-file`, you can use `dislocker` to FUSE-mount the decrypted NTFS in place:

```bash
mkdir -p /mnt/bitlocker-decrypted
dislocker -V disk.dd -ujacqueline -O0 -- /mnt/bitlocker-decrypted
mount -o ro /mnt/bitlocker-decrypted/dislocker-file /mnt/ntfs
cat /mnt/ntfs/flag.txt
```

**Method 3 — Skip the cracking step entirely with bdemount / libbde**

If you already know the password you can use `bdeinfo` and `bdemount` from `libbde`:

```bash
bdemount -p jacqueline -o ro disk.dd /mnt/bde
ls /mnt/bde
```

**Method 4 — Use the recovery password instead**

If you only have the 48-digit BitLocker recovery key (not the user password), the hash format is different (`$bitlocker$2$` / `$bitlocker$3$`) and hashcat cannot crack it directly. You have to decrypt with `dislocker -p<recovery_password>`.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify the BitLocker signature |
| `bitlocker2john` (Python) | Extract the cracking hash from the FVE metadata |
| `hashcat` | Crack the password hash with `rockyou.txt` |
| `rockyou.txt` | 14 M-password wordlist |
| `dislocker-file` | Decrypt the BitLocker volume to a raw NTFS file |
| `fls` (Sleuth Kit) | List files inside the decrypted NTFS |
| `icat` (Sleuth Kit) | Read the file contents by inode |

---

## Key Takeaways

- A `-FVE-FS-` OEM ID is the instant fingerprint of a BitLocker volume — you do not need to `mmls` it.
- `bitlocker2john` is **not** in the Debian `john` package. Either install `john-jumbo` or grab the Python script directly from the john GitHub.
- Hashcat mode **22100** handles both `$bitlocker$0$` (old) and `$bitlocker$1$` (new) password hashes.
- After decryption, you do not need to mount the NTFS — `fls` + `icat` from the Sleuth Kit read the file system directly, which is faster and avoids needing root.
- The flag wordplay reads `us3 b3tt3r p4ssw0rd5 pl5!` → **"use better passwords pls!"** — a tongue-in-cheek lesson, given the challenge password was literally just a first name.
- Always pipe file contents through `od -c` or `xxd` to verify flag characters before submitting. CTF flags sometimes use `0/O`, `1/l`, `5/s`, `3/e` etc. and it is easy to misread them in the terminal.
