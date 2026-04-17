# Transformation — picoCTF Writeup

**Challenge:** Transformation  
**Category:** Reverse Engineering  
**Difficulty:** Easy  
**Flag:** `picoCTF{16_bits_inst34d_of_8_b7f62ca5}`

---

## Description

> I wonder what this really is...
> Encoding used:
> ```python
> "".join([chr((ord(flag[i]) << 8) + ord(flag[i + 1])) for i in range(0, len(flag), 2)])
> ```
> Download the file: enc

**Hint shown in challenge:** `You may find some decoders online`

---

## Background Knowledge (Read This First!)

### What Did the Encoding Do?

Normal ASCII characters use **8 bits** (1 byte). Unicode characters can use **16 bits** (2 bytes).

The encoding combined 2 ASCII characters into 1 Unicode character:
```
flag[0] = 'p' = 112 in ASCII
flag[1] = 'i' = 105 in ASCII

Combined: (112 << 8) + 105 = 28777 = one Unicode character
```

To reverse it, we split each Unicode character back into its two original bytes:
```
28777 >> 8 = 112 = 'p'   (high byte)
28777 & 255 = 105 = 'i'  (low byte)
```

---

## Solution — Step by Step

### Step 1 — Decode Using CyberChef

I opened CyberChef at https://gchq.github.io/CyberChef/

1. Pasted the contents of the `enc` file into the **Input** box
2. Searched for **"Magic"** and dragged it into the Recipe
3. Checked the **"Intensive mode"** checkbox
4. Clicked **BAKE!**
5. Found the flag under `Encode_text('UTF-16BE (1201)')`

**Output:**
```
picoCTF{16_bits_inst34d_of_8_b7f62ca5}
```

Got the flag! 🎯

---

## Alternative Method — Python Script

```python
with open('enc', 'r', encoding='utf-8') as f:
    encoded = f.read().strip()

flag = ""
for char in encoded:
    val = ord(char)
    flag += chr(val >> 8)    # Get the high byte (first character)
    flag += chr(val & 0xFF)  # Get the low byte (second character)

print(flag)
```

Run it:
```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 decode.py
picoCTF{16_bits_inst34d_of_8_b7f62ca5}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| CyberChef | Decode using Magic / intensive mode |

---

## Key Takeaways

- The encoding combined 2 ASCII characters into 1 Unicode character using bit shifting
- CyberChef's **Magic** feature with **Intensive mode** can automatically detect and decode many encodings
- To reverse bit shifting, use `>> 8` to get the high byte and `& 0xFF` to get the low byte
- The flag name "16 bits instead of 8" directly describes what the encoding did
- Python is great for writing quick custom decoders when you understand the encoding formula
