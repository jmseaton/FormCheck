# CLAUDE.md — FormCheck project context

iOS app (SwiftUI) for asynchronous form feedback on bodybuilding exercise execution and posing. **`formcheck-spec.md` is the source of truth** — when implementation questions arise, check the spec first and cite the section in commit messages where relevant. If code needs to diverge from the spec, update the spec in the same PR/commit, never silently.

## Architecture (spec §7)

Hybrid pipeline, three stages:
- **Stage A** — on-device pose extraction (Apple Vision framework)
- **Stage B** — on-device kinematics (Swift package; rep segmentation, joint angles, tempo, symmetry)
- **Stage C** — cloud LLM vision (Anthropic Messages API) for qualitative coaching feedback, constrained by Stage B metrics

The Stage B kinematics package is consumed by BOTH the iOS app and the dev eval harness. Never fork the metric logic — device and harness must produce identical numbers from the same video.

## Non-negotiable invariants

1. **Two-layer reference model (spec §7.5).** Layer 1 (generic baseline checkpoint defs) is never modified based on one user's data. All personalization goes in Layer 2 user overlays. If a change to Layer 1 is proposed, it requires literature justification, not corpus fitting.
2. **No silent learning.** The app never self-adjusts thresholds or prompts at runtime. Calibration is an offline, human-reviewed, eval-gated workflow (spec §7.5.4).
3. **Eval gate on every calibration-relevant change.** Any change to checkpoint defs, overlays, or Stage C prompts must re-run the eval harness against the corpus; agreement rate must not regress (targets: ≥70% agreement, <10% false positives — spec §11).
4. **Privacy boundary (spec §9).** Raw video never leaves the device. Only downscaled keyframes + numeric metrics go to the LLM API. Do not widen this in any change.
5. **`coach-history/` is never committed.** It contains personal video and verbatim coach communications. It is gitignored; do not add exceptions, do not copy its contents elsewhere in the repo, do not quote corpus feedback verbatim in code, tests, or fixtures.
6. **Checkpoint definitions are data, not code.** Exercise defs live as JSON; adding an exercise must not require app code changes. Thresholds that vary with leverages are functions of anthropometric ratios, not constants (spec §7.5.1).
7. **Structured LLM output.** Stage C responses are schema-validated JSON (observations[], cues[], confidence, frame_refs[]). Feedback surfaces at most 1–3 prioritized cues per set. Low pose-tracking confidence → say so, never confabulate feedback.

## Repo layout (current & planned)

- `formcheck-spec.md` — the spec; versioned via git history
- `coach-history/` — (gitignored, local only) calibration corpus per spec §7.5.4: per-exercise dirs of (video + verbatim feedback) pairs + `manifest.csv` (`date, exercise, video, feedback, outcome, resolves`; outcome vocabulary `open|y|partial|n|regressed|n/a` — see spec §7.5.4 for semantics; `exercise` values must match directory names exactly, as they join to ExerciseDefs later)
- planned: `Kinematics/` (Swift package), `Harness/` (eval CLI), `App/` (iOS app), `ExerciseDefs/` (Layer 1 JSON)

## Current status / next milestone

Spec v0.2. Active milestone: **M0 — corpus assembly** (spec §10). M1 (pipeline proof, single exercise, dev harness) is blocked on M0 reaching 5–10 labeled pairs for the target lift, including negative ("looked good") examples.

## Conventions

- Swift / SwiftUI, iOS 17+ target (proposed; open question Q3), GRDB or SwiftData for persistence
- Commit messages reference spec sections when implementing or changing specified behavior
- Tests for kinematics use synthetic pose time series as fixtures — never corpus data
