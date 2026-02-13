# Cookies - picoCTF Writeup

**Challenge:** Cookies  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{3v3ry1_l0v3s_c00k135_a4dadb49}`

---

## Description

Who doesn't love cookies? Try to figure out the best one.

---

## Hints

(None)

---

## Solution

### Step 1: Open the Website

I navigated to the challenge URL:
```
http://wily-courier.picoctf.net:61983/
```

The page displayed a cookie search page with the text:
```
Welcome to my cookie search page. See how much I like different kinds of cookies!
```

There was a search box with "snickerdoodle" already typed in.

### Step 2: Search for Snickerdoodle

I clicked the **Search** button and the page displayed:
```
I love snickerdoodle cookies!
```

But no flag appeared. This means I need to try different cookies.

### Step 3: Inspect the Cookies

I pressed `F12` to open Developer Tools and went to the **Application** tab (or **Storage** tab in Firefox).

Under **Storage** → **Cookies** → `http://wily-courier.picoctf.net:61983/`, I found a cookie:

- **Name:** `name`
- **Value:** `0`

### Step 4: Change Cookie Values

I realized the cookie value controls which cookie type is displayed. I started changing the value manually:

- **Value: 0** → "I love snickerdoodle cookies!"
- **Value: 1** → "I love chocolate chip cookies!"
- **Value: 2** → "I love oatmeal raisin cookies!"
- **Value: 3** → "I love peanut butter cookies!"
- ...and so on

After each change, I refreshed the page (`F5`) to see the new result.

### Step 5: Find the Flag

I continued incrementing the cookie value:
```
0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18...
```

When I set the cookie value to **18** and refreshed the page, the flag appeared:
```
Flag: picoCTF{3v3ry1_l0v3s_c00k135_a4dadb49}
```

---

## Why This Works

* **Cookies** are small pieces of data stored in your browser by websites
* They can store session information, preferences, or tracking data
* Developers sometimes use cookies to control application behavior
* In this challenge, the cookie value determines which response is shown
* By manipulating the cookie value, we can access different pages/responses
* **Never trust client-side data** - users can modify cookies at any time

---

## What are Cookies?

Cookies are key-value pairs stored by your browser for each website:
* **Session cookies:** Temporary, deleted when browser closes
* **Persistent cookies:** Stored for a set time period
* **Security note:** Sensitive data should never be stored in cookies without encryption

**Common cookie uses:**
- Login sessions
- Shopping cart data
- User preferences
- Tracking/analytics

---

## Steps to Modify Cookies

### Method 1: Browser DevTools (Used in this challenge)
```
1. Press F12 to open Developer Tools
2. Go to Application tab (Chrome) or Storage tab (Firefox)
3. Expand Cookies → Select the website
4. Double-click the Value field to edit
5. Change the value
6. Refresh the page (F5)
```

### Method 2: Using Console
```javascript
// Set a cookie
document.cookie = "name=18";

// View all cookies
console.log(document.cookie);
```

---

## Flag

`picoCTF{3v3ry1_l0v3s_c00k135_a4dadb49}`

---

## Tools Used

* **Web Browser** - Access the challenge
* **Browser Developer Tools (F12)** - Inspect and modify cookies
* **Application/Storage Tab** - View and edit cookie values

---

## Key Takeaways

* Cookies can be easily viewed and modified by users
* Never store sensitive data or make security decisions based on cookies alone
* Cookie values should be validated server-side
* Users have full control over their cookies
* Always inspect cookies when investigating web applications
* Incrementing values (0, 1, 2, ...) is a common testing technique for finding hidden content
