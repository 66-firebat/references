# Fixing PDF page orientation on the MobiScribe Wave (and similar Android e-readers)

**Working output:** `principles_of_model_checking_portrait_upright.pdf`
**Source:** `principles_of_model_checking.pdf` (994 pages, 17 MB)
**Device:** MobiScribe Wave (Android 12), native PDF reader

---

## TL;DR

The book was a normal **upright portrait** PDF. The only anomaly was that **page 1
(the cover) had a landscape `MediaBox`** while the other 993 pages were portrait.
The Wave's native reader **ignores per-page `/Rotate`** and infers the whole app's
UI orientation from page geometry, so that single landscape cover forced the reader
into landscape mode and made the upright pages render sideways.

**Fix:** normalize **every** page to portrait (including the cover) and leave
`/Rotate 0`. No rotation was actually needed — the "rotation" symptom was the
reader's landscape-forcing behavior being applied to upright content.

---

## Symptoms observed

1. The book displayed rotated / sideways on the Wave.
2. Setting `/Rotate 270` (or 90 / 180) changed nothing — the reader ignores `/Rotate`.
3. Baking the rotation into the content (`*_baked_ccw90.pdf`) made the **content**
   readable, but the reader **UI** stayed in landscape mode.
4. Because the pages were then landscape, the app kept requesting a landscape Activity.

Root lesson: on this reader, **page aspect ratio drives UI orientation**, and
**`/Rotate` is not honored**. Both problems are solved by making all pages portrait.

---

## Environment / tooling

Available on the system (no installs were needed):

| Tool | Notes |
|---|---|
| `mutool` 1.27.2 | `show`, `draw`, and `run` (JavaScript engine) |
| `pdfinfo` | page count / page size / page rotation |
| `python3` | used for small pixel-diff checks |

**Not** available: `qpdf`, `pdftk`, `gs` (Ghostscript), `exiftool`, `pypdf`, `PyPDF2`, `fitz`.
`qpdf` was never needed.

---

## Diagnosis (commands that found the root cause)

```bash
# Basic info
pdfinfo principles_of_model_checking.pdf

# Every page's encoded rotation (there are 994, all zero)
mutool show -g principles_of_model_checking.pdf grep /Rotate \
  | grep -o '/Rotate[ ]*[-0-9]*' | sort | uniq -c
#   994 /Rotate 0

# Page-box distribution -> reveals the single landscape cover
mutool show -g principles_of_model_checking.pdf grep MediaBox \
  | grep -o 'MediaBox\[[^]]*\]' | sort | uniq -c
#   993 MediaBox[0 0 576 720]
#     1 MediaBox[0 0 1438.5 876]      <-- page 1

mutool show -g principles_of_model_checking.pdf grep CropBox \
  | grep -o 'CropBox\[[^]]*\]' | sort | uniq -c
#   993 CropBox[0 0 576 720]
#     1 CropBox[792.288 77.976 1368.3 797.952]   <-- page 1 (portrait, offset)

# Text is genuinely upright / horizontal, single column
mutool draw -q -F stext -o - principles_of_model_checking.pdf 20 | grep 'dir='
#   dir="1 0"
```

Findings:

- 994 pages, all `/Rotate 0`.
- 993 pages are portrait `576 × 720`; **page 1 is landscape `1438.5 × 876`**.
- Page 1's `CropBox` is a portrait, offset sub-region of that landscape box.
- Body text is horizontal (`dir="1 0"`), so the file is not intrinsically rotated.
- No outline/bookmarks to preserve.

---

## The fix (what `portrait_upright.pdf` does)

For the cover only (page 1):

1. Prepend a translation `q 1 0 0 1 (-cx0) (-cy0) cm` so the offset `CropBox`
   region moves to the origin.
2. Set `MediaBox`, `CropBox`, `BleedBox`, `TrimBox`, `ArtBox` to
   `[0, 0, CropBoxWidth, CropBoxHeight]` = `[0, 0, 576.012, 719.976]`.
3. Keep `/Rotate 0`.

All other pages are already portrait and untouched. Result: **994 portrait pages,
all `/Rotate 0`**, content upright and readable in portrait.

### The actual code that was run (verbatim)

The working file was produced by:

```bash
cd /home/firebat/Downloads && mutool run /tmp/portrait_build.js
```

`/tmp/portrait_build.js` contained two blocks; the fix is **block 1**, reproduced
verbatim below. (Block 2 built the unrelated letterboxed experiment, so it is
omitted.) This is the exact code that generated
`principles_of_model_checking_portrait_upright.pdf`.

```javascript
// ---- 1) Upright content, all pages portrait (readable in portrait) ----
(function(){
  var doc = new PDFDocument("principles_of_model_checking.pdf");
  var p1 = doc.loadPage(0).getObject();
  var cb = p1.get("CropBox");
  var cx0=cb.get(0).asNumber(), cy0=cb.get(1).asNumber();
  var cx1=cb.get(2).asNumber(), cy1=cb.get(3).asNumber();
  var W=cx1-cx0, H=cy1-cy0;
  var pre  = doc.addStream("q\n1 0 0 1 "+(-cx0)+" "+(-cy0)+" cm\n", doc.newDictionary());
  var post = doc.addStream("\nQ\n", doc.newDictionary());
  var arr = doc.newArray(); arr.push(pre);
  var old = p1.get("Contents");
  if (old.isArray()){ for (var k=0;k<old.length;k++) arr.push(old.get(k)); } else { arr.push(old); }
  arr.push(post);
  p1.put("Contents", arr);
  var nb = doc.newArray();
  nb.push(doc.newReal(0)); nb.push(doc.newReal(0)); nb.push(doc.newReal(W)); nb.push(doc.newReal(H));
  ["MediaBox","CropBox","BleedBox","TrimBox","ArtBox"].forEach(function(n){ if (p1.get(n)) p1.put(n, nb); });
  p1.put("Rotate", 0);
  doc.save("principles_of_model_checking_portrait_upright.pdf");
  print("upright portrait -> principles_of_model_checking_portrait_upright.pdf");
})();
```

To run the fix on its own, save block 1 to `/tmp/fix_portrait.js` and run:

```bash
cd /home/firebat/Downloads && mutool run /tmp/fix_portrait.js
```

---

## Verification checklist

```bash
F=principles_of_model_checking_portrait_upright.pdf

pdfinfo "$F" | grep -Ei 'pages|page size|page rot'
#   Pages: 994 ; Page size: 576.012 x 719.976 pts ; Page rot: 0

mutool show -g "$F" grep /Rotate | grep -o '/Rotate[ ]*[-0-9]*' | sort | uniq -c
#   994 /Rotate 0

mutool show -g "$F" grep MediaBox | grep -o 'MediaBox\[[^]]*\]' | sort | uniq -c
#   993 MediaBox[0 0 576 720]
#     1 MediaBox[0 0 576.012 719.976]

# text still horizontal and extractable
mutool draw -q -F stext -o - "$F" 20 | grep 'dir='
#   dir="1 0"
mutool draw -q -F txt "$F" 20 | head -c 60
#   Chapter 1\n\nSystem Verification\n\nOur reliance on the function
```

Cover render check: the new page 1 is pixel-identical to the original `CropBox`
view except for **0.165%** of pixels (sub-pixel anti-aliasing, best alignment
shift is `(0,0)` — no displacement).

---

## Companion artifacts in this folder

| File | Purpose |
|---|---|
| **`principles_of_model_checking_portrait_upright.pdf`** | **The working fix.** All 994 pages portrait, `/Rotate 0`, text horizontal. |
| `principles_of_model_checking.pdf` | Original source (untouched). |
| `principles_of_model_checking_portrait_letterboxed.pdf` | All pages portrait, but keeps the ccw90 content orientation (text rotated inside the page), scaled to 80% and centered. Only useful if a reader itself rotates page content. |
| `principles_of_model_checking_baked_ccw90.pdf` | Content physically rotated 90° CCW (landscape pages). Readable on the Wave, but keeps the UI landscape. |
| `principles_of_model_checking_baked_cw90.pdf` | Content physically rotated 90° CW (landscape pages). |
| `principles_of_model_checking_baked_180.pdf` | Content physically rotated 180° (portrait pages, upside down). |
| `principles_of_model_checking_rotated.pdf` | Earlier attempt that only set `/Rotate 270` (reader ignores it). Superseded. |
| `principles_of_model_checking_page1fixed.pdf` | Earlier cover-normalized file. Superseded by `..._portrait_upright.pdf`. |

---

## Gotchas for future agents

1. **`/Rotate` is only a hint.** Many e-readers ignore it. To guarantee orientation,
   *bake* the rotation into the page content and set `/Rotate 0`.
2. **Page aspect ratio can drive the reader's UI orientation.** If a single page is
   landscape, a reader may force the whole app into landscape. Keep box geometry
   uniform unless you intend that.
3. **`mutool draw -F pdf -R <angle>` does NOT bake rotation.** It writes `/Rotate`
   again *and* breaks text extraction (drops/garble ToUnicode mappings). Do not use
   it for this.
4. **`mutool run` `addStream` argument order is `(buffer, dict)`** — the opposite of
   what the name suggests. `doc.addStream(dict, buffer)` throws
   `Error: not a dict (name)`.
5. **`readStream()` returns an `fz_buffer`**, not a JS string (it has `.length` /
   `.slice`, but no `.substring`). To keep binary streams intact, avoid decoding and
   re-writing them; prepend/append separate streams instead (as done here).
6. **Baked-rotation matrices** in PDF order `[a b c d e f]`, for a box
   `[x0 y0 x1 y1]` with `W = x1-x0`, `H = y1-y0`:

   | Target | Matrix `[a b c d e f]` | New box |
   |---|---|---|
   | 90° CW  | `[0 -1 1 0 (-y0) (x0+W)]` | `[0,0,H,W]` |
   | 180°    | `[-1 0 0 -1 (x0+W) (y0+H)]` | `[0,0,W,H]` |
   | 270° (90° CCW) | `[0 1 -1 0 (y0+H) (-x0)]` | `[0,0,H,W]` |

   Baking procedure: prepend a stream `q <matrix> cm`, append a stream `Q`, rewrite
   `MediaBox`/`CropBox`/`BleedBox`/`TrimBox`/`ArtBox`, set `/Rotate 0`.
7. **Verify with rendering, not just metadata.** Compare `mutool draw` PNG/PPM
   output (md5 or pixel diff) between the baked file and the `/Rotate` equivalent;
   they should be pixel-identical.

### Full bake-all-angles script (for reference)

```javascript
// bakes 90 / 180 / 270 into page content; output files have /Rotate 0
var input = "principles_of_model_checking.pdf";

function bake(doc, ANGLE) {
  var n = doc.countPages();
  for (var i = 0; i < n; i++) {
    var obj = doc.loadPage(i).getObject();
    var box = obj.getInheritable("CropBox");
    if (!box) box = obj.getInheritable("MediaBox");
    var x0 = box.get(0).asNumber(), y0 = box.get(1).asNumber();
    var x1 = box.get(2).asNumber(), y1 = box.get(3).asNumber();
    var W = x1 - x0, H = y1 - y0;
    var a, b, c, d, e, f, nw, nh;
    if (ANGLE === 90)       { a = 0;  b = -1; c = 1;  d = 0;  e = -y0;    f = x0 + W; nw = H; nh = W; }
    else if (ANGLE === 180) { a = -1; b = 0;  c = 0;  d = -1; e = x0 + W; f = y0 + H; nw = W; nh = H; }
    else                    { a = 0;  b = 1;  c = -1; d = 0;  e = y0 + H; f = -x0;    nw = H; nh = W; }

    var mtx = a + " " + b + " " + c + " " + d + " " + e + " " + f;
    var pre  = doc.addStream("q\n" + mtx + " cm\n", doc.newDictionary());
    var post = doc.addStream("\nQ\n", doc.newDictionary());

    var arr = doc.newArray();
    arr.push(pre);
    var old = obj.get("Contents");
    if (old.isArray()) { for (var k = 0; k < old.length; k++) arr.push(old.get(k)); }
    else { arr.push(old); }
    arr.push(post);
    obj.put("Contents", arr);

    var nb = doc.newArray();
    nb.push(doc.newReal(0)); nb.push(doc.newReal(0));
    nb.push(doc.newReal(nw)); nb.push(doc.newReal(nh));
    ["MediaBox", "CropBox", "BleedBox", "TrimBox", "ArtBox"].forEach(function (name) {
      if (obj.get(name)) obj.put(name, nb);
    });
    obj.put("Rotate", 0);
  }
}

var jobs = [[90, "principles_of_model_checking_baked_cw90.pdf"],
            [180, "principles_of_model_checking_baked_180.pdf"],
            [270, "principles_of_model_checking_baked_ccw90.pdf"]];
for (var j = 0; j < jobs.length; j++) {
  var doc = new PDFDocument(input);
  bake(doc, jobs[j][0]);
  doc.save(jobs[j][1]);
}
```

---

## If the native reader still forces an orientation

A page-geometry fix only works if the reader derives orientation from the document.
If it overrides orientation regardless, use a third-party reader with an explicit
orientation lock. The Wave runs Android 12 and can install apps via the MobiScribe
App Store or Google Play. **KOReader** (free, open source, excellent on e-ink) has
`Screen → Orientation` to lock the UI independently of page rotation, plus its own
page rotate/crop controls. Librera, Moon+ Reader, Xodo, and Orion are alternatives.
