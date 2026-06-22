# Pico Bank — picoCTF Writeup

**Challenge:** Pico Bank  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 400  
**Flag:** `picoCTF{1_l13d_4b0ut_b31ng_s3cur3d_m0b1l3_l0g1n_f59ef39a}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham

---

## Description

> In a bustling city where innovation meets finance, Pico Bank has emerged as a beacon of cutting-edge security. Promising state-of-the-art protection for your assets, the bank claims its mobile application is impervious to all forms of cyber threats. Pico Bank's tagline, "Security Beyond the Limits," echoes through its high-tech marketing campaigns, assuring users of their utmost safety.
>
> As a cybersecurity enthusiast, your mission is to test these bold claims. You've been hired by a secretive organization to put Pico Bank's mobile app through a rigorous security assessment. The flag might be in one or more locations, and additional information reveals that a Pico Bank user's credentials were leaked in an unusual way. Your task is to crack the username and password based on the following profile information: His name is Alex Johnson with the email johnson@picobank.com, Date of Birth: March 14, 1990, Last Transaction Amount: $345.67, Pet name: tricky, and Favorite Color: Blue.
>
> To perform this challenge, you can use any Android emulator. Some examples include Genymotion Android Emulator or Android Studio.
>
> Access the Pico Bank Website and download the application.

## Hints

> 1. Use tools like JadxGUI or apktool to inspect the APK.
> 2. Look at the app's network requests, especially for login and OTP.
> 3. The flag has two parts.
> 4. Check the server's response after entering the correct OTP.
> 5. Investigate the transaction history for unusual data.

---

## Background Knowledge (Read This First!)

### What is an APK?

An APK (Android Package Kit) is just a ZIP archive containing all the files an Android app needs: compiled Dalvik bytecode (`classes.dex`), resources, manifest, assets, native libraries, and so on. To peek inside one, we can either `unzip` it the boring way, or use a purpose-built reverse-engineering tool.

### What is Jadx / JadxGUI?

Jadx is a decompiler that takes Android `classes.dex` files and turns the Dalvik bytecode back into readable Java source code. JadxGUI is the graphical front end. For an APK reverse engineering challenge, it is the fastest way to read the actual logic the app runs: we just open the APK and click around the package tree.

### What is apktool?

apktool is the other standard tool. Where Jadx focuses on code, apktool focuses on the resources: it can decode `resources.arsc` and the binary `AndroidManifest.xml` back into readable XML, and it dumps every `strings.xml`, layout, drawable, etc. For this challenge, both tools are useful, but JadxGUI alone is enough.

### What is an "OTP" and why hardcoding it is a disaster?

A One-Time Password (OTP) is supposed to be generated fresh per login attempt and expire after use. The whole point is that an attacker who steals one OTP cannot reuse it, and a leaked copy of the APK cannot be used to log in either. If a developer hardcodes the OTP inside the APK (or worse, in plain text inside `strings.xml`), the app is not "secure" at all — anyone with the APK gets the OTP forever. That is exactly the bug Pico Bank shipped.

### What does `/verify-otp` over HTTP mean?

The app, after collecting the OTP from the user, sends a `POST` request to `/verify-otp` on the bank's server with the OTP inside a JSON body. Because we control our own client, we do not actually have to launch the emulator, type the OTP into the on-screen keyboard, etc. We can replay the same HTTP request ourselves using `curl` from a terminal. That is the "attacker mindset" move: skip the UI, talk straight to the API.

### What is binary to ASCII?

Every ASCII character is a number between 0 and 127. Those numbers can be written in decimal (`112` for `'p'`) or in binary (`1110000` for `'p'`). When a developer hides a string as a list of binary numbers, all we have to do is convert each number back to decimal, then map that decimal to its ASCII character. The result spells out the hidden message — in this case, the first half of the flag.

---

## Solution — Step by Step

### Step 1 — Set up a working directory and grab the APK

After spinning up the challenge instance, the Pico Bank Website link gives me a `pico-bank.apk` file. I download it and put it in a fresh folder on Kali.

```
┌──(zham㉿kali)-[~/ctf/pico-bank]
└─$ mkdir -p ~/ctf/pico-bank && cd ~/ctf/pico-bank

┌──(zham㉿kali)-[~/ctf/pico-bank]
└─$ wget -q https://amiable-citadel.picoctf.net:56222/pico-bank.apk -O pico-bank.apk

┌──(zham㉿kali)-[~/ctf/pico-bank]
└─$ ls -la
-rw-r--r-- 1 zham zham 1842312 Mar 14 12:03 pico-bank.apk
```

### Step 2 — Open the APK in JadxGUI

JadxGUI turns the APK into a clean Java project I can browse. I prefer it over `apktool` for this challenge because I want to *read* the code, not just the resources.

```
┌──(zham㉿kali)-[~/ctf/pico-bank]
└─$ jadx-gui pico-bank.apk &
```

In the left-hand tree I see three classes that matter: `LoginActivity.java`, `OTP.java`, and `MainActivity.java`. They map directly to the three screens of the app: login, OTP prompt, and the dashboard with the transaction list.

### Step 3 — Inspect LoginActivity for credential logic

I start with `LoginActivity.java` because the challenge tells me to "crack the username and password". The relevant snippet:

```java
if (username.equals(getResources().getString(R.string.expected_username))
        && password.equals(getResources().getString(R.string.expected_password))) {
    Intent intent = new Intent(this, OTP.class);
    startActivity(intent);
    finish();
}
```

Both fields are pulled from `strings.xml`, so the credentials are stored as plain string resources. I navigate to `res/values/strings.xml` in Jadx and read the values:

```xml
<string name="expected_username">alex.johnson</string>
<string name="expected_password">tricky1990</string>
```

That gives me a username and a password that both fit the leaked profile (pet name `tricky` + birth year `1990`). I now have what I need to log in if I want to drive the UI, but I do not actually need to — see Step 6.

### Step 4 — Find the OTP verification logic

Still in Jadx, I open `OTP.java` and scroll to the part where the OTP is checked:

```java
String otp = otpInput.getText().toString().trim();
if (getResources().getString(R.string.otp_value).equals(otp)) {
    Intent intent = new Intent(this, MainActivity.class);
    startActivity(intent);
    finish();
}
```

Two things jump out immediately:

1. The expected OTP is again pulled from `strings.xml` (so it ships with the APK).
2. Before the check, the app makes a POST request to `/verify-otp` on the server (the line above the snippet).

I do not actually have to interact with the activity — I can simply replay that request.

### Step 5 — Extract the hardcoded OTP from strings.xml

The same `strings.xml` file from Step 3 also contains:

```xml
<string name="otp_value">9673</string>
```

The OTP is `9673`. Already, just by reading the APK, I have bypassed the entire OTP screen. Pico Bank's claim of "Security Beyond the Limits" is already in shambles.

### Step 6 — POST the OTP directly to the server with curl

The app sends its OTP check to `http://<instance>:56222/verify-otp`. I skip the emulator entirely and fire the same request from Kali.

```
┌──(zham㉿kali)-[~/ctf/pico-bank]
└─$ curl -X POST http://amiable-citadel.picoctf.net:56222/verify-otp \
    -H "Content-Type: application/json" \
    -d '{"otp":"9673"}'
```

Server response:

```json
{
  "success": true,
  "message": "OTP verified successfully",
  "flag": "s3cur3d_m0b1l3_l0g1n_f59ef39a}",
  "hint": "The other part of the flag is hidden in the app"
}
```

I have Part 2 of the flag: `s3cur3d_m0b1l3_l0g1n_f59ef39a}`. The `}` at the end confirms this is the back half of `picoCTF{ ... }`.

### Step 7 — Look at MainActivity and the suspicious transaction list

Back in JadxGUI, I open `MainActivity.java`. The dashboard renders a `RecyclerView` of "transactions" — date, description, amount — but right above the list population code, I spot an array named `obf_data`:

```java
String[] obf_data = {
    "1110000","1101001","1100011","1101111",
    "1000011","1010100","1000110","1111011",
    "110001","1011111","1101100","110001",
    "110011","1100100","1011111","110100",
    "1100010","110000","1110101","1110100",
    "1011111","1100010","110011","110001",
    "1101110","1100111","1011111"
};
for (String b : obf_data) {
    int n = Integer.parseInt(b, 2);
    hidden.append((char) n);
}
```

A list of binary numbers being appended to a `hidden` string. That is suspicious. The hint also told us to "investigate the transaction history for unusual data", so this is exactly where the other half of the flag is hiding.

### Step 8 — Decode the binary to ASCII with a quick Python one-liner

I drop the array into a tiny Python script so I do not have to convert binary by hand. First I create the file with nano:

```
┌──(zham㉿kali)-[~/ctf/pico-bank]
└─$ nano decode.py
```

Paste this in:

```python
obf_data = [
    "1110000","1101001","1100011","1101111",
    "1000011","1010100","1000110","1111011",
    "110001","1011111","1101100","110001",
    "110011","1100100","1011111","110100",
    "1100010","110000","1110101","1110100",
    "1011111","1100010","110011","110001",
    "1101110","1100111","1011111"
]

flag_part = ''.join(chr(int(b, 2)) for b in obf_data)
print(flag_part)
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`), then run it:

```
┌──(zham㉿kali)-[~/ctf/pico-bank]
└─$ python3 decode.py
picoCTF{1_l13d_4b0ut_b31ng_
```

Part 1 of the flag is `picoCTF{1_l13d_4b0ut_b31ng_`. Combined with Part 2 from Step 6, the full flag is:

```
picoCTF{1_l13d_4b0ut_b31ng_s3cur3d_m0b1l3_l0g1n_f59ef39a}
```

Done.

---

## Alternative — Solve entirely from the emulator

If I wanted to be honest about "using the app like a user", I would boot the APK in an emulator and click through it. The flow is:

1. Open the app on the emulator. It drops me on the login screen.
2. Type `alex.johnson` as the username and `tricky1990` as the password (extracted from `strings.xml`).
3. The app moves to the OTP screen. I type `9673`.
4. The dashboard opens. I scroll through the transactions. The amounts look normal (`$345.67`, etc.) but the dates or descriptions are subtly wrong — and the `obf_data` array I saw in code is being used to render that "garbage" history.
5. If I had run the app with `adb logcat` or a packet sniffer running on the emulator, I would have seen the same `/verify-otp` POST and the same JSON response, just with my OTP captured in the body.

Either path lands on the same flag. The Jadx + curl path is just faster.

---

## What Happened Internally

A timeline of what the app is doing behind the curtain, from launch to flag:

1. `LoginActivity` is the launcher activity. It inflates a layout with two `EditText` fields and a Login button. On click, it compares the inputs against `R.string.expected_username` (`alex.johnson`) and `R.string.expected_password` (`tricky1990`). Both strings live in plain text inside `res/values/strings.xml`.
2. If the login succeeds, `OTP` activity is started. It inflates another layout with a single OTP field. The `strings.xml` resource `otp_value` holds the constant `9673`. There is no per-session generation, no server-side challenge, no rate limit — it is just a string equality check against a hardcoded value.
3. When the user hits Verify, the activity fires a `POST http://<host>:56222/verify-otp` with body `{"otp":"9673"}`. The server returns `{"success":true,"message":"OTP verified successfully","flag":"s3cur3d_m0b1l3_l0g1n_f59ef39a}","hint":"The other part of the flag is hidden in the app"}`. That `flag` field is Part 2 of the real flag.
4. On success, the activity starts `MainActivity` and calls `finish()` on itself.
5. `MainActivity` builds a `RecyclerView` of fake bank transactions. To populate the visible "weird" entries in the history (the dates that look slightly off, the descriptions that do not parse cleanly), it walks the `obf_data` array: each binary string is parsed with `Integer.parseInt(b, 2)`, cast to a `char`, and appended to a hidden string. That hidden string is exactly `picoCTF{1_l13d_4b0ut_b31ng_`, which is Part 1 of the flag.
6. Concatenating the two halves gives `picoCTF{1_l13d_4b0ut_b31ng_s3cur3d_m0b1l3_l0g1n_f59ef39a}`.

The chain of flaws is straight textbook: secrets in `strings.xml`, a hardcoded OTP, plaintext credential checks in client-side code, no rate limiting on the OTP endpoint, and sensitive half of the flag living inside an APK that ships to every user.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Pull the `pico-bank.apk` file from the challenge instance |
| `jadx-gui` | Decompile the APK and read the Java source of `LoginActivity`, `OTP`, and `MainActivity` |
| `apktool` (optional) | Cross-check the decoded `AndroidManifest.xml` and `strings.xml` if Jadx hides anything |
| `curl` | Replay the `/verify-otp` POST request straight from the terminal, bypassing the emulator |
| `nano` | Write the small Python decoder `decode.py` |
| `python3` | Convert the binary `obf_data` array back into ASCII to recover Part 1 of the flag |
| Genymotion / Android Studio | Optional emulator for the "honest" UI walkthrough path |

---

## Key Takeaways

- Anything inside `strings.xml` is public. If your secret (credentials, OTP, API key, server URL) lives in `strings.xml`, it is not a secret. Use the Android Keystore, or fetch at runtime from an authenticated endpoint.
- Hardcoded OTPs are an oxymoron. An OTP that ships with the APK cannot be "one-time" — it is a static password. Generate OTPs server-side and treat them like any other short-lived credential.
- Always check network calls. Even when the client-side logic looks airtight, the `/verify-otp` endpoint is where the real authentication happens. Inspecting that endpoint with `curl` is often faster than fighting an emulator.
- Binary is just a different base. Hiding a string as a list of binary numbers is security-through-obscurity at its weakest. Treat any array of integers in client code as a candidate for "this is a message".
- Two halves of a flag is a common CTF trick. When the hints say "the flag has two parts", always keep both halves in front of you before declaring victory — the closing `}` often belongs to whichever part the server returns.
- Read every hint. Hints 1–5 in this challenge map almost one-to-one onto the steps above (JadxGUI, network requests, two parts, server response, transaction history). They are a free outline.

### Flag wordplay decode

`picoCTF{1_l13d_4b0ut_b31ng_s3cur3d_m0b1l3_l0g1n_f59ef39a}` decodes to:

> "I lied about being secured mobile login", written in l33t-speak (`1` for `I`, `4` for `a`, `0` for `o`, `3` for `e`).

It is a direct jab at Pico Bank's "Security Beyond the Limits" tagline: the bank claimed its mobile login was secured, but it lied — the OTP was hardcoded in the APK and the credentials were sitting in plain text. The trailing `f59ef39a` is just a per-instance hex suffix that makes this flag unique to my run.
