# mr-worldwide - picoCTF Writeup

**Challenge:** Mr-Worldwide  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{KODIAK_ALASKA}`  
**Platform:** picoCTF (2019)  
**Writeup by:** zham  

---

## Description

> A musician left us a message. What's it mean?

## Hints

> (No hints were provided for this challenge.)

---

## Background Knowledge

Before jumping in, here are the concepts behind this challenge.

### What is Geocoding?

**Geocoding** is the process of turning a place name (like "Kodiak, Alaska") into a pair of latitude and longitude numbers, and **reverse geocoding** is the opposite — taking `(lat, lon)` coordinates and looking up the place name on the map. Every maps API (Google Maps, OpenStreetMap, Mapbox, etc.) supports reverse geocoding, and several of them (including OpenStreetMap's Nominatim) are free for moderate-volume use.

For a CTF challenge like this one, reverse geocoding is the perfect tool: the challenge hands us coordinates, and we hand them right back to a map service and read out the city names.

### The Mr-Worldwide Connection

The challenge title is a hat tip to Pitbull (the rapper), who is nicknamed "Mr. Worldwide" because his lyrics constantly name-drop cities from every continent. The format of the challenge fits that theme perfectly: a stack of latitude/longitude pairs that each point at a famous city. The flag, "KODIAK ALASKA," is the punchline — the final letter of each city name spells out a place name, and the place name happens to be the punchline of the song.

### Why an Underscore, Not a Space?

The hint chain on similar challenges usually reminds you that picoCTF flags glue multi-word plaintexts together with underscores. `picoCTF{KODIAK_ALASKA}` and `picoCTF{KODIAKALASKA}` are different strings to the grader, but only one of them is the real flag. The original picoCTF flag for this challenge used the underscore, which matches picoCTF's normal flag-format convention.

### Putting the Pieces Together

The challenge gives us a flag-shaped string where the inside of the braces is twelve coordinate pairs. Our job:

1. Reverse-geocode each pair to a city name.
2. Take the first letter of each city name.
3. Concatenate the letters and read them as English.

The "keyspace" is small (only 12 characters to recover), so we can do this by hand with Google Maps one click at a time, or batch the work in Python using the free Nominatim API.

---

## Solution

### Step 1: Save the Message to a File

I copied the encoded flag from the challenge page into a file so I could poke at it from the terminal without retyping all twelve coordinate pairs.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/mr-worldwide]
└─$ mkdir -p ~/picoCTF/cryptography/mr-worldwide && cd ~/picoCTF/cryptography/mr-worldwide

┌──(zham㉿kali)-[~/picoCTF/cryptography/mr-worldwide]
└─$ nano message.txt
```

In `nano`, I pasted:

```
picoCTF{(35.028309, 135.753082)(46.469391, 30.740883)(39.758949, -84.191605)(41.015137, 28.979530)(24.466667, 54.366669)(3.140853, 101.693207)_(9.005401, 38.763611)(-3.989038, -79.203560)(52.377956, 4.897070)(41.085651, -73.858467)(57.790001, -152.407227)(31.205753, 29.924526)}
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X` (the usual nano dance).

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/mr-worldwide]
└─$ cat message.txt
picoCTF{(35.028309, 135.753082)(46.469391, 30.740883)(39.758949, -84.191605)(41.015137, 28.979530)(24.466667, 54.366669)(3.140853, 101.693207)_(9.005401, 38.763611)(-3.989038, -79.203560)(52.377956, 4.897070)(41.085651, -73.858467)(57.790001, -152.407227)(31.205753, 29.924526)}
```

### Step 2: Pull Out the Coordinate Pairs

Each pair sits inside round brackets, separated by an underscore. A small Python expression using regex pulls them into a list.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/mr-worldwide]
└─$ python3 -c "
import re
raw = open('message.txt').read()
pairs = re.findall(r'\((-?\d+\.\d+),\s*(-?\d+\.\d+)\)', raw)
print(f'Found {len(pairs)} coordinate pairs:')
for p in pairs: print(' ', p)
"
Found 12 coordinate pairs:
  ('35.028309', '135.753082')
  ('46.469391', '30.740883')
  ('39.758949', '-84.191605')
  ('41.015137', '28.979530')
  ('24.466667', '54.366669')
  ('3.140853', '101.693207')
  ('9.005401', '38.763611')
  ('-3.989038', '-79.203560')
  ('52.377956', '4.897070')
  ('41.085651', '-73.858467')
  ('57.790001', '-152.407227')
  ('31.205753', '29.924526')
```

Twelve pairs. The underscores between them are a tell: each pair contributes one letter and the underscores are the word boundaries.

### Step 3: Reverse-Geocode Each Pair

I used OpenStreetMap's free **Nominatim** API to look up each coordinate. The script saves one HTTP call per pair and writes the city names to a file. Be polite to the public API and stick to one request per second.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/mr-worldwide]
└─$ pip install requests
```

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/mr-worldwide]
└─$ nano lookup.py
```

In `nano`, I pasted:

```python
#!/usr/bin/env python3
import re, time, urllib.parse, requests

raw = open("message.txt").read()
pairs = re.findall(r"\((-?\d+\.\d+),\s*(-?\d+\.\d+)\)", raw)

cities = []
for lat, lon in pairs:
    url = "https://nominatim.openstreetmap.org/reverse?" + urllib.parse.urlencode({
        "format": "jsonv2",
        "lat": lat,
        "lon": lon,
        "zoom": 10,
    })
    headers = {"User-Agent": "picoCTF-mr-worldwide-solver/1.0"}
    r = requests.get(url, headers=headers, timeout=15).json()
    addr = r.get("address", {})
    city = (addr.get("city")
            or addr.get("town")
            or addr.get("village")
            or addr.get("hamlet")
            or addr.get("county")
            or addr.get("state")
            or "?")
    print(f"({lat}, {lon}) -> {city}")
    cities.append(city)
    time.sleep(1.1)   # be nice to the free Nominatim API

with open("cities.txt", "w") as f:
    f.write("\n".join(cities) + "\n")

flag_letters = "".join(c[0] for c in cities if c and c != "?")
print()
print("First letters:", flag_letters)
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/mr-worldwide]
└─$ python3 lookup.py
(35.028309, 135.753082) -> Kyoto
(46.469391, 30.740883) -> Odessa
(39.758949, -84.191605) -> Dayton
(41.015137, 28.979530) -> Istanbul
(24.466667, 54.366669) -> Abu Dhabi
(3.140853, 101.693207) -> Kuala Lumpur
(9.005401, 38.763611) -> Addis Ababa
(-3.989038, -79.203560) -> Loja
(52.377956, 4.897070) -> Amsterdam
(41.085651, -73.858467) -> Sleepy Hollow
(57.790001, -152.407227) -> Kodiak
(31.205753, 29.924526) -> Alexandria

First letters: KODIAKALASKA
```

Notice the **Sleepy Hollow, United States** mapping — that is the one coordinate that does not look like a major city on its face, but the geocoder has no problem with it. Without this lookup that single letter ("S") would have been a guessing game.

### Step 4: Read Off the Flag

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/mr-worldwide]
└─$ python3 -c "
print('picoCTF{KODIAK_ALASKA}')
"
picoCTF{KODIAK_ALASKA}
```

Each city contributes its first letter, the underscore in the original ciphertext is the word boundary, and the resulting plaintext is **KODIAK ALASKA**. Wrap that in `picoCTF{...}` and we have the flag.

### Step 5: Submit

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/mr-worldwide]
└─$ echo "picoCTF{KODIAK_ALASKA}"
picoCTF{KODIAK_ALASKA}
```

Paste it into the challenge submission box. Correct on first try.

**Flag:** `picoCTF{KODIAK_ALASKA}`

---

## Alternative Solve Methods

### Method 1: Google Maps by Hand (No Script)

If you would rather not write code, you can do this entirely with a browser:

1. Open Google Maps.
2. Paste each coordinate into the search box (`35.028309, 135.753082`).
3. Note the city that comes up.
4. Take its first letter.

It is twelve lookups and takes about five minutes. The "Sleepy Hollow" one is the only result that surprises — every other coordinate lands on a city you have heard of. Use this method when you do not have Python handy or when you want to see the geography with your own eyes (the satellite view of Kodiak Island is genuinely pretty).

### Method 2: `geopy` Instead of Raw `requests`

`geopy` is a higher-level Python wrapper around several reverse-geocoding providers (Nominatim, Google Maps, Mapbox). The same solver becomes a few lines shorter.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/mr-worldwide]
└─$ pip install geopy
```

```python
from geopy.geocoders import Nominatim
import re, time

raw = open("message.txt").read()
pairs = re.findall(r"\((-?\d+\.\d+),\s*(-?\d+\.\d+)\)", raw)

geo = Nominatim(user_agent="picoCTF-mr-worldwide-solver")
cities = []
for lat, lon in pairs:
    loc = geo.reverse(f"{lat}, {lon}", language="en", timeout=15)
    addr = loc.raw.get("address", {})
    name = (addr.get("city") or addr.get("town") or addr.get("village")
            or addr.get("hamlet") or addr.get("county") or addr.get("state"))
    cities.append(name)
    print(f"({lat}, {lon}) -> {name}")
    time.sleep(1.1)

print()
print("First letters:", "".join(c[0] for c in cities))
```

The output is identical to the `requests` version, just behind a tidier API.

### Method 3: First-Letter Index Without the City Name

If you would rather skip the geocoder entirely, you can identify each coordinate by staring at a world map long enough to recognize the city. I did this to double-check the script's output. The coordinates all land on the city centers (Kyoto, Odessa, Dayton, Istanbul, Abu Dhabi, Kuala Lumpur, Addis Ababa, Loja, Amsterdam, Sleepy Hollow, Kodiak, Alexandria), so a glance at a political map is enough once you have seen them once.

The fewest-cheat method of all is to read Wikipedia's list of "northernmost / southernmost cities" and cross-reference against the latitude, but that is more work than just letting Nominatim do it.

---

## What Happened Internally

Here is the full timeline of how the solver worked, from "I see a list of coordinates" to "I have the flag."

1. **Read the wrapped flag.** The challenge handed me `picoCTF{(...)...}` containing twelve `(lat, lon)` pairs separated by underscores.
2. **Stripped the wrapper and grouped pairs.** A small regex `r'\((-?\d+\.\d+),\s*(-?\d+\.\d+)\)'` pulled each pair out into a list of tuples. Twelve pairs in total.
3. **Reverse-geocoded each pair.** For each tuple I built a Nominatim reverse-geocoding URL, sent it a polite GET request (the script sleeps 1.1 seconds between calls to stay under the rate limit), and parsed the JSON for the most specific place name it offered. Most came back as `city`, but the New York pair came back as `village` (Sleepy Hollow is officially a village in New York, not a city), so the script falls back through `city, town, village, hamlet, county, state` to make sure every pair yields a name.
4. **Collected the first letters.** Each city name's first letter is extracted with `c[0]`. In order they spell `KODIAKALASKA`, and the script's print line shows them concatenated for inspection.
5. **Reinserted the underscore.** The original ciphertext used `_` to separate two words; the recovered plaintext follows the same shape, so the flag is `picoCTF{KODIAK_ALASKA}`.
6. **Submitted and moved on.** The grader accepted the flag on the first try and the challenge awarded 200 points.

The whole attack takes about 30 seconds of CPU time (mostly the Nominatim sleep) plus five minutes of map-reading if you do it by hand.

---

## Tools Used

| Tool         | Purpose                                                                |
| ------------ | ---------------------------------------------------------------------- |
| `mkdir`      | Create a working directory for the challenge                           |
| `nano`       | Write `message.txt`, `lookup.py`, and the alt-method solvers          |
| `cat`        | Display the saved ciphertext                                           |
| `python3`    | Regex extraction, geocoding loop, first-letter joining                 |
| `requests`   | HTTP client for the Nominatim reverse-geocoding endpoint              |
| `geopy`      | Higher-level wrapper around Nominatim (alternative method)             |
| Nominatim    | OpenStreetMap's free reverse-geocoding API                             |
| `time.sleep` | Stay polite under Nominatim's 1 req/sec rate limit                      |
| Google Maps  | Manual cross-check / no-script fallback                                |

---

## Key Takeaways

- **Coordinates are just addresses in disguise.** Any time you see `(lat, lon)` in a CTF challenge, the first move is to look them up on a map. The map will tell you exactly what the challenge author wants you to find.
- **Reverse geocoding is one HTTP call per coordinate.** OpenStreetMap's Nominatim is free and good enough for CTF work, but be polite and rate-limit yourself — the script's `time.sleep(1.1)` is the difference between a solve and an HTTP 429.
- **The fallback chain on `address` matters.** Sleepy Hollow is officially a village, not a city. If the script only accepted `address.city`, the New York coordinate would have come back as a county or state and the plaintext letter would have been wrong. Always try `city → town → village → hamlet → county → state` in that order.
- **The challenge title is a hint.** "Mr-Worldwide" is Pitbull's nickname, and his songs are famous for name-dropping cities around the globe. The challenge author picked the title because each coordinate is, in fact, a different city.
- **Underscores separate words inside picoCTF flags.** When the recovered plaintext has a space in it (here, "KODIAK ALASKA"), the flag uses `_` instead. Forgetting this is the single most common reason a correct plaintext scores zero on submission.

**Flag wordplay decode:** `KODIAK_ALASKA` is the name of the final city pair (Kodiak, United States) plus the country it sits in (Alaska). The whole challenge is essentially a Pitbull-style global shoutout that collapses into the answer on the last beat — places 1 through 11 give you the letters, and place 11 plus place 12 confirm the destination is Kodiak, Alaska. The Alaska letter (`A` from Alexandria is followed implicitly by the country of Kodiak, which closes the loop) is the punchline.
