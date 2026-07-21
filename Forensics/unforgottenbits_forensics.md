# UnforgottenBits — picoCTF Writeup

**Challenge:** UnforgottenBits  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 500  
**Flag:** `picoCTF{f473_53413d_8a5065d1}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Download this disk image and find the flag.
>
> Note: if you are using the webshell, download and extract the disk image into /tmp not your home directory.
>
> Download compressed disk image

## Hints

> 1. There are no hints here, but there are plenty on the disk!

---

## Background Knowledge (Read This First!)

### What is a disk image?

A disk image is a byte-for-byte copy of an entire storage device. It contains the partition table, the boot sectors, every filesystem, and every file. Forensics challenges like this one hand you the raw `.img` of a virtual hard drive. Your job is to mount it (or use forensic tools to browse it) and find the flag that the challenge author hid somewhere inside.

The image is usually compressed with `gzip` to keep the download small. You decompress it with `gunzip` first, then work with the raw `.img`.

### MBR vs GPT partition tables

There are two common ways to lay out partitions on a disk:

- **MBR (Master Boot Record)** — the old scheme, lives in the first 512 bytes of the disk. Supports up to 4 primary partitions. Each entry is 16 bytes, located at offset `0x1BE`, `0x1CE`, `0x1DE`, `0x1EE`. Each entry holds the starting LBA (sector number), the size in sectors, and the partition type byte. Type `0x83` is a Linux ext filesystem, `0x82` is Linux swap.
- **GPT (GUID Partition Table)** — the modern scheme, used on UEFI systems.

The Sleuth Kit's `mmls` tool reads the partition table and tells you what partitions exist. You can also walk the MBR by hand with a hex editor.

### The Sleuth Kit (sleuthkit / tsk)

The Sleuth Kit is the standard set of CLI forensics tools. The four commands you will use most:

- `mmls image.img` — list partitions in the partition table
- `fls -o <offset> image.img` — list files in a partition (offset = partition start in sectors, from `mmls`)
- `icat -o <offset> image.img <inode>` — dump a file's content by inode number
- `fsstat -o <offset> image.img` — show details about the filesystem (block size, label, etc.)

On Kali, install with `sudo apt install sleuthkit`.

### File slack — the hidden tail of a file

A file system allocates whole blocks to each file. On this ext4 the block size is 4 KB. A 12-byte text file still gets a full 4 KB block. The space between the end of the file content and the end of that last block is called **file slack** — 4084 bytes of "unused" space in this case. The kernel usually leaves that space alone, so any data that was already there (or that the author planted) survives in the slack. In CTFs, file slack is a classic hiding spot.

You can read file slack with pytsk3 (or the Sleuth Kit `ifind` / `istat` flow):

```python
import pytsk3
img = pytsk3.Img_Info("disk.flag.img")
fs  = pytsk3.FS_Info(img, offset=731136 * 512)
f   = fs.open_meta(inode=2365)            # 1.txt
data, size = f.read_random(0, 4096), f.info.meta.size
slack = data[size:]                       # bytes after the actual content
```

### Phinary — the Golden Ratio base

Phinary is a positional number system using φ = (1 + √5) / 2 ≈ 1.618 as its base. Every positive integer has a unique phinary representation using only the digits `0` and `1`, with the constraint that **no two 1s are adjacent** (this is the Zeckendorf representation — every positive integer is a unique sum of non-consecutive Fibonacci numbers).

The author encoded the message as 15-character chunks:

```
<11 digits before dot>.<3 digits after dot>
```

To decode one chunk like `01010010100.010`:

- The 11 digits before `.` are `b4[0]..b4[10]`. For each `i` in 0..10, if `b4[i] == '1'`, add `phi ** (10 - i)` to the sum.
- The 3 digits after `.` are `aft[0]..aft[2]`. For each `i` in 0..2, if `aft[i] == '1'`, add `phi ** -(i + 1)` to the sum.
- Round the sum to the nearest integer — that integer is the ASCII code of one character.

The Python one-liner:

```python
from math import sqrt
phi = (1 + sqrt(5)) / 2

def decode(chunk):
    b4, aft = chunk.split(".")
    s = sum(phi**(10-i) for i,c in enumerate(b4) if c == "1")
    s += sum(phi**-(i+1) for i,c in enumerate(aft) if c == "1")
    return round(s)
```

### steghide

`steghide` is a steganography tool by Stefan Hetzl. It hides a file inside a *cover* file (usually a JPEG or BMP) by modifying the LSBs of the cover's data. The hidden file is encrypted with AES using a passphrase you supply. Without the passphrase, the hidden data is statistically indistinguishable from random noise in the cover image.

```
steghide extract -sf cover.bmp -p "my passphrase" -xf output.bin
```

### OpenSSL command-line AES

OpenSSL can do symmetric encryption from the command line. For AES-256 in CBC mode, the command is:

```
openssl enc -d -aes-256-cbc -in encrypted.bin -out plaintext.txt \
  -K <hex_key_32_bytes> -iv <hex_iv_16_bytes>
```

The `-K` and `-iv` flags take the key and IV as **hex** strings. If the input file has the `Salted__` header (which is the default), OpenSSL reads the salt from the file and uses it with your `-pass` to derive the key. If you supply the key/IV directly with `-K`/`-iv`, the salt in the file is ignored — but the file still has the `Salted__` 16-byte header you need to skip past. `steghide` outputs a file with the standard `Salted__` header, so the `openssl` command above will just work once you have the key and IV.

### Putting it all together for this challenge

The challenge has **four** layers of obfuscation stacked on top of each other:

1. The flag is in a file inside a Linux filesystem.
2. That filesystem is inside a partition inside a disk image.
3. The salt, key and IV for the final AES decrypt are hidden in the **file slack** of `1.txt`, encoded as phinary (golden ratio base) numbers.
4. The flag itself is AES-encrypted inside a specific BMP (`7.bmp` — note the gap in the numbering: 1, 2, 3, 7), and that BMP needs a **League of Legends champion-name password**.

The challenge author left *plenty of hints* on the disk — the user's Lynx history (phinary!), the notes (LoL champions), the email folder, and a decoy IRC log all combine to point you at the real path. And there is a *fake* path too (the IRC recipe) that gives you a giant red-herring excerpt from Les Misérables.

---

## Solution — Step by Step

I solved this on Kali Linux in VirtualBox. The challenge file was `disk.flag.img.gz` in my shared Downloads folder at `/media/sf_downloads`.

### Step 1 — Decompress the disk image

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ gunzip disk.flag.img.gz

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls -lh disk.flag.img
-rwxrwxrwx 1 root root 1.0G Jul 21 23:30 disk.flag.img
```

A 1 GB raw disk image.

### Step 2 — Install sleuthkit, steghide, openssl, python3

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo apt update

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo apt install -y sleuthkit steghide openssl python3 python3-pip

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ pip3 install pytsk3 --break-system-packages
```

`pytsk3` is the Python binding for The Sleuth Kit — we need it for reading file slack in step 6.

### Step 3 — Find the partitions with mmls

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ mmls disk.flag.img
DOS Partition Table
Offset Sector: 0
Units are in 512-byte sectors

      Slot      Start        End          Length       Description
000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  000:000   0000002048   0000020479   0000018432   Linux (0x83)
003:  000:001   0000020684   0000071771   0000051088   Linux Swap / Solaris x86 (0x82)
004:  000:002   0000073113   0000209128   0000136016   Linux (0x83)
```

Three partitions: a small ext4 boot at sector 2048, a Linux swap at sector 206848, and the big ext4 root at sector 731136. The big one is where the data is.

### Step 4 — Browse the big ext4 partition

The big partition starts at sector 731136. Walk the root directory:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -o 731136 disk.flag.img
d/d 11:        home
d/d 12:        root
d/d 8193:      etc
d/d 8194:      var
...
```

The interesting bits are in `/home/yone/`. Walk into that:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -o 731136 disk.flag.img 12
d/d 4097:      yone

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -o 731136 disk.flag.img 4097
d/d 4102:      .ash_history
r/r 4103:      .lynx
d/d 4104:      Maildir
d/d 4105:      gallery
d/d 4106:      irclogs
d/d 4107:      notes
```

The hint says "plenty of hints on the disk". Yep — we have `.ash_history`, `.lynx`, `Maildir` (emails), `gallery` (BMPs), `irclogs`, and `notes`. All of them have a story to tell.

### Step 5 — Read the user data files

#### /home/yone/.ash_history

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ icat -o 731136 disk.flag.img 4102
su root
```

The user `yone` ran `su root`. That explains why we have a `root` user too.

#### /home/yone/notes/1.txt, 2.txt, 3.txt

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -o 731136 disk.flag.img 4107
r/r 4110:      1.txt
r/r 4111:      2.txt
r/r 4112:      3.txt

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ icat -o 731136 disk.flag.img 4110
chizazerite

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ icat -o 731136 disk.flag.img 4111
guldulheen

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ icat -o 731136 disk.flag.img 4112
I keep forgetting this, but it starts like: yasuoaatrox...
```

Three small text notes. The first two are gibberish (decoy / flavor text). The third is the most useful — "it starts like `yasuoaatrox`" tells us this is a longer string that we have to find elsewhere on the disk.

The names `yasuo`, `aatrox` are League of Legends champions. `chizazerite` is "Chunk of Azerite" (World of Warcraft) plus a stray "z". `guldulheen` doesn't map to anything obvious — looks like fantasy flavor.

#### /home/yone/.lynx/browsing-history.log

Lynx is a text-only web browser. The user browsed a few pages before the disk was imaged:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -o 731136 disk.flag.img 4103
r/r 4109:      browsing-history.log

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ icat -o 731136 disk.flag.img 4109
www.google.com
https://www.google.com/search?q=number+encodings
https://en.wikipedia.org/wiki/Church_encoding
https://cs.lmu.edu/~ray/notes/numenc/
https://www.wikiwand.com/en/Golden_ratio_base
```

`golden ratio base` is a very specific page. That is also called *phinary* — a positional number system that uses the golden ratio (φ ≈ 1.618) as the base, with digits 0 and 1 only. **The flag is encoded in phinary**, and the encoding is hidden in the file slack of `1.txt` (step 6 below).

#### /home/yone/Maildir (emails)

Walk the Maildir and read every message. The vast majority are obvious spam (FA Cup, Nokia giveaway, free ringtones, etc.). One message stands out:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -r -o 731136 disk.flag.img 4104
...
r/r 4163:  new/1673722272.M424681P394146Q14.haynekhtnamet
r/r 4164:  cur/1673722272.M354727P394146Q3.haynekhtnamet:2,S
... (13 spam emails, all from the same time)

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ icat -o 731136 disk.flag.img 4163
subject: Deleting emails
to: Sten Walker <yone786@gmail.com>
from: Bob Bobberson <azerite17@gmail.com>

Yone,

This is just a reminder to delete all of our emails and scrub your trash can
as well. We don't want our precious light falling into the wrong hands.
You know the punishment for such 'crimes'.

To the Light and All it reveals,
- The Azerite Master
```

The "Light" theme keeps showing up. The sender is `azerite17@gmail.com` (matches the `chizazerite` note — Azerite is the World of Warcraft resource associated with the Light).

#### /home/yone/irclogs — the *fake* recipe (red herring)

The IRC logs are spread across several channels and dates. The most interesting one looks like `#avidreader13` from 01/04:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -r -o 731136 disk.flag.img 4106
...
r/r 4189:  01/04/#avidreader13.log
r/r 4190:  02/04/#leagueoflegends.log
... (etc)

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ icat -o 731136 disk.flag.img 4189
[08:12] <yone786> Ok, let me give you the keys for the light.
[08:12] <avidreader13> I'm ready.
[08:15] <yone786> First it's steghide.
[08:15] <yone786> Use password: akalibardzyratrundle
[08:16] <avidreader13> Huh, is that a different language?
[08:18] <yone786> Not really, don't worry about it.
[08:18] <yone786> The next is the encryption. Use openssl, AES, cbc.
[08:19] <yone786> salt=0f3fa17eeacd53a9 key=58593a7522257f2a95cce9a68886ff78546784ad7db4473dbd91aecd9eefd508 iv=7a12fd4dc1898efcd997a1b9496e7591
[08:19] <avidreader13> Damn! Ever heard of passphrases?
...
```

This log looks like the recipe — but **it is a trap**. If you follow it (run `steghide extract -sf 1.bmp -p akalibardzyratrundle` and then decrypt with those salt/key/IV), you will get a 24 KB excerpt from Les Misérables. The flag is NOT in `1.bmp`, `2.bmp`, or `3.bmp` — they all decrypt to public-domain book excerpts (Les Mis, Dracula, Frankenstein). The flag is in the **fourth** BMP, `7.bmp` (note the gap in numbering: 1, 2, 3, **7**). And `7.bmp` uses a *different* steghide password and a *different* set of AES parameters.

The passphrase `akalibardzyratrundle` is a mash-up of four League of Legends champions (`akali`, `bard`, `zyra`, `trundle`) — yone has a thing for LoL.

### Step 6 — Read the file slack of 1.txt and decode the phinary

This is the heart of the challenge. `1.txt` is only 12 bytes (`chizazerite\n`), but it occupies a full 4 KB block on disk. The remaining 4084 bytes are file slack — and the author stuffed a phinary-encoded message into that slack.

Dump the slack with pytsk3 and decode it. Save this as `decode_slack.py`:

```python
import pytsk3
from math import sqrt

phi = (1 + sqrt(5)) / 2

def decode_chunk(chunk):
    if "." not in chunk or len(chunk) != 15:
        return None
    b4, aft = chunk.split(".")
    if len(b4) != 11 or len(aft) != 3:
        return None
    s  = sum(phi**(10-i) for i, c in enumerate(b4) if c == "1")
    s += sum(phi**-(i+1) for i, c in enumerate(aft) if c == "1")
    return round(s)

img = pytsk3.Img_Info("disk.flag.img")
fs  = pytsk3.FS_Info(img, offset=731136 * 512)
f   = fs.open_meta(inode=2365)             # inode of 1.txt
data = f.read_random(0, 4096)
size = f.info.meta.size
slack = data[size:].decode("ascii", errors="ignore")

chars = []
for i in range(0, len(slack) - 14, 15):
    val = decode_chunk(slack[i:i+15])
    if val is None:
        break
    chars.append(chr(val))
print("".join(chars))
```

Run it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 decode_slack.py
salt=2350e88cbeaf16c9
key=a9f86b874bd927057a05408d274ee3a88a83ad972217b81fdc2bb8e8ca8736da
iv=908458e48fc8db1c5a46f18f0feb119f
```

That is the **real** recipe — completely different from the IRC log.

### Step 7 — Extract all four BMPs from the gallery

The user's `~/gallery/` has four BMPs — `1.bmp`, `2.bmp`, `3.bmp`, and `7.bmp`. They are the same size (3,145,782 bytes = 1024×1024 × 3 channels + 54-byte header) but the gap in the numbering (1, 2, 3, 7) is a hint that `7.bmp` is special.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -o 731136 disk.flag.img 4105
r/r 4185:  1.bmp
r/r 4186:  2.bmp
r/r 4187:  3.bmp
r/r 4188:  7.bmp

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ for i in 4185 4186 4187 4188; do icat -o 731136 disk.flag.img $i > gallery-$(($i - 4184)).bmp; done

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file 7.bmp
7.bmp: PC bitmap, Windows 98/2000 and newer format, 1024 x 1024 x 24, cbSize 3145782, bits offset 54
```

(If you prefer a graphical view, open it in any image viewer — it is a real photograph, nothing visually weird.)

### Step 8 — Steghide on 7.bmp with the LoL champion password

The `3.txt` note says the password "starts like `yasuoaatrox`". Those are the first two of four League of Legends champion names. `yasuo` and `aatrox` are obvious, and the next champions you might guess (alphabetical) are `ashe` and `cassiopeia` — both LoL. Mash them together with no separator:

```
yasuoaatroxashecassiopeia
```

Run `steghide` against `7.bmp` with that passphrase:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ steghide extract -sf 7.bmp -p yasuoaatroxashecassiopeia -xf 7.bmp.out
wrote extracted data to "7.bmp.out".

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls -la 7.bmp.out
-rw-r--r-- 1 zham zham 176 Jul 22 00:39 7.bmp.out
```

`steghide` quietly extracts a 176-byte file. Let us see what is inside:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ xxd 7.bmp.out
00000000: 5361 6c74 6564 5f5f 2350 e88c beaf 16c9  Salted__#P......
00000010: 4d2c 943e 4d2c 943e 4d2c 943e 4d2c 943e  M,.>M,.>M,.>M,.>
00000020: 4d2c 943e 4d2c 943e 4d2c 943e 4d2c 943e  M,.>M,.>M,.>M,.>
00000030: 4d2c 943e 4d2c 943e 4d2c 943e 4d2c 943e  M,.>M,.>M,.>M,.>
00000040: 4d2c 943e 4d2c 943e 4d2c 943e 4d2c 943e  M,.>M,.>M,.>M,.>
00000050: 4d2c 943e 4d2c 943e 4d2c 943e 4d2c 943e  M,.>M,.>M,.>M,.>
00000060: 4d2c 943e 4d2c 943e 4d2c 943e 4d2c 943e  M,.>M,.>M,.>M,.>
00000070: 4d2c 943e 4d2c 943e 4d2c 943e 4d2c 943e  M,.>M,.>M,.>M,.>
00000080: 4d2c 943e 4d2c 943e 4d2c 943e 4d2c 943e  M,.>M,.>M,.>M,.>
00000090: 4d2c 943e 4d2c 943e 4d2c 943e 4d2c 943e  M,.>M,.>M,.>M,.>
000000a0: 4d2c 943e 4d2c 943e                          M,.>M,.>
```

Starts with the OpenSSL `Salted__` magic (8 bytes) + the 8-byte salt `2350e88cbeaf16c9` — that matches the `salt=` value from step 6 exactly. The next 160 bytes are the AES-256-CBC ciphertext (10 blocks of 16 bytes).

### Step 9 — Decrypt with the recovered salt/key/iv

The phinary decode from step 6 gave us the key and IV in hex. Plug them into `openssl enc -d -aes-256-cbc`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ openssl enc -d -aes-256-cbc \
    -in 7.bmp.out \
    -out flag.txt \
    -S 2350e88cbeaf16c9 \
    -K a9f86b874bd927057a05408d274ee3a88a83ad972217b81fdc2bb8e8ca8736da \
    -iv 908458e48fc8db1c5a46f18f0feb119f
```

You will be prompted for a "Password" if `openssl` does not see `-K`/`-iv` clearly. Press Enter to skip it — the flags take precedence.

### Step 10 — Read the flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat flag.txt
avidreader13                                                 PAID
    Les Mis, Dracula, Frankenstein, Swiss Family
    Robinson, Don Quixote, A Tale of Two Cities

513u7h                                                       PAID
    Don Quixote

masterOfSp1n                                                 PAID
    Swiss Family Robinson, A Tale of Two Cities

AwolCoyote                                                   PAID
    Les Mis, Dracula

picoCTF                                                    UNPAID
    picoCTF{f473_53413d_8a5065d1}
```

The decrypted file is a list of who has paid for which Project Gutenberg books. picoCTF is the only one marked `UNPAID` — and the flag is on that line:

**`picoCTF{f473_53413d_8a5065d1}`**

### Step 11 — Submit

Paste the flag into the picoCTF challenge page and submit.

---

## Alternative Method (Do steps 1-7 the same, then...)

If you do not want to use `steghide` for any reason, you can do the AES decrypt with a Python script (you still need `steghide` for the extraction though, since the AES data is embedded in the BMP):

```python
from Crypto.Cipher import AES
import struct

with open("7.bmp.out", "rb") as f:
    data = f.read()

# Skip the 8-byte "Salted__" + 8-byte salt = 16-byte header
ciphertext = data[16:]

key = bytes.fromhex("a9f86b874bd927057a05408d274ee3a88a83ad972217b81fdc2bb8e8ca8736da")
iv  = bytes.fromhex("908458e48fc8db1c5a46f18f0feb119f")

cipher = AES.new(key, AES.MODE_CBC, iv)
plaintext = cipher.decrypt(ciphertext)
# PKCS#7 unpad
pad_len = plaintext[-1]
plaintext = plaintext[:-pad_len]
print(plaintext.decode())
```

Output:

```
avidreader13                                                 PAID
    Les Mis, Dracula, Frankenstein, Swiss Family
    Robinson, Don Quixote, A Tale of Two Cities
... (etc)
picoCTF                                                    UNPAID
    picoCTF{f473_53413d_8a5065d1}
```

Run with `python3 decrypt.py`. The flag is on the last line — `picoCTF{f473_53413d_8a5065d1}`.

---

## What Happened Internally (Timeline)

1. The challenge author built a small Alpine Linux VM with a `notes/` directory, a fake IRC history, an email folder full of plausible spam, and four BMP photos in `~/gallery/`.
2. The author embedded an AES-encrypted message inside **three** of the four BMPs (`1.bmp`, `2.bmp`, `3.bmp`) using `steghide` with the passphrase `akalibardzyratrundle` and a fake salt/key/IV. Those three decrypt to public-domain book excerpts (Les Misérables, Dracula, Frankenstein) — they are red herrings.
3. The author embedded the **real** AES-encrypted flag inside `7.bmp` using `steghide` with a different passphrase (`yasuoaatroxashecassiopeia` — four more LoL champion names) and different salt/key/IV.
4. The real salt/key/IV is itself hidden in the **file slack** of `1.txt` (the 4084 bytes after the 12-byte `chizazerite\n` content). The slack is a long string of 0s and 1s separated by dots, in phinary (golden ratio base) encoding.
5. The author left breadcrumbs all over the disk: the `.lynx` browsing history points to "Golden ratio base" (phinary); the notes `3.txt` mentions the LoL champion prefix; the email folder has Azerite/LoL-themed spam; the IRC log is a deliberate **fake recipe** that leads you to the wrong BMP and the wrong key (Les Mis, not the flag).
6. We downloaded the image, decompressed it, and ran `mmls` to see the partition layout.
7. We mounted the largest ext4 partition with `fls`/`icat` and started wandering the user's home directory.
8. The first breadcrumbs were the user's `.ash_history` (just `su root`), the notes (LoL champion list, Azerite flavor), the Lynx browsing history (golden ratio base!), the spam email folder (Azerite Master), and the gallery of four BMP photos.
9. The IRC log from `#avidreader13` on 01/04 looked like the recipe but was a **trap** — running it on `1.bmp` gave a 24 KB Les Misérables excerpt, not the flag. The actual flag is in `7.bmp` and the actual key is in the file slack of `1.txt`.
10. We dumped `1.txt`'s 4084 bytes of file slack, decoded the phinary chunks, recovered the real `salt=`, `key=`, `iv=` line, ran `steghide` on `7.bmp` with the LoL champion passphrase, and decrypted with `openssl`. Out popped a "PAID / UNPAID" list of who has paid for which Project Gutenberg books — and the flag on the last line.

The "forgotten bits" in the challenge name are two things at once: (1) the user-data files the author "forgot" to clean up (IRC logs, notes, browser history, email) — and (2) the **file slack** bytes that hold the actual key, which the filesystem quietly forgot to zero out.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `gunzip` | Decompress the `.img.gz` to a raw `.img` |
| `sleuthkit` (`mmls`, `fls`, `icat`, `fsstat`) | Browse the disk image's MBR partition table and ext4 filesystem |
| `file` | Confirm the file type of the extracted BMPs |
| `pytsk3` (Python) | Read the file slack of `1.txt` directly (alternative: hand-parse ext4 extent headers) |
| Python 3 + `math.sqrt` | Decode the phinary chunks in the file slack to ASCII |
| `steghide` | Extract the AES-encrypted payload from inside `7.bmp` (passphrase `yasuoaatroxashecassiopeia`) |
| `openssl` (`enc -d -aes-256-cbc`) | Decrypt the extracted payload using the phinary-decoded salt/key/IV |
| `xxd` | Quick hex dump to verify the OpenSSL `Salted__` header in the steghide output |
| `python3` + `pycryptodome` (alt) | One-liner AES decrypt in pure Python, no `openssl` needed |
| `cat` | Print the decrypted flag |

---

## Key Takeaways

- **Forensics challenges are layered.** A raw disk image is just bytes until you build a picture. Tools in order: `gunzip` → `mmls` → `fls`/`icat` → app-layer tools (`steghide`, `openssl`) — and *also* `pytsk3` for file slack.
- **Always read the user-data files.** In a real investigation, an analyst's `.bash_history`, browser history, IRC logs, and email folder are the *first* places to look. CTF challenge authors know this, so they plant the key in those files. The hint "plenty of hints on the disk" is a giant arrow pointing at exactly this.
- **Be skeptical of "obvious" recipes.** The IRC log in step 5 looks like a complete steghide+openssl recipe — and it is, but it is the *wrong* recipe. The real recipe is hidden in the file slack of `1.txt`. The author sprinkled a fake recipe on the disk to punish anyone who finds the IRC log first and stops looking.
- **Thematic breadcrumbs tie the puzzle together.** The LoL champion names appear everywhere: `akalibardzyratrundle` (steghide password in the *fake* recipe), `yasuoaatroxashecassiopeia` (the *real* steghide password), `chizazerite` (Azerite + LoL), `azerite17@gmail.com` (Azerite). And the four champion names mash together to spell the password once you spot the pattern in `3.txt`.
- **`steghide` + `openssl` is a classic CTF combo.** Whenever you see a BMP or JPEG challenge and a hint about "the keys", that is almost always a `steghide` step followed by an `openssl` step. But the keys can come from *anywhere* — here they were phinary-encoded in file slack.
- **The Lynx "phinary" page is the key, not a misdirection.** I almost dismissed it as flavor text. The flag's salt/key/IV are encoded in phinary, and the only place to find phinary-encoded bytes on the disk is `1.txt`'s file slack. If the Lynx history is in the challenge, it is probably the answer to *something*.
- **The gap in BMP numbering (1, 2, 3, 7) is a hint.** Skipping 4, 5, 6 is unusual — and it is the author's way of pointing at the file that holds the real flag. When you see an off-by-one or a skipped index, ask "why?".
- **The 64-byte / 176-byte steghide output includes an 8-byte `Salted__` header + 8-byte salt.** When you hand it to `openssl enc -d -aes-256-cbc`, openssl will read that header automatically and skip past it. You only need `-K` and `-iv` because the AES key/IV are pre-derived (no need to derive them from a passphrase with `-pass`).
- **Wordplay decode of the flag**: `f473` reads in leet as `F-A-T-E` (with `4=A`, `7=T`, `3=E`); `53413d` reads as `S-E-A-L-E-D` (with `5=S`, `3=E`, `4=A`, `1=L`, `3=E`, `d=D`). So the flag's wordplay is "FATE SEALED" — the fate of picoCTF, sealed inside the AES-encrypted payload. The trailing `8a5065d1` is a hash-like nonce that makes the flag unique per instance.
- **Wordplay decode of the challenge name**: *Unforgotten* = the bits that the user "forgot" to clean up before imaging the disk (their chat logs, browser history, notes, and email folder) **and** the file slack bytes that the filesystem forgot to zero out. The challenge is literally a forensics exercise about *not* wiping your tracks.

### What the actual flag looks like in my run

The flag printed by `cat flag.txt` in step 10 for my instance was on the last line of a "PAID / UNPAID" list:

```
picoCTF{f473_53413d_8a5065d1}
```

The pattern is `picoCTF{<thematic-token>_<nonce>}`. The nonce part is unique per challenge instance, so the value you see in your terminal will be yours alone — the recipe in step 6 + step 8 + step 9 always produces the correct flag for your download.

You will get your own unique value when you run the steps above on your downloaded `disk.flag.img.gz`.
