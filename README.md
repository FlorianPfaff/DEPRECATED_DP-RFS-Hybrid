# DP-RFS Hybrid (Discontinued)

## Discontinued: 2026-11-11

**This project is deprecated. We discontinued the DP-specific paper direction
after the frozen coherent-model study failed its predefined tracking criteria.**
The code, tests, raw results, and historical experiments remain available for
reproducibility and reuse. No further tuning or full-PMBM integration is planned.

### What we learned

- Prior-free track-history likelihoods, uncertain birth times, and analytically
  integrated Gaussian centers provide a defensible conditional model. The
  replacement passed all 53 tests and the exact-reference particle-accuracy
  checks. Filtered posteriors must not simply be reused as independent raw
  observations: they already contain the tracking prior.
- Correct inference and better density prediction do not guarantee better
  tracking. On held-out seeds 200-299, DP's primary RMS GOSPA was **5.0006 versus
  4.7626 for the development-selected KDE baseline**, or **5.00% worse**
  (paired 95% interval: **3.78% to 6.24% worse**). The protocol required at least
  a 3% improvement. Nonrecurring-birth GOSPA was **56.95% worse**, also failing
  the 5% degradation guardrail.
- Replay predictive log loss improved by **0.7313 nats** (95% interval:
  **0.6844 to 0.7784**), but a finite Bayesian mixture nearly matched that gain.
  This does not establish a uniquely DP-specific tracking contribution.
- Confirmation does not eliminate selection bias, clutter histories, or
  fragmentation. Shrinking DP residual mass combined with birth admission is
  a plausible explanation for missed novel regions, not a proven causal result.

These results justify stopping this project, not a universal claim that DP
birth learning cannot work. The exact reference, likelihood construction,
tests, and negative results remain useful research infrastructure; a separate
paper contribution from them has not been established.

See the [frozen results and provenance](results/coherent_birth_20261003/README.md)
and the [full decision memo with uncertainty, failure examples, and limitations](https://github.com/FlorianPfaff/DEPRECATED_2026-07-DP-RFS-Hybrid-Paper/blob/main/notes/coherent-birth-20261003/decision.md).

## Historical Documentation

The material below describes the earlier prototypes and is retained for
reproduction. Its positive historical comparisons do not supersede the final
controlled study or justify submission-ready claims.

Reference sandbox for Dirichlet-process layers inside RFS-style multitarget trackers.

The first prototype keeps the RFS side intentionally modest: an independent labeled multi-Bernoulli-style tracker with greedy association. The Bayesian nonparametric side now contains two deliberately separated models:

- a DP birth model for reusable birth regions inferred from unexplained measurements;
- clutter density models for structured measurement-space clutter, including a fixed Gaussian-mixture baseline and an adaptive DP clutter model learned from soft clutter responsibilities.

This repository was the implementation sandbox for the companion paper notes in
[`FlorianPfaff/DEPRECATED_2026-07-DP-RFS-Hybrid-Paper`](https://github.com/FlorianPfaff/DEPRECATED_2026-07-DP-RFS-Hybrid-Paper).
Upstreaming to PyRecEst was an earlier goal, not an active commitment.

## Design principle

The RFS layer estimates physical targets: existence, labels, survival, death, missed detections, measurement-to-track association, and trajectory continuity.

The DP layer estimates unknown distributions used by the RFS layer. DP atoms are not targets. In this repository:

```text
DP birth atoms   = recurring birth regions
DP clutter atoms = recurring clutter regimes
```

For birth and clutter, the implementation keeps normalized densities separate from RFS intensity/rate scales:

```text
D_B(x)   = lambda_B * b(x)
kappa(z) = lambda_C * c(z)
```

`DirichletProcessBirthModel` estimates a normalized posterior-predictive birth density and stores the birth mass as `birth_rate`. `DirichletProcessClutterModel` estimates `c(z)` and stores `lambda_C` as `rate`. `FixedGaussianMixtureClutterModel` implements the same density/intensity interface without learning, so it can be used as a hand-specified baseline.

The tracker can optionally consume a clutter model: association odds, Bernoulli existence updates, and birth decisions use `clutter_model.intensity(z)` instead of a fixed scalar clutter intensity. The tracker then feeds fractional clutter responsibilities back into adaptive clutter models. Responsibility learning can be attenuated or gated via `clutter_responsibility_learning_rate` and `min_clutter_responsibility_to_learn`.

Birth learning can also run on two time scales. With `delayed_birth_learning=True`, an accepted birth measurement still spawns a tentative Bernoulli track immediately, but the DP birth atoms are updated only after that track reaches `birth_confirmation_age` and `birth_confirmation_existence`.

Confirmed Gaussian states are scored against both occupied atoms and the DP
residual branch. A nearby occupied winner is updated by count-weighted moment
matching; a residual winner creates a new atom. This keeps repeated confirmed
births from producing an append-only list of components.

## Install

```bash
python -m pip install -e .
```

For tests:

```bash
python -m pip install -e ".[test]"
pytest
```

The plotting script expects `matplotlib` in the active environment.

## Quick Demos

DP birth learning:

```bash
python examples/run_synthetic_birth_demo.py
```

Structured DP clutter learning:

```bash
python examples/run_structured_clutter_demo.py
```

The birth demo simulates two recurring birth regions, clutter, and a compact tracker that learns birth atoms online. The clutter demo compares fixed scalar clutter against an adaptive DP clutter model around a persistent measurement-space hotspot.

## Metrics and plots

Generate a multi-seed CSV for the structured-clutter comparison:

```bash
python experiments/structured_clutter_metrics.py --seeds 20 --scans 20 --output results/structured_clutter_metrics.csv
```

The metrics script compares:

```text
fixed_scalar_clutter
fixed_gmm_clutter
adaptive_dp_clutter
```

Generate PDF plots from the CSV:

```bash
python experiments/plot_structured_clutter_metrics.py --input results/structured_clutter_metrics.csv --output-dir results/figures
```

The most paper-relevant first plot is `structured_clutter_hotspot_track_steps.pdf`, which summarizes false-track pressure near the persistent clutter hotspot.

Run the recurring-birth benchmark used for delayed-learning ablations:

```bash
python experiments/recurring_birth_metrics.py \
  --seed-start 100 --seeds 100 --scans 96 \
  --output results/recurring_birth_metrics_seed100_199.csv
```

This paired benchmark compares a fixed broad birth model, measurement-driven
birth (MDB), immediate DP learning, delayed learning without reclustering,
delayed DP learning with reclustering, and an oracle birth mixture. It reports
GOSPA with localization/missed/false decomposition and birth-region recovery
diagnostics. Seeds 0--99 are the development campaign that fixed the method and
selected the DP concentration. Headline comparisons use the fresh, locked seeds
100--199. See `docs/recurring_birth_benchmark.md` for the recurrence ablation
and official PMBM reference workflow.

On the locked campaign, delayed DP learning with confirmed-state reclustering
scores `4.493 +/- 0.068 m` RMS GOSPA, compared with `4.841 +/- 0.050 m` for MDB
and `5.009 +/- 0.051 m` for fixed broad birth. Raw locked CSVs, paired analyses,
run provenance, and paper-ready figures are under `results/`.

## Reusable Experiment API

The structured-clutter demo is backed by a reusable experiment helper:

```python
from dp_rfs_hybrid import run_structured_clutter_experiment

result = run_structured_clutter_experiment(scans=20, seed=11)
rows = result.as_rows()
```

The result reports cumulative births, final active tracks, active clutter atoms, and per-scan records that can be exported to paper tables or figures.

## Package Layout

```text
src/dp_rfs_hybrid/
  gaussian.py      # linear-Gaussian prediction, update, likelihood
  dp_birth.py      # truncated DP birth atom model
  dp_clutter.py    # fixed GMM and DP clutter density models
  lmb_tracker.py   # small RFS-style multi-Bernoulli tracker
  experiments.py   # reusable structured-clutter experiment harness
examples/
  run_synthetic_birth_demo.py
  run_structured_clutter_demo.py
experiments/
  structured_clutter_metrics.py
  plot_structured_clutter_metrics.py
tests/
```

## Current Limitations

- Greedy nearest-neighbor association, not GLMB/PMBM hypothesis management.
- Finite active DP atoms, not full posterior sampling of the random measure.
- Gaussian birth/clutter atoms and linear-Gaussian tracking only.
- Heuristic birth-vs-clutter odds threshold.
- DP clutter feedback is still approximate and single-scan; it does not yet use smoothing or a full multi-hypothesis responsibility calculation.
- The experiment tracker still uses greedy association; PMBM is evaluated as a
  separate reference implementation rather than being integrated as the main
  backend.

Those restrictions are intentional for the first artifact: the goal is to isolate whether adaptive nonparametric nuisance structure helps before adding a larger RFS backend.
