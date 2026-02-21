# dont-use-client-side - picoCTF Writeup

**Challenge:** dont-use-client-side  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{no_clients_plz_2eb02b45}`

---

## Description

Can you break into this super secure portal?

---

## Hints

1. Never trust the client

---

## Solution

### Step 1: Open the Website

I navigated to the challenge URL:
```
http://fickle-tempest.picoctf.net:57747
```

The page displayed a login form:
```
This is the secure login portal
Enter valid credentials to proceed
[Password Field] [Verify Button]
```

### Step 2: Try Random Input

I tried entering a random password, but got:
```
Incorrect password
```

### Step 3: Inspect the Page Source

I pressed `Ctrl+U` to view the page source, and found JavaScript code that validates the password:
```javascript
function verify() {
    checkpass = document.getElementById("pass").value;
    split = 4;
    if (checkpass.substring(0, split) == 'pico') {
      if (checkpass.substring(split*6, split*7) == 'eb02') {
        if (checkpass.substring(split, split*2) == 'CTF{') {
         if (checkpass.substring(split*4, split*5) == 'ts_p') {
          if (checkpass.substring(split*3, split*4) == 'lien') {
            if (checkpass.substring(split*5, split*6) == 'lz_2') {
              if (checkpass.substring(split*2, split*3) == 'no_c') {
                if (checkpass.substring(split*7, split*8) == 'b45}') {
                  alert("Password Verified")
                }
              }
            }
          }
        }
      }
    }
  }
```

### Step 4: Analyze the JavaScript Logic

The code checks the password in chunks of 4 characters (`split = 4`):

- `substring(0, 4)` = `'pico'`
- `substring(4, 8)` = `'CTF{'`
- `substring(8, 12)` = `'no_c'`
- `substring(12, 16)` = `'lien'`
- `substring(16, 20)` = `'ts_p'`
- `substring(20, 24)` = `'lz_2'`
- `substring(24, 28)` = `'eb02'`
- `substring(28, 32)` = `'b45}'`

### Step 5: Reconstruct the Password

I combined all the parts in order:
```
pico + CTF{ + no_c + lien + ts_p + lz_2 + eb02 + b45}
```

**Complete password/flag:** `picoCTF{no_clients_plz_2eb02b45}`

### Step 6: Verify the Password

I entered the password in the form and clicked "Verify". The page showed:
```
Password Verified
```

---

## Why This Works

* **Client-side validation is never secure** - all the code runs in the user's browser
* The password verification logic is visible in the JavaScript source code
* Anyone can read the source and reconstruct the password
* This challenge demonstrates why sensitive validation should ALWAYS happen on the server-side
* **Never trust the client** - users can view, modify, and bypass any client-side code

---

## Security Lesson

**Bad Practice (This Challenge):**
```javascript
// Client-side validation - INSECURE
if (password == "secret123") {
    alert("Access granted");
}
```

**Good Practice:**
```javascript
// Send to server for validation - SECURE
fetch('/api/login', {
    method: 'POST',
    body: JSON.stringify({password: userInput})
})
```

The server should:
- Validate credentials securely
- Use proper authentication mechanisms
- Never expose validation logic to the client

---

## Flag

`picoCTF{no_clients_plz_2eb02b45}`

---

## Tools Used

* **Web Browser** - Access the challenge
* **View Page Source (Ctrl+U)** - Inspect JavaScript code
* **Brain** - Analyze the validation logic

---

## Key Takeaways

* Client-side validation can be easily bypassed
* Never store or validate passwords in JavaScript
* Always inspect page source for client-side validation logic
* The hint "Never trust the client" is a fundamental security principle
* Sensitive operations must be performed server-side
* This challenge teaches why client-side security is an oxymoron
