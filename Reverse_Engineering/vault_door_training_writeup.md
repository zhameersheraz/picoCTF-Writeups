# vault-door-training — picoCTF Writeup

**Challenge:** vault-door-training  
**Category:** Reverse Engineering  
**Difficulty:** Easy  
**Flag:** `picoCTF{w4rm1ng_Up_w1tH_jAv4_000AXPNPN0i}`

---

## Description

> Your mission is to enter Dr. Evil's laboratory and retrieve the blueprints for his Doomsday Project. You will need to read the source code for each vault's computer to figure out what the password is.
> Download the file: VaultDoorTraining.java

---

## Background Knowledge (Read This First!)

### What are Hardcoded Credentials?

**Hardcoded credentials** means a password or secret is stored directly in the source code as plain text. This is a critical security mistake — anyone who gets access to the source code can simply read the password without running the program at all.

---

## Solution — Step by Step

### Step 1 — Download and Read the Source Code

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat VaultDoorTraining.java
```

**Key part of the code:**
```java
// The password is below. Is it safe to put the password in the source code?
// -Minion #9567
public boolean checkPassword(String password) {
    return password.equals("w4rm1ng_Up_w1tH_jAv4_000AXPNPN0i");
}
```

### Step 2 — Find the Password

The password was hardcoded right inside the `checkPassword` function:

```
w4rm1ng_Up_w1tH_jAv4_000AXPNPN0i
```

### Step 3 — Build the Flag

The code strips `picoCTF{` and `}` before checking the password. I just wrapped it with the flag format:

```
picoCTF{w4rm1ng_Up_w1tH_jAv4_000AXPNPN0i}
```

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read the Java source code file |

---

## Key Takeaways

- Never hardcode passwords or secrets directly in source code
- Source code can be stolen or leaked, exposing any hardcoded credentials
- Always read the source code carefully in Reverse Engineering challenges
- Sometimes the flag is just sitting there in plain text with no decoding needed
- The comment left by "Minion #9567" was a direct hint that the password was unsafe
