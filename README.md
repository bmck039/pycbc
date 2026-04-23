# PyCBC — `optimized_match` Bug Fix

This is a fork of [gwastro/pycbc](https://github.com/gwastro/pycbc), a Python toolkit for gravitational wave data analysis used by the LIGO/Virgo/KAGRA scientific collaborations.

This fork contains a bug fix for `pycbc.filter.matchedfilter.optimized_match()`, discovered during undergraduate research on distinguishing gravitational lensing from orbital precession in binary black hole merger waveforms.

> **Issue:** [#5144](https://github.com/gwastro/pycbc/issues/5144) — Extreme discontinuities in `optimized_match` that are not present in `match`
> **Pull Request:** [#5146](https://github.com/gwastro/pycbc/pull/5146) — Open, reviewed and approved by PyCBC contributors
> **Branch:** [`optimized_match-debug`](https://github.com/bmck039/pycbc/tree/optimized_match-debug)

---

## The Bug

### How it was discovered

While computing the mismatch (`1 - match`) between lensed and unlensed gravitational wave signals across a parameter space of chirp masses and time delays, extreme discontinuities appeared in the output of `optimized_match()` that were completely absent when using the standard `match()` function on the same waveforms.

The discontinuities appeared as sharp spikes in 1D mismatch plots and as localized high-mismatch regions in 2D contour plots over parameter space — both highly anomalous results for what should be a smoothly varying quantity.

### Root cause

`optimized_match()` uses `scipy.optimize.minimize_scalar` to refine the subsample time-shift that maximizes the match between two waveforms. The original implementation passed a `bracket` parameter to the optimizer instead of `bounds`, meaning the minimizer was **not constrained** to the valid time-shift window `(-delta_t, delta_t)`. In certain regions of waveform parameter space, the unconstrained optimizer found local minima outside this window and returned them as the result — producing physically incorrect match values.

This is a rare failure mode, but one with real scientific consequences: incorrect match values directly corrupt mismatch calculations used in parameter estimation and template bank studies.

### The fix

**`pycbc/filter/matchedfilter.py`** — Changed the `minimize_scalar` call in `optimized_match()` to use the `'bounded'` method with `bounds=(-delta_t, delta_t)`, ensuring the optimizer always respects the valid time-shift window. Added a high-precision convergence step (`xatol=1e-8`) to maintain numerical accuracy equivalent to the original implementation.

**`test/test_matchedfilter.py`** — Added `test_optimized_match_valid()`, a regression test using a known-tricky waveform pair that previously triggered the bug, asserting that `optimized_match() >= match()` as required by definition.

**`test/testing_data.hdf5`** — Waveform and PSD fixture data for the regression test, stored in HDF5 format per PyCBC contributor recommendation.

---

## Development History

The fix involved 16 commits across a two-day development sprint and an extended review cycle with PyCBC contributors:

| Step | What happened |
|------|---------------|
| Bug discovery | Observed discontinuities in mismatch contour plots during lensing research |
| Root cause analysis | Identified that `bracket=` does not enforce bounds in `minimize_scalar` |
| Initial fix | Switched to `method='bounded'` with `bounds=(-delta_t, delta_t)` |
| CI feedback | Contributor [@jacopok](https://github.com/jacopok) identified `bracket` → `bounds` keyword mismatch |
| Precision issue | Bounded method initially failed 4-decimal accuracy test; iterated on `xatol` and convergence strategy |
| Test data | Refactored fixture from Python pickle → inline Python → HDF5 at contributor request |
| Rebase | Rebased onto `gwastro:master` (April 2026) to resolve unrelated CI failures |
| Final review | Approved by [@jacopok](https://github.com/jacopok); remaining CI failure confirmed unrelated to the fix |

---

## Status

The PR is **open and approved** by PyCBC contributors. The only remaining CI failure is an unrelated 404 on a virtual environment image mirror — not caused by any change in this PR.

---

## Context: The Research That Found This

This bug was found during a Summer 2025 REU at the University of Texas at Dallas, investigating how to distinguish **gravitational lensing** effects from **regular orbital precession** in binary black hole merger gravitational wave signals. The research used `optimized_match()` to quantify the similarity between lensed and unlensed waveforms across a parameter space of chirp masses and lensing time delays.

The anomalous mismatch spikes initially appeared to be a physical phenomenon. Realizing they were a numerical artifact of the optimizer — and then isolating, reproducing, and fixing the bug in a production codebase with an active test suite and contributor review process — was an unexpected but significant outcome of the project.

---

## About PyCBC

PyCBC is an open-source Python toolkit for gravitational wave astronomy used by LIGO, Virgo, and KAGRA researchers worldwide. It provides tools for matched filtering, parameter estimation, template bank generation, and detector characterization.

- Repository: [github.com/gwastro/pycbc](https://github.com/gwastro/pycbc)
- Documentation: [pycbc.org](http://pycbc.org)
