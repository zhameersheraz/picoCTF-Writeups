# logon - picoCTF Writeup

**Challenge:** logon  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{th3_c0nsp1r4cy_l1v3s_4d184b0d}`

---

## Description

The factory is hiding things from all of its users. Can you login as Joe and find what they've been looking at?

---

## Hints

1. Hmm it doesn't seem to check anyone's password, except for Joe's?

---

## Solution

### Step 1: Open the Website

I navigated to the challenge URL:
```
http://fickle-tempest.picoctf.net:57775
```

The page displayed a login form:
```
Factory Login
[Username field]
[Password field]
[Sign In button]
```

### Step 2: Try Logging In as Joe

Based on the challenge description, I need to login as Joe. I entered:

- **Username:** `Joe`
- **Password:** (any password, like `password`)

Then clicked **Sign In**.

### Step 3: Check the Response

After logging in, I got a green success message:
```
Success: You logged in! Not sure you'll be able to see the flag though.
```

But the page showed:
```
No flag for you
```

This means I'm logged in, but I don't have permission to see the flag!

### Step 4: Inspect the Application Storage

I pressed `F12` to open Developer Tools and went to the **Application** tab.

Under **Storage** → **Cookies** → `http://fickle-tempest.picoctf.net:57775`, I found these cookies:

- **Name:** `admin` | **Value:** `False`
- **Name:** `password` | **Value:** (some value)
- **Name:** `username` | **Value:** (some value)

The key cookie here is `admin` with value `False`!

### Step 5: Change Admin Cookie to True

I double-clicked on the **Value** field of the `admin` cookie and changed it from:
```
False  →  True
```

### Step 6: Refresh the Page

I pressed `F5` to refresh the page.

### Step 7: Capture the Flag

After refreshing with `admin=True`, the page now displayed:
```
Flag: picoCTF{th3_c0nsp1r4cy_l1v3s_4d184b0d}
```

---

## Why This Works

* The website uses **client-side authentication** to check admin status
* After login, it sets an `admin` cookie to `False`
* The page checks this cookie to determine if you can see the flag
* Since cookies are stored in your browser, **you have full control** over them
* By changing `admin` from `False` to `True`, we bypass the access control
* **Never trust client-side authorization** - always validate permissions server-side

---

## The Security Flaw

**Bad Practice (This Challenge):**
```javascript
// Client-side check - INSECURE
if (getCookie("admin") == "True") {
    showFlag();
}
```

**Good Practice:**
```javascript
// Server-side check - SECURE
fetch('/api/get-flag', {
    headers: { 'Authorization': 'Bearer ' + sessionToken }
})
// Server validates if user has admin privileges
```

The problem:
- Authorization decisions are made in the browser
- Users can modify cookies freely
- No server-side validation of admin status

---

## Steps to Modify Cookies

### Method 1: Browser DevTools (Used in this challenge)
```
1. Press F12 to open Developer Tools
2. Go to Application tab (Chrome) or Storage tab (Firefox)
3. Expand Cookies → Select the website
4. Find the "admin" cookie
5. Double-click the Value field
6. Change "False" to "True"
7. Refresh the page (F5)
```

### Method 2: Using Console
```javascript
// Set admin cookie to true
document.cookie = "admin=True";

// Refresh the page
location.reload();
```

---

## Flag

`picoCTF{th3_c0nsp1r4cy_l1v3s_4d184b0d}`

---

## Tools Used

* **Web Browser** - Access the challenge
* **Browser Developer Tools (F12)** - Inspect and modify cookies
* **Application/Storage Tab** - View and edit cookie values

---

## Key Takeaways

* Client-side authorization checks are fundamentally insecure
* Users can modify any data stored in their browser (cookies, local storage, etc.)
* Never trust client-side data for security decisions
* Authorization must always be validated server-side
* Cookies are easily manipulated - never use them alone for access control
* This demonstrates why authentication ≠ authorization
* Always verify user permissions on the server before granting access
