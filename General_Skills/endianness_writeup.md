# endianness - picoCTF Writeup

**Challenge:** endianness  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{3ndi4n_sw4p_su33ess_cfe38ef0}`

---

## Description

Know of little and big endian?

Connect with:
```
nc titan.picoctf.net 53225
```

---

## Hints

1. You might want to check the ASCII table to first find the hexadecimal representation of characters before finding the endianness.

---

## Solution

### Step 1: Connect to the Server

```bash
nc titan.picoctf.net 53225
```

The server gave me a word and asked for both its **Little Endian** and **Big Endian** hex representations.

**Word given:** `hynyy`

### Step 2: Convert the Word to Hex Using CyberChef

I went to **CyberChef** at https://gchq.github.io/CyberChef/ and:
1. Searched for **"To Hex"** and dragged it into the Recipe
2. Set **Delimiter** to `None` and **Bytes per line** to `0`
3. Typed `hynyy` in the Input box
4. Got the output: `68796e7979`

This output is the **Big Endian** value!

### Step 3: Get Little Endian

**Little Endian** is just the Big Endian byte pairs in **reversed order**:

```
Big Endian:    68 79 6e 79 79
Little Endian: 79 79 6e 79 68
```

So Little Endian = `79796e7968`

### Step 4: Go Back to Kali and Submit Both Answers

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

## Why This Works

### What is Endianness?

**Endianness** is the order in which bytes are stored or read in memory.

**Big Endian** stores bytes from the **most significant (first) to least significant (last)** — just like how we normally read left to right.

**Little Endian** stores bytes from the **least significant (last) to most significant (first)** — the reversed order.

**Example with "hynyy":**
```
Normal (Big Endian):    68 79 6e 79 79
Reversed (Little Endian): 79 79 6e 79 68
```

### Simple Breakdown

```
Connect to server
      |
      v
Get a word (e.g. "hynyy")
      |
      v
Convert each letter to hex using ASCII table
      |
      v
Big Endian = normal hex order left to right
Little Endian = reversed hex order right to left
      |
      v
Submit both answers
      |
      v
Flag is revealed!
```

---

## Alternative Method 1: CyberChef

1. Go to https://gchq.github.io/CyberChef/
2. Search for **"To Hex"** and drag it into the Recipe
3. Set **Delimiter** to `None` and **Bytes per line** to `0`
4. Type the word in the Input box
5. The output is your **Big Endian** value
6. Manually **reverse** the pairs to get **Little Endian**

Example with `hynyy`:
```
CyberChef output: 68796e7979  (Big Endian)
Reversed pairs:   79796e7968  (Little Endian)
```

## Alternative Method 2: Python Script

You can automate the conversion with Python:

```python
word = "hynyy"

# Convert each character to hex
hex_values = [format(ord(c), '02x') for c in word]
print("Hex values:", hex_values)

# Big Endian = normal order
big_endian = ''.join(hex_values)
print("Big Endian:", big_endian)

# Little Endian = reversed order
little_endian = ''.join(reversed(hex_values))
print("Little Endian:", little_endian)
```

**Output:**
```
Hex values: ['68', '79', '6e', '79', '79']
Big Endian: 68796e7979
Little Endian: 79796e7968
```

---

## Commands Used

```bash
# Connect to the server
nc titan.picoctf.net 53225
```

---

## Flag

```
picoCTF{3ndi4n_sw4p_su33ess_cfe38ef0}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nc` (netcat) | Connect to the challenge server |
| ASCII table | Convert characters to hex values |

---

## Key Takeaways

- **Big Endian** = normal byte order (most significant byte first)
- **Little Endian** = reversed byte order (least significant byte first)
- Most modern CPUs (Intel/AMD x86) use Little Endian by default
- Always convert letters to hex using the ASCII table first, then apply the endian order
- Little Endian is just the reverse of Big Endian byte order
