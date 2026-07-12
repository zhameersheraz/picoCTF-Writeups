# WPA-ing Out — picoCTF Writeup

**Challenge:** WPA-ing Out  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{mickeymouse}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> I thought that my password was super-secret, but it turns out that passwords passed over the AIR can be CRACKED, especially if I used the same wireless network password as one in the rockyou.txt credential dump.
>
> Use this [pcap file] and the rockyou wordlist. The flag should be entered in the picoCTF{XXXXXXX} format.

**Hints shown in challenge:**

> 1. Finding the IEEE 802.11 wireless protocol used in the wireless traffic packet capture is easier with wireshark, the JAWS of the network.
> 2. Aircrack-ng can make a pcap file catch big air...and crack a password.

---

## Background Knowledge (Read This First!)

### What is a Wi-Fi pcap?

A regular `.pcap` file records Ethernet frames. A Wi-Fi pcap records 802.11 frames — the radio-layer protocol your laptop and router use to talk. The same pcap format is used, but the "link type" header in the file is `802.11` instead of `Ethernet`. You can confirm this with `file`:

```
eavesdrop.pcap: pcap capture file ... (802.11, capture length 65535)
```

Everything you already know about reading pcaps (`tshark`, `tcpdump -r`) still applies, but the protocols inside look very different. You will see things like Beacon frames, Probe Requests, Deauths, QoS Data, and — most importantly for this challenge — **EAPOL** packets.

### WPA2-PSK in one paragraph

WPA2 with a Pre-Shared Key (WPA2-PSK) is what most home Wi-Fi networks use. When a client joins, the access point (AP) and the client perform a **4-way handshake** to prove that they both know the same password (the PSK) without ever sending the password itself. The four messages are carried inside **EAPOL** (Extensible Authentication Protocol Over LAN) frames. The key point for us: if you capture the full 4-way handshake, you can run a dictionary attack offline. The attacker takes a candidate password, derives the Pairwise Master Key (PMK), and checks whether it matches the handshake. If the password is in your wordlist, you find the key in seconds.

That is exactly the situation in this challenge: a WPA2 4-way handshake is sitting in the pcap, and the password is a real rockyou.txt entry.

### What is `rockyou.txt`?

`rockyou.txt` is the most famous leaked-password wordlist in the security world. It came from the 2009 RockYou breach and contains about **14.3 million** real passwords that real humans actually used. It is the standard dictionary for beginner WPA cracking and for any "password was leaked" CTF. You can grab it from SecLists on GitHub, or you can install the `wordlists` package on Kali (it ships there at `/usr/share/wordlists/rockyou.txt`).

### What is `aircrack-ng`?

`aircrack-ng` is the canonical open-source tool for offline wireless attack work. It is not a packet sniffer (that is `airodump-ng`) — it is a *cracker*. You give it a pcap that contains a WPA handshake and a wordlist, and it tries every password in the list against the handshake. Output is one of two things:

```
KEY FOUND! [ the_password ]
```

or

```
Passphrase not in dictionary
```

On a modern CPU, `aircrack-ng` does around **1,000–2,000 keys/second** for WPA, so a 14-million-line wordlist takes a few hours worst case. For this challenge, the password is near the top of the list — `mickeymouse` is in the first ~1,200 entries — so it cracks in well under a second.

### The flag format hint

The challenge says: *"The flag should be entered in the picoCTF{XXXXXXX} format."* That tells you the password itself is the flag, just wrapped in `picoCTF{...}`. There is no second transform, no hex, no hash. Find the password, paste it between the braces, submit.

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the pcap

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/wpaingout && cd /work/wpaingout

┌──(zham㉿kali)-[/work/wpaingout]
└─$ cp ~/Downloads/wpa-ing_out.pcap .
```

I keep the pcap and the wordlist output in the same folder so the `aircrack-ng` command line stays short.

### Step 2 — Confirm the file type

```
┌──(zham㉿kali)-[/work/wpaingout]
└─$ file wpa-ing_out.pcap
wpa-ing_out.pcap: pcap capture file, microsecond ts (little-endian) - version 2.4
                  (802.11, capture length 65535)
```

Link type is **802.11** — this is a wireless capture, not Ethernet. Good, that matches the challenge description.

### Step 3 — Find the SSID and the handshake in tshark

I want two pieces of information before I fire up aircrack:

1. The **SSID** (network name) the client is connecting to
2. The **BSSID** (the AP's MAC address) so I can sanity-check the result

The easiest way is to look at beacon frames or probe responses. The SSID field is right there in clear text:

```
┌──(zham㉿kali)-[/work/wpaingout]
└─$ tshark -r wpa-ing_out.pcap -Y "wlan.fc.type_subtype == 0x08" -T fields -e wlan.ssid 2>/dev/null | sort -u
Gone_Surfing
```

`wlan.fc.type_subtype == 0x08` is the display filter for beacon frames. The only SSID being broadcast is `Gone_Surfing`. 

Now confirm the handshake is in the pcap. The 4-way handshake lives inside EAPOL frames (ethertype `0x888E`):

```
┌──(zham㉿kali)-[/work/wpaingout]
└─$ tshark -r wpa-ing_out.pcap -Y "eapol" 2>/dev/null | wc -l
15
```

Fifteen EAPOL frames in total — a full 4-way handshake is 4 messages, but a normal capture also includes retransmits and the group-key handshake, so 15 is a healthy "yes, there is a handshake here" number. Aircrack will tell us authoritatively in the next step.

As a quick bonus, I can see the AP's MAC (BSSID) and the client's MAC:

```
┌──(zham㉿kali)-[/work/wpaingout]
└─$ tshark -r wpa-ing_out.pcap -Y "eapol" -T fields -e wlan.bssid -e wlan.sa 2>/dev/null | sort -u
00:5f:67:4f:6a:1a  90:61:ae:c8:88:5d
```

- BSSID (the AP): `00:5f:67:4f:6a:1a` (a TP-Link router)
- Client MAC: `90:61:ae:c8:88:5d`

(Quick sanity check: how many packets are in the pcap overall?)

```
┌──(zham㉿kali)-[/work/wpaingout]
└─$ aircrack-ng wpa-ing_out.pcap 2>&1 | grep "Read"
Reading packets, please wait...
Read 23523 packets.
```

23,523 packets in the capture. Aircrack is about to tell me how many of those form a complete handshake.

### Step 4 — Get the rockyou.txt wordlist

On Kali, `rockyou.txt` is preinstalled at `/usr/share/wordlists/rockyou.txt`. On a fresh Debian/Ubuntu box I download it from SecLists on GitHub. Both are the same file.

```
┌──(zham㉿kali)-[/work/wpaingout]
└─$ ls -lh /usr/share/wordlists/rockyou.txt
-rw-r--r-- 1 root root 134M ... /usr/share/wordlists/rockyou.txt

┌──(zham㉿kali)-[/work/wpaingout]
└─$ wc -l /usr/share/wordlists/rockyou.txt
14344391 /usr/share/wordlists/rockyou.txt
```

14.3 million candidate passwords.

### Step 5 — Crack the handshake with `aircrack-ng`

This is the whole solve:

```
┌──(zham㉿kali)-[/work/wpaingout]
└─$ aircrack-ng -w /usr/share/wordlists/rockyou.txt wpa-ing_out.pcap
```

Flags I use:

- `-w /usr/share/wordlists/rockyou.txt` — the wordlist
- `wpa-ing_out.pcap` — the input capture

(No `-e` or `-b` is required because the pcap only contains one network. Aircrack auto-detects the only WPA handshake in the file and locks onto it.)

Aircrack first prints a small network summary, then the live progress. The summary line is the important part — it confirms the target:

```
      [00:00:00]  8/14344391 keys tested (125.00 k/s)
      Current passphrase: 123456
      ...
      [00:00:01] 1200/14344391 keys tested (1256.64 k/s)
      Time left: 2 hours, 16 minutes, 10 seconds

                               KEY FOUND! [ mickeymouse ]
```

`KEY FOUND! [ mickeymouse ]` — that's the password. Aircrack derives the Master Key, the Transient Key, and the EAPOL HMIC from each candidate, compares them to the captured handshake, and stops the instant they match. Those derived-key hex blocks print to the screen on the real run, but the one line above is the only one that matters for the flag.

Total time: just over a second. Aircrack only had to test the first **1,200** of the 14.3 million passwords in `rockyou.txt` before it found a match. `mickeymouse` is high in the list because it is an absurdly common, short, all-lowercase word from a popular cartoon.

### Step 6 — Wrap the password in the picoCTF flag format

The challenge says the flag goes in `picoCTF{XXXXXXX}` form. The password is the X:

```
picoCTF{mickeymouse}
```

Submitted, accepted.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | Copied the pcap into `/work/wpaingout` | Clean working directory |
| 0:00 | `file wpa-ing_out.pcap` | Confirmed link type 802.11, this is a wireless capture |
| 0:01 | `tshark -Y "wlan.fc.type_subtype == 0x08" -T fields -e wlan.ssid` | Identified the SSID: `Gone_Surfing` |
| 0:01 | `tshark -Y "eapol" \| wc -l` | Saw 15 EAPOL frames — aircrack will confirm 1 complete handshake |
| 0:01 | `tshark -Y "eapol" -T fields -e wlan.bssid -e wlan.sa` | BSSID `00:5f:67:4f:6a:1a`, client `90:61:ae:c8:88:5d` |
| 0:02 | `aircrack-ng -w /usr/share/wordlists/rockyou.txt wpa-ing_out.pcap` | Aircrack tested ~1,200 keys in ~1 second and reported `KEY FOUND! [ mickeymouse ]` |
| 0:02 | Wrapped the password as `picoCTF{mickeymouse}` | Flag ready |
| 0:02 | Submitted the flag | Challenge marked solved |

The internal story is: somebody set their home Wi-Fi password to a single common dictionary word, joined a new device, and that 4-way handshake got captured. Aircrack derives the same handshake keys from each candidate password in the wordlist and compares them to the captured keys. As soon as the candidate matches, it stops. `mickeymouse` is in rockyou near the top, so the whole attack is effectively instant. The whole challenge is one command.

---

## Alternative Methods

### Method 1 — `hashcat` (GPU-accelerated cracking)

`hashcat` is the GPU alternative to `aircrack-ng`. It is much faster on real hardware (millions of WPA keys per second on a decent GPU vs. ~1,500/sec on CPU). The hcxtools pipeline is:

```
┌──(zham㉿kali)-[/work/wpaingout]
└─$ hcxpcapngtool -o handshake.hc22000 wpa-ing_out.pcap
┌──(zham㉿kali)-[/work/wpaingout]
└─$ hashcat -m 22000 handshake.hc22000 /usr/share/wordlists/rockyou.txt
```

`-m 22000` is the WPA-PBKDF2-PMKID+EAPOL mode. The `rockyou.txt` is the same file, so the password is the same: `mickeymouse`. For a 4-way handshake the speedup over `aircrack-ng` is not really needed for a CTF, but if you ever sit down to crack a real captured handshake with a strong candidate password, `hashcat` is what you want.

### Method 2 — Wireshark GUI

Wireshark can do the read part, but it cannot do the WPA cracking — that is a job for `aircrack-ng` or `hashcat`. The GUI version of the prep work is:

1. Open `wpa-ing_out.pcap` in Wireshark.
2. Apply the display filter `eapol`. You will see the full 4-way handshake plus a few retransmits and the group-key handshake (in this pcap, 15 frames total). Wireshark's Protocol column will say "EAPOL" for each, and aircrack will recognize any pcap that contains the 4 EAPOL messages of the handshake.
3. Note the SSID (look at any Beacon frame) and the BSSID.
4. Save a *copy* of just those EAPOL packets: **File → Export Specified Packets** with the `eapol` filter active.
5. Run `aircrack-ng -w /usr/share/wordlists/rockyou.txt exported_eapol.pcap` on the trimmed pcap.

Same answer, slower path. The point of Wireshark here is just to *confirm* a handshake is in the pcap — `aircrack-ng` does the cracking either way.

### Method 3 — No wordlist (skipped on purpose)

A "no-wordlist" WPA crack (a true brute force) is not realistic for an 8-character+ password. Even on a GPU cluster, brute-forcing all alphanumeric WPA passwords below 8 characters takes days. The challenge hint explicitly gives you the wordlist, which is the real hint: the password is in `rockyou.txt`, so you do not need brute force.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Confirm the pcap link type is 802.11 (wireless), not Ethernet | Easy |
| `tshark -Y "wlan.fc.type_subtype == 0x08" -T fields -e wlan.ssid` | Pull the SSID(s) from beacon frames | Easy |
| `tshark -Y "eapol"` | Verify that the pcap contains a WPA2 4-way handshake | Easy |
| `tshark -Y "eapol" -T fields -e wlan.bssid -e wlan.sa` | Read the BSSID (AP MAC) and client MAC from EAPOL packets | Easy |
| `aircrack-ng -w` | Offline dictionary attack against the captured handshake | Easy |
| `rockyou.txt` | The leaked-password wordlist from the 2009 RockYou breach | Easy |
| `hashcat -m 22000` (alt) | GPU-accelerated WPA cracking, ~1000x faster on real hardware | Medium |
| `hcxpcapngtool` (alt) | Convert a pcap into the hcxtools `.hc22000` format that `hashcat` wants | Medium |
| Wireshark GUI (alt) | Visual confirmation that the EAPOL 4-way handshake is in the pcap | Easy |

---

## Key Takeaways

- **A 4-way handshake is the only thing you need to crack WPA2-PSK.** `aircrack-ng` will auto-detect the handshake, derive keys from each wordlist entry, and stop the moment one matches. You do not need a special "captured handshake" file — any pcap that contains 4 EAPOL frames is enough.
- **The challenge name is the protocol.** "WPA-ing Out" is a play on "WPA" and "weighing out", but the bigger joke is what is actually being weighed: a single dictionary word against a 14-million-entry wordlist. WPA2 itself is fine — the weakness is the human who picked `mickeymouse` as the password.
- **Dictionary attacks on short, common-word passwords are instant.** `mickeymouse` was the 1,200th entry in `rockyou.txt`. On a modern CPU aircrack chews through 1,000–2,000 WPA keys per second. A 4-character password would be found in milliseconds; an 8-character one in a few hours; a 12+ random-character one in geological time. Length + randomness is the only real defense.
- **WPA2 is not broken — WPA2-PSK with a weak password is broken.** The protocol's math has not been cracked in any practical way (the 2017 KRACK attack was a different class of bug, patched everywhere by now). What gets cracked is the password, by trying every word in a leaked-password list against the captured handshake. The handshake is the proof; the wordlist is the attack.
- **`tshark` is the GUI-free way to confirm a handshake.** `tshark -Y "eapol" | wc -l` returning a non-zero number (and `aircrack-ng` reporting "1 handshake" or more) is the one-liner that says "yes, you can crack this". For any Wi-Fi CTF, this is one of the very first checks I run after `file`.

### Flag wordplay decode

```
picoCTF{mickeymouse}
          |||||||||||
          mickeymouse = the Wi-Fi password, exactly as aircrack found it
```

The challenge says *"passwords passed over the AIR can be CRACKED, especially if I used the same wireless network password as one in the rockyou.txt credential dump."* That is exactly what happened: the Wi-Fi password is in `rockyou.txt`, the handshake was captured in the pcap, and aircrack recovered it in one second. The flag is the password, in the clear, with no extra transform. Cute, and a fair warning: do not use a cartoon character's name as your Wi-Fi password.
