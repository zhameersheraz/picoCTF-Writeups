# Unminify - picoCTF Writeup

**Challenge:** Unminify  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{pr3tty_c0d3_622b2c88}`

---

## Description

I don't like scrolling down to read the code of my website, so I've squished it. As a bonus, my pages load faster!

Browse here, and find the flag!

---

## Hints

1. The code is "squished" - this means it's been minified
2. The website says "I just deliver flags, I don't know how to read them..."
3. Try using developer tools or a proxy to see the full response

---

## Solution

### Step 1: Open the Website

I navigated to the challenge URL:
```
http://titan.picoctf.net:55983/
```

The page displayed:
```
Welcome to my flag distribution website!

If you're reading this, your browser has successfully received the flag.

I just deliver flags, I don't know how to read them...
```

This is a hint - the flag was already sent to my browser!

### Step 2: Try Inspecting the Page Source

I tried viewing the page source with `Ctrl+U`, but the code was all squished together on one line (minified). I couldn't easily spot the flag.

### Step 3: Use Burp Suite to Intercept

I used Burp Suite to see the full HTTP response:

1. Open **Burp Suite** and go to the **Proxy** tab
2. Turn on **Intercept**
3. Open **Burp's browser** and navigate to the URL
4. Go to **Target** tab in Burp Suite
5. Find `titan.picoctf.net:55983` in the site map (bottom section)
6. **Right-click** on it → **Send to Repeater**
7. Go to **Repeater** tab
8. Click **Send** button
9. Scroll down in the **Response** section

### Step 4: Find the Flag

In the response body, I searched through the minified JavaScript code (or used `Ctrl+F` to search for "picoCTF{") and found:
```
picoCTF{pr3tty_c0d3_622b2c88}
```

---

## Why This Works

* **Minification** removes spaces and line breaks to make code smaller and faster to load
* It makes code harder to read, but doesn't hide it
* The flag is embedded in the JavaScript code sent to the browser
* **Client-side code cannot hide secrets** - anything sent to the browser can be viewed
* Burp Suite helps analyze HTTP traffic and view responses clearly

---

## Alternative Method

You can also solve this using just the browser:
* Right-click → "View Page Source" (or press `Ctrl+U`)
* Press `Ctrl+F` and search for `picoCTF{`
* The flag will be found in the minified code

---

## Flag

`picoCTF{pr3tty_c0d3_622b2c88}`

---

## Tools Used

* **Burp Suite** - Intercept and analyze HTTP requests/responses
* **Web Browser** - Access the challenge instance

---

## Key Takeaways

* Minified code is harder to read but not secure
* Never store sensitive data (like flags) in client-side code
* Burp Suite is a powerful tool for web traffic analysis
* Always check page source and JavaScript - flags can be hidden there
* Anything sent to the browser can always be viewed by the user
