# Bases — picoCTF Writeup

**Challenge:** Bases  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{l3arn_th3_r0p35}`  

---

## Description

> What does this bDNhcm5fdGgzX3IwcDM1 mean? I think it has something to do with bases.

**Hint 1:** `Submit your answer in our flag format. For example, if your answer was 'hello', you would submit 'picoCTF{hello}' as the flag.`

---

## Background Knowledge (Read This First!)

### What is Base64?

**Base64** encodes binary data into a string using 64 printable characters (A-Z, a-z, 0-9, +, /). It's recognisable by its character set and often ends with `=` padding. The string `bDNhcm5fdGgzX3IwcDM1` is Base64 — no `=` padding because the length happens to be exact.

### What are "bases"?

The challenge title hints at **number bases** and **encoding bases** — different ways to represent data. Base64 is one of the most common encodings in computing, used everywhere from emails to web APIs.

---

## Solution — Step by Step

### Step 1 — Decode the Base64 string

```
┌──(zham㉿kali)-[~]
└─$ echo "bDNhcm5fdGgzX3IwcDM1" | base64 -d
l3arn_th3_r0p35
```

### Step 2 — Wrap in picoCTF format

Per Hint 1, wrap the decoded text in `picoCTF{...}`:

```
picoCTF{l3arn_th3_r0p35}
```

✅ Got the flag! 🎯

---

## Alternative Method — Python

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "import base64; print(base64.b64decode('bDNhcm5fdGgzX3IwcDM1').decode())"
l3arn_th3_r0p35
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `base64 -d` | Decode the Base64 string | ⭐ Easy |

---

## Key Takeaways

- **`echo "string" | base64 -d`** decodes any Base64 string in one command
- The flag `l3arn_th3_r0p35` → "learn the ropes" — an introduction to Base64 encoding

