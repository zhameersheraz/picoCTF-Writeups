# Bitlocker 2 — picoCTF Writeup

**Challenge:** Bitlocker 2  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{B1tl0ck3r_dr1v3_d3crypt3d_9029ae5b}`  
**Platform:** picoCTF 2025  
**Writeup by:** zham  

---

## Description

> Jacky has learnt about the importance of strong passwords and made sure to encrypt the BitLocker drive with a very long and complex password. We managed to capture the RAM while this drive was opened however. See if you can break through the encryption!
>
> Download the disk image here and the RAM dump here.

**Hint given in the challenge:**

- `Try using a volatility plugin`

---

## Background Knowledge (Read This First!)

### What is BitLocker?

BitLocker is a full-disk encryption feature built into Windows (Vista and newer). When a drive is encrypted with BitLocker, every byte of user data is encrypted with a key called the **Full Volume Encryption Key (FVEK)**. Without that key, the data on the drive is just random-looking ciphertext. The FVEK itself is never stored in plaintext on disk — it is wrapped by another key called the **Volume Master Key (VMK)**, which is in turn protected by one or more **key protectors**:

| Protector | Code | Meaning |
|-----------|------|---------|
| TPM | `0x0100` | Trusted Platform Module releases the key |
| Startup key | `0x0200` | USB key file |
| TPM + PIN | `0x0500` | TPM plus user PIN |
| Recovery password | `0x0800` | The 48-digit recovery key |
| **User password** | `0x2000` | A plain-text password |
| **Clear key** | `0x0000` | No encryption (test only) |

```
[user password / TPM / recovery password / startup key]
            |
            v  (decrypts VMK)
[VMK] ---> (decrypts FVEK) ---> [FVEK]
                                  |
                                  v  (decrypts every sector)
                              [user data]
```

### The `-FVE-FS-` signature

A BitLocker-encrypted volume does **not** have a normal MBR partition table. The very first bytes of the volume are the BitLocker header, and the `file` command recognises it by the OEM ID `-FVE-FS-`:

```
$ file bitlocker-2.dd
bitlocker-2.dd: DOS/MBR boot sector, code offset 0x58+2, OEM-ID "-FVE-FS-", ...
```

If you see `-FVE-FS-`, it is BitLocker. You cannot mount the raw `.dd` directly — you must decrypt it first.

### Why is RAM useful here?

While the BitLocker volume is **mounted** (i.e. unlocked and in use), Windows keeps the FVEK in kernel memory so it can decrypt reads and encrypt writes on the fly. If you have a memory dump of a running machine with the drive unlocked, you can pull the FVEK straight out of RAM and decrypt the whole volume offline — no need to crack the long complex password or the 48-digit recovery key. This is why `Bitlocker 1` (crack the password) and `Bitlocker 2` (steal the FVEK from RAM) are two different challenges for the same disk-encryption feature.

### What is Volatility?

Volatility is the de-facto open-source memory forensics framework. You point it at a RAM dump, it walks the kernel structures, and you run "plugins" to extract things — running processes, registry hives, network connections, loaded kernel modules, cryptographic keys, and so on.

For BitLocker there are a few community plugins:

| Plugin | Framework | Notes |
|--------|-----------|-------|
| `elceef/bitlocker` | Volatility 2 | Original, Windows Vista / 7 |
| `tribalchicken/volatility-bitlocker` | Volatility 2 | Adds Windows 8.1 / 10 |
| `breppo/Volatility-BitLocker` | Volatility 2 | Outputs Dislocker-format `.fvek` |
| `lorelyai/volatility3-bitlocker` | **Volatility 3** | Validates candidate AES key schedules |

None of them ship with the upstream framework — you have to install them yourself. We use the Volatility 3 one for this challenge because the RAM dump is Windows 10.

### What is AES-XTS?

BitLocker on modern Windows uses **AES-XTS**. XTS is a tweakable block-cipher mode designed for disk encryption. The "key" is really two AES-128 keys: `k1` (the FVEK itself) and `k2` (the tweak key). Decryption of a 512-byte sector works like this:

```
tweak = AES_k2(sector_number)
for each 16-byte block in the sector:
    plaintext = AES_k1_decrypt(ciphertext_block XOR tweak) XOR tweak
    tweak = tweak * alpha      (multiplication in GF(2^128))
```

The volatility plugin returns the FVEK and the TWEAK as a single 32-byte blob, and the `0x8087865bead0-Dislocker.fvek` file the plugin also writes is the dislocker format: 2-byte cipher id (`0x8005` = AES-128 XTS) ‖ 16-byte FVEK ‖ 16-byte TWEAK ‖ 32 zero bytes.

### The on-disk layout of a BitLocker volume

```
+--------------------+  0x00000000  -- (unencrypted) BitLocker boot code (OEM "-FVE-FS-")
|  FVE metadata #1   |
+--------------------+  0x02100000
|  16 boot sectors   |             -- the real NTFS VBR, encrypted with the FVEK
|  (VBR backup)      |                (the "boot_sectors_backup" region)
+--------------------+  0x02120000
|                    |
|  Encrypted user    |
|  data (NTFS)       |
|                    |
+--------------------+
```

A key detail: the first 16 sectors of the original NTFS volume are stored encrypted in the **boot sectors backup** region inside the FVE metadata. That is what `bdeinfo`/`bdemount`/`dislocker` read first to figure out the NTFS volume size and geometry. If you naively decrypt sector 0 of the `.dd` you get the BitLocker boot code, not `NTFS    ` — you have to decrypt the sector at `boot_sectors_backup / 512` instead.

### Tools you need

| Tool | Purpose | Install |
|------|---------|---------|
| `vol` (Volatility 3) | Walk the kernel structures and run the bitlocker plugin | `pip install volatility3` |
| `lorelyai/volatility3-bitlocker` | Volatility 3 plugin that pulls the FVEK + TWEAK from kernel pool memory | `git clone` and copy the `.py` into Volatility's plugins folder |
| `bdeinfo` (libbde) | Read the FVE metadata and confirm the cipher | `apt install libbde-utils` |
| `dislocker` | Decrypt the volume and expose it as a FUSE filesystem | `apt install dislocker` |
| `ntfs-3g` | Mount the decrypted NTFS | `apt install ntfs-3g` |
| `fls` / `icat` (Sleuth Kit) | Read the decrypted NTFS without mounting it | `apt install sleuthkit` |
| `python3` + `pycryptodome` | (Alternative) hand-rolled AES-XTS decryptor | `pip install pycryptodome` |

---

## Solution — Step by Step

### Step 1 — Copy the image to `/tmp`

BitLocker images are big. Always work in `/tmp`:

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /tmp/bitlocker2 && cd /tmp/bitlocker2
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ cp ~/Downloads/bitlocker-2.dd .
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ cp ~/Downloads/memdump.mem.gz .
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ gunzip -k memdump.mem.gz
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ ls -la
-rw-r--r--  134217728  bitlocker-2.dd
-rw-r--r-- 1845788672  memdump.mem
```

### Step 2 — Confirm the .dd is BitLocker

```
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ file bitlocker-2.dd
bitlocker-2.dd: DOS/MBR boot sector, code offset 0x58+2, OEM-ID "-FVE-FS-", ...
```

The `-FVE-FS-` OEM ID confirms this is a BitLocker volume, not a normal NTFS or FAT volume.

### Step 3 — Identify the OS in the RAM dump

```
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ vol -f memdump.mem windows.info
...
Variable        Value

Kernel Base     0xf8006280e000
DTB             0x1ad000
...
Major/Minor     15.19041
NtMajorVersion  10
NtMinorVersion  0
```

So this is a Windows 10 (build 19041) image. We need a BitLocker plugin that works on Windows 10.

### Step 4 — Install Volatility 3 and the bitlocker plugin

Volatility 3 does not ship with a bitlocker plugin. The community plugin we need is `lorelyai/volatility3-bitlocker`, which validates candidate AES key schedules against the standard AES key-expansion:

```
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ pip3 install volatility3
...
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ git clone https://github.com/lorelyai/volatility3-bitlocker.git
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ sudo cp volatility3-bitlocker/bitlocker.py \
        /usr/local/lib/python3.11/dist-packages/volatility3/plugins/windows/bitlocker.py
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ vol windows.bitlocker.BitlockerFVEKScan --help
...
This is a Volatility 3 rewrite of the classic Volatility 2 BitLocker pool
scanner that detected AES keys by checking for valid expanded key schedules.
```

### Step 5 — Run the bitlocker plugin

```
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ vol -f memdump.mem windows.bitlocker.BitlockerFVEKScan \
        --tags FVEc Cngb None dFVE --dislocker
```

Abridged output:

```
Progress:  100.00  PDB scanning finished
Volatility 3 Framework 2.28.0

PoolOffset     PoolTag   Cipher    FVEK                                Tweak                              PoolSize

0x40d857c90    Cngb      AES-128   d40582190eb6f067691120bbbe55e511                                       672
0x40de7ece0    Cngb      AES-128   039e111586d5f9d974a571190474d097                                       672
0x40dfeab80    Cngb      AES-256   65b8064ec7acea96726aa18d294213176fd513a62a95c80720648f0590211364    672
...
0x8087865bead0 None      AES-128   4f79d4a00d5e9b25965b89581a6a599c    4109b89a973fd5c65ed75841404e7c39    1280
```

Two things to notice:

- **Most hits have `Cngb` pool tag and no Tweak value.** Those are not the FVEK — they are other CNG (Crypto Next Gen) AES keys Windows keeps in memory (TLS, DPAPI, EFS, etc.). Ignore them.
- **The last hit, `0x8087865bead0`, has the `None` pool tag, AES-128, and BOTH an FVEK and a Tweak.** That is the real key. On Windows 10 BitLocker stores the key material in un-tagged (None) kernel pool allocations, and a valid FVEK candidate for XTS must come with the matching tweak key.

The plugin also wrote a Dislocker-format `.fvek` file for that hit:

```
0x8087865bead0-Dislocker.fvek
```

66 bytes: 2-byte cipher id (`0x8005` = AES-128 XTS) ‖ 16-byte FVEK ‖ 16-byte TWEAK ‖ 32 zero bytes.

Our keys:

```
FVEK  = 4f79d4a00d5e9b25965b89581a6a599c
TWEAK = 4109b89a973fd5c65ed75841404e7c39
```

### Step 6 — Confirm the encryption method with `bdeinfo`

```
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ sudo apt install -y libbde-utils
...
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ bdeinfo bitlocker-2.dd
bdeinfo 20190102

BitLocker Drive Encryption information:
        Encryption method       : AES-XTS 128-bit
        Volume identifier       : dc6328f2-22b7-47ff-8d5b-cfcd63f010db
        Creation time           : Mar 10, 2025 02:41:57 UTC
        Description             : DESKTOP-73SUB54 New Volume 3/9/2025
        Number of key protectors: 2

Key protector 0:
        Type    : Password
Key protector 1:
        Type    : Recovery password

Unable to unlock volume.
```

The encryption is **AES-XTS 128-bit**, which matches the plugin output. Two key protectors — a (long, complex) password and a recovery password. We do not need either: we have the FVEK straight from RAM.

### Step 7 — Decrypt the volume

There are two ways from here. **Method A** is the easy one-liner on a normal Kali box. **Method B** is the path I actually took because the sandbox I was working in did not let me open `/dev/fuse`.

#### Method A — Dislocker (FUSE)

```
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ sudo apt install -y dislocker ntfs-3g
...
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ sudo mkdir -p /mnt/dislocker /mnt/ntfs
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ sudo dislocker -k 0x8087865bead0-Dislocker.fvek \
                    -V bitlocker-2.dd /mnt/dislocker
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ sudo mount -t ntfs-3g /mnt/dislocker/dislocker-file /mnt/ntfs
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ ls /mnt/ntfs/
'$Recycle.Bin'   'System Volume Information'   bootmgr   BOOTNXT   flag.txt   ...
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ cat /mnt/ntfs/flag.txt
picoCTF{B1tl0ck3r_dr1v3_d3crypt3d_9029ae5b}
```

#### Method B — Hand-rolled AES-XTS in Python

When FUSE is not available (sandbox, no root, etc.), a 60-line Python script implementing AES-XTS is enough.

**a. Find the FVE block and the boot-sectors backup offset**

```
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ python3 - <<'PY'
import struct
with open('bitlocker-2.dd','rb') as f:
    f.seek(0x2100000)  # primary FVE metadata block
    h = f.read(0x40)
print('encrypted_volume_size        =', struct.unpack('<Q', h[0x10:0x18])[0])
print('number_of_volume_header_secs =', struct.unpack('<I', h[0x1c:0x20])[0])
print('volume_header_offset         = 0x{:x}'.format(struct.unpack('<Q', h[0x38:0x40])[0]))
PY
encrypted_volume_size        = 208666624
number_of_volume_header_secs = 16
volume_header_offset         = 0x2110000
```

So the NTFS VBR backup is at `0x2110000` (16 sectors = 8 KiB), and the encrypted user data starts at `0x2120000`.

**b. Write the decryptor**

```
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ nano decrypt_bl.py
```

Paste (Ctrl+Shift+V), then Ctrl+O, Enter, Ctrl+X:

```python
#!/usr/bin/env python3
import struct
from Crypto.Cipher import AES

FVEK  = bytes.fromhex('4f79d4a00d5e9b25965b89581a6a599c')
TWEAK = bytes.fromhex('4109b89a973fd5c65ed75841404e7c39')
FVE_BLOCK = 0x02100000
IMG_SIZE  = 134217728

def gf_mul_x(val):
    # Multiply a 128-bit value by alpha in GF(2^128)
    # Reduction polynomial: x^128 + x^7 + x^2 + x + 1 (mask 0x87)
    # 'val' is a 16-byte little-endian byte string.
    carry = val[15] >> 7
    out = bytearray(16)
    out[0] = (val[0] << 1) & 0xff
    for i in range(1, 16):
        out[i] = ((val[i] << 1) | (val[i-1] >> 7)) & 0xff
    if carry:
        out[0] ^= 0x87
    return bytes(out)

def xts_decrypt(k1, k2, ct, sector_num):
    c2 = AES.new(k2, AES.MODE_ECB)
    c1 = AES.new(k1, AES.MODE_ECB)
    tweak = c2.encrypt(struct.pack('<QQ', sector_num, 0))
    out = bytearray()
    for i in range(0, len(ct), 16):
        block = bytearray(ct[i:i+16])
        x = bytes(a ^ b for a, b in zip(block, tweak))
        p = c1.decrypt(x)
        pt = bytes(a ^ b for a, b in zip(p, tweak))
        out.extend(pt)
        tweak = gf_mul_x(tweak)
    return bytes(out)

with open('bitlocker-2.dd', 'rb') as fi, open('decrypted.img', 'wb') as fo:
    # 1. Boot-sectors backup (16 sectors at 0x2110000) -> decrypted sectors 0..15
    fi.seek(0x2110000)
    boot = fi.read(8192)
    for i in range(16):
        fo.write(xts_decrypt(FVEK, TWEAK, boot[i*512:(i+1)*512],
                              0x2110000 // 512 + i))

    # 2. Encrypted user data starting at 0x2120000
    fi.seek(0x2120000)
    off = 0x2120000
    while True:
        chunk = fi.read(65536)
        if not chunk:
            break
        for i in range(0, len(chunk), 512):
            fo.write(xts_decrypt(FVEK, TWEAK, chunk[i:i+512],
                                 (off + i) // 512))
        off += len(chunk)

    # 3. Pad to original image size with zeros
    pos = fo.tell()
    if pos < IMG_SIZE:
        fo.write(b'\x00' * (IMG_SIZE - pos))
```

```
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ pip install pycryptodome
...
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ python3 decrypt_bl.py
Done in 59.4s
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ file decrypted.img
decrypted.img: DOS/MBR boot sector, code offset 0x52+2, OEM-ID "NTFS    ",
sectors/cluster 8, ..., sectors 262143, $MFT start cluster 16981, ...
```

The decrypted image is now a real NTFS volume.

### Step 8 — Read the flag

**Method A — mount with `ntfs-3g`**

```
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ sudo mount -t ntfs-3g decrypted.img /mnt/ntfs
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ cat /mnt/ntfs/flag.txt
picoCTF{B1tl0ck3r_dr1v3_d3crypt3d_9029ae5b}
```

**Method B — read it directly with the Sleuth Kit (no mount needed)**

```
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ sudo apt install -y sleuthkit
...
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ fls -r decrypted.img | grep -iE "flag|pico"
r/r 38-128-1: flag.txt
┌──(zham㉿kali)-[/tmp/bitlocker2]
└─$ icat decrypted.img 38
picoCTF{B1tl0ck3r_dr1v3_d3crypt3d_9029ae5b}
```

Got the flag.

---

## What Happened Internally (Timeline)

1. **Jacky's machine boots.** The TPM releases the VMK to Windows because the boot measurements match.
2. **Jacky's drive is unlocked.** Windows decrypts the FVEK with the VMK and keeps the FVEK in kernel pool memory (an un-tagged `None`-tag allocation) so it can encrypt reads and decrypt writes on the fly.
3. **An attacker captures the RAM.** They run something like `WinPmem`, `FTK Imager` or `Magnet RAM Capture` and get a copy of physical memory while the volume is still mounted.
4. **The attacker runs `windows.bitlocker.BitlockerFVEKScan`.** The plugin enumerates kernel pool allocations, looks for AES key schedules that match either the legacy `FVEc` tag (Windows 7) or the `None`/`Cngb` tags (Windows 10), validates the schedule with the standard AES key-expansion, and emits the candidate FVEK + TWEAK.
5. **The attacker feeds the FVEK to `dislocker`.** Dislocker parses the FVE metadata at the head of the disk image, uses the boot-sectors backup region to read the original NTFS VBR (decrypting sector-by-sector), then exposes the encrypted user data through a FUSE filesystem that decrypts reads on the fly.
6. **`ntfs-3g` mounts the decrypted stream** (or `fls` + `icat` read it directly). It walks the MFT and sees `flag.txt` in the root directory. Plaintext.

A neat detail: `dislocker` does **not** decrypt sectors 0..15 from offset 0 of the .dd. It reads them from the boot-sectors backup region inside the FVE metadata, where the **encrypted** copy of the original NTFS boot sectors was stashed during encryption. That is also why the `volume_header_offset` field of the FVE block header matters so much — it is the offset of the only place in the .dd that decrypts into something starting with `EB 52 90 4E 54 46 53`.

A second neat detail: AES-XTS ties the tweak to the **physical** sector offset (not the logical NTFS sector offset), so a single 512-byte block of data at offset `X` in the image is decrypted using tweak = `X / 512`, regardless of where that block sits in the volume's logical geometry. You can decrypt sectors one at a time without knowing the full NTFS layout.

A third neat detail: the GF(2^128) reduction polynomial in the tweak update is a common place to introduce a bug. In little-endian, the low byte (byte 0) is the LSB, so the `0x87` reduction mask has to XOR into byte 0, not byte 15. Mixing those up will produce a first block that decrypts correctly (the very first tweak is `AES_k2(sector_num)` straight from the cipher) and then garbage everywhere from block 2 onwards. Worth remembering when you write your own.

---

## Alternative Methods

**Method 1 — Mount via Dislocker + ntfs-3g (the one-liner)**

The version shown in Step 7 — `dislocker -k .fvek -V volume.dd /mnt/dislocker` followed by `mount -t ntfs-3g ...` — is the canonical path. Use it whenever you have a Kali box with `/dev/fuse` available.

**Method 2 — Use the Sleuth Kit instead of mounting**

If you do not want to mount the decrypted image (e.g. it complains about being "dirty"), `fls -r decrypted.img` and `icat decrypted.img <inode>` read the NTFS directly. Inode `38` is `flag.txt` here.

**Method 3 — `bdemount` from `libbde`**

If you prefer the libbde path, `bdemount -k FVEK:TWEAK volume.dd /mnt/bde` exposes the decrypted NTFS through FUSE the same way `dislocker` does. The FVEK and TWEAK come out of the `0x8087865bead0-Dislocker.fvek` file the volatility plugin wrote: 16 bytes from offset 2 and 16 bytes from offset 18, separated by a colon.

**Method 4 — Walk the `$MFT` manually in Python**

If you cannot mount the NTFS for any reason (the kernel module refuses to load, the MFT is "dirty", the image is smaller than a real volume, ...), just parse the `$MFT` by hand. In this challenge `flag.txt` is inode 38 and its data attribute is **resident** (lives inside the MFT entry itself), so the plaintext is in the MFT record:

```python
data = open('decrypted.img','rb').read()
off  = data.index(b'FILE')
while data[off:off+4] == b'FILE':
    rec = data[off:off+1024]
    if b'flag.txt' in rec:
        print(rec[rec.index(b'picoCTF'):rec.index(b'}')+1].decode())
        break
    off += 1024
```

That is the path I used as a sanity check after the AES-XTS decrypt.

**Method 5 — Use the recovery password (defeats the point of the challenge)**

If for some reason you had the 48-digit BitLocker recovery password instead of the RAM dump, you would crack the `$bitlocker$2$` / `$bitlocker$3$` hash out of the FVE metadata with `bitlocker2john` and `hashcat -m 22100`, and then `dislocker -V bitlocker-2.dd -p<recovery_password> /mnt/dislocker`. But you do not have the recovery password here — Jacky made sure it was 48 random digits — so this is theoretical.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify the `-FVE-FS-` BitLocker signature on the .dd |
| `vol` (Volatility 3) | Walk the Windows 10 kernel structures of the RAM dump |
| `lorelyai/volatility3-bitlocker` | Extract the FVEK + TWEAK from kernel pool memory |
| `bdeinfo` (libbde) | Confirm the encryption method (AES-XTS 128) and read the FVE metadata |
| `dislocker` | Decrypt the volume via FUSE and feed the NTFS to a real file system |
| `ntfs-3g` | Mount the decrypted NTFS volume and read `flag.txt` |
| `fls` / `icat` (Sleuth Kit) | Read the decrypted NTFS without mounting it |
| `python3` + `pycryptodome` | (Alternative) hand-rolled AES-XTS decryptor and `$MFT` walker |

---

## Key Takeaways

- **Cold drives are safe; warm drives are not.** BitLocker's security model assumes the attacker does not have access to a memory dump of a running, unlocked system. The moment they do, the FVEK is recoverable and the entire volume falls. This is why features like Secure Boot + TPM PCR measurements, "pre-boot authentication only" mode, and TPM + PIN exist.
- **The Windows 10 FVEK is un-tagged.** On Windows 10 BitLocker stores the key material in an un-tagged (`None` pool tag) kernel allocation. A valid candidate has BOTH a 16-byte FVEK and a 16-byte TWEAK. The other AES-128 hits with the `Cngb` tag and no tweak are unrelated CNG keys (TLS, DPAPI, EFS, etc.) and will produce garbage if you try to use them.
- **AES-XTS uses two 128-bit keys, not one 256-bit key.** The volatility plugin and the `.fvek` file format give you FVEK and TWEAK separately. For the actual cipher you concatenate them into a 32-byte key: `key = FVEK || TWEAK`, and the sector number is the tweak input.
- **Layout matters, not magic.** A BitLocker `.dd` does not start with the NTFS VBR. The BitLocker boot code lives at offset 0 and the encrypted NTFS VBR is stashed at `boot_sectors_backup` (here `0x2110000`). Decrypting sector 0 of the .dd gives you back the BitLocker boot code, not `NTFS    `. Decrypting the sector at `boot_sectors_backup / 512` is what produces `EB 52 90 4E 54 46 53`.
- **The GF(2^128) reduction is the classic bug.** In little-endian the low byte (byte 0) is the LSB, so the `0x87` reduction mask has to XOR into byte 0, not byte 15. A wrong-direction reduction will give a first block that decrypts correctly and then garbage from block 2 onwards — easy to miss for 30 minutes.
- **Dislocker or roll your own.** `dislocker -k .fvek -V volume.dd /mnt/dislocker` plus `mount -t ntfs-3g /mnt/dislocker/dislocker-file /mnt/decrypted` is the one-liner that gets you the flag on any normal Kali box. When you cannot use FUSE (sandbox, no root, etc.), a 60-line Python script implementing AES-XTS is enough.
- The flag wordplay reads `B1tl0ck3r dr1v3 d3crypt3d` → **"BitLocker drive decrypted"** — a fitting summary of the attack: pull the FVEK out of RAM, decrypt the whole drive, read the flag. With the usual leetspeak (`1` for `i`, `3` for `e`) and a random-looking suffix `9029ae5b` to make sure you actually did the work.
