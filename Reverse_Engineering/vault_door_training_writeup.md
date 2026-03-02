# vault-door-training - picoCTF Writeup

**Challenge:** vault-door-training  
**Category:** Reverse Engineering  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{w4rm1ng_Up_w1tH_jAv4_000AXPNPN0i}`

---

## Description

Your mission is to enter Dr. Evil's laboratory and retrieve the blueprints for his Doomsday Project. The laboratory is protected by a series of locked vault doors. Each door is controlled by a computer and requires a password to open. Unfortunately, our undercover agents have not been able to obtain the secret passwords for the vault doors, but one of our junior agents obtained the source code for each vault's computer! You will need to read the source code for each level to figure out what the password is for that vault door. As a warmup, we have created a replica vault in our training facility.

Download the file: VaultDoorTraining.java

---

## Hints

None

---

## Solution

### Step 1: Download and Read the Source Code

I downloaded the Java file and read its contents:

```bash
cat VaultDoorTraining.java
```

**Contents:**
```java
import java.util.*;

class VaultDoorTraining {
    public static void main(String args[]) {
        VaultDoorTraining vaultDoor = new VaultDoorTraining();
        Scanner scanner = new Scanner(System.in); 
        System.out.print("Enter vault password: ");
        String userInput = scanner.next();
        String input = userInput.substring("picoCTF{".length(), userInput.length()-1);
        if (vaultDoor.checkPassword(input)) {
            System.out.println("Access granted.");
        } else {
            System.out.println("Access denied!");
        }
    }

    // The password is below. Is it safe to put the password in the source code?
    // -Minion #9567
    public boolean checkPassword(String password) {
        return password.equals("w4rm1ng_Up_w1tH_jAv4_000AXPNPN0i");
    }
}
```

### Step 2: Find the Password

The password was hardcoded right inside the `checkPassword` function:

```java
return password.equals("w4rm1ng_Up_w1tH_jAv4_000AXPNPN0i");
```

✅ Password found: `w4rm1ng_Up_w1tH_jAv4_000AXPNPN0i`

### Step 3: Build the Flag

The code strips `picoCTF{` from the beginning and `}` from the end before checking the password. So to get the full flag, I just wrapped the password with the flag format:

```
picoCTF{w4rm1ng_Up_w1tH_jAv4_000AXPNPN0i}
```

Got the flag! 🎯

---

## Why This Works

The developer stored the password directly in the source code as plain text. This is called **hardcoded credentials** and is a very common and dangerous mistake.

Anyone who gets access to the source code can simply read the password without needing to run the program at all.

The developer even left a comment warning about this:

```java
// The password is below. Is it safe to put the password in the source code?
// What if somebody stole our source code? Then they would know what our
// password is.
// -Minion #9567
```

### Simple Breakdown

```
Download source code
        |
        v
Read the checkPassword function
        |
        v
Password is hardcoded in plain text
        |
        v
Wrap it with picoCTF{} format
        |
        v
Flag found!
```

---

## Flag

```
picoCTF{w4rm1ng_Up_w1tH_jAv4_000AXPNPN0i}
```

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
