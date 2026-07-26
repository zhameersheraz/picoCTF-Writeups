# m00nwalk2 — picoCTF Writeup

**Challenge:** m00nwalk2  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 300  
**Flag:** `picoCTF{the_answer_lies_hidden_in_plain_sight}`  
**Platform:** picoCTF 2019  
**Author:** Joon  
**Writeup by:** zham

---

## Description

> Revisit the last transmission. We think this transmission contains a hidden message. There are also some clues clue1, clue2, clue3.

**Attachments:** `message.wav` (the original m00nwalk transmission, with something hidden in the audio) + `clue1.wav`, `clue2.wav`, `clue3.wav` (three SSTV images that explain how to extract the hidden thing)

---

## Hints

> 1. Use the clues to extract the another flag from the wav file

---

## Background Knowledge (Read This First!)

### SSTV, again

Like `m00nwalk`, all four `.wav` files are Slow-Scan Television (SSTV) audio transmissions. Decoding them with `sstv -d file.wav -o file.png` (from the [colaclanth/sstv](https://github.com/colaclanth/sstv) Python tool) or any GUI SSTV decoder (QSSTV, RX-SSTV) turns them into images. Each wav uses a different SSTV mode:

| File           | SSTV mode  |
|----------------|------------|
| `clue1.wav`    | Martin 1   |
| `clue2.wav`    | Scottie 2  |
| `clue3.wav`    | Martin 2   |
| `message.wav`  | Scottie 1  |

### Audio steganography

The 3 clue images, when decoded and read together, point you to:
1. A password (`hidden_stegosaurus`)
2. A hint that there's something hidden in the audio ("the quieter you are, the more you can hear")
3. A specific steganography tool / website (Alan Eliasen's "Future Boy" stegano decoder, which wraps `steghide`)

`steghide` is a classic steganography tool that hides files inside WAV, JPEG, BMP, AU formats. It uses a passphrase to extract — without the right passphrase, the embedded file is unrecoverable. The Future Boy website at `https://futureboy.us/stegano/decinput.html` is a web frontend that runs `steghide` for you in the browser.

---

## Solution — Step by Step

### Step 1 — Decode the 4 wav files as SSTV

```
$ sstv -d clue1.wav  -o clue1.png
$ sstv -d clue2.wav  -o clue2.png
$ sstv -d clue3.wav  -o clue3.png
$ sstv -d message.wav -o message.png
```

You get 4 PNG images. `message.png` is the same one from `m00nwalk` (`picoCTF{beep_boop_im_in_space}`). The three clue images contain text:

- **clue1.png**: shows the words "password is `hidden_stegosaurus`"
- **clue2.png**: shows the words "the quieter you are the more you can hear"
- **clue3.png**: shows the words "Alan Eliasen the Future Boy"

### Step 2 — Combine the clues

The three clues tell you to:

1. Use the password **`hidden_stegosaurus`** to unlock a hidden file
2. The hidden file is *inside* the audio of `message.wav` (not in the decoded image)
3. The tool to extract it is on Alan Eliasen's **Future Boy** stegano website, which uses `steghide` under the hood

### Step 3 — Extract the hidden file

**Via the Future Boy web tool (easiest):**
1. Go to `https://futureboy.us/stegano/decinput.html`
2. Upload `message.wav`
3. Enter the passphrase `hidden_stegosaurus`
4. Click "Decrypt"
5. The output file `stegano_payload.<id>.txt` contains the flag

**Via the `steghide` CLI:**
```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ steghide extract -sf message.wav -p hidden_stegosaurus -xf flag.txt
wrote extracted data to "flag.txt".

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat flag.txt
picoCTF{the_answer_lies_hidden_in_plain_sight}
```

### Step 4 — Submit

```
picoCTF{the_answer_lies_hidden_in_plain_sight}
```

---

## What Happened Internally (Timeline)

1. The challenge author started from the `m00nwalk` setup: an SSTV transmission that decodes to a "beep boop im in space" image, with the obvious flag `picoCTF{beep_boop_im_in_space}` in the rendered picture.
2. The author then **embedded a second flag** *inside the raw audio bytes* of the same transmission, using `steghide` with the passphrase `hidden_stegosaurus`. The image you see after SSTV decoding is identical to the m00nwalk one — the second flag is invisible to the eye.
3. The three clue images are part of a treasure hunt: clue1 gives you the password, clue2 hints that the secret is in the audio (not the picture), clue3 names the tool to use.
4. You, the solver, decode all 4 wavs, read the three clues, use the password to extract the steghide payload from `message.wav`, and recover the second flag.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `sstv` (Python) | Decode the 4 wav files into PNG images | Easy |
| `steghide` (CLI) | Extract the password-protected hidden file from `message.wav` | Easy |
| `futureboy.us/stegano/decinput.html` | Web frontend for steghide if you don't have it installed | Easy |
| `cat` | Print the extracted flag | Easy |

---

## Key Takeaways

- **SSTV is only the first layer.** Just like m00nwalk, the wav files decode to images. But unlike m00nwalk, the *real* flag is not in the image — it's hidden in the audio bytes using a completely different steganography tool.
- **`steghide` is the de facto audio-stego tool.** It supports WAV/JPEG/BMP/AU, uses a passphrase, and is the answer to most "hidden in audio" forensics challenges in CTF series. If a forensics challenge gives you a wav and a clue about "the quieter you are the more you can hear", reach for steghide first.
- **The three clues are a treasure hunt, not a hint.** Each one is necessary: clue1 = the password, clue2 = where to look, clue3 = which tool. If you only had clue1 (the password), you wouldn't know to use steghide or to look in `message.wav`. The challenge is teaching you to read clues in combination.
- **"The answer lies hidden in plain sight"** is the meta-joke. The flag is literally sitting in the audio of the file you just decoded — the same audio, already on your disk. You spent 5 minutes SSTV-decoding it and *still* missed it the first time.
- **Always check multiple stego layers.** The m00nwalk → m00nwalk2 progression is the classic CTF author move: hide a second flag in the *same* artifact using a different tool. If a forensics challenge only has one layer of stego, you're not done. Look at the raw audio, the EXIF, the file tail, the file slack space — anywhere data could hide.
