# Inspect HTML - picoCTF Writeup

**Challenge:** Inspect HTML  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{1n5p3t0r_0f_h7ml_fd5d57bd}`

---

## Description

This challenge provides a simple webpage and asks whether you can find the hidden flag. The task is to inspect the website and discover anything unusual within the page source.

---

## Hints

1. The challenge title itself is a big hint: "Inspect HTML"
2. Try viewing the page source
3. Look for HTML comments

---

## Solution

### Step 1: Open the Website

I navigated to the provided challenge URL:
```
http://saturn.picoctf.net:64474/
```

The page displayed a short passage about Histiaeus along with a source citation. Nothing on the visible page indicated the presence of a flag.

### Step 2: Inspect the Page Source

To view the HTML source, I:
* Right-clicked anywhere on the webpage
* Selected **Inspect** (or pressed **F12** to open Developer Tools)

### Step 3: Search Through the HTML

Inside the HTML structure, I found a hidden comment near the bottom of the page:
```html
<!-- picoCTF{1n5p3t0r_0f_h7ml_fd5d57bd} -->
```

### Step 4: Capture the Flag

The flag was directly embedded inside the HTML comment. I copied it exactly as shown.

---

## Why This Works

* **HTML comments are invisible to users** but remain accessible in the source code
* Developers sometimes forget to remove test data, notes, or sensitive information before deployment
* Many beginner CTF challenges rely on basic inspection skills to build foundational web security knowledge
* This teaches the importance of never storing sensitive data in client-side code

---

## Alternative Methods

You can also view the page source by:
* Right-click → "View Page Source"
* Press `Ctrl+U` (Windows/Linux) or `Cmd+Option+U` (Mac)
* Add `view-source:` before the URL in your browser

---

## Flag

`picoCTF{1n5p3t0r_0f_h7ml_fd5d57bd}`

---

## Tools Used

* **Browser Developer Tools (F12)** - Inspect webpage source
* **Web Browser** - Access the challenge instance

---

## Key Takeaways

* Always check the HTML source code when investigating web pages
* HTML comments can contain sensitive information
* Basic inspection is a fundamental skill in web security
* Never trust client-side code to hide secrets - anything sent to the browser can be viewed
