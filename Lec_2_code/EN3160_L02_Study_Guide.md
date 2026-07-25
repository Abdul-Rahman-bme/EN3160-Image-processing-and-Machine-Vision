# EN3160 — L02: Point Operations — Study Guide

Scope: this covers exactly what's in your L02 slide deck (digital images, color, camera/Bayer,
color models, intensity transformations, histograms). At the end I map each topic to Tutorial 1
questions — I explain the *method* with a different worked example, not the tutorial's actual
numbers, since tutorials are graded (20% of your continuous assessment).

---

## 1. What a Digital Image Actually Is

A grayscale digital image is a 2D array of integers. Each entry is a **pixel**, and for an 8-bit
image each pixel is an integer in `[0, 255]` (because 2⁸ = 256 distinct values, stored as `uint8`
— unsigned, 8-bit).

**Coordinate convention** (this trips people up constantly, so nail it early):
- The array is indexed `(i, j)` where **i is the row** (vertical axis, going down) and **j is the
  column** (horizontal axis, going right).
- `(0, 0)` is the **top-left** pixel — not bottom-left like a math graph.
- So pixel `(2, 3)` means: go down 2 rows, right 3 columns.

A color image is **three** such arrays stacked — one per channel — giving shape `(rows, cols, 3)`.

**Worked example:** Say you have a 4×5 image (4 rows, 5 columns). Its shape in code is
`(4, 5)`. The pixel at `(1, 4)` is row 1, column 4 (the last column, 0-indexed). If it were a
color image, shape would be `(4, 5, 3)` and `im[1, 4]` would return three values, not one.

---

## 2. Color Images Are BGR in OpenCV (not RGB)

- Grayscale image = 1 plane.
- Color image = 3 planes: **Blue, Green, Red**, in that order for OpenCV (`cv2`).
- Each plane is `[0, 255]`, so a pixel can be any of 2⁸ × 2⁸ × 2⁸ = 16,777,216 colors.

**Why does order matter?** `matplotlib` expects RGB order. OpenCV stores BGR. If you `imshow()`
a `cv.imread()`'d image directly in matplotlib without converting, red and blue channels get
swapped and the colors look wrong (skin looks blue, sky looks orange, etc.). That's why you
constantly see:

```python
ax.imshow(cv.cvtColor(im, cv.COLOR_BGR2RGB))
```

If you display with `cv.imshow()` instead (OpenCV's own window), **no conversion needed** —
OpenCV is internally consistent with itself; the mismatch only appears when you hand BGR data
to a library (matplotlib) that assumes RGB.

---

## 3. Creating Images in Code

```python
im = np.zeros((6, 8), dtype=np.uint8)   # grayscale, 6 rows x 8 cols, all black
im[2, 3] = 255                          # row 2, col 3 -> fully white
```

For color:
```python
im = np.zeros((6, 8, 3), dtype=np.uint8)
im[2, 3] = (255, 190, 203)              # (B, G, R) — this is pink
```

Why `np.uint8`? Because pixel values only ever need `[0, 255]`, and using the smallest
sufficient data type saves memory — a 3712×5568 image at `uint8` is ~62 MB; at default `int64`
it would be ~496 MB for no benefit.

**Image properties example** — for shape `(2641, 1761, 3)`:
- `im.shape` → `(2641, 1761, 3)` (rows, cols, channels)
- `im.dtype` → `uint8`
- `im.size` → 2641 × 1761 × 3 = 13,952,403 (total number of stored numbers, not pixels)

---

## 4. Extracting a Single Color Plane

```python
im_blue = im.copy()
im_blue[:, :, 1] = 0   # zero the Green channel
im_blue[:, :, 2] = 0   # zero the Red channel
```
Only the Blue channel survives, so `im_blue` displays as a blue-tinted version of the original.
This is a *masking* operation — you're not extracting a grayscale plane, you're **suppressing**
the other two channels while keeping the array 3-channel (so it still displays as color).

If you wanted the *actual* blue-channel intensities as a single-plane grayscale image, you'd do
`im[:, :, 0]` (no assignment, just indexing) — that gives you a 2D array you could display with
`cmap='gray'`.

---

## 5. Increasing Brightness — and the Overflow Trap

Naive approach:
```python
im2 = im1 + 100
```
This is **wrong**. `im1` is `uint8`. Adding 100 to a pixel already at 200 gives 300, but `uint8`
can't represent 300 — it **wraps around**: `300 mod 256 = 44`. So a bright pixel (200) becomes a
dark one (44) instead of clipping at 255. This is why bright regions in the "wrong" result
picture turn into black patches — a "sawtooth" wraparound, not a graceful clip.

**Correct approach:**
```python
im2 = cv.add(im1, 100)
```
`cv.add` performs **saturating arithmetic**: any sum over 255 is clipped to 255, not wrapped.
200 + 100 → 255 (not 44).

**Numeric check for yourself:** pick pixel value 180. Plain `+100` (with wraparound) → `280 mod
256 = 24`. `cv.add` → `min(280, 255) = 255`. Try this arithmetic on a few values yourself (e.g.
50, 160, 250) with both rules — that's exactly the reasoning the tutorial's gamma/transform
questions will expect from you when judging "what's wrong with this."

---

## 6. Working of a Camera — Bayer Filter and Demosaicing

A camera sensor (CMOS) only measures **light intensity**, not color — it's colorblind. To get
color, a **Bayer filter** (a mosaic of tiny R/G/B color filters) sits in front of the sensor, so
each individual sensor cell only receives one color's worth of light. The classic Bayer pattern
uses **twice as many green filters as red or blue**, because human vision is more sensitive to
green — this improves the *perceived* resolution and reduces noise in the luminance channel.

The result: each pixel location only has *one* of R, G, or B recorded directly. **Demosaicing**
is the process of estimating the two missing color values at each pixel from neighboring pixel
values (interpolation) — e.g., a pixel that only measured "Green" gets its Red and Blue values
estimated from its red/blue neighbors.

---

## 7. Color Models

A color model = (1) a coordinate system + (2) a subspace, so every color maps to one point.

| Model | Use | Notes |
|---|---|---|
| **RGB** | Image capture (cameras) | Additive; wavelength-based |
| **CMYK** | Printing | Subtractive (Cyan, Magenta, Yellow, Black) |
| **HSV** | How humans describe color | Decouples color (Hue, Saturation) from brightness (Value) |

**Why HSV matters for point operations:** if you want to change *brightness* without shifting
*hue*, you convert to HSV, modify only the V channel, then convert back — modifying R, G, B
directly (like brightness addition above) shifts all three channels and can distort color
balance, especially near clipping (255).

---

## 8. Intensity Transformations — the General Framework

**Key property:** the output value of a pixel depends *only* on that pixel's own input value —
never on its neighbors. (This is what distinguishes point operations from spatial/linear
filtering, which you'll cover next.)

```
Input image:  f(x)
Output image: g(x)
Transform:    g(x) = T(f(x))
```

`T` is just a function from `[0,255] → [0,255]`. In code, the standard trick is to **precompute
T as a 256-element lookup table (LUT)**, then apply it to the whole image at once:

```python
t = np.arange(256, dtype=np.uint8)   # this particular t is the identity transform
g = t[f]                              # index the LUT using every pixel value in f as an index
```

**Why this works:** `f` is an array of values each in `[0,255]`. `t[f]` uses *fancy indexing* —
NumPy replaces every element of `f` with `t[that element]`. So if `f[i,j] = 130`, then
`g[i,j] = t[130]`. This is equivalent to, but far faster than, looping over every pixel and
applying `T` individually (the "slow" nested-for-loop version in the slides exists specifically
to contrast with this).

`cv.LUT(img_orig, transform)` does the identical thing via OpenCV's own optimized C++ routine
instead of NumPy fancy indexing — same math, different implementation. **This is worth being
able to explain in your own words for the tutorial: `cv.LUT` and `transform[img_orig]` are two
implementations of the same lookup-table indexing operation.**

---

## 9. Identity and Negative Transforms

- **Identity:** `T(f) = f`, i.e. `g(x) = f(x)`. No change. `t = np.arange(256, dtype=np.uint8)`.
- **Negative:** `g(x) = 255 − f(x)`. Dark ↔ bright flips. `t = np.arange(255, -1, -1,
  dtype=np.uint8)`. This is a straight line from (0,255) to (255,0) on the transform graph.
  Used in radiology because certain structures (e.g. subtle tissue variations) become easier for
  the eye to pick out when displayed as their negative.

**Self-check:** what is `t[25]` for the negative transform? `255 − 25 = 230`. Confirm you can
compute a handful of these by hand before you trust code.

---

## 10. Intensity Windowing (piecewise-linear contrast stretch)

**Goal:** stretch a narrow *input* range to occupy a wider *output* range, so contrast within
that range of interest increases. Outside that range, detail is sacrificed (clipped to flat
low/high).

The transform is a **3-segment piecewise-linear function** defined by two corner points
`(x1, y1)` and `(x2, y2)`:
- for `f < x1`: linear ramp from `(0, 0)` to `(x1, y1)`
- for `x1 ≤ f ≤ x2`: steep linear ramp from `(x1, y1)` to `(x2, y2)` — this is the "stretched"
  region
- for `f > x2`: linear ramp from `(x2, y2)` to `(255, 255)`

**Worked example (different numbers from the slide, so you can practice the same steps):**
Suppose corner points are `(80, 20)` and `(180, 220)`.

- Segment 1 (`0→80` maps to `0→20`): slope = 20/80 = 0.25
- Segment 2 (`80→180` maps to `20→220`): slope = 200/100 = 2.0 ← the steep, contrast-boosting
  part
- Segment 3 (`180→255` maps to `220→255`): slope = 35/75 ≈ 0.47

For an input pixel `f = 130` (inside the steep segment):
`g = 20 + (130 − 80) × 2.0 = 20 + 100 = 120`

Try computing `g` for `f = 40` and `f = 220` yourself using segments 1 and 3, respectively — that's
the exact operation the tutorial's windowing-style questions test.

**Code pattern** (`c` holds the two corner points):
```python
c = np.array([(80, 20), (180, 220)])
t1 = np.linspace(0, c[0,1], c[0,0] + 1 - 0).astype('uint8')
t2 = np.linspace(c[0,1] + 1, c[1,1], c[1,0] - c[0,0]).astype('uint8')
t3 = np.linspace(c[1,1] + 1, 255, 255 - c[1,0]).astype('uint8')
transform = np.concatenate((t1, t2, t3), axis=0).astype('uint8')
```

---

## 11. Gamma Correction

```
g = f^γ ,   f ∈ [0, 1]   (image must be normalized to [0,1] first!)
```

- `0 < γ < 1` → **brightens**: maps a *narrow* range of dark pixels to a *wider* range → dark
  regions get more visible detail/contrast.
- `γ > 1` → **darkens**: opposite effect — compresses dark pixels together, expands the bright
  end.
- `γ = 1` → identity, no change.

**Worked example:** normalized pixel `f = 0.25`.
- `γ = 0.5` → `g = 0.25^0.5 = 0.5` (pixel got *brighter* — pulled up)
- `γ = 2` → `g = 0.25^2 = 0.0625` (pixel got *darker* — pushed down)

**Intuition:** think of a room photographed against a bright window. The *inside* of the room
is dark relative to outside. To reveal detail *inside* the room, you want to spread out (expand)
the narrow range of dark values the room occupies — that's `γ < 1`. If you wanted to see detail
*outside* the bright window instead, you'd want to expand the narrow range of *bright* values
instead — that requires `γ > 1`. This directly informs the "why did window detail change / not
change" and "possible reasons for no visible effect" style tutorial questions — the deciding
factor is **where in the intensity range the region of interest sits**, and whether that region
is a narrow or already-wide slice of the histogram.

**Code:**
```python
t = np.array([(i/255.0)**gamma * 255 for i in np.arange(0, 256)]).astype(np.uint8)
g = cv.LUT(f, t)
```

---

## 12. Histograms

- `h(rₖ)` = number of pixels with intensity level `rₖ`.
- Normalizing by total pixel count `n = M×N` gives the **probability mass function**:
  `p(rₖ) = h(rₖ) / n` — an estimate of how likely a random pixel is to have that intensity.

**Worked example** (different from the slide's): a 2×4 image (8 pixels), 3-bit range `[0,7]`,
with values `[1, 1, 3, 3, 5, 5, 5, 7]`.

`h(0)=0, h(1)=2, h(2)=0, h(3)=2, h(4)=0, h(5)=3, h(6)=0, h(7)=1`

Check: `Σh(i) = 2+2+3+1 = 8` ✓ (must always equal total pixel count — use this to catch
arithmetic mistakes).

`p(1) = 2/8 = 0.25`, `p(5) = 3/8 = 0.375`, etc.

**Reading histogram shape:**
- Dark image → mass concentrated at low intensities (left).
- Bright image → mass concentrated at high intensities (right).
- Two separated bright/dark regions → **bimodal** histogram.
- Flat histogram → uniform distribution of all intensities → high contrast, "vibrant" image
  (what histogram equalization tries to produce).

---

## 13. Histogram Equalization

**Idea:** find a transform `T` that reshapes an arbitrary histogram into one that's as flat as
possible, by using the **cumulative distribution function (CDF)** of the input intensities as
the transform itself.

Discrete formula:
```
s = T(k) = (L−1)/(MN) × Σ_{j=0}^{k} n_j ,   k = 0, ..., L−1
```
where `L` = number of gray levels (256 for 8-bit, 8 for 3-bit), `MN` = total pixel count, and
`n_j` = count of pixels at level `j`. The sum term is the running (cumulative) pixel count up to
level `k` — i.e., PDF → CDF. Multiplying by `(L−1)/MN` rescales that cumulative count back into
the valid intensity range `[0, L−1]`.

**Worked example, full walkthrough** (a fresh small example so you can check your own tutorial
work against a method you've seen solved start to finish): 2-bit image (`L = 4`, levels `[0,3]`),
size 2×2 = 4 pixels (`MN = 4`), with counts `n₀=1, n₁=1, n₂=1, n₃=1` (perfectly uniform, just to
make arithmetic transparent).

| k | nₖ | cumulative Σn | (L−1)/MN × Σn = 3/4 × Σn | rounded |
|---|----|----------------|---------------------------|---------|
| 0 | 1 | 1 | 0.75 | 1 |
| 1 | 1 | 2 | 1.50 | 2 |
| 2 | 1 | 3 | 2.25 | 2 |
| 3 | 1 | 4 | 3.00 | 3 |

New lookup table: `t = [1, 2, 2, 3]`. Notice: since this input was already uniform, equalization
barely changes it (as expected — a flat histogram is the target, and this one started flat).

**What to notice in general** (this is the part worth understanding conceptually, not just
mechanically):
1. The transform curve is always **non-decreasing** (since it's a running sum) — order of
   intensities is preserved, dark stays darker than bright.
2. Because the mapping is built from *integer* rounding of a continuous-looking curve, **several
   input levels can round to the same output level** — this is exactly why, in a real
   equalization, some output bins end up empty (input levels get "merged") while others gain
   extra mass. A perfectly flat output histogram is therefore not actually achievable for
   discrete data — only approximately flat.
3. Where the input histogram is *sparse* (few pixels at a level), the CDF barely rises there, so
   that whole range gets compressed onto very few output levels — visually, images with large
   flat/empty histogram regions equalize more dramatically than images with rich detail
   everywhere.

**Code — two equivalent ways:**
```python
g = cv.equalizeHist(f)                                          # built-in
t = np.array([(L-1)/(M*N)*cdf[k] for k in range(256)], dtype=np.uint8)
g = t[f]                                                          # manual, using the formula
```

---

## How this maps onto Tutorial 1

Going through Tutorial 1's structure and pointing you to the right tool above — **without
solving the graded questions for you**:

- **Q1(a)** (gamma value + justification, window visibility): pure application of §11. Look at
  whether the *overall* image got brighter or darker, then reason about which regions (dark vs.
  bright) show more detail afterward — that tells you whether γ<1 or γ>1 was used. For "no
  visual enhancement on a bright window" — think about what a γ transform does to values that
  are *already* near 255, and what fraction of the histogram the window pixels represent.
- **Q1(b)** (zooming/interpolation, nearest neighbor vs. bilinear) is **not** in this L02
  deck — it's interpolation/sampling content (likely your L01 or a separate lecture). I can
  teach that separately if you share those slides; the core ideas (nearest-neighbor rounding vs.
  weighted averaging of 4 neighbors) are straightforward once you have the reference material.
- **Q1(c)** (Gaussian kernels, rank, repeated smoothing) is linear filtering, not point
  operations — also outside this deck's scope. Flag it for your linear filtering lecture.
- **Q2(a)–(b)** (intensity transform examples, `s = c·ln(r)` log transform, LUT application):
  directly §8–§9. The log transform behaves like an extreme case of gamma correction with γ→0 —
  work out its shape (concave, compresses bright values, expands dark ones) using the same
  reasoning as §11's gamma discussion, then reuse the LUT code pattern in §8 to think about how
  you'd implement it. For "reason for incorrectness" in a plotted result, check for exactly the
  kind of overflow/clipping/normalization issue discussed in §5 and §11 (image not normalized to
  [0,1] before applying the formula is the classic bug).
- **Q2(c)** (windowing with L/U from cumulative PMF, "drop bottom/top 10%"): this *is* §10, but
  driven by the histogram/CDF from §12–13 rather than fixed corner points — find L and U from
  where the CDF crosses 0.1 and 0.9, then apply the exact 3-segment piecewise formula from §10.
- **Q2(d)** (Laplacian kernel, discretization) is linear filtering — outside L02, flag for later.
- **Q3(a)** (sketch histograms of processed images, identify the operation): pure §12–§13
  pattern-matching — compare each processed histogram's *shape* (shifted left/right, compressed,
  spread out/flattened) against the effects catalogued in §9–§11 and §13.
- **Q3(b)** (transfer function + log transform combined): same tools as Q2(b), applied twice in
  sequence — work out each transform's LUT, then compose them (apply the second transform to the
  output of the first).
- **Q3(c)**, **Q4**: Laplacian of Gaussian, HDR windowing, foreground/shadow compositing,
  morphology — these span linear filtering, HDR-specific windowing (an extension of §10 to a
  16-bit input range), and morphological operations, none of which are in this L02 deck. Q4(a)
  is a windowing problem structurally identical to §10 but with input range up to 65,535 instead
  of 255 — same 3-segment logic, larger numbers.

If you upload the linear filtering / interpolation slide deck, I can build the same kind of
guide for those topics so the rest of Tutorial 1 is fully covered too.
