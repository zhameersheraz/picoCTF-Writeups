# Includes - picoCTF Writeup

**Challenge:** Includes  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{1nclu51v17y_1of2_f7w_2of2_df589022}`

---

## Description

Can you get the flag?

Go to this website and see what you can discover.

---

## Hints

1. Is there more code than what the inspector initially shows?

---

## Solution

### Step 1: Open the Website

I opened the challenge URL and checked the webpage. It looked normal and did not show the flag.

---

### Step 2: View Page Source

I pressed `Ctrl + U` to view the page source. Inside the HTML, I noticed the site was loading external files:

```html
<link rel="stylesheet" href="style.css">
<script src="script.js"></script>
```

This suggested that the flag might be hidden inside those files.

---

### Step 3: Check style.css

I opened **style.css** directly in the browser. At the bottom of the file, I found part of the flag inside a comment:

```css
/* picoCTF{1nclu51v17y_1of2_ */
```

---

### Step 4: Check script.js

Next, I opened **script.js** and found the remaining part of the flag:

```javascript
// f7w_2of2_df589022}
```

---

### Step 5: Combine the Flag

After combining both parts, the complete flag is:

```
picoCTF{1nclu51v17y_1of2_f7w_2of2_df589022}
```

---

## Why This Works

Websites often load external resources such as CSS and JavaScript files. These files must be accessible to the browser, which means anyone can open and inspect them.

In this challenge, the flag was hidden inside comments across two separate files. Checking only the main HTML page would not reveal it.

---

## Alternative Method

You can also find these files using **Inspect Element**:

1. Press `F12`
2. Go to the **Sources** tab
3. Open `style.css` and `script.js`
4. Look for comments containing the flag

---

## Flag

`picoCTF{1nclu51v17y_1of2_f7w_2of2_df589022}`

---

## Tools Used

* Web Browser  
* View Page Source  
* Inspect Element  

---

## Key Takeaways

* Always check external files like CSS and JavaScript  
* Flags are commonly hidden inside comments  
* Do not focus only on the HTML page  
* Anything sent to the browser can be viewed
