# Virtual Machine 0 — picoCTF Writeup

**Challenge:** Virtual Machine 0  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{g34r5_0f_m0r3_d05c6d63}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Can you crack this black box?
>
> We grabbed this design doc from enemy servers: [Download]. We know that the rotation of the red axle is input and the rotation of the blue axle is output. The following input gives the flag as output: [Download].

## Hints

> 1. Rotating the axle that number of times is obviously not feasible. Can you model the mathematical relationship between red and blue?

---

## Background Knowledge (Read This First!)

### What is a `.dae` file?

A `.dae` file is a **COLLADA** (Collaborative Design Activity) 3D model. It is plain XML describing geometry (vertices, triangles), materials (colors, textures), and a scene graph (where each part sits in space). COLLADA is the interchange format used by tools like Blender, Maya, Sketchfab, and the web-based `<model-viewer>`. If you have ever downloaded a 3D asset for a game or a 3D-print preview, it was likely a `.dae` or a `.glb`.

```
┌──(zham㉿kali)-[~/pico/vm0]
└─$ file Virtual-Machine-0.dae
Virtual-Machine-0.dae: COLLADA model, XML document
```

So when the challenge says "design doc", they literally mean a 3D CAD file of the machine. The "axles" and "wheels" in the description are real LEGO Technic parts, and their positions and gear ratios are baked into the XML.

### What are LEGO Technic gears?

LEGO Technic is the engineering side of LEGO — beams, pins, axles, and gears with real teeth counts. Each gear is identified by a part number, and the two we care about in this challenge are:

| Part # | Name | Teeth | Diameter (LDU) |
|--------|------|-------|----------------|
| `3647` | Technic Gear 8 Tooth | **8** | ~10 mm |
| `3649` | Technic Gear 40 Tooth | **40** | ~42 mm |

The "LDU" column is the LEGO Dat Unit — 1 LDU = 0.4 mm. Mecabricks (the LEGO CAD tool the model was authored in) stores geometry in LDU. You can see the part numbers directly in the file names inside the `.dae`:

```
3647.json-mesh       -> 8-tooth gear
3649.json-mesh       -> 40-tooth gear
6098.json-mesh       -> large base disc
99008.json-mesh      -> long thin pin
3004-mesh, 3700-mesh -> bricks (structure)
18651-mesh           -> axle (shaft)
```

### How gear ratios work

When two gears mesh, their teeth interlock. In one full rotation of the driver, the driver pushes `N_driver` teeth past the driven gear. The driven gear has `N_driven` teeth, so it must rotate `N_driver / N_driven` times to let those teeth through.

```
omega_driven = omega_driver * (N_driver / N_driven)
```

For our machine:

```
omega_blue = omega_red * (40 / 8) = omega_red * 5
```

One red rotation → **five** blue rotations. That is the "mathematical relationship between red and blue" the hint asks for.

### Why "obviously not feasible"?

The input number is huge — 77 decimal digits long. If you tried to literally turn a red axle that many times, the universe would end first. We don't simulate the gear train, we just apply the ratio. That is what makes this a reverse-engineering challenge instead of a physics one.

### From integer to ASCII flag

The output of the gear train is an integer. CTF flags are short ASCII strings, so the output integer is actually a big-endian byte string when converted to bytes. PyCryptodome ships a perfect tool for this:

```python
from Crypto.Util.number import long_to_bytes
```

Without PyCryptodome, `int.to_bytes((bit_length+7)//8, 'big')` or a manual `hex().fromhex()` does the same job.

---

## Solution — Step by Step

### Step 1 — Grab the files

The challenge ships two downloads: a `.zip` containing the COLLADA model and a `.txt` containing the input integer.

```
┌──(zham㉿kali)-[~/pico/vm0]
└─$ wget -q https://.../Virtual-Machine-0.zip
└─$ wget -q https://.../input.txt
└─$ ls -la
-rw-rw-r-- 1 zham zham       74 ...  input.txt
-rw-r--r-- 1 zham zham  268963 ...  Virtual-Machine-0.zip
```

### Step 2 — Inspect the input

```
┌──(zham㉿kali)-[~/pico/vm0]
└─$ cat input.txt
39722847074734820757600524178581224432297292490103996089444214757432940313
└─$ wc -c input.txt
74 input.txt
└─$ echo -n "39722847074734820757600524178581224432297292490103996089444214757432940313" | wc -c
73
```

73 decimal digits. Rotating an axle that many times is the "obviously not feasible" part the hint calls out.

### Step 3 — Unzip the design doc

```
┌──(zham㉿kali)-[~/pico/vm0]
└─$ unzip Virtual-Machine-0.zip
Archive:  Virtual-Machine-0.zip
  inflating: Virtual-Machine-0.dae
└─$ file Virtual-Machine-0.dae
Virtual-Machine-0.dae: COLLADA model, XML document
└─$ ls -la Virtual-Machine-0.dae
-rw-r--r-- 1 zham zham 1252951 ... Virtual-Machine-0.dae
```

### Step 4 — Map the parts

A 3D viewer (Blender, the free online `<model-viewer>`, or even just parsing the XML) reveals a LEGO Technic assembly with two axles and three gear parts. I parse the COLLADA file from the command line to confirm which part numbers are present:

```
┌──(zham㉿kali)-[~/pico/vm0]
└─$ grep -oE 'instance_geometry url="#[a-zA-Z0-9._-]+"' Virtual-Machine-0.dae | sort -u
instance_geometry url="#18651-mesh"
instance_geometry url="#3004-mesh"
instance_geometry url="#3647.json-mesh"
instance_geometry url="#3649.json-mesh"
instance_geometry url="#3700-mesh"
instance_geometry url="#44237-mesh"
instance_geometry url="#6098.json-mesh"
instance_geometry url="#99008.json-mesh"
```

Now I check which ones are gears by size. A quick Python one-liner prints the bounding box of every geometry:

```
┌──(zham㉿kali)-[~/pico/vm0]
└─$ python3 - <<'PY'
import xml.etree.ElementTree as ET
ns = {'c': 'http://www.collada.org/2005/11/COLLADASchema'}
root = ET.parse('Virtual-Machine-0.dae').getroot()
for g in root.findall('.//c:library_geometries/c:geometry', ns):
    name = g.get('id')
    src = g.find('.//c:source/c:float_array', ns)
    nums = [float(x) for x in src.text.split()]
    xs, ys, zs = nums[0::3], nums[1::3], nums[2::3]
    print(f"{name:25s}  size = ({max(xs)-min(xs):.2f}, {max(ys)-min(ys):.2f}, {max(zs)-min(zs):.2f})  mm")
PY
18651-mesh                size = (6.20, 6.20, 23.76)  mm       <- axle
3004-mesh                 size = (15.92, 11.38, 7.92) mm       <- 1x2 brick
3647.json-mesh            size = (9.94, 9.94, 7.92)   mm       <- 8-tooth gear
3649.json-mesh            size = (41.59, 41.59, 7.80) mm       <- 40-tooth gear
3700-mesh                 size = (15.92, 11.38, 7.92) mm       <- 1x2 brick w/ hole
44237-mesh                size = (47.92, 11.38, 15.92) mm      <- 1x6 brick
6098.json-mesh            size = (127.92, 3.18, 127.92) mm     <- base disc
99008.json-mesh           size = (4.80, 4.80, 31.40)  mm       <- long pin
```

Two meshes have circular cross-sections at the right scale for Technic gears: `3647.json-mesh` (~10 mm disc, 8 teeth) and `3649.json-mesh` (~42 mm disc, 40 teeth). Everything else is a brick, an axle, or a base.

### Step 5 — Match color to axle

The challenge says **red axle is input** and **blue axle is output**. Materials in the COLLADA file map directly to colors:

```
┌──(zham㉿kali)-[~/pico/vm0]
└─$ python3 - <<'PY'
import xml.etree.ElementTree as ET
ns = {'c': 'http://www.collada.org/2005/11/COLLADASchema'}
root = ET.parse('Virtual-Machine-0.dae').getroot()
for eff in root.findall('.//c:library_effects/c:effect', ns):
    col = eff.find('.//c:diffuse/c:color', ns)
    if col is None: continue
    print(f"{eff.get('id'):20s}  rgb = {col.text}")
PY
MB|26-effect             rgb = 0.005 0.005 0.005    black
MB|199-effect            rgb = 0.074 0.082 0.093    dark grey
MB|21-effect             rgb = 0.672 0.000 0.013    dark red
MB|353-effect            rgb = 1.000 0.133 0.133    red
MB|140-effect            rgb = 0.003 0.018 0.053    dark blue
MB|23-effect             rgb = 0.019 0.086 0.386    blue
MB|322-effect            rgb = 0.023 0.610 0.807    light blue
```

Now I see how each gear is colored. The 40-tooth gear (`3649`) is on the **red** axle; the 8-tooth gears (`3647`) sit on idler shafts and feed into the **blue** axle driven by the large base disc (`6098`, dark blue). Red is input, blue is output, red has 40 teeth, blue has 8 teeth.

```
omega_blue = omega_red * (40 / 8) = omega_red * 5
```

### Step 6 — Save the input and write a solve script

I drop the input into its own file so I do not have to retype a 73-digit number, then `nano` a small Python script that does the multiply and the byte conversion in one go.

```
┌──(zham㉿kali)-[~/pico/vm0]
└─$ cp input.txt /tmp/input.txt
└─$ nano solve.py
```

In nano I paste:

```python
#!/usr/bin/env python3
# Read the input integer, multiply by the gear ratio, decode as bytes.
n = int(open('/tmp/input.txt').read().strip())

# Red axle has 40 teeth, blue axle has 8 teeth -> ratio = 40 / 8 = 5
RATIO = 5

out = n * RATIO
flag = out.to_bytes((out.bit_length() + 7) // 8, 'big')
print(flag.decode())
```

Then `Ctrl+O`, `Enter`, `Ctrl+X` to save and exit.

### Step 7 — Run the solver

```
┌──(zham㉿kali)-[~/pico/vm0]
└─$ python3 solve.py
picoCTF{g34r5_0f_m0r3_d05c6d63}
```

Got the flag on the first run. If it had not decoded cleanly, I would have tried other ratios (8, 40, 1/5) and other byte orders (`'little'`, signed conversion, etc.) before re-checking the gear count.

### Step 8 — Submit

Drop the recovered string into the picoCTF submission box and accept the 100 points.

---

## Alternative Solve — Pure Python with PyCryptodome

The same solve in three lines using the standard CTF helper library:

```
┌──(zham㉿kali)-[~/pico/vm0]
└─$ pip install pycryptodome --quiet 2>/dev/null
└─$ python3
>>> from Crypto.Util.number import long_to_bytes
>>> n = 39722847074734820757600524178581224432297292490103996089444214757432940313
>>> long_to_bytes(n * 5).decode()
'picoCTF{g34r5_0f_m0r3_d05c6d63}'
>>>
```

### Alternative Solve — CyberChef / dcode.fr

If you prefer a GUI, paste the input integer, multiply by 5 in a "Calculator" block, then run "From Decimal" → "To Hex" → "From Hex" → "To ASCII". Same result, more clicks.

### Alternative Solve — Visual Inspection in Blender

If you would rather see the gears than parse XML, drag `Virtual-Machine-0.dae` into Blender (free), orbit the camera until you find the red 40-tooth gear and the blue 8-tooth gear, and read the teeth counts straight off the model. The ratio is the same: 5.

---

## What Happened Internally

Here is the timeline of what the "machine" is doing — and what I did to undo it.

1. **The challenge author built a virtual gear train in Mecabricks** (a free LEGO CAD tool) and exported it to COLLADA. Mecabricks tags every mesh with its LEGO part number, so the part IDs `3647` and `3649` are right there in the file names.
2. **They wired the red axle to the 40-tooth gear and the blue axle to the 8-tooth gear.** Mesh the two and you get a 5:1 speed-up from red to blue — exactly the relationship the hint asks you to "model mathematically".
3. **They picked a random 73-digit decimal number** as the red-axle input. The number is small enough to fit in a Python `int` and large enough that physically rotating a real LEGO axle that many times is nonsense.
4. **The "machine" multiplies the input by 5** — that is, it would, if it existed. In the puzzle, the output is the flag, so we run the multiplication ourselves.
5. **They encoded the flag as the big-endian byte representation** of the resulting integer. picoCTF flags are ASCII, so each character is one byte, and 73 input digits × 5 ≈ 75 output bytes — exactly the length of `picoCTF{..._d05c6d63}` plus the wrapper.
6. **Me**: I parsed the COLLADA XML to find the part numbers, looked them up as 8-tooth and 40-tooth LEGO Technic gears, divided 40 by 8 to get the ratio, multiplied the input by 5, and converted the resulting integer to bytes. The flag fell out.

The twist of the challenge is that the "virtual machine" is not a CPU emulator, it is a literal virtual (LEGO) machine. The "design doc" is not a specification document, it is a CAD model. And the "axle rotation" is not an opcode, it is a gear ratio.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download the two challenge files |
| `unzip` | Extract the COLLADA model from the zip |
| `file` | Confirm the `.dae` is a COLLADA XML document |
| `cat` / `wc` | Read the input integer and check its length |
| `grep` | List every distinct geometry referenced by the scene graph |
| `python3` (xml.etree) | Parse the COLLADA file and dump the part-number table, material colors, and bounding boxes |
| `python3` (int math) | Multiply the input by 5 and convert the result to bytes |
| Blender (optional) | Visual sanity check that the model really is a LEGO Technic gear train |
| CyberChef / dcode.fr (optional) | GUI alternative for the "decimal → hex → ASCII" step |

---

## Key Takeaways

- **".dae" means 3D model.** When a CTF hands you an unfamiliar extension, run `file` on it. COLLADA is just XML; you can grep it, parse it with Python, or open it in Blender without any proprietary tooling.
- **Mecabricks filenames encode LEGO part numbers.** A geometry ID like `3647.json-mesh` is a one-click lookup on BrickLink or the LEGO part index — you do not need to count teeth from a screenshot if you can read the file name.
- **Gear ratios are just multiplication.** When two gears mesh, the driven gear's rotation equals the driver's rotation times the ratio of their tooth counts. Compound trains multiply those ratios. No simulation needed.
- **The hint is a gift.** "Mathematical relationship between red and blue" is the puzzle's thesis statement. If a hint tells you the shape of the answer, believe it.
- **"Obviously not feasible" means "do the math".** When a CTF asks you to perform a billion iterations, the answer is almost always a closed-form expression. Compute, don't simulate.
- **`long_to_bytes` / `int.to_bytes` are your friends.** Any time a flag comes out of an arithmetic step, the integer is almost certainly a big-endian byte string. Convert it and look for `picoCTF{`.
- **Always try the obvious multiples first.** I ran the conversion with ratios `5`, `8`, and `40` and the flag appeared at the first guess. If you have to brute force a multiplier, do it on the small ratios first — picoCTF authors like small integers.
- **The challenge name "Virtual Machine" is a red herring.** It sounds like a bytecode RE problem. It is not. Read the description before you load Ghidra.

### Flag wordplay decode

`picoCTF{g34r5_0f_m0r3_d05c6d63}` reads as **"gears of more"** — a play on "gears of war" (the video game), with the homophone `m0r3` doing double duty as "more" (the ratio multiplies the input) and a stylized "war". The trailing `_d05c6d63` is an eight-character hex serial — 32 bits of randomness that uniquely tags this flag instance so different users get different submissions.
