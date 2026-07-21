# m00nwalk — picoCTF Writeup

**Challenge:** m00nwalk  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 250  
**Flag:** `picoCTF{beep_boop_im_in_space}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> Decode this message from the moon.

## Hints

> 1. How did pictures from the moon landing get sent back to Earth?
> 2. What is the CMU mascot?, that might help select a RX option

---

## Background Knowledge (Read This First!)

### What is SSTV?

SSTV stands for **Slow-Scan Television**. It is a method that was used to send pictures over radio, often by amateur (ham) radio operators. The pictures are sent line by line as audio tones. Each tiny audio frequency corresponds to a pixel's brightness or color value. When you decode the audio, the pixel values get reassembled into an image.

SSTV was the technology NASA used during the Apollo missions to send photographs from the moon back to Earth. That is exactly what hint 1 is hinting at. If you have ever seen the classic black and white moon landing photos on TV, those were transmitted using SSTV.

### Why a WAV file?

SSTV is just audio under the hood. Anyone with a software defined radio (SDR) or even an audio recording of a radio transmission can save the signal as a WAV file. That is what the challenge gives us. The flag is hidden inside the audio, encoded as an SSTV image.

### SSTV Modes (Scottie, Martin, Robot, etc.)

There are several SSTV modes. The two most common ones are:

- **Scottie S1** — used by the hint, sends a 320x256 color image
- **Martin M1** — also 320x256 color, slightly different timing

Each mode uses a different sequence of tones for the header, the color calibration, and the actual scan lines. The decoder has to know which mode was used, otherwise the image comes out garbled.

### The CMU Mascot Hint

Hint 2 says: "What is the CMU mascot?, that might help select a RX option". CMU is Carnegie Mellon University, and the mascot is **Scotty the Scottie dog**. This is a pun that points us to the **Scottie S1** mode. When you open an SSTV receiver like `qsstv` or use the command line decoder, you pick a "RX mode" (receive mode). The hint is telling you to pick **Scottie S1**.

---

## Solution — Step by Step

I solved this on Kali Linux running in VirtualBox. The WAV file was already in my Downloads folder, shared with Kali as a VirtualBox shared folder at `/media/sf_downloads`.

### Step 1 — Confirm the WAV File

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls -la message.wav
-rwxrwxrwx 1 root root 11066998 Jul 21 22:18 message.wav

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file message.wav
message.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16-bit, mono 48000 Hz
```

A 48 kHz mono WAV, about 11 MB. Roughly 115 seconds long, which is right in the ballpark for a Scottie S1 image (around 110 seconds per frame).

### Step 2 — Install a Command Line SSTV Decoder

I do not run `qsstv` (the GUI app) inside my headless test setup, so I went with a pure command line decoder. The `colaclanth/sstv` project on GitHub works well.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ pipx install git+https://github.com/colaclanth/sstv.git
  installed package sstv 0.1, installed using Python 3.11.9
  These apps are now globally available
    - sstv
done!

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sstv --version
sstv 0.1
```

If you do not have `pipx`, you can use `pip install --user` instead, or just `apt install qsstv` if you want a GUI.

### Step 3 — Decode the WAV File

The hint points to Scottie S1, but the `sstv` tool can also auto-detect. I let it auto-detect first and it picked up the signal correctly.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sstv -d message.wav -o message.png
[sstv] Searching for calibration header... Found.
[sstv] Found Scottie S1 signal
[sstv] Decoding image... Done
[sstv] Image saved to message.png
```

Auto-detect saw the Scottie S1 calibration header and decoded a 320x256 color image. The tool did not need me to specify the mode manually, but if it failed to auto-detect, I would have used:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sstv -d message.wav -o message.png --mode scottie-s1
```

### Step 4 — Open the Image

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file message.png
message.png: PNG image data, 320 x 256, 8-bit/color RGB, non-interlaced

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ xdg-open message.png
```

The image showed a folded tan and orange towel sitting on a dark surface, with a metal key ring, a yellow keychain, and what looks like a small white plastic part. The text was running **vertically** along the bottom of the image (rotated 90 degrees), so I rotated it to read it normally.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "from PIL import Image; im=Image.open('message.png'); im.rotate(-90, expand=True).save('message_rot.png')"

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ xdg-open message_rot.png
```

Now the text was readable. The flag was written in handwritten style across the orange portion of the towel:

```
CTF{beep_boop_im_in_space}
```

Since the platform is picoCTF, the full flag to submit is the standard picoCTF format:

```
picoCTF{beep_boop_im_in_space}
```

---

## Alternative Solve (Optional)

If you prefer a GUI workflow, you can use `qsstv` on Kali.

### Install qsstv

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo apt update

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo apt install qsstv
```

### Configure Audio and Mode

1. Open `qsstv`.
2. Go to **Options → Sound Input** and pick your sound device.
3. Go to **Mode** in the menu bar and select **Scottie S1**. This is the RX mode the hint told us about.
4. Click **Start** to begin receiving.

### Decode the File

Since `qsstv` listens from a live audio source, the easiest way to feed it a file is:

1. In `qsstv`, set the sound input to a **virtual audio sink** like `pulse` or `null`.
2. Open the WAV in another program that supports audio output (like Audacity or `paplay`).
3. Play the file while `qsstv` is listening.

Or, if `qsstv` supports a "file" source directly in your version, just point it at the WAV. In my experience the command line approach is more reliable on a headless VM, which is why I used `colaclanth/sstv`.

---

## What Happened Internally

1. **SSTV Encoding Side (the challenger)**: An image (a real photo of a towel and keychain) was fed into an SSTV encoder set to **Scottie S1** mode. The encoder converted each pixel into a specific audio frequency based on its brightness and color, line by line. The image was 320x256 pixels, so 256 scan lines were generated. Before the actual image data, the encoder added a special calibration header — a sequence of tones that identifies the mode and helps the receiver lock onto the signal.
2. **Transmission Side**: The audio stream was either transmitted over the air (in the real world) or just saved as a WAV file (in this challenge). Our WAV file is the recording of that audio.
3. **Decoding Side (me)**: I opened the WAV in an SSTV decoder. The decoder:
    - Scanned the audio for the calibration header
    - Identified it as Scottie S1
    - Measured the timing of each tone to recover the line scan data
    - Mapped each tone frequency back to a pixel value
    - Assembled all 256 lines into the final 320x256 PNG
4. **The Flag**: Once the image was rebuilt, the flag was visible as handwritten text on the orange part of the towel. I just rotated the image and read it.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Confirm the WAV format and properties |
| `sstv` (colaclanth) | Decode the SSTV audio signal into a PNG image |
| `pipx` | Install the `sstv` Python tool in an isolated environment |
| Python 3 + Pillow | Rotate the decoded image 90 degrees so the flag text was readable |
| `xdg-open` | Open the PNG in the default image viewer |
| `qsstv` (optional) | GUI alternative for the same decode |

---

## Key Takeaways

- **SSTV is a real, historical tech**: NASA actually used it for the Apollo missions. Knowing this gives you a fast path into any "audio file that hides a picture" challenge.
- **Always read every hint literally**: hint 2 was a pun on "Scotty" the CMU mascot, which directly named the SSTV mode. Reading the hints slowly is faster than brute forcing modes.
- **The flag was the literal contents of the image, not hidden inside the file**: there was no steganography layer, no LSB tricks, no metadata. Just decode the image and read it.
- **Wordplay decode of the flag**: `beep_boop` is classic robot sound, and `im_in_space` is exactly where the message was claimed to come from — the moon. So the whole flag is "I am a robot beeping from space", which fits the SSTV (radio transmission) theme perfectly.
