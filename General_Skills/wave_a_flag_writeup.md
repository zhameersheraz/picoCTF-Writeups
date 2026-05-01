# Wave a flag — picoCTF Writeup

**Challenge:** Wave a flag  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{b1scu1ts_4nd_gr4vy_ac5832c}`  

---

## Description

> Can you invoke help flags for a tool or binary? This program has extraordinarily helpful information...
> Download: `warm`

**Hint 1:** `This program will only work in the webshell or another Linux computer.`

---

## Background Knowledge (Read This First!)

### What is a help flag?

Almost every Linux program accepts a **help flag** — a special argument you pass when running it to get usage information. The two most common are:

```bash
program -h
program --help
```

When you run a program with `-h` or `--help`, it prints instructions about how to use it instead of doing its normal function. In this challenge, the binary was programmed to also print the flag when asked for help.

### What is `chmod +x`?

When you download a binary file, Linux may not allow you to run it yet — it needs **execute permission**. The `chmod +x` command adds that permission:

```bash
chmod +x filename   # makes the file executable
```

Without this step, running `./warm` gives a "Permission denied" error.

### What is `./`?

`./` means "run this file from the current directory." Linux doesn't automatically look in the current folder for programs — you need `./` to tell it where to find the file.

---

## Solution — Step by Step

### Step 1 — Copy the file to your working directory

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cp warm /home/zham/warm
```

### Step 2 — Make it executable

```
┌──(zham㉿kali)-[~]
└─$ chmod +x warm
```

### Step 3 — Run it with the help flag

```
┌──(zham㉿kali)-[~]
└─$ ./warm -h
Oh, help? I actually don't do much, but I do have this flag here: picoCTF{b1scu1ts_4nd_gr4vy_ac5832c}
```

✅ Got the flag! 🎯

---

## Alternative Method — Try `--help` too

Some programs use `--help` instead of `-h`:

```bash
./warm --help
```

Both are worth trying when you don't know which format a binary accepts. In this case `-h` works.

---

## What Happens Without `chmod +x`?

If you forget to make the file executable and try to run it:

```
┌──(zham㉿kali)-[~]
└─$ ./warm -h
bash: ./warm: Permission denied
```

Always run `chmod +x` on any downloaded binary before executing it.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `chmod +x` | Give the binary execute permission | ⭐ Easy |
| `./warm -h` | Run the binary with the help flag | ⭐ Easy |

---

## Key Takeaways

- **`-h` and `--help`** are the universal help flags — try both when exploring an unknown binary
- **`chmod +x filename`** is required before running any downloaded binary on Linux
- **`./`** is needed to run a file in the current directory — Linux doesn't look there automatically
- The challenge name is the hint: **"Wave a flag"** → pass a flag (`-h`) to the program
- The flag `b1scu1ts_4nd_gr4vy` → "biscuits and gravy" — a warm comfort food, matching the binary name `warm`!
