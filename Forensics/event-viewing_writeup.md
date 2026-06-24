# Event-Viewing — picoCTF Writeup

**Challenge:** Event-Viewing  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{Ev3nt_vi3wv3r_1s_a_pr3tty_us3ful_t00l_81ba3fe9}`  
**Platform:** picoCTF 2024 (picoGym Exclusive)  
**Writeup by:** zham  

---

## Description

> One of the employees at your company has their computer infected by malware! Turns out every time they try to switch on the computer, it shuts down right after they log in. The story given by the employee is as follows:
>
> 1. They installed software using an installer they downloaded online
> 2. They ran the installed software but it seemed to do nothing
> 3. Now every time they bootup and login to their computer, a black command prompt screen quickly opens and closes and their computer shuts down instantly.
>
> See if you can find evidence for each of these events and retrieve the flag (split into 3 pieces) from the correct logs!
>
> Download the Windows Log file here.

**Hint 1:** `Try to filter the logs with the right event ID`
**Hint 2:** `What could the software have done when it was ran that causes the shutdowns every time the system starts up?`

---

## Background Knowledge (Read This First!)

### What is a Windows Event Log (`.evtx`)?

Windows records everything that happens on the system in structured logs called **Event Logs**. They live in `C:\Windows\System32\winevt\Logs\` as `.evtx` files. The main channels:

- **System** — service starts/stops, driver loads, shutdowns
- **Application** — software installers (MSI), crashes
- **Security** — logons, process creation, registry changes, scheduled tasks (requires audit policy)

Every event has an **Event ID** — a small integer that tells you what kind of event it is. Knowing the right ID is the difference between finding your evidence in minutes vs. scrolling through thousands of records.

### Key Event IDs for This Challenge

| Event ID | Channel | Meaning |
|----------|---------|---------|
| **1033** | Application | MSI installer **started** installing a product |
| **1040** | Application | MSI installer **finished** installing a product |
| **1074** | System | User-initiated **shutdown/restart** (with comment in `param6`) |
| **4657** | Security | A **registry value was modified** |
| **4688** | Security | A **process was created** |
| **4698** | Security | A **scheduled task was created** |
| **4624** | Security | Successful **logon** |

Hint 1 explicitly tells us to filter by the right Event ID — so learning these is mandatory, not optional.

### How Do You Read an `.evtx` on Linux?

Windows has Event Viewer (`eventvwr.msc`). On Kali we use **python-evtx**, a Python library that parses `.evtx` files and dumps each record as XML. Install with:

```bash
sudo apt install python3-pip
pip3 install --break-system-packages python-evtx
```

Then iterate records like this:

```python
import Evtx.Evtx as evtx
with evtx.Evtx('log.evtx') as log:
    for record in log.records():
        print(record.xml())
```

Each record's XML has `<System>` (metadata like EventID, TimeCreated, Provider) and `<EventData>` (the actual fields).

### What is Base64 in Event Data?

The challenge author encoded each flag piece as a **Base64** string and stuffed it into a Data field that the Windows log API can carry any text in (like the comment on a shutdown, or a registry value name, or an MSI product field). Base64 looks like:

```
cGljb0NURntFdjNudF92aTN3djNyXw==
```

Decode with `base64 -d` or `echo ... | base64 -d`. Always be on the lookout for it — Windows logs love it.

---

## Solution — Step by Step

The challenge splits the flag across three events that match the employee's story. I hunt each piece in order.

### Step 1 — Set Up and Open the Log

I make a working folder and drop the `.evtx` in:

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/event-viewing && cd /work/event-viewing

┌──(zham㉿kali)-[/work/event-viewing]
└─$ cp /media/sf_downloads/Windows_Logs.evtx .

┌──(zham㉿kali)-[/work/event-viewing]
└─$ file Windows_Logs.evtx
Windows_Logs.evtx: MS Windows 10-11 Event Log, version 3.2, 99 chunks
```

I install the parser:

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ pip3 install --break-system-packages python-evtx
Successfully installed hexdump-3.3 python-evtx-0.8.1
```

### Step 2 — Get a Bird's-Eye View of Every Event ID

I write a small script that counts every Event ID in the log so I know what is in there:

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ nano scan.py
```

Paste:

```python
import Evtx.Evtx as evtx
import sys, re

counts = {}
with evtx.Evtx(sys.argv[1]) as log:
    for record in log.records():
        xml = record.xml()
        m = re.search(r'<EventID[^>]*>(\d+)</EventID>', xml)
        if m:
            counts[m.group(1)] = counts.get(m.group(1), 0) + 1

for eid, c in sorted(counts.items(), key=lambda x: -x[1]):
    print(f'{eid}: {c}')
```

Save with `Ctrl+O`, Enter, `Ctrl+X`, then run:

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ python3 scan.py Windows_Logs.evtx
1033: 13
1040: 13
4657: 1195
4688: 44
1074: 4
4698: 1
4624: 243
...
```

The interesting ones jump out: `1033`/`1040` for the installer story, `4657` for the registry tampering, and `1074` for the shutdowns.

### Step 3 — Piece 1: Find the Installer Event (Event ID 1033)

Story clue #1: "They installed software using an installer they downloaded online."

I filter for Event ID 1033 (MSI installer started):

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ nano find_1033.py
```

Paste:

```python
import Evtx.Evtx as evtx, sys, re

with evtx.Evtx(sys.argv[1]) as log:
    for record in log.records():
        xml = record.xml()
        if '<EventID>1033</EventID>' in xml:
            # Pretty-print the Data fields
            for d in re.findall(r'<Data[^>]*>([^<]+)</Data>', xml):
                print(d)
            print('---')
```

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ python3 find_1033.py Windows_Logs.evtx | grep -A1 'Totally_Legit_Software' | head -20
Totally_Legit_Software
1.3.3.7
0
0
cGljb0NURntFdjNudF92aTN3djNyXw==
```

The product is `Totally_Legit_Software` version `1.3.3.7` (lol). One of the Data fields holds `cGljb0NURntFdjNudF92aTN3djNyXw==`. I decode it:

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ echo cGljb0NURntFdjNudF92aTN3djNyXw== | base64 -d
picoCTF{Ev3nt_vi3wv3r_
```

**Piece 1 of the flag:** `picoCTF{Ev3nt_vi3wv3r_`

### Step 4 — Piece 2: Find What the Software Did After Installing (Event ID 4657)

Story clue #2: "They ran the installed software but it seemed to do nothing."

Hint 2 says "What could the software have done when it was ran that causes the shutdowns every time the system starts up?" — that is a classic **persistence-via-registry-Run-key** trick. So I look at Event ID 4657 (registry value modified) and search for our malware's path:

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ nano find_4657.py
```

Paste:

```python
import Evtx.Evtx as evtx, sys, re

with evtx.Evtx(sys.argv[1]) as log:
    for record in log.records():
        xml = record.xml()
        if '<EventID>4657</EventID>' in xml and 'Totally_Legit_Software' in xml:
            print(xml)
            print('===')
```

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ python3 find_4657.py Windows_Logs.evtx
```

The malware-added record shows:

```
<ObjectName>\REGISTRY\MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Run</ObjectName>
<ObjectValueName>Immediate Shutdown (MXNfYV9wcjN0dHlfdXMzZnVsXw==)</ObjectValueName>
<NewValue>C:\Program Files (x86)\Totally_Legit_Software\custom_shutdown.exe</NewValue>
<ProcessName>C:\Program Files (x86)\Totally_Legit_Software\Totally_Legit_Software.exe</ProcessName>
```

Confirmed attack chain: `Totally_Legit_Software.exe` ran, then wrote a value named `Immediate Shutdown (MXNfYV9wcjN0dHlfdXMzZnVsXw==)` to the **`HKLM\...\Run`** key, pointing at `custom_shutdown.exe`. Windows reads the `Run` key at every logon, so `custom_shutdown.exe` will fire next time the user signs in — instant shutdown.

Decode the Base64 inside the value name:

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ echo MXNfYV9wcjN0dHlfdXMzZnVsXw== | base64 -d
1s_a_pr3tty_us3ful_
```

**Piece 2 of the flag:** `1s_a_pr3tty_us3ful_`

### Step 5 — Piece 3: Find the Shutdown Event Itself (Event ID 1074)

Story clue #3: "Every time they bootup and login ... a black command prompt screen quickly opens and closes and their computer shuts down instantly."

Hint 1 — filter by Event ID. Event ID **1074** records user-initiated shutdowns, and the comment the attacker passes to `shutdown.exe` ends up in `param6`.

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ nano find_1074.py
```

Paste:

```python
import Evtx.Evtx as evtx, sys, re

with evtx.Evtx(sys.argv[1]) as log:
    for record in log.records():
        xml = record.xml()
        if '<EventID>1074</EventID>' in xml:
            print(xml)
            print('===')
```

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ python3 find_1074.py Windows_Logs.evtx | grep -E 'param[1-7]|shutdown'
```

Output:

```
<Data Name="param1">C:\Windows\system32\shutdown.exe (DESKTOP-EKVR84B)</Data>
<Data Name="param5">shutdown</Data>
<Data Name="param6">dDAwbF84MWJhM2ZlOX0=</Data>
```

`param6` is the comment string the attacker passed to `shutdown.exe /c "..."`. Decode it:

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ echo dDAwbF84MWJhM2ZlOX0= | base64 -d
t00l_81ba3fe9}
```

**Piece 3 of the flag:** `t00l_81ba3fe9}`

### Step 6 — Assemble the Flag

Concatenate in order (1 → 2 → 3):

```
┌──(zham㉿kali)-[/work/event-viewing]
└─$ echo -n 'picoCTF{Ev3nt_vi3wv3r_'; \
        echo -n '1s_a_pr3tty_us3ful_'; \
        echo -n 't00l_81ba3fe9}'
picoCTF{Ev3nt_vi3wv3r_1s_a_pr3tty_us3ful_t00l_81ba3fe9}
```

The flag is:

```
picoCTF{Ev3nt_vi3wv3r_1s_a_pr3tty_us3ful_t00l_81ba3fe9}
```

---

## Alternative Methods

### Method 1 — Windows Event Viewer (Native GUI)

If you happen to be on a Windows box, just double-click the `.evtx` file. Event Viewer opens it, then:

- Click **Filter Current Log** on the right pane
- Type the Event IDs `1033, 4657, 1074` into the filter box
- Look at the Data fields manually

Pros: zero install. Cons: you have to be on Windows.

### Method 2 — `evtx_dump` (Command-line XML dumper)

If you can install the `python-evtx` scripts properly:

```bash
evtx_dump.py Windows_Logs.evtx | grep -A2 'EventID>1033'
```

This dumps every record as XML on stdout, then `grep`/`xmllint` filter down to what you want. Slower than writing a 10-line Python script but works without coding.

### Method 3 — Microsoft PowerShell

```powershell
Get-WinEvent -Path .\Windows_Logs.evtx -FilterXPath '*[System[EventID=1033]]' |
    ForEach-Object { $_.Message }
```

PowerShell can parse `.evtx` natively. Same idea: filter by EventID, then inspect `Message` (which is the rendered version of all Data fields concatenated).

### Method 4 — Libsodium's `evtx` Rust Parser

There is a fast Rust-based parser called `evtx` by Omer Ben-Amram, plus a Rust binary called `evtx_dump`. If you parse very large logs regularly it is much faster than python-evtx, but the output format is identical.

---

## What Happened Internally (Timeline)

A clean reconstruction of the attack, pulled from the event timestamps:

1. **15:55:54 — User logs in** (`EventID 4624`, type 2 interactive logon by `DESKTOP-EKVR84B\user`).
2. **15:55:55 — MSI installer launched** (`EventID 1040` begin, `C:\Users\user\Desktop\Totally_Legit_Software.msi`).
3. **15:55:57 — Product recorded** (`EventID 1033`, `Totally_Legit_Software` v1.3.3.7, hidden Base64 in Data field). **Piece 1 of the flag** lives here.
4. **15:55:57 — Install complete** (`EventID 11707`, "Installation completed successfully").
5. **15:56:19 — Malware runs** (`EventID 4657`, `ProcessName = Totally_Legit_Software.exe`, `SubjectUserName = user`). The exe writes a new value under `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` named `Immediate Shutdown (MXNfYV9wcjN0dHlfdXMzZnVsXw==)` whose data is the path to `custom_shutdown.exe`. **Piece 2 of the flag** lives inside the value name.
6. **16:33:46 — Visual Studio installer creates a scheduled task** (`EventID 4698`, task `\Microsoft\VisualStudio\Updates\BackgroundDownload`) — this is a legitimate Visual Studio action, NOT part of the malware. Easy to mistake for malicious persistence. Don't fall for it.
7. **16:46:14 — User shuts down manually** (`EventID 1074`, `param1 = RuntimeBroker.exe`, `param5 = restart`). Restart to test.
8. **16:46:35 — Boot begins** (`EventID 4688` chain: `smss.exe` → `wininit.exe` → `services.exe` → `lsass.exe` → `winlogon.exe` → userinit).
9. **16:46:48 — User logs in again** (`EventID 4624`). The `Run` key fires. A black `cmd.exe` window opens (the "black command prompt screen quickly opens and closes"), runs `custom_shutdown.exe`, and...
10. **16:59:18 — Second manual shutdown** — same pattern, used to validate the persistence.
11. **17:01:05 — Third shutdown** — `param1 = C:\Windows\system32\shutdown.exe`, comment in `param6 = dDAwbF84MWJhM2ZlOX0=`. **Piece 3 of the flag** lives in the shutdown comment.
12. **17:02:35 — Fourth shutdown** — same payload, same comment. The attacker is taunting the user (or just confirming it works on every boot).

The whole attack is "install → set Run key → shut down next logon". The flag pieces are split across three different log channels (Application → Security → System), which is the puzzle: you have to know **which Event ID** corresponds to **which clue** in the story.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `python-evtx` | Parse the `.evtx` file and walk every record as XML | Medium |
| `python3` (custom scripts) | Filter by EventID, extract Data fields, decode Base64 | Easy |
| `base64` | Decode the flag pieces hidden inside Data fields | Easy |
| `file` | Confirm the input is a real Windows Event Log file | Easy |
| `grep` / `xmllint` (alt) | Filter XML output if you use `evtx_dump` | Easy |
| Windows Event Viewer (alt) | Native GUI, click-and-filter if you have Windows | Easy |
| PowerShell `Get-WinEvent` (alt) | Native PowerShell parsing with XPath filter | Medium |

To install everything on Kali:

```bash
sudo apt update
sudo apt install -y python3-pip
pip3 install --break-system-packages python-evtx
```

---

## Key Takeaways

- **Know your Event IDs.** The first hint literally tells you to filter by Event ID — so memorizing the common ones (1033/1040/11707 for installers, 1074 for shutdowns, 4657 for registry, 4688 for process create, 4698 for scheduled tasks, 4624/4625 for logon) is non-negotiable for Windows forensics.
- **Persistence = startup mechanism.** "Every time the system starts up" almost always means one of three things: a `Run`/`RunOnce` registry key, a scheduled task, or a service. Event IDs 4657 (registry) and 4698 (task) are your first stops.
- **Base64 hides everywhere.** Windows event logs happily carry arbitrary strings in their Data fields, and Base64 is the universal "wrap arbitrary bytes as text" trick. If you see `==` at the end of a long string in a log, try decoding it.
- **Read filenames, not just process paths.** The malware's process path is `Totally_Legit_Software\Totally_Legit_Software.exe` and the payload it drops is `custom_shutdown.exe` — both designed to look boring. The evidence that gives them away is the registry value name with the Base64 chunk in it.
- **Not every suspicious event is malicious.** Event 4698 was a *legitimate* Visual Studio scheduled task, not the malware's persistence. Read the task content (`Command` field) — `BackgroundDownload.exe` from `Microsoft Visual Studio\Installer\...` is a real VS component.
- **The flag wordplay is the punchline.** `Ev3nt_vi3wv3r_1s_a_pr3tty_us3ful_t00l` decodes to **"Event Viewer is a pretty useful tool"** — which is exactly the lesson: with the right Event IDs and a bit of Base64 decoding, Event Viewer (or its CLI equivalents) tells you everything you need to reconstruct the attack.
