# WebDecode - picoCTF Writeup

**Challenge:** webDecode  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{web_succ3ssfully_d3c0ded_07b91c79}`

---

## Description

Can you find the flag hidden in this website?

---

## Hints

1. Try inspecting different pages on the website
2. Look for encoded text
3. Base64 is a common encoding method

---

## Solution

### Step 1: Open the Website

I navigated to the challenge URL and saw a simple website with two navigation links: **Home** and **About**.

The homepage didn't show anything suspicious.

### Step 2: Check the About Page

I clicked on the **About** link to navigate to the About page. The visible content showed some text about notpetya, but nothing that looked like a flag.

### Step 3: Inspect the About Page

I inspected the About page source:
* Right-clicked on the page
* Selected **Inspect** (or pressed **F12**)

### Step 4: Find the Encoded Flag

While examining the HTML source code on the About page, I found a Base64-encoded string:
```
cGljb0NURnt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfMDdiOTFjNzl9
```

### Step 5: Decode the Base64 String

I used an online Base64 decoder (like https://www.base64decode.org/) or the command line:
```bash
echo "cGljb0NURnt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfMDdiOTFjNzl9" | base64 -d
```

**Output:**
```
picoCTF{web_succ3ssfully_d3c0ded_07b91c79}
```

---

## Why This Works

* **Base64 encoding** is a common way to encode binary data into ASCII text
* It's not encryption - just encoding, so anyone can decode it
* Developers sometimes use Base64 to "hide" data, but it provides no security
* The flag was hidden in the About page's HTML source
* **Always check all pages** on a website, not just the homepage

---

## What is Base64?

Base64 is an encoding scheme that converts binary data into ASCII text using 64 characters (A-Z, a-z, 0-9, +, /). It's commonly used to:
* Encode binary data for transmission over text-based protocols
* Embed images in HTML/CSS
* Obfuscate (but not secure) text

**Common indicators of Base64:**
* Ends with `=` or `==` (padding)
* Contains only alphanumeric characters, `+`, and `/`
* Length is always a multiple of 4

---

## Command Line Decoding
```bash
# Decode Base64 string
echo "cGljb0NURnt3ZWJfc3VjYzNzc2Z1bGx5X2QzYzBkZWRfMDdiOTFjNzl9" | base64 -d

# Output:
# picoCTF{web_succ3ssfully_d3c0ded_07b91c79}
```

---

## Flag

`picoCTF{web_succ3ssfully_d3c0ded_07b91c79}`

---

## Tools Used

* **Web Browser** - Access the challenge and navigate pages
* **Browser Developer Tools (F12)** - Inspect HTML source
* **Base64 Decoder** - https://www.base64decode.org/ or `base64` command

---

## Key Takeaways

* Always explore all pages on a website, not just the homepage
* Check the HTML source code of every page
* Base64 encoding is easily reversible and provides no security
* Never use Base64 as a way to hide sensitive information
* Common encoding patterns (like Base64) are easy to recognize and decode
