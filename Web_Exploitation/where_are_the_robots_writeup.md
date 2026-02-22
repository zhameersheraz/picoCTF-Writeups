# where are the robots - picoCTF Writeup

**Challenge:** where are the robots  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{ca1cu1at1ng_Mach1n3s_cc6b1}`

---

## Description

Can you find the robots?

---

## Hints

(None)

---

## Solution

### Step 1: Open the Website

I navigated to the challenge URL:
```
http://fickle-tempest.picoctf.net:54038
```

The page displayed:
```
Welcome

Where are the robots?
```

This is a big hint! The challenge is asking about "robots" - this refers to the `robots.txt` file.

### Step 2: Check robots.txt

I navigated to the robots.txt file by adding `/robots.txt` to the URL:
```
http://fickle-tempest.picoctf.net:54038/robots.txt
```

I found:
```
User-agent: *
Disallow: /cc6b1.html
```

This tells me that there's a file called `cc6b1.html` that the website doesn't want search engines to index!

### Step 3: Visit the Disallowed Page

I changed the URL to visit the disallowed page:
```
http://fickle-tempest.picoctf.net:54038/cc6b1.html
```

The page displayed:
```
Guess you found the robots
picoCTF{ca1cu1at1ng_Mach1n3s_cc6b1}
```

---

## Why This Works

* **robots.txt** is a file that tells search engine crawlers which pages they should or shouldn't visit
* The `Disallow` directive tells crawlers not to index certain pages
* However, **robots.txt is publicly accessible** and anyone can read it
* Ironically, by trying to hide pages from search engines, robots.txt often reveals interesting or sensitive pages
* This is called "security through obscurity" and it doesn't work!

---

## What is robots.txt?

The `robots.txt` file is a standard used by websites to communicate with web crawlers and search engines:

**Format:**
```
User-agent: *           # Applies to all bots
Disallow: /admin/       # Don't crawl admin pages
Disallow: /secret.html  # Don't crawl this specific file
```

**Common uses:**
- Prevent search engines from indexing admin panels
- Hide duplicate content from search results
- Reduce server load by limiting what gets crawled
- Protect sensitive directories

**Security issue:**
- robots.txt is **not** a security measure
- Anyone can read it and see "hidden" pages
- Never rely on robots.txt to protect sensitive content

---

## robots.txt Syntax
```
# Allow all bots to crawl everything
User-agent: *
Disallow:

# Block all bots from everything
User-agent: *
Disallow: /

# Block specific directory
User-agent: *
Disallow: /admin/

# Block specific file
User-agent: *
Disallow: /secret.html

# Block specific bot
User-agent: Googlebot
Disallow: /private/
```

---

## Flag

`picoCTF{ca1cu1at1ng_Mach1n3s_cc6b1}`

---

## Tools Used

* **Web Browser** - Access the challenge and robots.txt

---

## Key Takeaways

* Always check `robots.txt` when investigating a website
* robots.txt often reveals hidden or interesting pages
* Never use robots.txt as a security measure
* robots.txt is publicly accessible - it's not for hiding sensitive content
* The challenge name "where are the robots" directly hints at checking robots.txt
* Security by obscurity doesn't work - use proper authentication instead
* If you don't want something indexed, don't put it on a public web server
