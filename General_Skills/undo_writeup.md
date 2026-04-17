# Undo — picoCTF Writeup

**Challenge:** Undo  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{Revers1ng_t3xt_Tr4nsf0rm@t10ns_7a89a9da}`  

---

## Description

> Can you reverse a series of Linux text transformations to recover the original flag?
> Start searching for the flag here `nc foggy-cliff.picoctf.net 51418`

**Hint shown in challenge:** For text translation and character replacement, see `tr` command documentation.

---

## Background Knowledge (Read This First!)

### What is this challenge?

Connecting via `nc` (netcat) drops you into an interactive server that applies a series of text transformations to the flag — one at a time. Each step shows you the **current (transformed) flag** and a **hint** describing what was done. Your job is to type the correct Linux command to reverse that transformation before moving to the next step.

### What is netcat (`nc`)?

`nc` (netcat) is a networking tool that opens a raw TCP connection to a server. It lets you interact with challenge servers directly from the terminal:

```
nc <host> <port>
```

### What transformations can appear?

| Transformation | Reverse Command |
|---|---|
| Base64 encoded | `base64 -d` |
| Reversed the text | `rev` |
| Replaced `_` with `-` | `tr '-' '_'` |
| Replaced `{}` with `()` | `tr '()' '{}'` |
| Applied ROT13 | `tr 'A-Za-z' 'N-ZA-Mn-za-m'` |

---

## Solution — Step by Step

### Step 1 — Connect to the server

```
┌──(zham㉿kali)-[~]
└─$ nc foggy-cliff.picoctf.net 51418
===Welcome to the Text Transformations Challenge!===
Your goal: step by step, recover the original flag.
At each step, you'll see the transformed flag and a hint.
Enter the correct Linux command to reverse the last transformation.
```

### Step 2 — Reverse Base64

```
--- Step 1 ---
Current flag: KW5xOW45OG43LWZhMDFnQHplMHNmYTRlRy1nazNnLXRhMWZlcmlyRShTR1BicHZj
Hint: Base64 encoded the string.
Enter the Linux command to reverse it: base64 -d
Correct!
```

Base64 is a common encoding that turns binary data into printable ASCII text. `base64 -d` decodes it back.

### Step 3 — Reverse rev

```
--- Step 2 ---
Current flag: )nq9n98n7-fa01g@ze0sfa4eG-gk3g-ta1ferirE(SGPbpvc
Hint: Reversed the text.
Enter the Linux command to reverse it: rev
Correct!
```

`rev` reverses a string character by character. Running `rev` again on a reversed string restores the original.

### Step 4 — Restore underscores

```
--- Step 3 ---
Current flag: cvpbPGS(Eriref1at-g3kg-Ge4afs0ez@g10af-7n89n9qn)
Hint: Replaced underscores with dashes.
Enter the Linux command to reverse it: tr '-' '_'
Correct!
```

The `tr` command translates characters. The transformation swapped `_` → `-`, so reversing it swaps `-` → `_`.

### Step 5 — Restore curly braces

```
--- Step 4 ---
Current flag: cvpbPGS(Eriref1at_g3kg_Ge4afs0ez@g10af_7n89n9qn)
Hint: Replaced curly braces with parentheses.
Enter the Linux command to reverse it: tr '()' '{}'
Correct!
```

Same idea — the transformation swapped `{` → `(` and `}` → `)`, so reversing it swaps them back.

### Step 6 — Reverse ROT13

```
--- Step 5 ---
Current flag: cvpbPGS{Eriref1at_g3kg_Ge4afs0ez@g10af_7n89n9qn}
Hint: Applied ROT13 to letters.
Enter the Linux command to reverse it: tr 'A-Za-z' 'N-ZA-Mn-za-m'
Correct!
Congratulations! You've recovered the original flag:
>>> picoCTF{Revers1ng_t3xt_Tr4nsf0rm@t10ns_7a89a9da}
```

ROT13 rotates every letter by 13 positions. Since the alphabet has 26 letters, applying ROT13 twice brings you back to the start — it is its own inverse. ✅ Got the flag! 🎯

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nc` (netcat) | Connect to the challenge server | ⭐ Easy |
| `base64 -d` | Decode Base64 encoded string | ⭐ Easy |
| `rev` | Reverse a string | ⭐ Easy |
| `tr '-' '_'` | Swap dashes back to underscores | ⭐ Easy |
| `tr '()' '{}'` | Swap parentheses back to curly braces | ⭐ Easy |
| `tr 'A-Za-z' 'N-ZA-Mn-za-m'` | Decode ROT13 | ⭐ Easy |

---

## Key Takeaways

- **Read the hint carefully** — it tells you exactly what was done, so you only need to think about how to undo it
- **`tr` is your best friend** for character substitution — just swap the source and destination sets to reverse any `tr` transformation
- **ROT13 is self-inverse** — you apply the exact same `tr` command to both encode and decode
- **`rev` is also self-inverse** — reversing a reversed string restores the original
- **`base64 -d`** is the standard way to decode any Base64 string on Linux
