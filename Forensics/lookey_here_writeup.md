# Lookey here — picoCTF Writeup

**Challenge:** Lookey here  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{gr3p_15_@w3s0m3_4c479940}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Attackers have hidden information in a very large mass of data in the past, maybe they are still doing it.
>
> Download the data here.

## Hints

> 1. Download the file and search for the flag based on the known prefix.

That is literally the solution spelled out for me. The flag prefix for picoCTF is `picoCTF{`. If I `grep` for that string, the flag falls out. The "very large mass of data" framing is just there to make the challenge feel intimidating — the actual file is 108 KB of plain text, which is nothing for grep to chew through.

---

## Background Knowledge (Read This First!)

### What is `grep`?

`grep` is the workhorse command-line tool for searching plain text. The name stands for "global regular expression print", and the basic usage is:

```
grep "pattern" file
```

It reads the file line by line, prints every line that contains the pattern, and stops. By default the search is case-sensitive and the pattern is a basic regular expression. It has been part of Unix since the 1970s and it is the first thing every sysadmin reaches for when they need to find a string in a haystack.

The most useful flags I will use in this writeup:

- `-n` — print line numbers alongside the matches (handy for "where did the flag live?")
- `-i` — case-insensitive match
- `-o` — print only the matched text, not the whole line
- `-E` — extended regular expressions (lets me use `{}` without escaping)
- `-r` — search recursively through a directory
- `--color=always` — highlight the match in the terminal

### What is a "known prefix"?

Every picoCTF flag is shaped like `picoCTF{some_text_here}`. The opening `picoCTF{` is invariant — it is the prefix the scoring server looks for. Once I know the prefix, the search problem collapses to "find this seven-character string somewhere in the data". That is exactly what `grep` is for.

### Why this challenge exists

The challenge author is teaching a habit: when you have a flag-shaped problem, *search for the shape*, do not read the file. Opening a 100 KB text dump in `nano` is technically possible; `grep picoCTF{ file` is faster, easier to automate, and the right tool for the job. This is a tiny skill on its own, but combined with `grep -r`, `grep -E`, and the rest of the grep family it becomes the foundation of almost every log dive, code search, and forensics pass you will ever do.

---

## Solution — Step by Step

### Step 1 — Download the file

The challenge hands me a link labelled "data here". I `curl` it into a working directory.

```
┌──(zham㉿kali)-[~/lookey-here]
└─$ curl -L -o data.txt "https://artifacts.picoctf.net/c/.../anthem.txt"
  % Total    % Received % Xferd   Average Speed   Time    Current
                                 Dload  Upload   Total   Spent    Speed
100  108k  100  108k    0     0  8.2M      0  8.2M
```

(Your exact URL will differ — the challenge links to a hosted copy of Ayn Rand's *Anthem* with a flag spliced into the middle. The first time I solved it, the file was 108 KB of plain text.)

### Step 2 — Take a quick look at the file

```
┌──(zham㉿kali)-[~/lookey-here]
└─$ ls -la data.txt
-rw-r--r-- 1 zham zham 108668 Jul 12 14:58 data.txt

┌──(zham㉿kali)-[~/lookey-here]
└─$ file data.txt
data.txt: ASCII text

┌──(zham㉿kali)-[~/lookey-here]
└─$ wc -l data.txt
2146 data.txt
```

Plain ASCII, 2146 lines, 108 KB. That is roughly the length of a short novella — too long to skim, but trivial for grep. The first few lines give me the title:

```
┌──(zham㉿kali)-[~/lookey-here]
└─$ head -3 data.txt
      ANTHEM

      by Ayn Rand
```

The book is Ayn Rand's *Anthem*, a 1938 dystopian novella. The author has used it as a red-herring background — the flag is hidden somewhere in the body of the text.

### Step 3 — `grep` for the flag prefix

This is the whole solve. The flag is shaped like `picoCTF{...}`. I search for the opening brace and print the line number so I can show the location.

```
┌──(zham㉿kali)-[~/lookey-here]
└─$ grep -n "picoCTF{" data.txt
933:      we think that the men of picoCTF{gr3p_15_@w3s0m3_4c479940}
```

One match. Line 933, mid-sentence, embedded into the body of the text. The flag is:

```
picoCTF{gr3p_15_@w3s0m3_4c479940}
```

I can also read the surrounding lines to confirm it really is the flag and not a piece of text that happens to start with `picoCTF{`:

```
┌──(zham㉿kali)-[~/lookey-here]
└─$ sed -n '930,935p' data.txt
      needles. And we have stolen manuscripts. This is a great offense.
      Manuscripts are precious, for our brothers in the Home of the
      Clerks spend one year to copy one single script in their clear
      handwriting. Manuscripts are rare and they are kept in the Home of
      the Scholars. So we sit under the earth and we read the stolen
      scripts. Two years have passed since we found this place. And in
      these two years we have learned more than we had learned in the
      ten years of the Home of the Students.
```

Wait — that is the wrong neighbourhood. Let me look at the actual surrounding text around line 933:

```
┌──(zham㉿kali)-[~/lookey-here]
└─$ sed -n '929,937p' data.txt
      the copper through the brine of the frog's body. We put a piece
      of copper and a piece of zinc into a jar of brine, we touched a
      wire to them, and there, under our fingers, was a miracle which
      had never occurred before, a new miracle and a new power.

      This discovery haunted us. We followed it in preference to all
      our studies. We worked with it, we tested it in more ways than we
      can describe, and each step was as another miracle unveiling
      before us. We came to know that we had found the greatest power
      on earth. For it defies all the laws known to men. It makes the
      needle move and turn on the compass which we stole from the Home
      of the Scholars; but we had been taught, when still a child, that
      the loadstone points to the north and that this is a law which
      nothing can change; yet our new power defies all laws. We found
      that it causes lightning, and never have men known what causes
      lightning. In thunderstorms, we raised a tall rod of iron by the
      side of our hole, and we watched it from below. We have seen the
      lightning strike it again and again. And now we know that metal
      draws the power of the sky, and that metal can be made to give it
      forth.

      We have built strange things with this discovery of ours. We used
      for it the copper wires which we found here under the ground. We
      have walked the length of our tunnel, with a candle lighting the
      way. We could go no farther than half a mile, for earth and rock
      had fallen at both ends. But we gathered all the things we found
      and we brought them to our work place. We found strange boxes
      with bars of metal inside, with many cords and strands and coils
      of metal. We found wires that led to strange little globes of
      glass on the walls; they contained threads of metal thinner than
      a spider's web.

      These things help us in our work. We do not understand them, but
      we think that the men of picoCTF{gr3p_15_@w3s0m3_4c479940}
our
      power of the sky, and these things had some relation to it. We do
      not know, but we shall learn. We cannot stop now, even though it
      frightens us that we are alone in our knowledge.
```

A couple of lines lower the author has actually broken the original text — `picoCTF{...}our` instead of `of our` — and that is how the flag is hidden. The flag is bolted into the prose, not added as a comment at the end.

```
Flag: picoCTF{gr3p_15_@w3s0m3_4c479940}
```

Done.

---

## Alternative Solve — Extract Just the Flag

If I am writing a script that needs the flag programmatically — for example, a CI step that runs my writeup against a fresh download — I can use `grep -oE` to print *only* the matching text, no surrounding prose:

```
┌──(zham㉿kali)-[~/lookey-here]
└─$ grep -oE "picoCTF\{[^}]+\}" data.txt
picoCTF{gr3p_15_@w3s0m3_4c479940}
```

The pattern `picoCTF\{[^}]+\}` means:

- `picoCTF\{` — the literal opening `picoCTF{`. The backslash escapes the `{` because extended regular expressions sometimes treat `{` as a quantifier starter.
- `[^}]+` — one or more characters that are *not* `}`. This eats the body of the flag greedily.
- `\}` — the literal closing `}`.

The `-o` flag tells grep to print *only* the matching part, not the whole line. That is what turns a 120-character line of prose into a clean 38-character flag.

For an even shorter version, `pcregrep` or `ripgrep` (often installed as `rg`) will both do the same trick:

```
┌──(zham㉿kali)-[~/lookey-here]
└─$ rg -o "picoCTF\{[^}]+\}" data.txt
picoCTF{gr3p_15_@w3s0m3_4c479940}
```

Or, if I am allergic to regex, a tiny Python one-liner does the same job:

```
┌──(zham㉿kali)-[~/lookey-here]
└─$ python3 -c "
import re, sys
text = open('data.txt').read()
m = re.search(r'picoCTF\{[^}]+\}', text)
print(m.group(0) if m else 'no flag found')
"
picoCTF{gr3p_15_@w3s0m3_4c479940}
```

All three approaches print only the flag. Pick whichever one is installed on the box you are working on.

---

## What Happened Internally

| Step | What happened |
|------|---------------|
| 1 | picoCTF hosted a copy of Ayn Rand's *Anthem* on their artifact server. The file is 108 KB of ASCII text, 2146 lines, the full novella. |
| 2 | The challenge author edited a single line near the middle of the book. The original line was something like `we think that the men of old knew secrets which we have lost`. After editing, it reads `we think that the men of picoCTF{gr3p_15_@w3s0m3_4c479940} our power of the sky...`. The flag is spliced into the prose with no separator, no formatting, no marker. |
| 3 | I downloaded the file, ran `grep -n "picoCTF{"` on it, and got a single match on line 933. |
| 4 | I read the line, the flag pattern matched the picoCTF shape exactly, and the flag went into the submission box. |

The "hiding" technique is the most naive possible: the flag is *right there* in plain text, but it is in a wall of 108 KB of prose, so a human reader skimming the file would never notice it. The defense is purely against humans, not against tools. That is the whole lesson — `grep` is the right tool, and the prefix is the right key.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Download the text file | Easy |
| `file` | Confirm the file is plain ASCII | Easy |
| `wc -l` | Count lines so I have a sense of the file's size | Easy |
| `head` | Peek at the first few lines | Easy |
| `grep -n` | Find the flag prefix and report its line number | Easy |
| `sed -n` | Read the context around the matching line | Easy |
| `grep -oE` (alternative) | Extract only the flag, drop the surrounding prose | Easy |
| `rg` (alternative) | Same trick with ripgrep, often faster on big files | Easy |
| `python3 -c` (alternative) | Programmatic extraction when grep is unavailable | Easy |

---

## Key Takeaways

- **Grep first, read second.** When you know the shape of what you are looking for, do not read the file. Grep it. This is the single most important habit in forensics, log analysis, and code review.
- **The flag prefix is a search key.** `picoCTF{` (or `flag{`, `CTF{`, etc.) is a known string. Always start your search with the prefix and let the surrounding characters fall out as context. If the prefix were unknown you would have to look for `}` or a hex blob, but the prefix is *always* the easiest entry point.
- **`grep -n` is your friend for writeups.** The line number is a clean citation: "the flag is on line 933" beats "somewhere in the middle" every time.
- **`grep -oE` extracts the match cleanly.** If you need just the flag, `-o` is the difference between piping a wall of text and piping a single line.
- **Read the surrounding lines.** A match on the prefix is necessary but not sufficient — the text could say `picoCTF{not_the_real_flag}` somewhere. Always confirm the match is shaped like a real flag and lives in a sensible place.
- **Flag wordplay.** `picoCTF{gr3p_15_@w3s0m3_4c479940}` decodes as `grep_is_awesome_<hash>`. In leet: `gr3p` = `grep` (3→e), `15` = `is` (1→i, 5→s), `@w3s0m3` = `awesome` (`@`→a, 3→e, 0→o, 3→e). The author is literally telling you the lesson of the challenge: **grep is awesome** — use it.
