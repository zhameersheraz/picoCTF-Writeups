# Insp3ct0r - picoCTF Writeup

**Challenge:** Inspect Me  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{tru3_d3t3ct1ve_0r_ju5t_lucky?302945a7}`

---

## Description

This website is hiding something. Can you find it by inspecting the page?

---

## Hints

1. The flag is split across multiple files
2. Check HTML, CSS, and JavaScript files
3. Use "View Page Source" or Developer Tools

---

## Solution

### Step 1: Open the Website

I navigated to the challenge URL:
```
http://fickle-tempest.picoctf.net:49923/
```

Or used:
```
view-source:http://fickle-tempest.picoctf.net:49923/
```

The page showed a simple website with two tabs: "What" and "How".

### Step 2: View Page Source (Find Part 1/3)

I pressed `Ctrl+U` to view the page source. Inside the HTML, I found a comment:
```html
<!-- Html is neat. Anyways have 1/3 of the flag: picoCTF{tru3_d3 -->
```

**Part 1/3:** `picoCTF{tru3_d3`

### Step 3: Check the CSS File (Find Part 2/3)

I noticed the HTML linked to an external CSS file:
```html
<link rel="stylesheet" type="text/css" href="mycss.css">
```

I clicked on `mycss.css` or navigated to:
```
http://fickle-tempest.picoctf.net:49923/mycss.css
```

At the bottom of the CSS file, I found a comment:
```css
/* You need CSS to make pretty pages. Here's part 2/3 of the flag: t3ct1ve_0r_ju5t */
```

**Part 2/3:** `t3ct1ve_0r_ju5t`

### Step 4: Check the JavaScript File (Find Part 3/3)

The HTML also linked to a JavaScript file:
```html
<script type="application/javascript" src="myjs.js"></script>
```

I clicked on `myjs.js` or navigated to:
```
http://fickle-tempest.picoctf.net:49923/myjs.js
```

At the bottom of the JavaScript file, I found:
```javascript
/* Javascript sure is neat. Anyways part 3/3 of the flag: _lucky?302945a7} */
```

**Part 3/3:** `_lucky?302945a7}`

### Step 5: Combine All Parts

I combined all three parts in order:
```
Part 1: picoCTF{tru3_d3
Part 2: t3ct1ve_0r_ju5t
Part 3: _lucky?302945a7}
```

**Complete Flag:** `picoCTF{tru3_d3t3ct1ve_0r_ju5t_lucky?302945a7}`

---

## Why This Works

* Web pages consist of three main components: **HTML** (structure), **CSS** (styling), and **JavaScript** (functionality)
* Each file can contain comments that are invisible to users but visible in source code
* Developers sometimes leave notes or sensitive information in these files
* **Always inspect all linked resources**, not just the main HTML page
* This challenge teaches you to thoroughly examine all parts of a website

---

## File Breakdown

| File | Purpose | Flag Part |
|------|---------|-----------|
| **index.html** | Page structure | Part 1/3: `picoCTF{tru3_d3` |
| **mycss.css** | Styling | Part 2/3: `t3ct1ve_0r_ju5t` |
| **myjs.js** | Functionality | Part 3/3: `_lucky?302945a7}` |

---

## Navigation Methods

### Method 1: View Source
```
1. Press Ctrl+U (View Page Source)
2. Look for linked files (href="mycss.css", src="myjs.js")
3. Click on each file to view its contents
```

### Method 2: Developer Tools
```
1. Press F12 (Open Developer Tools)
2. Go to "Sources" or "Debugger" tab
3. Expand the file tree to see all resources
4. Click on mycss.css and myjs.js to view contents
```

### Method 3: Direct URL Access
```
http://fickle-tempest.picoctf.net:49923/mycss.css
http://fickle-tempest.picoctf.net:49923/myjs.js
```

---

## Flag

`picoCTF{tru3_d3t3ct1ve_0r_ju5t_lucky?302945a7}`

---

## Tools Used

* **Web Browser** - Access the challenge
* **View Page Source (Ctrl+U)** - Inspect HTML and linked files
* **Browser Developer Tools (F12)** - Alternative inspection method

---

## Key Takeaways

* Always inspect **all** files used by a webpage (HTML, CSS, JS)
* Comments in code can contain sensitive information
* External resources (CSS, JS files) can hide important data
* Use "View Source" to see all linked files easily
* Multi-part flags teach you to be thorough in your inspection
* Never leave sensitive data in client-side comments
