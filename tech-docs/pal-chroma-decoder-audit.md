# PAL chroma-decoder audit (2026-06-10)

Audit of `src/ld-chroma-decoder/` (palcolour, transformpal2d, transformpal3d)
driven by the handoff notes in `dp111/ld-decode`
`tech-docs/pal-audit-handoff.md` (branch `claude/upstream-branch-sync-jnqcx7`).
That handoff covered RF-side work in ld-decode and listed open items to check
on the chroma-decoder side; this document records the result of working
through them.

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
  the ±1.3 MHz chroma band. The `z_ref` reflection carries an in-code `XXX`
  note questioning `ZTILE/4`, but it is consistent with the documented 18.75 Hz
  axis and reflection target; left as-is (pre-existing, out of scope for this
  audit).

## Not addressed (belong to other work)

The remaining handoff open items — MTF over-equalisation (~+1.5 dB at 4.8 MHz),
fold-over distortion, and the multi-disc MTF calibration campaign — are
ld-decode RF-side items requiring a calibration dataset, not chroma-decoder
code changes.
