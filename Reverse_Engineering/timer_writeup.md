# Timer — picoCTF Writeup

**Challenge:** Timer  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{t1m3r_r3v3rs3d_succ355fully_17496}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> You will find the flag after analysing this apk
>
> Download [here](https://artifacts.picoctf.net/c/449/timer.apk).

## Hints

> 1. Decompile
> 2. mobsf or jadx

---

## Background Knowledge (Read This First!)

### What is an APK?

An APK (Android Package Kit) is the file format Android uses to distribute and install apps. Think of it as Android's version of a `.zip` or a `.tar.gz`. In fact, an APK is literally just a ZIP archive with a specific internal layout. If you rename `timer.apk` to `timer.zip` and unzip it, you get the raw contents the Android system would normally install for you.

That little fact is the entire key to this challenge.

### What is inside an APK?

When you unzip an APK you will see something like:

```
AndroidManifest.xml      # a binary XML describing the app (name, permissions, etc.)
classes.dex              # the compiled app code (Dalvik bytecode)
classes2.dex             # more compiled code if the app is big
classes3.dex             # even more compiled code
resources.arsc           # a binary table of strings the app uses
res/                     # icons, layouts, drawables, etc.
META-INF/                # signature files
kotlin/                  # Kotlin standard library classes (if the app uses Kotlin)
```

The interesting bit is `classes*.dex`. Android apps are written in Java or Kotlin, compiled to JVM `.class` files, then re-compiled by the `dx`/`d8` tool into **DEX** (Dalvik Executable) bytecode that the Dalvik or ART runtime on the phone can actually execute. A single DEX file can pack many classes together.

### How do you read a DEX file?

A DEX file is binary. You have a few options:

- **`strings <file>`** — print every printable ASCII sequence in the binary. Fastest way to spot a flag that the developer left as a plain string somewhere.
- **`grep -r picoCTF <dir>`** — after unzipping, search recursively through every extracted file for the flag prefix.
- **Decompile with `jadx`** — turn DEX back into readable Java/Kotlin source. Heavier tool, but lets you see what the app actually does, not just the strings.
- **Decompile with `apktool` / `mobsf`** — turn DEX into smali (Android assembly) plus the original resource layout. Even heavier, used when you need to repackage and modify an APK.

### Why does the flag live in `classes3.dex`?

The app was written in **Kotlin** (you can tell because there is a `kotlin/` folder in the APK, and the decompiled class file is annotated with `@Metadata`). Because Kotlin adds the Kotlin standard library on top of the app's own code, a small app like this one actually produces more than one DEX file: the standard library lives in `classes.dex` / `classes2.dex`, and the app's own code lives in `classes3.dex`. The flag string was placed in the app's `BuildConfig.VERSION_NAME` field, which is part of the app's own code, which is why it ends up in `classes3.dex`.

### What is `BuildConfig.VERSION_NAME`?

Android Studio auto-generates a `BuildConfig.java` class for every app. By default `VERSION_NAME` is set to whatever string you put under `versionName` in `build.gradle`. Developers sometimes hide Easter eggs in there, and that is exactly what the challenge author did: the flag was dropped into `versionName "picoCTF{...}"`.

---

## Solution — Step by Step

### Step 1 — Download and confirm the file

```
┌──(zham㉿kali)-[~/ctf]
└─$ wget https://artifacts.picoctf.net/c/449/timer.apk
--2025-01-01 00:00:00--  https://artifacts.picoctf.net/c/449/timer.apk
Resolving artifacts.picoctf.net (artifacts.picoctf.net)... 172.64.155.69, ...
Connecting to artifacts.picoctf.net (artifacts.picoctf.net)|172.64.155.69|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4678467 (4.5M) [application/octet-stream]
Saving to: 'timer.apk'

timer.apk                100%[===================>]   4.46M  --.-K/s   in 0.1s

2025-01-01 00:00:00 (47.4 MB/s) - 'timer.apk' saved [4678467/4678467]
```

A quick `file` check confirms what we have:

```
┌──(zham㉿kali)-[~/ctf]
└─$ file timer.apk
timer.apk: Android package (APK), with ZIP compression
```

The output literally tells us: this is a ZIP file with a fancy name. We do not even need `apktool` or `jadx` to open it.

### Step 2 — Unzip the APK

```
┌──(zham㉿kali)-[~/ctf]
└─$ mkdir timer_extracted
┌──(zham㉿kali)-[~/ctf]
└─$ cd timer_extracted
┌──(zham㉿kali)-[~/ctf/timer_extracted]
└─$ unzip ../timer.apk
Archive:  ../timer.apk
  inflating: AndroidManifest.xml
  inflating: classes.dex
  inflating: classes2.dex
  inflating: classes3.dex
   creating: META-INF/
   creating: META-INF/com/android/build/gradle/
  inflating: META-INF/com/android/build/gradle/app-metadata.properties
   creating: kotlin/
   creating: res/
   ...
```

Check what we got:

```
┌──(zham㉿kali)-[~/ctf/timer_extracted]
└─$ ls -la
total 8731
drwxr-xr-x  5 zham zham    4096 Jan  1 00:00 .
drwxr-xr-x  4 zham zham    4096 Jan  1 00:00 ..
-rw-r--r--  1 zham zham    3300 Jan  1  1981 AndroidManifest.xml
drwxr-xr-x  3 zham zham    4096 Jan  1 00:00 META-INF
-rw-r--r--  1 zham zham 7591452 Jan  1  1981 classes.dex
-rw-r--r--  1 zham zham  472460 Jan  1  1981 classes2.dex
-rw-r--r--  1 zham zham    6848 Jan  1  1981 classes3.dex
drwxr-xr-x  3 zham zham    4096 Jan  1 00:00 kotlin
drwxr-xr-x 44 zham zham    4096 Jan  1 00:00 res
-rw-r--r--  1 zham zham  854796 Jan  1  1981 resources.arsc
```

Three DEX files. The smallest one (`classes3.dex`, only 6.8 KB) is almost always the app's own code when the app is small — the bulk of the bytes in the bigger DEX files comes from the Kotlin/AndroidX standard library.

### Step 3 — Hunt for the flag

The fastest approach is to grep every extracted file for the flag prefix. Since `classes*.dex` are binary files, regular `grep` will just say "binary file matches" without showing the line. That is fine for narrowing down the file, but we want to actually see the flag string.

```
┌──(zham㉿kali)-[~/ctf/timer_extracted]
└─$ grep -rl picoCTF .
./classes3.dex
```

One hit: `classes3.dex`. Now let us read the line with `strings` so we see the actual flag:

```
┌──(zham㉿kali)-[~/ctf/timer_extracted]
└─$ strings classes3.dex | grep picoCTF
*picoCTF{t1m3r_r3v3rs3d_succ355fully_17496}
```

There it is: `picoCTF{t1m3r_r3v3rs3d_succ355fully_17496}`. The `*` in front of `picoCTF{...}` is just `strings` being a little overzealous about including a leading punctuation character from the byte before the printable sequence — it is not part of the flag.

If you want to see where in the file the flag lives (useful when you have a giant binary), add the `-t d` flag to print decimal offsets:

```
┌──(zham㉿kali)-[~/ctf/timer_extracted]
└─$ strings -t d classes3.dex | grep picoCTF
   5647 *picoCTF{t1m3r_r3v3rs3d_succ355fully_17496}
```

That is the whole solve. Submit the flag and you are done.

---

## Alternative — Decompile with jadx (the hinted approach)

The hints specifically call out `jadx` and `mobsf`, so let me also walk through the jadx path. The string-search trick above is faster, but the decompile approach is more instructive because you actually see the Kotlin source the developer wrote.

### Step 1 — Install jadx

```
┌──(zham㉿kali)-[~/ctf]
└─$ wget https://github.com/skylot/jadx/releases/download/v1.5.0/jadx-1.5.0.zip
┌──(zham㉿kali)-[~/ctf]
└─$ unzip -q jadx-1.5.0.zip -d jadx
┌──(zham㉿kali)-[~/ctf]
└─$ ls jadx/bin/
jadx  jadx-gui  jadx-gui.bat  jadx.bat
```

### Step 2 — Decompile the APK

```
┌──(zham㉿kali)-[~/ctf]
└─$ jadx/bin/jadx -d jadx_out timer.apk
INFO  - loading ...
INFO  - processing ...
INFO  - progress: 1987 of 1987 (100%)
ERROR - finished with errors, count: 3
```

(Those three errors are normal — jadx could not fully resolve every method reference in the third-party Kotlin stdlib classes, but the app's own code is fine.)

### Step 3 — Look at the decompiled sources

```
┌──(zham㉿kali)-[~/ctf]
└─$ ls jadx_out/sources/com/example/timer/
BuildConfig.java  MainActivity.java  R.java
```

Open `BuildConfig.java`:

```
┌──(zham㉿kali)-[~/ctf]
└─$ cat jadx_out/sources/com/example/timer/BuildConfig.java
package com.example.timer;

/* loaded from: classes3.dex */
public final class BuildConfig {
    public static final String APPLICATION_ID = "com.example.timer";
    public static final String BUILD_TYPE = "debug";
    public static final boolean DEBUG = Boolean.parseBoolean("true");
    public static final int VERSION_CODE = 1;
    public static final String VERSION_NAME = "picoCTF{t1m3r_r3v3rs3d_succ355fully_17496}";
}
```

`VERSION_NAME` is the flag. Same answer, just a more "official" way to get there.

### Step 4 — Look at MainActivity for context

While we are here, let me look at `MainActivity.java` to understand what the app actually does:

```
┌──(zham㉿kali)-[~/ctf]
└─$ head -25 jadx_out/sources/com/example/timer/MainActivity.java
package com.example.timer;

import android.os.Bundle;
import android.os.CountDownTimer;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;
import android.widget.TextView;
import androidx.appcompat.app.AppCompatActivity;

public final class MainActivity extends AppCompatActivity {
    public EditText hours;
    public EditText minutes;
    public Button playbtn;
    public EditText seconds;
    public TextView textView;

    // ...onCreate wires up the EditTexts and the play button...
```

The interesting bit is what the play button does:

```java
private static final void m30onCreate$lambda0(MainActivity this$0, View it) {
    String hoursentered = this$0.getHours().getText().toString();
    String minutesentered = this$0.getMinutes().getText().toString();
    String secondsentered = this$0.getSeconds().getText().toString();
    int actual_seconds =
        (Integer.parseInt(hoursentered) * 3600)
      + (Integer.parseInt(minutesentered) * 60)
      + Integer.parseInt(secondsentered);
    this$0.startCountingDown(actual_seconds * 1000);
}
```

So the app is a simple countdown timer: you type hours/minutes/seconds, you press play, and it ticks down using `android.os.CountDownTimer`. There is no flag-check logic in the running app at all. The flag was never meant to appear on screen — the challenge is purely "find this string that the author hid in the binary".

The countdown logic itself lives in `startCountingDown`:

```java
private final void startCountingDown(int starttime) {
    final long j = starttime;
    new CountDownTimer(j) {
        @Override public void onTick(long millisUntilFinished) {
            long seconds_remaining = millisUntilFinished / 1000;
            // ...split seconds_remaining into h/m/s and update the EditTexts...
        }

        @Override public void onFinish() {
            MainActivity.this.getTextView().setText("done!");
        }
    }.start();
}
```

When the timer reaches zero the app just prints "done!". Nothing more. No flag, no validation, nothing to brute force. The flag lives only in the APK on disk.

---

## What Happened Internally

A timeline of what the app and the binary actually do behind the curtain:

1. The Android developer wrote a tiny Kotlin app with three `EditText` boxes (hours, minutes, seconds), a `Button`, and a `TextView`. The whole thing lives in `MainActivity.kt`.
2. They also set `versionName "picoCTF{t1m3r_r3v3rs3d_succ355fully_17496}"` in `build.gradle`. Android Studio auto-generated `BuildConfig.java` from that, baking the flag string into the compiled output.
3. At build time, `kotlinc` compiled `MainActivity.kt` to JVM `.class` files. The Kotlin standard library was compiled too. The Android Gradle plugin then ran `d8` to convert everything to DEX bytecode.
4. Because the standard library is much bigger than the app itself, `d8` split the output into multiple DEX files: `classes.dex` and `classes2.dex` ended up with the Kotlin/AndroidX library code, and `classes3.dex` ended up with the app's own classes (`MainActivity`, `BuildConfig`, `R`).
5. Everything was zipped into `timer.apk`, with `AndroidManifest.xml` at the root describing the package and activities.
6. We downloaded the APK, treated it like a ZIP (because it is one), extracted the three DEX files, and ran `strings` on them. The flag was sitting in `classes3.dex` as a literal UTF-8 string.
7. Submitting the flag completes the challenge.

Note that the actual running app has zero involvement in the flag — the user is never asked to enter anything flag-related. The challenge is a "find the secret the developer hid in the binary" exercise, not a "crack the running app" exercise.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download `timer.apk` from the picoCTF artifact server |
| `file` | Confirm the download is really an APK/ZIP and not a corrupted file |
| `unzip` | Extract `AndroidManifest.xml`, `classes*.dex`, `resources.arsc`, etc. out of the APK (no special APK tool needed — APK is just a ZIP) |
| `ls` | List the extracted files and confirm the layout |
| `grep -rl` | Recursively find which file inside the extracted folder contains the flag prefix |
| `strings` | Pull every printable ASCII sequence out of a binary DEX file and surface the flag string |
| `jadx` | Decompile the APK back into readable Java source code so we can confirm `BuildConfig.VERSION_NAME` is where the flag lives (alternative solve) |
| `cat` | Read the decompiled `BuildConfig.java` to see the flag inline |

---

## Key Takeaways

- An APK is just a ZIP with a different extension. You do not always need `apktool`, `jadx`, or `mobsf` to peek inside one — `unzip` plus `strings`/`grep` is enough for a lot of beginner reverse engineering.
- `strings <file> | grep picoCTF` is the single fastest move for any CTF challenge that hands you a binary and hides a flag inside. Always try it first.
- When a binary has multiple sections (here, three DEX files), start with the smallest one if the bigger files are obviously libraries. Small = the challenge author's own code = where the flag lives.
- Apps written in Kotlin end up with a `kotlin/` folder inside the APK and typically more than one `classes*.dex` because the Kotlin stdlib is large. Knowing this saves you from wasting time grepping the wrong DEX file.
- `BuildConfig.java` is generated by Android Studio and embeds values like `APPLICATION_ID`, `VERSION_CODE`, `VERSION_NAME`, and `DEBUG` as static `public static final` fields. Any string the developer puts into `versionName` ends up baked into the APK's `classes*.dex` for everyone to read.
- Decompilation tools like `jadx` are not magic. They are just `dex2jar` + a Java decompiler under the hood. They shine when the flag is hidden behind runtime logic, but here the flag is a literal string, so the lighter `strings` approach wins.
- The hints said "Decompile" and "mobsf or jadx". They were not the only path. PicoCTF hints are nudges, not mandatory tool requirements. If a faster method gets you the flag, that is fine — but it is still worth doing the hinted approach at least once so you learn the "official" workflow.

### Flag wordplay decode

`picoCTF{t1m3r_r3v3rs3d_succ355fully_17496}` decodes to **"timer reversed successfully"**, written in l33t-speak:

- `t1m3r` -> "timer" (`1` for `i`, `3` for `e`)
- `r3v3rs3d` -> "reversed" (`3` for `e` three times)
- `succ355fully` -> "successfully" (`3` for `e`, `55` for `ss`)
- `17496` -> the unique hex suffix that makes this flag different from anyone else's copy of the challenge. (`17496` in decimal is `0x4458`, the ASCII bytes `D` `X` — possibly a fun little Easter egg, possibly just a random ID.)

So the whole flag is bragging that we successfully reversed the timer app. The joke being: there was no timer logic to reverse at all, we just `strings`'d the binary.
