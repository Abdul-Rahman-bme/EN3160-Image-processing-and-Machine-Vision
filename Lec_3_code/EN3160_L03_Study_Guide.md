# EN3160 — L03: Spatial Filtering — Study Guide

Same approach as the L02 guide: everything below matches your L03 deck, with fresh worked
examples (different numbers from your slides/tutorial) so you do the arithmetic yourself.
Tutorial mapping is at the end.

---

## 1. Point Operations vs. Spatial Filtering — the One Distinction Everything Rests On

- **Point operation** (L02): `g(x) = T(f(x))` — output pixel depends **only on that same input
  pixel**. No neighbors involved.
- **Spatial filtering** (L03): output pixel depends on a **neighborhood** of input pixels around
  it — a window slides across the image, and at each position it combines the pixels under it
  into a single output value. That's why it's called "spatial" — the space around the pixel now
  matters.

Two uses you'll keep coming back to: **enhancement** (smoothing/noise removal, sharpening) and
**template matching** (finding where a small pattern occurs in a larger image, by sliding it
around and measuring similarity at each position).

---

## 2. The Sliding-Window Mechanic — Worked by Hand

Take a small 5×6 image (rows×cols) and a 3×3 averaging kernel (all entries 1/9):

```
f =
0   0   0   0   0   0
0  60  120  60   0   0
0 120  180 120   0   0
0  60  120  60   0   0
0   0   0   0   0   0
```

The kernel's **center** cell aligns with the output pixel being computed. To get the output at
position `(2,2)` (0-indexed, the "180" location), take the 3×3 neighborhood centered there:

```
  0   60  120
 60  180  120
  0   60  120
```

Multiply elementwise by 1/9 and sum: `(0+60+120+60+180+120+0+60+120)/9 = 720/9 = 80`.

That's the entire operation, repeated at every valid pixel position. **Try computing the output
at `(1,1)` yourself** using the same 3×3 neighborhood centered on the "60" at `(1,1)` — this is
exactly the manual "drag the kernel across the grid" exercise your slides walk through visually.

**Key fact worth internalizing:** at the exact center position where a bright/dark isolated spot
sits (like the "180" here), the averaging kernel spreads that value out over its neighborhood —
this is precisely why box/Gaussian filters blur: sharp, localized values get diluted into their
surroundings.

---

## 3. Boundaries — Why You Need Padding, and What Kind

If you only compute outputs where the kernel fully fits inside the image, the output shrinks —
for a 3×3 kernel you lose 1 pixel of border all around; for a 5×5 kernel you lose 2 pixels all
around (**general rule: kernel size `k` → lose `(k−1)/2` pixels on each side**, so a 7×7 kernel
loses 3 pixels per side).

To keep output the **same size** as input, you pad the image first. OpenCV border modes (with
`|` marking the real image boundary):

| Mode | Pattern | Behavior |
|---|---|---|
| `BORDER_CONSTANT` | `iii\|abcdefgh\|iii` | Fixed value (often 0 — "zero padding") outside |
| `BORDER_REPLICATE` | `aaa\|abcdefgh\|hhh` | Repeats the edge pixel |
| `BORDER_REFLECT` | `cba\|abcdefgh\|hgf` | Mirrors, **including** the edge pixel itself |
| `BORDER_REFLECT_101` | `dcb\|abcdefgh\|gfe` | Mirrors, **excluding** the edge pixel (the default `filter2D` uses) |
| `BORDER_WRAP` | `fgh\|abcdefgh\|abc` | Treats the image as periodic (wraps to the opposite edge) |

**Worked check:** for a 1D signal `[1,2,3,4,5]`, padding by 2 on each side:
- `BORDER_REPLICATE` → `[1,1,1,2,3,4,5,5,5]`
- `BORDER_REFLECT` → `[2,1,1,2,3,4,5,5,4]`
- `BORDER_REFLECT_101` → `[3,2,1,2,3,4,5,4,3]`
- `BORDER_WRAP` → `[4,5,1,2,3,4,5,1,2]`

Match each result to its rule yourself before trusting the table — that's the actual skill being
tested when a tutorial gives you a border-type question.

**Which mode for what?** Zero-padding is standard for edge detection (keeps derivative filters
honest at the boundary — no artificial gradient introduced by fabricated pixel values).
Replicate/reflect are better for smoothing filters on natural photos (avoids a hard black edge
after blurring). Wrap is really only appropriate for genuinely periodic data.

---

## 4. Kernel Choice Determines What You're Measuring

Two kernels used on the same image produce very different outputs because they're measuring
different things:

- **Averaging kernel** (all 1/9): computes a local mean → smooths/blurs, suppresses noise **and**
  fine detail together (it can't tell them apart).
- **Sobel kernel** (e.g. the vertical-edge kernel `[[-1,-2,-1],[0,0,0],[1,2,1]]`): computes a
  weighted difference between the pixels below and above the center → responds strongly at
  **edges**, near zero on flat regions.

The intuition tying these together: **correlation output = a dot product between the kernel and
the local patch of image**, which measures how *similar* the two are. A patch that looks like
the kernel produces a high output; a patch orthogonal to (very different from) the kernel
produces near zero. So: an averaging kernel is "looking for" a uniform patch (and any patch
scores fairly high, since it's just measuring average brightness); a Sobel kernel is "looking
for" a specific edge orientation, so only patches with that gradient pattern score high. This is
the same principle **template matching** relies on.

---

## 5. Convolution vs. Correlation — the Flip

**Correlation** (what "sliding and dot-producting" naturally computes):
```
(w ⊛ f)[m,n] = Σ_s Σ_t w[s,t] · f[m+s, n+t]
```

**Convolution** (the "proper" signal-processing operation):
```
(w * f)[m,n] = Σ_s Σ_t w[s,t] · f[m−s, n−t]
```

The only difference: convolution **flips the kernel 180°** (both axes) before sliding it. If the
kernel is symmetric under 180° rotation (true for averaging, Gaussian, and — check this
yourself — the Laplacian kernel), flipping does nothing, so correlation and convolution give
identical results. For an **asymmetric** kernel (Sobel is asymmetric), they differ, and mixing
them up is a real source of bugs. `cv.filter2D` performs **correlation**, not convolution — it
does **not** flip the kernel internally.

**Full worked example** (fresh numbers, different from your slides): image is a 5×5 array of
zeros with a single `1` at position `(1,3)` (row 1, col 3):
```
f =
0 0 0 0 0
0 0 0 1 0
0 0 0 0 0
0 0 0 0 0
0 0 0 0 0
```
kernel:
```
w =
9 8 7
6 5 4
3 2 1
```

**Step 1 — pad** `f` by 1 pixel of zeros on every side (so a 3×3 kernel produces same-size
output) → 7×7 padded array with the `1` now at `(2,4)`.

**Step 2 — correlation.** Since `f` is zero everywhere except one pixel, correlating simply
**stamps the kernel itself**, unflipped, centered at that pixel's location:
```
(w ⊛ f) — 5×5 region around where the 1 was:
0 0 0 0 0
0 9 8 7 0
0 6 5 4 0
0 3 2 1 0
0 0 0 0 0
```

**Step 3 — convolution.** Convolution flips the kernel 180° first:
```
flipped w =
1 2 3
4 5 6
7 8 9
```
then stamps *that* at the impulse location:
```
(w * f) —
0 0 0 0 0
0 1 2 3 0
0 4 5 6 0
0 7 8 9 0
0 0 0 0 0
```

Notice: correlation gives you the kernel as-written; convolution gives you the kernel
**rotated 180°**. This single-impulse trick — "convolving with a delta function just stamps the
(possibly flipped) kernel at that location" — is worth keeping as a mental shortcut; it's exactly
why your slides use a single "1" pixel to demonstrate the difference.

---

## 6. Convolution's Algebraic Properties (and why they matter practically)

- **Commutative:** `a*b = b*a` — kernel and image are interchangeable, conceptually.
- **Associative:** `a*(b*c) = (a*b)*c` — apply several filters in sequence = apply one combined
  filter.
- **Distributes over addition:** `a*(b+c) = a*b + a*c`.
- **Linear + shift-invariant** ⇒ (by a real theorem) any operator with those two properties
  *can* be written as a convolution — this is *why* convolution is the natural tool, not an
  arbitrary choice.

**Why associativity matters for computation cost — worked example (different numbers from your
slides):** suppose you filter a 2000×2000 image with two 5×5 kernels applied in sequence.

- Applying them one after another: `2 × (5² × 2000²) = 2 × 25 × 4,000,000 = 200,000,000`
  multiply-adds.
- Combining the two 5×5 kernels first (`b*c`, a 5×5 * 5×5 convolution → produces up to a 9×9
  combined kernel) costs `5² × 5² = 625`, then applying that combined kernel to the image costs
  `9² × 2000² = 324,000,000`.

Here combining first is actually *worse*, because the combined kernel (9×9) is bigger than
either original — associativity only saves computation when the *combined* kernel stays small
relative to the image, which is usually true only when you're combining many *tiny* kernels
(e.g. repeated small blurs) rather than kernels that are already a meaningful fraction of the
image. **Compute this trade-off yourself for two 3×3 kernels on a 1000×1000 image (mirroring the
handwritten example in your slides) before assuming "combine first" is always cheaper** — it
isn't automatically.

---

## 7. Sharpening — Unsharp Masking

**Idea:** an image = low-frequency content (smooth regions) + high-frequency content (edges,
fine detail). Smoothing *removes* high-frequency content. So:

```
high-frequency detail = original − smoothed
sharpened = original + (original − smoothed)
```

You're literally re-adding the edge information back on top of the original, boosting contrast
right at edges. This is why sharpening halos appear near strong edges — you're adding a
scaled copy of the edge-detector output there.

**Code:**
```python
smoothed = cv.GaussianBlur(im, (5,5), sigmaX=1)
high_freq = cv.subtract(im, smoothed)     # use saturating subtract, not im - smoothed
sharpened = cv.add(im, high_freq)          # and saturating add, for the same overflow reasons as L02 §5
```

---

## 8. Box Filter vs. Gaussian Filter

Both are "averaging" filters, but weighted differently:
- **Box filter:** every pixel in the window gets **equal weight** (flat).
- **Gaussian filter:** weight falls off smoothly from the center — nearby pixels matter more
  than far ones.

**Why this matters (this is the "what's wrong with this" question your slides pose):** a box
filter's hard cutoff (sharp edges of the window) introduces artifacts — in the frequency domain
a box function's response is a `sinc`, which has strong side-lobes (ringing, related to the
Gibbs phenomenon) — you can sometimes see faint repeated edges/ringing near sharp boundaries
after box filtering. A Gaussian's frequency response is itself a Gaussian — no sharp cutoff, no
side-lobes, so it smooths without ringing. **Gaussian filtering is generally preferred over box
filtering for this reason**, even though box filtering is cheaper to compute.

---

## 9. Building a Gaussian Kernel by Hand

```
G(x,y) = 1/(2πσ²) · exp( −(x²+y²) / (2σ²) )
```

The `1/(2πσ²)` normalizing constant can be **dropped** during construction — you renormalize by
dividing by the kernel's own sum at the end anyway, so it cancels out.

**Worked example, σ = 1, 3×3 grid**, coordinates centered at 0 (so the grid spans
`x,y ∈ {−1,0,1}`), un-normalized:

At `(0,0)`: `exp(0) = 1`
At `(±1,0)` or `(0,±1)` (distance² = 1): `exp(−1/2) = 1/√e ≈ 0.6065`
At `(±1,±1)` (distance² = 2): `exp(−1) = 1/e ≈ 0.3679`

```
0.3679  0.6065  0.3679
0.6065  1.0000  0.6065
0.3679  0.6065  0.3679
```

**Normalize** by dividing every entry by the sum of all nine (`≈ 4.8976`), so the kernel sums to
1 — this stops a flat white region from overflowing (getting brighter) after filtering, since a
kernel that sums to 1 preserves overall brightness on a constant region.

**Try this yourself for σ = 2 on a 3×3 grid** (same coordinate grid, different σ) — you'll find
the values are much closer to each other (flatter falloff) than the σ=1 case above, which is the
concrete version of "larger σ → more blur, weights spread more evenly."

**Rule of thumb for kernel size:** set the half-width to about `3σ`, so total kernel size ≈
`2(3σ)+1 = 6σ+1`. For σ=1: `6(1)+1 = 7` (a 7×7 kernel would capture ~99.7% of the Gaussian's
mass — beyond 3σ contributes almost nothing). **Compute the recommended kernel size for σ=2 and
σ=4 yourself** using this formula before checking any reference.

---

## 10. Separability of the Gaussian — Why It's Computationally Special

```
G(x,y) = [1/√(2πσ²) · exp(−x²/2σ²)] × [1/√(2πσ²) · exp(−y²/2σ²)]
```

The 2D Gaussian factors exactly into a product of two identical 1D Gaussians (one in x, one in
y). This means a 2D Gaussian convolution can be done as **two 1D convolutions** — filter every
row with the 1D kernel, then filter every column of *that result* with the same 1D kernel — and
the outcome is mathematically identical to the full 2D convolution.

**Cost comparison, worked with different numbers than your slides:** filtering an `n×n = 500×500`
image with an `m×m = 9×9` Gaussian kernel:
- Direct 2D convolution: `O(n²m²) = 500² × 9² = 250,000 × 81 = 20,250,000` multiply-adds.
- Separable (two 1D passes): `O(n²m) = 500² × 9 × 2 = 250,000 × 18 = 4,500,000` multiply-adds
  (the factor of 2 is for the two passes — row pass then column pass, each `O(n²m)`).

That's roughly a **4.5× speedup** for a 9×9 kernel — and the gap widens as `m` grows (it scales
as `m` vs `m²`). **Redo this comparison yourself for a 15×15 kernel on the same 500×500 image**
to see how much bigger the gap gets — this is exactly the kind of complexity question your
tutorials test.

---

## 11. Noise Models and Why the Right Filter Matters

- **Gaussian noise:** models the sum of many small independent random effects (sensor noise,
  thermal noise) — additive, zero-mean, follows a normal distribution at each pixel
  independently. Gaussian *smoothing* reduces it, at the cost of blurring real detail — larger σ
  removes more noise but blurs more.
- **Salt-and-pepper noise:** isolated pixels randomly slammed to pure black or pure white
  (0 or 255) — not additive Gaussian noise, a completely different corruption model. **Gaussian
  filtering barely helps here** — averaging a 255-spike into a 3×3 neighborhood just smears that
  spike into its neighbors rather than removing it; you still see speckle, just blurred speckle.

**Median filtering fixes salt-and-pepper specifically:** instead of averaging the window, **sort**
the pixel values in the window and take the **middle** one.

**Worked example (different window from your slides):**
```
window =
14  200   9
 11   8   250
 12  13   10
```
Sorted: `8, 9, 10, 11, 12, 13, 14, 200, 250` → median (5th of 9 values) = **12**. The two extreme
corrupted values (200, 250) get completely discarded rather than dragging the average up — this
is the mechanism by which median filtering removes salt-and-pepper spikes without blurring
everything else the way averaging would.

**Median filtering is non-linear** (`median(a) + median(b) ≠ median(a+b)` in general) — it
cannot be expressed as a convolution, so none of §6's convolution algebra (associativity,
combining kernels, separability tricks) applies to it.

---

## 12. Bilateral Filter — Smoothing That Respects Edges

Gaussian smoothing blurs *everything* equally, including real edges you might want to keep
sharp. The bilateral filter fixes this by weighting neighbors by **two** factors simultaneously:

```
I_filtered(x) = (1/Wp) Σ G_σs(‖xᵢ−x‖) · G_σr(‖I(xᵢ)−I(x)‖)
```

- `G_σs(‖xᵢ−x‖)` — the usual **spatial** Gaussian weight (closer pixels count more), exactly
  like a regular Gaussian blur.
- `G_σr(‖I(xᵢ)−I(x)‖)` — a **range/intensity** Gaussian weight: pixels with *similar intensity*
  to the center pixel count more, pixels with very different intensity (i.e., across an edge)
  count less, regardless of how spatially close they are.

**Intuition:** a pixel just across a sharp edge is spatially close but intensity-different, so
the range term suppresses its contribution — the edge doesn't get blurred across. A pixel in a
smooth region nearby is both spatially close and intensity-similar, so it contributes fully —
that region still gets smoothed normally. The net effect: noise in flat regions is reduced (like
Gaussian blur) while edges stay sharp (unlike Gaussian blur).

If `σr` is very large, the range term becomes nearly constant everywhere (since even large
intensity differences map to a weight near 1), so the bilateral filter degenerates into an
ordinary Gaussian blur. If `σr` is very small, only near-identical-intensity neighbors
contribute at all, and almost no smoothing happens even in flat regions.

---

## How this maps onto Tutorial 1 (the parts covered by L03)

Same rule as before — I'm pointing you to the right method with a *different* worked example,
not solving the graded question:

- **Q1(c)i–ii** (1-D Gaussian kernel, σ=1, size 3; then the 3×3 2D Gaussian): directly §9. I
  worked the exact σ=1, 3×3 case above with full numbers — the tutorial's version differs only
  in kernel *size* (size 3 for the 1D part specifically), so build the 1D version first (three
  values along one axis using the same `exp(−x²/2σ²)` formula, x ∈ {−1,0,1}), then form the 2D
  kernel as the **outer product** of that 1D kernel with itself (§10 tells you why this works —
  separability).
- **Q1(c)iii** (rank of the 3×3 Gaussian kernel): use §10's separability fact directly — a
  kernel that's an outer product of two vectors is, by construction, **rank 1** (every row is a
  scalar multiple of every other row). Confirm this by inspecting your own 3×3 result from §9:
  check whether row 2 is a scalar multiple of row 1 and row 3.
- **Q1(c)iv–v** (repeated runs of the kernel, expected resultant σ, and why you don't actually
  get it): this is about **variance addition under convolution** — convolving two Gaussians of
  std devs σ₁ and σ₂ gives a new Gaussian with `σ_result = √(σ₁² + σ₂²)`. Compute what running a
  σ=1 kernel *twice* should theoretically give using that formula, then think about §9's
  kernel-size discussion (the 3×3 kernel truncates the Gaussian's infinite tail at ±1, i.e. way
  short of the recommended `3σ` half-width from §9) — that mismatch between the theoretical
  (infinite-support) Gaussian and the actual truncated discrete kernel is the reason the expected
  σ isn't fully realized.
- **Q2(d)** (Laplacian: forward+backward difference → symmetric second-derivative expression,
  then the 3×3 kernel, then applying it with a specific image, then the zero-padded sum): this
  wasn't explicitly derived in this deck (only the *result* — the `[[1,1,1],[1,−8,1],[1,1,1]]`
  kernel — appeared in your L02 tutorial slide, as an example of a linear filter). The derivative
  itself comes from combining forward difference `f'(x)≈f(x+1)−f(x)` and backward difference
  `f'(x)≈f(x)−f(x−1))`, then differencing those to get the second derivative — work this through
  algebraically yourself; it directly parallels the central-difference note in your slides for
  Sobel (`[−1,0,+1]` as an approximation to a derivative). For the "sum of the Laplacian image
  under zero-padding" question: think about what happens to a kernel that itself sums to
  **zero** (check: `1+1+1+1−8+1+1+1+1 = 0`) when applied to any region, including padded
  boundary regions — a zero-sum kernel applied via correlation/convolution to a flat region
  always outputs zero there.
- **Q3(c)** (Laplacian of Gaussian, un-normalized 3×3 kernel with σ=1.4, filling entries, why
  positive): plug the given formula into the same coordinate grid you used in §9
  (`x,y ∈ {−1,0,1}`), but note LoG values can be *negative* near the center and positive further
  out (it's a "Mexican hat" shape) — re-read the formula's sign structure (`1 − (x²+y²)/2σ²`)
  before assuming any of the 3×3 entries you're computing "should" be positive; work out for
  which `(x,y)` the bracket term is positive vs negative before filling the grid, rather than
  computing blindly.

If you upload the next lecture (sampling/interpolation, or morphology/segmentation), I'll build
the matching guide for those so you eventually have full Tutorial 1 coverage.
