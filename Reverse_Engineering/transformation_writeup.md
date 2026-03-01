# Transformation - picoCTF Writeup

**Challenge:** Transformation  
**Category:** Reverse Engineering  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{16_bits_inst34d_of_8_b7f62ca5}`

---

## Description

I wonder what this really is...

Encoding used:
```python
"".join([chr((ord(flag[i]) << 8) + ord(flag[i + 1])) for i in range(0, len(flag), 2)])
```

Download the file: enc

---

## Hints

1. You may find some decoders online

---

## Solution

### Step 1: Understand the Encoding

The challenge gave us the encoding formula. Let me break it down simply:

- It takes the flag **2 characters at a time**
- It combines them into **1 Unicode character**
- It does this by shifting the first character left by 8 bits and adding the second character

So every 2 ASCII characters become 1 Unicode character. This means the encoded file has half the number of characters as the original flag.

### Step 2: Decode Using CyberChef (My Method)

I opened **CyberChef** at https://gchq.github.io/CyberChef/

Steps I followed:
1. Pasted the contents of the `enc` file into the **Input** box
2. Searched for **"Magic"** in the Operations search bar and dragged it into the Recipe
3. Checked the **"Intensive mode"** checkbox
4. Clicked **BAKE!**
5. Scrolled through the output and found the flag under `Encode_text('UTF-16BE (1201)')`

**Output:**
```
picoCTF{16_bits_inst34d_of_8_b7f62ca5}
```

Got the flag! 🎯

---

## Alternative Method 1: Python Script

You can write a simple Python script to reverse the encoding:

```python
# Read the encoded file
with open('enc', 'r', encoding='utf-8') as f:
    encoded = f.read().strip()

# Decode: split each Unicode character back into 2 ASCII characters
flag = ""
for char in encoded:
    val = ord(char)
    flag += chr(val >> 8)    # Get the high byte (first character)
    flag += chr(val & 0xFF)  # Get the low byte (second character)

print(flag)
```

Run it:
```bash
python3 decode.py
```

**Output:**
```
picoCTF{16_bits_inst34d_of_8_b7f62ca5}
```

### Alternative Method 2: Python One-liner

```bash
python3 -c "
data = open('enc', 'r', encoding='utf-8').read().strip()
print(''.join([chr(ord(c) >> 8) + chr(ord(c) & 0xFF) for c in data]))
"
```

---

## Why This Works

### What Did the Encoding Do?

Normal ASCII characters use **8 bits** (1 byte). Unicode characters can use **16 bits** (2 bytes).

The encoding combined 2 ASCII characters into 1 Unicode character like this:

```
flag[0] = 'p' = 112 in ASCII
flag[1] = 'i' = 105 in ASCII

Combined: (112 << 8) + 105 = 28777 = one Unicode character
```

To reverse it, we just split each Unicode character back into its two original bytes:

```
28777 >> 8 = 112 = 'p'   (high byte)
28777 & 255 = 105 = 'i'  (low byte)
```

### Simple Breakdown

```
Encoded file has weird Unicode characters
        |
        v
Each Unicode character = 2 original ASCII characters
        |
        v
Split each character into high byte and low byte
        |
        v
Flag is revealed!
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| CyberChef | Decode the encoded file using Magic mode |

---

## Flag

```
picoCTF{16_bits_inst34d_of_8_b7f62ca5}
```

---

## Key Takeaways

- The encoding combined 2 ASCII characters into 1 Unicode character using bit shifting
- CyberChef's **Magic** feature with **Intensive mode** can automatically detect and decode many encodings
- To reverse bit shifting, use `>> 8` to get the high byte and `& 0xFF` to get the low byte
- The flag name "16 bits instead of 8" directly describes what the encoding did
- Python is great for writing quick custom decoders when you understand the encoding formula
