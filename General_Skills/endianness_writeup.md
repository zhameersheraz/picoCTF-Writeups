# endianness — picoCTF Writeup

**Challenge:** endianness  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{3ndi4n_sw4p_su33ess_cfe38ef0}`

---

## Description

> Know of little and big endian?
> Connect with: `nc titan.picoctf.net 53225`

**Hint shown in challenge:** `You might want to check the ASCII table to first find the hexadecimal representation of characters before finding the endianness.`

---

## Background Knowledge (Read This First!)

### What is Endianness?

**Endianness** is the order in which bytes are stored or read in memory.

- **Big Endian** — stores bytes from most significant (first) to least significant (last). This is how we normally read left to right.
- **Little Endian** — stores bytes from least significant (last) to most significant (first). The reversed order.

Example with `hynyy`:
```
Normal (Big Endian):      68 79 6e 79 79
Reversed (Little Endian): 79 79 6e 79 68
```

---

## Solution — Step by Step

### Step 1 — Connect to the Server

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nc titan.picoctf.net 53225
```

The server gave a word and asked for both its Little Endian and Big Endian hex representations.

**Word given:** `hynyy`

### Step 2 — Convert the Word to Hex Using CyberChef

I went to CyberChef (https://gchq.github.io/CyberChef/) and:
1. Searched for **"To Hex"** and dragged it into the Recipe
2. Set **Delimiter** to `None` and **Bytes per line** to `0`
3. Typed `hynyy` in the Input box
4. Got the output: `68796e7979`

This output is the **Big Endian** value!

### Step 3 — Get Little Endian

Little Endian is just the Big Endian byte pairs in **reversed order**:

```
Big Endian:    68 79 6e 79 79
Little Endian: 79 79 6e 79 68
```

So Little Endian = `79796e7968`

### Step 4 — Submit Both Answers

```
Enter the Little Endian representation: 79796e7968
Correct Little Endian representation!
Enter the Big Endian representation: 68796e7979
Correct Big Endian representation!
Congratulations! You found both endian representations correctly!
Your Flag is: picoCTF{3ndi4n_sw4p_su33ess_cfe38ef0}
```

Got the flag! 🎯

---

## Alternative Method — Python Script

```python
word = "hynyy"
hex_values = [format(ord(c), '02x') for c in word]
big_endian = ''.join(hex_values)
little_endian = ''.join(reversed(hex_values))
print("Big Endian:", big_endian)
print("Little Endian:", little_endian)
```

**Output:**
```
Big Endian: 68796e7979
Little Endian: 79796e7968
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nc` (netcat) | Connect to the challenge server |
| CyberChef | Convert characters to hex |

---

## Key Takeaways

- **Big Endian** = normal byte order (most significant byte first)
- **Little Endian** = reversed byte order (least significant byte first)
- Most modern CPUs (Intel/AMD x86) use Little Endian by default
- Always convert letters to hex using the ASCII table first, then apply the endian order
- Little Endian is just the reverse of Big Endian byte pairs
