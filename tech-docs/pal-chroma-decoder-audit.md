# PAL chroma-decoder audit (2026-06-10)

Audit of `src/ld-chroma-decoder/` (palcolour, transformpal2d, transformpal3d)
driven by the handoff notes in `dp111/ld-decode`
`tech-docs/pal-audit-handoff.md` (branch `claude/upstream-branch-sync-jnqcx7`).
That handoff covered RF-side work in ld-decode and listed open items to check
on the chroma-decoder side; this document records the result of working
through them.

A second pass added an **empirical** investigation ("is there a better
decoder?") backed by a reproducible measurement harness; see "Empirical
validation" below. The headline conclusion is that the existing PAL decoders
are already well-tuned: the defaults that looked improvable on noise-free
synthetic data do not survive a realistic-noise test, and the one in-code
"this looks wrong" note (`transformpal3d` `z_ref`) is actually correct. The
concrete code changes are two correctness items (the `in5` guard below and the
misleading `z_ref` comment) plus the new benchmark script.

## Bug found and fixed

### `palcolour.cpp`: `in5` vertical-tap boundary guard off by one

`PalColour::decodeLine()` builds a 7-line vertical aperture (`in0` = line *n*,
`in1/in2` = *n*∓1, `in3/in4` = *n*∓2, `in5/in6` = *n*∓3). Each pointer is
clamped to a black line when the tap falls outside the active region. The
`in5` clamp tested the wrong line:

```cpp
// before
in5 = (line.number - 2) <  firstLine ? blackLine : (chromaData + ((line.number - 3) * videoParameters.fieldWidth));
// after
in5 = (line.number - 3) <  firstLine ? blackLine : (chromaData + ((line.number - 3) * videoParameters.fieldWidth));
```

The pointer addresses line *n*−3 but the guard tested *n*−2, so the guard was
asymmetric with its mirror `in6` (which correctly tests *n*+3) and with the
intended −3 tap.

**Effect.** The divergence occurs on exactly one line per field — when
`line.number == firstLine + 2` — where `in5` read the line *just above* the
active region (`firstLine - 1`) instead of the black line. It is the
outermost vertical tap, weighted by the smallest coefficients (`cfilt[b][3]`),
and `firstLine - 1` is always ≥ 0, so this was never a memory-safety issue —
just a small luma/chroma artifact on the third active line near the top of
each field. It is most visible in native PALcolour mode, where that line is
real composite (sync/blanking-adjacent) content; in Transform-PAL mode the
prefiltered chroma buffer is zero outside the active region, so the wrong read
is ≈ 0 and the artifact is negligible.

This bug is also present on `origin/main`; it is a long-standing latent issue,
not a regression introduced on this branch.

## Handoff items checked — no change required

- **Group-delay double-correction.** The handoff flagged that ld-decode now
  applies an IEC 60856 9.1.6 group-delay equaliser end-to-end, so any *legacy*
  GD compensation in the chroma decoder would now double-correct. Searched the
  whole decoder: there is no group-delay / phase-equalisation stage. The only
  filter delays are the symmetric-FIR delays of `doYNR` (luma NR) and the
  Simple-PAL UV post-filter, both compensated locally. **No double-correction
  risk.**

- **Chroma-gain / subcarrier calibration vs burst.** The reference carrier
  (`sine`/`cosine` LUTs) and the burst phase detector both use
  `videoParameters.fSC`. For PAL the default is `283.75 × 15625 + 25 =
  4 433 618.75 Hz` (`lddecodemetadata.cpp`), i.e. exactly 4.43361875 MHz — the
  exact PAL subcarrier the handoff calls for. Chroma gain is normalised
  per-line against the measured burst magnitude (`burstNorm`), so saturation
  tracks burst amplitude rather than an absolute constant. **Correct.**

- **Filter symmetry around fSC (band-tilt interaction).** Both PALcolour's
  raised-cosine FIR (`buildLookUpTables`) and the Transform-PAL reflection
  filters assume symmetry about fSC. The handoff notes the TBC now delivers a
  mild residual tilt (≈ −1.4 dB at fSC vs LF, falling fast above 5.3 MHz). This
  is a property of the upstream RF response, not a decoder defect: the filters
  are correctly centred on fSC. It would only bias *saturation slightly low* on
  the highest chroma frequencies; correcting it belongs in the RF/MTF
  calibration campaign the handoff already lists as open, not here.

- **V-switch / line-pairing.** `detectBurst` determines V-switch by comparing
  the burst-vector difference between the current line and the
  opposite-V-switch neighbour average against the burst magnitude; the ±2-line
  averaging keeps phase reference on same-V-switch lines. Logic is internally
  consistent and unchanged. No issue found.

- **Transform-PAL tiling / window overlap.** 2D uses 32×16 tiles with
  half-tile overlap and a symmetric raised-cosine window so inverse tiles sum
  without an inverse window; 3D extends this in Z with documented
  look-behind/look-ahead. X-bin range (`XTILE/8 … XTILE/4`, 0.5–1.5 fSC) matches
  the ±1.3 MHz chroma band. The `z_ref` temporal reflection carried an in-code
  `XXX` note suggesting `ZTILE/4` was wrong; it is in fact correct (verified
  empirically — see "Empirical validation"), and the misleading comment has
  been rewritten.

## Not addressed (belong to other work)

The remaining handoff open items — MTF over-equalisation (~+1.5 dB at 4.8 MHz),
fold-over distortion, and the multi-disc MTF calibration campaign — are
ld-decode RF-side items requiring a calibration dataset, not chroma-decoder
code changes.

## Empirical validation ("can we do better?")

To answer this with numbers rather than opinion, a self-contained benchmark was
added: `scripts/measure-pal-chroma`. It generates a synthetic ground-truth
image, encodes it to a TBC with `ld-chroma-encoder`, optionally adds Gaussian
noise, decodes it with `ld-chroma-decoder`, and compares the result to the
original. Two patterns are used:

- **Wide colour bars** (bandwidth-friendly) — reconstruction PSNR measures
  chroma/luma fidelity.
- **Greyscale frequency sweeps** — since the source has no chroma, any U/V in
  the decode is luma->chroma cross-colour; mean |UV| (lower = better) and the
  luma PSNR measure Y/C separation.

The same encode/decode path was also confirmed to run on the real PAL captures
in `test-data/pal/` (the GGV multiburst and the Kagemusha colour-bar leadout
referenced by the handoff).

### Decoder baseline (noise-free; numbers in dB unless noted)

```
decoder        barRGB    barY    barU    barV | greyXcolor(|UV|)  greyY
pal2d           27.74   37.48   31.27   30.58 |          384.2   20.02
transform2d     28.20   47.40   31.31   30.68 |           29.5   41.47
transform3d     28.23   49.63   31.33   30.68 |           20.7   44.93
```

This matches expectations: Transform PAL separates luma far better than the
PALcolour 2D FIR (47-50 dB vs 37 dB barY; 20-30 vs 384 cross-colour), and 3D
beats 2D on static content. Chroma fidelity (barU/barV) is similar across all
three — the decoders differ mainly in luma / cross-colour, not in chroma.

### Transform threshold (default 0.4) — confirmed well-chosen

On noise-free data, *raising* the threshold (stricter symmetry test, keeps
fewer bins) reduced cross-colour and improved greyscale luma with no colour
loss, which naively suggests the default is too low. But the threshold's real
job is noise rejection, so it was re-swept with injected RF noise:

```
transform2d, greyscale luma PSNR (dB) vs threshold
 noise     0.3    0.4    0.5    0.6    0.7
 40 dB   37.23  38.68  38.96  39.19  39.59
 34 dB   34.48  34.97  34.90  34.76  34.64
 30 dB   31.56  31.66  31.45  31.18  30.90
```

At realistic LaserDisc SNRs (30-34 dB) luma PSNR peaks at ~0.4 and colour-bar
fidelity degrades monotonically as the threshold rises. The noise-free gain was
a mirage. **Default 0.4 stands.**

### PALcolour chroma bandwidth (`1.1 MHz / 0.93`) — Pareto, not a bug

Sweeping the bandwidth constant trades cross-colour against chroma sharpness
with no free lunch: narrower (~1.05 MHz) lowers cross-colour and lifts luma but
softens saturated colour; wider does the reverse. The shipped ~1.18 MHz sits in
a sensible spot favouring colour fidelity. Picking a different point is an
aesthetic/real-disc-calibration decision (the handoff's deferred campaign), not
a correctness fix, so the value is left unchanged.

### `transformpal3d` `z_ref` reflection axis — code is correct, comment fixed

An in-code `XXX` claimed the temporal reflection should be about 18.75 Hz
(`(6*ZTILE)/8`) rather than the shipped 6.25 Hz (`ZTILE/4`). Testing all three
candidate centres on a static loopback:

```
z_ref reflection centre            barY/greyY luma PSNR
 6.25 Hz  (ZTILE/4,   shipped)        ~37-49 dB   <-- best by far
12.5 Hz  (ZTILE/2)                    ~15-16 dB
18.75 Hz ((6*ZTILE)/8, "suggested")  ~12-13 dB
```

The shipped value wins by ~24 dB. The reason is physical: a static PAL picture
repeats over the 8-field sequence, so its chroma temporal carrier is at the
field rate / 8 = 6.25 Hz = bin `ZTILE/8`, and the reflection constant is twice
that, `ZTILE/4`. The `18.75 Hz` in the comment was a misidentification (it is
the 3rd harmonic). The misleading comment has been corrected so the working
code is not "fixed" into a 24 dB regression later.

### `in5` guard fix — verified scope

The `palcolour.cpp` fix above was confirmed with the harness to change exactly
two lines per frame (the 3rd active line of each field, lines 4 and 5), as
predicted, with no effect elsewhere.

### Reproducing

```
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release && ninja -C build \
    ld-chroma-decoder ld-chroma-encoder
scripts/measure-pal-chroma --build-dir build --noise 0 376 750 1190
```

## Rightward "ghost" on sharp transitions — diagnosis and an adaptive canceller

Symptom: a displaced replica to the right of high-contrast (especially
black<->white) transitions.

**It is not the chroma decoder.** A clean encode->decode loopback reproduces a
sharp black/white edge with no displaced replica (transform2d/3d are essentially
perfect; PALcolour shows only small symmetric edge ripple). The decoder's FIR /
FFT filters are horizontally symmetric, so they cannot produce a one-sided
right-hand echo. A rightward echo is therefore a *composite-domain* linear
distortion already present in the TBC — a single-bounce reflection in the
player / RF / capture chain:

    composite[x] = clean[x] + a * clean[x - d]      (0 < a < 1, d > 0 samples)

Because it is upstream of and independent of colour decoding, it is best removed
on the composite signal (which fixes luma *and* chroma together) by the exact
inverse recursive filter:

    clean[x] = composite[x] - a * clean[x - d]

`scripts/deghost-tbc` implements this as a TBC->TBC pre-decode filter:

- **Cancellation core is exact.** Injecting a known echo into a synthetic TBC
  and cancelling with the true (a, d) restores the decoded PSNR to the
  echo-free baseline to within 0.01 dB (e.g. colour bars 28.20 -> 17.20 with a
  0.15/20 echo -> 28.20 after cancel).
- **Amplitude is auto-fitted** at a given delay by a 1-D search that nulls the
  residual echo, measured on the composite luma band (an 8-tap boxcar nulls the
  0.5-1.5 fSC chroma band). On pure-luma content this is accurate (a 0.20/15
  edge echo: fitted 0.188, decoded 23.1 -> 48.8 dB; a greyscale 0.12/22 echo:
  18.8 -> 31.8 dB). On fully-saturated colour bars, where every luma edge
  coincides with a chroma edge, it under-corrects (~0.5x) but still improves;
  pass `--amplitude` to override.
- **Delay** is supplied by the user (`--delay N`, the ghost offset in 4fSC
  samples, which is directly readable from the picture) — the reliable mode.
  `--auto` adds experimental blind delay estimation from the autocorrelation of
  the decoded-luma gradient (decoding first removes chroma). This is reliable on
  natural content / isolated edges but can lock onto regular periodic structure
  (e.g. coherent colour-bar edge-ringing), so the reported delay should always
  be checked against the visible ghost offset.

Caveat: the model is a single first-order echo at integer-sample delay. Multiple
or fractional-delay ghosts would need a short adaptive FIR equaliser; the same
residual-nulling framework extends to that.
