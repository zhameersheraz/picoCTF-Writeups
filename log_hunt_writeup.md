---

# Log Hunt - picoCTF Writeup

**Challenge:** Log Hunt  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{us3_y0urlinux_sk1lls_cedfa5fb}`

---

## Description
This challenge provides a server log file that contains scattered pieces of a flag.  
The flag fragments are repeated throughout the log and need to be extracted and reconstructed.  
The goal is to find all the flag parts and assemble them in the correct order.

---

## Approach
The challenge description mentioned that flag parts are "scattered and sometimes repeated" in the logs.  
This suggested using text filtering tools like `grep` to search for a specific pattern.  
I needed to progressively narrow down the log entries to find the flag fragments.

---

## Solution

### Step 1: Download the Log File
I downloaded `server.log` from the challenge link provided.

### Step 2: View the Entire Log
First, I viewed the entire log file to understand its structure:

```bash
cat server.log
```

The log contained many entries with timestamps, log levels, and various messages.

### Step 3: Filter INFO Messages
To reduce the noise, I filtered only INFO level messages:

```bash
cat server.log | grep INFO
```

This showed many INFO entries, but I noticed some contained "FLAGPART" in them.

### Step 4: Search for Flag Fragments
I filtered specifically for lines containing "FLAGPART":

```bash
cat server.log | grep FLAGPART
```

**Output:**
```
[1990-08-09 10:00:10] INFO FLAGPART: picoCTF{us3_
[1990-08-09 10:02:55] INFO FLAGPART: y0urlinux_
[1990-08-09 10:05:54] INFO FLAGPART: sk1lls_
[1990-08-09 10:05:55] INFO FLAGPART: sk1lls_
[1990-08-09 10:10:54] INFO FLAGPART: cedfa5fb}
[1990-08-09 10:10:58] INFO FLAGPART: cedfa5fb}
[1990-08-09 10:11:06] INFO FLAGPART: cedfa5fb}
(repeated entries...)
```

### Step 5: Identify the Flag Parts
From the output, I identified four distinct flag fragments in chronological order:
1. `picoCTF{us3_`
2. `y0urlinux_`
3. `sk1lls_`
4. `cedfa5fb}`

### Step 6: Reconstruct the Flag
I concatenated the fragments in order:

```
picoCTF{us3_ + y0urlinux_ + sk1lls_ + cedfa5fb}
```

**Result:**
```
picoCTF{us3_y0urlinux_sk1lls_cedfa5fb}
```

---

## Why This Works
Server logs often contain valuable information hidden among normal operational data.  
By progressively filtering the log (all entries → INFO entries → FLAGPART entries), I narrowed down the search to find the flag fragments.  
The fragments appeared in chronological order based on timestamps, making reconstruction straightforward.

---

## Flag
`picoCTF{us3_y0urlinux_sk1lls_cedfa5fb}`

---

## Tools Used
* `cat` - Display file contents
* `grep` - Search for patterns in text
* Pipe (`|`) - Chain commands together

---

## Key Takeaways
* Use `cat` to view file contents when you need to understand the overall structure.
* Progressive filtering with `grep` helps narrow down large log files efficiently.
* Log analysis often requires reconstructing information from scattered fragments.
* Understanding command-line text processing tools is crucial for forensics challenges.
