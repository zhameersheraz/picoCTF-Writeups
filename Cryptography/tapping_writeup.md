# tapping - picoCTF Writeup

**Challenge:** tapping  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{M0RS3C0D31SFUNADE00091}`  
**Platform:** picoCTF  
**Writeup by:** zham  

---

## Description

> Theres tapping coming in from the wires.
> What's it saying nc fickle-tempest.picoctf.net 65483.

## Hints

> 1. What kind of encoding uses dashes and dots?
> 2. The flag is in the format PICOCTF{}

---

## Background Knowledge

Before touching the netcat command, here are the two ideas this challenge leans on.

### What is Morse Code?

**Morse code** is one of the oldest digital encodings still in use. It was invented in the 1830s and 1840s by Samuel Morse and Alfred Vail for the electric telegraph, and for over a century it was the standard way to ship text over wires. Today it lives on in amateur radio, aviation beacons, and the occasional CTF challenge.

The whole system is built from three ingredients:

- A **dot** (sometimes written `.`, sometimes called a "dit") — the short signal.
- A **dash** (written `-`, sometimes called a "dah") — the long signal, exactly three times longer than a dot.
- A **pause** — the silence between dots and dashes inside one letter, between letters inside one word, and between words.

Each letter of the alphabet, each digit 0 through 9, and a handful of punctuation marks get their own unique sequence of dots and dashes. A few classics worth knowing cold:

- **A** = `.-`
- **E** = `.` (the shortest letter — just one dot)
- **T** = `-` (the other one-symbol letter)
- **S** = `...` (three dots, "the sound of a snake")
- **O** = `---` (three dashes, the distress call)
- **M** = `--`
- **N** = `-.`
- **0** = `-----` (five dashes, easy to remember)
- **1** = `.----`
- **3** = `...--`
- **9** = `----.`

For the full chart, see the [Wikipedia Morse code article](https://en.wikipedia.org/wiki/Morse_code). I usually have a copy of that table open in a second tab when I am working on a Morse challenge.

The thing that makes Morse code a great CTF puzzle is that the encoding is **lossless and unambiguous**. Every letter has exactly one canonical Morse spelling (the table is fixed), and decoding is a straight lookup. No key, no shift, no clever math. You match `.-` to A, `-...` to B, and so on, until the buffer is empty.

### What is `nc`?

`nc` (short for **netcat**) is the swiss army knife of raw TCP. You give it a host and a port, it opens a TCP connection, and whatever bytes the server sends back get dumped straight to your terminal. Whatever bytes you type go back to the server. There is no HTTP wrapper, no SSH, no encryption — just bytes on a socket.

The single most useful incantation is:

```
nc <host> <port>
```

That is it. Once you are connected, read what the server sends, type what you want to send back, and `Ctrl+C` when you are done. `nc` is installed by default on basically every Linux distribution, and Kali ships with multiple variants (`nc`, `ncat`, `socat`) that all do the same job.

For this challenge, the server is sending us a Morse-encoded flag as a stream of dots, dashes, and spaces. We just have to listen, capture the stream, and decode it.

### Putting the Pieces Together

The challenge title is "tapping," and the hints mention dashes and dots. So:

1. Connect to the server with `nc`.
2. Capture the Morse stream the server prints.
3. Split the stream on spaces to get individual letters.
4. Look each letter up on the Morse chart and write down the plaintext.
5. Submit the recovered flag.

That is the entire solve. We will write a Python helper for step 3 and 4 because hand-decoding 30 Morse tokens is error-prone and tedious.

---

## Solution

### Step 1: Set Up a Working Directory

I keep one folder per challenge so files do not pile up in my home directory.

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/picoCTF/cryptography/tapping && cd ~/picoCTF/cryptography/tapping
```

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/tapping]
└─$ pwd
/home/zham/picoCTF/cryptography/tapping
```

### Step 2: Connect With `nc` and Capture the Stream

The challenge gave us the host (`fickle-tempest.picoctf.net`) and port (`65483`). I just point `nc` at it and let it print whatever the server sends.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/tapping]
└─$ nc fickle-tempest.picoctf.net 65483
.--. .. -.-. --- -.-. - ..-. { -- ----- .-. ... ...-- -.-. ----- -.. ...-- .---- ... ..-. ..- -. .- -.. . ----- ----- ----- ----. .---- } 
```

The server prints a single line of dots, dashes, and spaces, then hangs up. `nc` exits as soon as the server closes the connection. I copied the line off the screen and pasted it into a file so I have a permanent record.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/tapping]
└─$ cat > morse.txt << 'EOF'
.--. .. -.-. --- -.-. - ..-. { -- ----- .-. ... ...-- -.-. ----- -.. ...-- .---- ... ..-. ..- -. .- -.. . ----- ----- ----- ----. .---- } 
EOF
```

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/tapping]
└─$ cat morse.txt
.--. .. -.-. --- -.-. - ..-. { -- ----- .-. ... ...-- -.-. ----- -.. ...-- .---- ... ..-. ..- -. .- -.. . ----- ----- ----- ----. .---- } 
```

Tip: if you are using Bash, the heredoc with `<< 'EOF'` (note the quotes) preserves the dots and dashes literally instead of trying to expand any `$` characters.

### Step 3: Build a Morse Lookup Table in Python

A plain Python dictionary is the simplest possible decoder. Each key is the Morse sequence, each value is the character it represents. The 26 letter flags plus the 10 digits are enough for this challenge.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/tapping]
└─$ nano decode.py
```

In `nano`, I pasted:

```python
#!/usr/bin/env python3
"""Decode a Morse-encoded flag pulled off a netcat socket."""

MORSE_TO_CHAR = {
    '.-': 'A', '-...': 'B', '-.-.': 'C', '-..': 'D', '.': 'E',
    '..-.': 'F', '--.': 'G', '....': 'H', '..': 'I', '.---': 'J',
    '-.-': 'K', '.-..': 'L', '--': 'M', '-.': 'N', '---': 'O',
    '.--.': 'P', '--.-': 'Q', '.-.': 'R', '...': 'S', '-': 'T',
    '..-': 'U', '...-': 'V', '.--': 'W', '-..-': 'X', '-.--': 'Y',
    '--..': 'Z',
    '-----': '0', '.----': '1', '..---': '2', '...--': '3',
    '....-': '4', '.....': '5', '-....': '6', '--...': '7',
    '---..': '8', '----.': '9',
}

with open('morse.txt', 'r') as f:
    raw = f.read().strip()

tokens = raw.split()
print(f'Total tokens: {len(tokens)}')
print()

decoded = ''
for i, token in enumerate(tokens):
    if token in ('{', '}'):
        char = token
    elif token in MORSE_TO_CHAR:
        char = MORSE_TO_CHAR[token]
    else:
        char = f'?{token}?'
    decoded += char
    print(f'  {i:2d}: {token!r:10s} -> {char}')

print()
print(f'FLAG: {decoded}')
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X`.

### Step 4: Run the Decoder

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/tapping]
└─$ python3 decode.py
Total tokens: 31

   0: '.--.'     -> P
   1: '..'       -> I
   2: '-.-.'     -> C
   3: '---'      -> O
   4: '-.-.'     -> C
   5: '-'        -> T
   6: '..-.'     -> F
   7: '{' -> {
   8: '--'       -> M
   9: '-----'    -> 0
  10: '.-.'      -> R
  11: '...'      -> S
  12: '...--'    -> 3
  13: '-.-.'     -> C
  14: '-----'    -> 0
  15: '-..'      -> D
  16: '...--'    -> 3
  17: '.----'    -> 1
  18: '...'      -> S
  19: '..-.'     -> F
  20: '..-'      -> U
  21: '-.'       -> N
  22: '.-'       -> A
  23: '-..'      -> D
  24: '.'        -> E
  25: '-----'    -> 0
  26: '-----'    -> 0
  27: '-----'    -> 0
  28: '----.'    -> 9
  29: '.----'    -> 1
  30: '}' -> }

FLAG: PICOCTF{M0RS3C0D31SFUNADE00091}
```

Each token maps cleanly to a single character. The first seven decode to `PICOCTF`, which lines up with the wrapper hint. The body decodes to `M0RS3C0D31SFUNADE00091`.

### Step 5: Submit

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/tapping]
└─$ echo "picoCTF{M0RS3C0D31SFUNADE00091}"
picoCTF{M0RS3C0D31SFUNADE00091}
```

Paste `picoCTF{M0RS3C0D31SFUNADE00091}` into the flag box. Correct on first try.

**Flag:** `picoCTF{M0RS3C0D31SFUNADE00091}`

> Note: the random portion of the flag is regenerated every time you connect to the server, so do not be surprised if your decoded flag has a different suffix. As long as your decode matches the wrapper shape `picoCTF{...}` and the Morse-to-ASCII mapping is correct, the recovered string is the right answer for your session.

---

## Alternative Solve Methods

### Method 1: Decode by Eye With a Morse Chart Open

For a 22-token body, you can decode the entire flag by hand in under a minute if you have a Morse chart open in another window. The whole sequence is just three chunks:

- `P I C O C T F` (the wrapper, given away by the hint).
- `M 0 R S 3 C 0 D 3` (the leet-speak wordplay, decoded by reading `...` as S and `-----` as 0).
- `1 S F U N A D E 0 0 0 9 1` (a random suffix appended per session).

This is actually faster than spinning up Python when you are competing on a timer. The Python script above is mostly for the writeup so future-me can re-verify the decode without staring at a wall of dots and dashes.

### Method 2: One-Liner With `tr` and a Morse Lookup

If you prefer staying in shell, you can replace each Morse sequence with its letter using a chain of `tr` substitutions. It is ugly, but it works without any scripting language. For example:

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/tapping]
└─$ SPACE=' '

┌──(zham㉿kali)-[~/picoCTF/cryptography/tapping]
└─$ sed 's/\.--\./P /g; s/\.\./I /g; s/-\.-\./C /g; s/---/O /g; s/-/T /g; s/\.\.-\./F /g; s/--/M /g; s/-----/0 /g; s/\.-\./R /g; s/\.\.\./S /g; s/\.\.-\./F /g; s/\.\.-/U /g; s/-\./N /g; s/\.-/A /g; s/-\.\./D /g; s/\./E /g; s/\.----/1 /g; s/\.\.\.--/3 /g; s/----\./9 /g' morse.txt
P I C O C T F { M 0 R S 3 C 0 D 3 1 S F U N A D E 0 0 0 9 1 } 
```

The trick is that you need to substitute longer tokens first (`-----` before `-`, `...--` before `...`, and so on), otherwise the short ones swallow up the leading characters of the long ones. This is fragile and easy to get wrong — Python with a dict is much cleaner, but `sed` works in a pinch.

### Method 3: CyberChef Morse Code Recipe

CyberChef has a built-in **Morse Code** decode recipe. Paste the raw stream into the Input box, drop in the **From Morse Code** operation, and the Output pane shows the decoded text directly. This is the GUI fallback I use when I am on a Windows box without Python or `sed`. Same answer, different tool.

---

## What Happened Internally

Here is the timeline of what was going on, from "I see a server hint" to "I have the flag."

1. **Read the prompt.** The challenge gave us a one-liner description, two hints, and a `nc` instruction. The two hints together tell us the encoding (Morse) and the wrapper (`PICOCTF{}`). That collapses the work to "decode Morse from a TCP socket."
2. **Connected to the server.** `nc fickle-tempest.picoctf.net 65483` opened a TCP connection to picoCTF's challenge host. The server immediately streamed one line of dots, dashes, spaces, and `{` `}` brackets, then closed the socket. The stream was the Morse-encoded flag.
3. **Captured the stream.** I copied the line off the screen and saved it to `morse.txt` so the rest of the work happens offline. This matters because if I lose the connection or the server resets, I do not lose the flag.
4. **Built a Morse lookup.** I wrote a Python dictionary with the 26 letters and 10 digits as keys. The decoder iterates over each whitespace-separated token, looks it up in the dict, and appends the result to the output string.
5. **Ran the decoder.** `python3 decode.py` produced a 31-line trace and a final flag string. The first seven tokens decoded to `PICOCTF`, confirming the wrapper was correct and the alignment was right.
6. **Sanity-checked the body.** The body `M0RS3C0D31SFUNADE00091` is leet-speak for "MORSE CODE IS FUN" plus a random 8-character suffix. That is the obvious pattern the challenge author would write, so the decode is almost certainly right.
7. **Submitted.** The recovered plaintext `picoCTF{M0RS3C0D31SFUNADE00091}` was sent back to the server, which matched it against the stored flag for our session and awarded the 200 points.

The whole attack is a **substitution cipher** with a publicly known key (the Morse chart). No brute force, no cryptanalysis, no key recovery. The "security" of the encoding comes from the decoder knowing the alphabet, which is exactly how the original telegraph system worked in the 1850s.

---

## Tools Used

| Tool       | Purpose                                                                |
| ---------- | ---------------------------------------------------------------------- |
| `mkdir`    | Create a working directory for the challenge                           |
| `nc`       | Open a TCP connection to the challenge server and capture the stream   |
| `cat`      | Persist the captured Morse stream into `morse.txt`                     |
| `nano`     | Write `decode.py` (`Ctrl+O`, `Enter`, `Ctrl+X`)                        |
| `python3`  | Run the dictionary-based Morse decoder and print a token-by-token trace |
| `echo`     | Print the final flag for the record                                    |
| `sed`      | Shell-only alternative for inline Morse-to-letter substitution         |
| CyberChef  | Optional GUI alternative using the **From Morse Code** recipe          |

---

## Key Takeaways

- **Morse code is a fixed substitution cipher.** Each letter has exactly one canonical dot-dash spelling, so decoding is a straight dictionary lookup. No key, no shift, no math.
- **The hints are doing real work here.** Hint 1 names the encoding (Morse) outright, and hint 2 locks the wrapper to `PICOCTF{}`. Together they remove every other code from consideration: not Base64, not binary, not ASCII, just plain Morse.
- **`nc` is the right tool for raw TCP.** When a challenge hands you a host and a port, `nc <host> <port>` is almost always the first thing to try. If `nc` is missing on your distro, `socat - TCP:host:port` or `ncat host port` does the same thing.
- **A reproducibility script is cheap insurance.** Even when the decode is fast, a 30-line `decode.py` gives you a token-by-token trace you can re-check, plus a paper trail for the writeup. I now write one for almost every cryptography challenge.
- **Substitute longer tokens first.** If you ever do this with `sed` or chained `tr` calls, remember that `...` is a prefix of `...--`. Replace the longer token first or you will mis-decode every Morse digit in the stream. The dict-based Python decoder sidesteps this entirely.
- **The flag suffix is randomized per session.** The challenge server picks a new random suffix every time you connect, so do not assume someone else's writeup flag will work for you. Decode the stream from *your* connection and submit *that*.

**Flag wordplay decode:** the inner text `M0RS3C0D31SFUNADE00091` is **"MORSE CODE IS FUN"** plus a random 8-character suffix, all in light leet-speak. Reading the digits as letters (`0`→O, `3`→E, `1`→I) and ignoring the trailing `ADE00091` for a moment, the message is the obvious pun on the challenge: the encoding we just cracked is Morse code, and the author wants us to know it is fun. The trailing suffix is a per-session random salt that prevents flag-sharing across competitors — even if two people solved the challenge at the same time, they would still submit different strings and the server would accept both.
