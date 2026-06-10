# FormAnalysis

Spec and design docs for **FormCheck** — an iOS app providing asynchronous form feedback on bodybuilding exercise execution and posing, replacing a week-long coach feedback loop with ~60-second turnaround while preserving (and learning from) the coaching relationship.

## Contents

- [`formcheck-spec.md`](formcheck-spec.md) — product & engineering spec (v0.2)

## Core design decisions

- **Hybrid analysis engine:** on-device pose estimation (Apple Vision) for quantitative kinematics, constraining a cloud LLM vision pass for qualitative coaching feedback. Raw video never leaves the device — only downscaled keyframes and metrics.
- **Cue tracking as the differentiator:** coaching cues persist as open items; subsequent sets are evaluated against them and marked resolved/persisting, closing the loop between feedback and verified change.
- **Two-layer reference model:** a literature-derived generic baseline (Layer 1, anthropometrically parameterized) plus a per-user calibration overlay (Layer 2) mined from coach feedback — personalization never contaminates the shipped baseline.
- **Deliberate calibration:** the app never self-adjusts. Labeled (video + coach feedback) pairs accumulate through use, and recalibration is an offline, eval-gated workflow.

## Repo conventions

- `coach-history/` (gitignored) holds the calibration corpus: per-exercise (video + verbatim coach feedback) pairs and a manifest with outcome labels. Structure defined in spec §7.5.4. **Contains personal media — never committed.**
- Spec versions are tracked through commit history; see `git log -p formcheck-spec.md` for the evolution of design decisions.

## Status

Spec draft v0.2. Next milestone: **M0 — corpus assembly** (spec §10).
