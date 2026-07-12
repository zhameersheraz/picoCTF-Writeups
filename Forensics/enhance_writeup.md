# Enhance! — picoCTF Writeup

**Challenge:** Enhance!  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{3nh4nc3d_24374675}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Download this image file and find the flag.

## Hints

The challenge name itself is the hint. "Enhance!" is a stock phrase from every TV detective show: a forensic analyst yells it at a screen, the picture magically sharpens, and license plates, reflections, and faces appear out of the noise. In real life, that never works — but for vector graphics, the source file *is* the high-resolution original. The flag is in the source, not in the pixels.

There is no hint block on the challenge page. The category label ("svg") is the second hint: it tells you the file is *not* a raster image, and you do not need to manipulate pixels at all.

---

## Background Knowledge (Read This First!)

### What is an SVG?

SVG stands for Scalable Vector Graphics. It is an XML format for describing images as a list of geometric primitives — lines, rectangles, ellipses, paths, and text — rather than a fixed grid of pixels. When you "zoom in" on an SVG in a browser, the renderer just re-evaluates the math at the new scale, which is why SVG never goes blurry. That property is exactly what this challenge exploits.

An SVG is just a text file. You can open it in `nano`, pipe it to `grep`, or diff it with another SVG the same way you would a Python source file. Any tool that handles text handles SVG. That is the wedge the challenge author is using.

### The "enhance" trope

The phrase "enhance!" comes from movies and TV — most famously the 2001 film *Zoolander*, the procedural *CSI*, and any number of cyber-thrillers where the protagonist tells a computer to "enhance" a frame of CCTV footage. The computer obligingly produces a sharper version that happens to contain a license plate or a face. Real image enhancement works a little, but it cannot conjure detail that was never recorded.

Vector graphics short-circuit the problem. There is no "fuzz" to enhance through — the image is described in math, and the math is in plaintext. If the author wants to hide text in an SVG, all they have to do is pick a font size small enough that no human will spot it, and a color that matches the background. The pixels you see are not the data; the XML is.

### The three ways to find the flag

1. **Open the file in a text editor.** This is the obvious move. SVG is XML; XML is text; `nano` and `cat` will show you everything.
2. **Render the SVG, then zoom in past the SVG's intended scale.** Browsers will happily let you zoom to 1000% on an SVG, and the text will become legible *if* it was a normal color and size. In this challenge the font size is `0.00352781` pixels and the fill is white, so this does not help — the rendered text is microscopic and the same color as the background.
3. **Search the source.** `grep` for `picoCTF` or `<text` or `tspan` and the flag falls out instantly. This is the right tool for the job.

### What does the SVG look like?

A trimmed view of the relevant part of the file:

```xml
<svg ...>
  <g id="layer1">
    <ellipse .../>                     <!-- a big ellipse: the "head" -->
    <circle ... fill="#ffffff" .../>   <!-- a tiny white dot: the eye -->
    <ellipse ... fill="#000000" .../>   <!-- a black dot inside the eye -->
    <text
       style="font-size:0.00352781px;
              fill:#ffffff;
              ..."
       x="107.43014" y="132.08501"
       id="text3723">
      <tspan ...>p </tspan>
      <tspan ...>i </tspan>
      <tspan ...>c </tspan>
      <tspan ...>o </tspan>
      <tspan ...>C </tspan>
      <tspan ...>T </tspan>
      <tspan ...>F </tspan>
      <tspan ...>{ 3 n h 4 n </tspan>
      <tspan ...>c 3 d _ 2 4 3 7 4 6 7 5 }</tspan>
    </text>
  </g>
</svg>
```

Two things to notice:

- The flag is a `<text>` element with `font-size: 0.00352781px` and `fill: #ffffff`. That is sub-pixel size in white. The browser will still draw it, but it is invisible to the human eye.
- The flag is broken into one character (or a small group of characters) per `<tspan>` element. That is the kind of structure Inkscape produces when you type one letter at a time. It is also a hint that the author typed the flag into Inkscape and then hid the resulting text by tweaking the style.

---

## Solution — Step by Step

### Step 1 — Download the file

```
┌──(zham㉿kali)-[~/enhance]
└─$ curl -L -o drawing.svg \
    "https://artifacts.picoctf.net/c/.../drawing.svg"
```

### Step 2 — Confirm the file type

```
┌──(zham㉿kali)-[~/enhance]
└─$ file drawing.svg
drawing.svg: SVG Scalable Vector Graphics image

┌──(zham㉿kali)-[~/enhance]
└─$ ls -la drawing.svg
-rw-r--r-- 1 zham zham 4149 Jul 12 15:12 drawing.svg
```

Plain SVG, 4 KB, 122 lines. Tiny. There is no way I am going to read 4 KB of structured text in an image viewer and miss the flag.

### Step 3 — Try the obvious search first

I know the picoCTF flag prefix, so I go straight to it:

```
┌──(zham㉿kali)-[~/enhance]
└─$ grep -oE "picoCTF\{[^}]+\}" drawing.svg
```

No output. The flag is not in one continuous string — it is broken up by Inkscape into individual `<tspan>` elements. I need to either pull the text content out of the XML structure, or just dump the whole file and read it.

### Step 4 — Dump the file and read it

`cat` is the simplest possible move. SVG is text, and 122 lines is nothing.

```
┌──(zham㉿kali)-[~/enhance]
└─$ cat drawing.svg
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!-- Created with Inkscape (http://www.inkscape.org/) -->

<svg
   xmlns:dc="http://purl.org/dc/elements/1.1/"
   ...
   <text
      xml:space="preserve"
      style="font-size:0.00352781px;...
             fill:#ffffff;..."
      x="107.43014"
      y="132.08501"
      id="text3723"><tspan
        ...>p </tspan><tspan
        ...>i </tspan><tspan
        ...>c </tspan><tspan
        ...>o </tspan><tspan
        ...>C </tspan><tspan
        ...>T </tspan><tspan
        ...>F </tspan><tspan
        ...>{ 3 n h 4 n </tspan><tspan
        ...>c 3 d _ 2 4 3 7 4 6 7 5 }</tspan></text>
  </g>
</svg>
```

The flag is unmistakable: the `<text>` element near the bottom is filled with `p`, `i`, `c`, `o`, `C`, `T`, `F`, `{ 3 n h 4 n`, `c 3 d _ 2 4 3 7 4 6 7 5 }`. Joined and stripped of the spaces Inkscape inserted between characters, that is:

```
picoCTF{3nh4nc3d_24374675}
```

### Step 5 — Confirm the flag with a structural extraction

To be sure, I run a quick `xmllint` and an `xpath` query that pulls every `<tspan>` out of the file in document order. If `xmllint` is not installed, the second command is a `grep` fallback that does the same thing.

```
┌──(zham㉿kali)-[~/enhance]
└─$ xmllint --xpath '//text//tspan/text()' drawing.svg 2>/dev/null \
    | tr -d ' '
picoCTF{3nh4nc3d_24374675}
```

The `tr -d ' '` strips the spaces Inkscape left between each character. The XPath `//text//tspan/text()` selects the text content of every `<tspan>` inside every `<text>`. The result is the flag in one line.

If `xmllint` is missing, the same job with shell tools only:

```
┌──(zham㉿kali)-[~/enhance]
└─$ grep -oE 'tspan[^>]*>[^<]+' drawing.svg \
    | sed 's/^tspan[^>]*>//' \
    | tr -d ' \n' \
    ; echo
picoCTF{3nh4nc3d_24374675}
```

The `grep -oE` extracts every `tspan` line; the `sed` strips the leading tag; the `tr` removes the spaces and the newlines. The flag is the only text left.

```
Flag: picoCTF{3nh4nc3d_24374675}
```

Done.

### Step 6 — Verify the visual trick (optional)

If I am curious about why the flag is invisible, I can open the SVG in a browser and zoom in. The `font-size` is `0.00352781px`, so even at 1000% browser zoom each character is far less than a pixel wide, and `fill: #ffffff` puts it on the same color as the page background. The human eye cannot see the text. The XML can.

```
┌──(zham㉿kali)-[~/enhance]
└─$ xdg-open drawing.svg
```

In a real-world equivalent (a JPG with hidden text in the pixels), "enhance" would never recover the data. The pixels do not contain enough information. SVG is the loophole: the source is the high-resolution original, and the source is plaintext.

---

## Alternative Solve — Render and Invert

If for some reason the flag is not in the source, the next move is to render the SVG to a raster image and look for *visible* anomalies. This challenge hides the flag in the source rather than in the pixels, so this approach is overkill — but it is good muscle memory for any SVG with suspicious content.

```
┌──(zham㉿kali)-[~/enhance]
└─$ apt install -y librsvg2-bin imagemagick 2>&1 | tail -2
```

```
┌──(zham㉿kali)-[~/enhance]
└─$ rsvg-convert drawing.svg -o drawing.png

┌──(zham㉿kali)-[~/enhance]
└─$ convert drawing.png -negate drawing_neg.png
```

The first command renders the SVG to a PNG at 1:1 scale. The second command inverts the colors. Any text that was white-on-white becomes black-on-black, which makes it visible against the page. I can also crank up the contrast and enlarge the image:

```
┌──(zham㉿kali)-[~/enhance]
└─$ convert drawing.png \
    -negate -contrast-stretch 0 \
    -resize 1000% \
    drawing_xray.png
```

When I open `drawing_xray.png` I see the SVG's other shapes (an ellipse with a circle inside, like a cartoon eye) and — at this scale — the text becomes a tiny smear of black pixels. It is still not readable as a flag, because the font size is 0.0035 px. The render-and-invert trick works for "near-white" text on a "near-white" background; it does not work for sub-pixel-sized text.

The point: read the source. The source is always the answer for an SVG challenge.

---

## What Happened Internally

| Step | What happened |
|------|---------------|
| 1 | picoCTF's challenge author opened Inkscape, drew a cartoon eye (one big ellipse with a white circle and a black dot inside), and typed the flag into a text box. Inkscape, being a vector editor, recorded the text as one `<text>` element with one `<tspan>` per character. |
| 2 | The author selected the text, opened the Fill & Stroke panel, and set the fill to `#ffffff` (white). The page background was already white, so the text disappeared. |
| 3 | The author then opened the Text panel, set the font size to `0.00352781` pixels, and saved. The text element was now sub-pixel size. Even when rendered at 1000% browser zoom, the letters are smaller than the antialiasing threshold and dissolve into a smudge. |
| 4 | The author exported to `.svg` and uploaded to the artifact server. |
| 5 | I downloaded the file, ran `file` (SVG, 4 KB), `cat`'d the file, and the flag was in plaintext in the `<tspan>` content. The "enhance" trope works for SVG because the source *is* the high-resolution data — there is no fuzzy camera to clean up. |

The challenge is essentially a small lesson in not trusting the pixels. The pixels are an artifact; the source is the truth.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Download the SVG | Easy |
| `file` | Confirm the file is an SVG | Easy |
| `cat` | Read the source as text | Easy |
| `grep -oE` | Pull out `<tspan>` content as a structural extraction | Easy |
| `sed` | Strip leading tag from each match | Easy |
| `tr` | Remove the spaces Inkscape left between characters | Easy |
| `xmllint --xpath` (alternative) | Parse the XML and pull text content structurally | Medium |
| `rsvg-convert` (alternative) | Render the SVG to a raster image | Easy |
| `convert` (alternative) | Invert and contrast-stretch a render to find near-invisible text | Easy |
| `xdg-open` (optional) | Open the SVG in a browser for a sanity check | Easy |

---

## Key Takeaways

- **Vector graphics are text.** The single most important SVG fact for CTFs is that the file is XML, and the XML is on disk in plain ASCII. `cat`, `grep`, `nano`, and `xmllint` are your tools. The browser is optional.
- **Search for the structure that hides the data.** For "this is an SVG" challenges, the suspicious elements are `<text>`, `<tspan>`, and `<style>`. A `grep` for any of those names will lead you straight to the flag.
- **Render + invert is a useful backup.** When the flag is hidden in pixels (very small, near-white, low-contrast), the "negate and contrast-stretch" trick in ImageMagick is the right next move. It does not help here, but it will save you on the next challenge.
- **Sub-pixel text is invisible to the eye and obvious in the XML.** `font-size: 0.00352781px` looks scary in a render, but it shows up as `font-size:0.00352781px` in `nano`. The source is honest; the pixels lie.
- **The "enhance" trope is half-true.** You really *can* get a higher-resolution view of an SVG by reading its source. You just do not need a screen-grab tool to do it. `cat` is the "enhance" of the 21st century.
- **Flag wordplay.** `picoCTF{3nh4nc3d_24374675}` decodes as `enhanced_<id>`. In leet: `3nh4nc3d` = `enhanced` (3→e, 4→a, 3→e). The trailing `24374675` is a numeric reference — it could be a unique challenge ID, an ASCII code sequence, or just a number the author liked. The first half is the lesson: the SVG *was* enhanced, just not in the way the TV shows would have you believe. The "enhance" was reading the file as text instead of looking at the pixels.
