# FormCheck — Product & Engineering Spec

**Version:** 0.2 (Draft)
**Date:** 2026-06-10
**Owner:** Jared
**Status:** Draft for review
**Changelog:** 0.2 — added reference provenance & two-layer calibration model (§7.5), coach feedback attachment (§6.4), user overlay + anthropometric parameterization in data model (§8), calibration corpus workflow in milestones (§10), resolved Q1.

---

## 1. Problem Statement

Remote bodybuilding coaching operates on a weekly feedback loop. Exercise execution and posing videos are recorded during the week, reviewed at check-in, and feedback arrives days after the rep was performed. By then, the proprioceptive memory of the lift is gone — it's difficult to connect "what it felt like" to "what the coach saw," and equally difficult to verify that incorporated feedback actually changed the movement.

**Core insight:** the value isn't just faster feedback — it's feedback delivered while the kinesthetic memory is still fresh, plus an objective record that closes the loop between coach cue → attempted correction → verified change.

## 2. Goals

1. Record (or import) exercise execution video and receive structured form feedback within ~60 seconds of submission.
2. Same pipeline for bodybuilding posing — accepting video or photos.
3. Maintain a longitudinal library: track the same exercise/pose over time, compare sessions side-by-side, and verify whether a coaching cue was successfully incorporated.
4. Let the user attach subjective notes ("felt heavy off the chest," "lost lat tension at lockout") to pair the feel with the footage.
5. Capture ongoing coach feedback as labeled ground truth, enabling deliberate, testable calibration over time.

## 3. Non-Goals (v1)

- **Real-time feedback on live video.** Analysis is post-hoc on submitted media only.
- Coach-in-the-loop features (coach accounts, direct annotation, in-app sharing). **Deferred to Phase 2** — but v1 data model must not preclude it.
- Multi-user **product surface** (onboarding, accounts, distribution). Single-user is v1 scope, but the data model and reference architecture must not assume a single user — personalization lives in a per-user overlay, never in the shipped baseline (§7.5).
- Automatic/silent learning: the app never self-adjusts thresholds from feedback. All calibration changes are human-reviewed and eval-gated.
- Programming/periodization advice, load recommendations, or nutrition.
- Android / web clients.

## 4. Users

- **v1:** Single user (owner). Advanced trainee, technically sophisticated, training in a home garage gym (fixed camera positions are feasible and should be exploited).
- **Design constraint:** a hypothetical second user with no calibration history must receive correct (if more generic) feedback from the shipped baseline alone.
- **Phase 2:** Remote coach as a second persona (reviewer/annotator); additional lifters.

## 5. User Stories

| ID | Story | Priority |
|----|-------|----------|
| US-1 | As a lifter, I record a set in-app (or import from Photos) and tag it with an exercise, so analysis knows what to evaluate. | P0 |
| US-2 | As a lifter, I receive per-rep and per-set form feedback within ~60s of submission. | P0 |
| US-3 | As a lifter, I see quantitative metrics (joint angles, ROM, tempo, bar path where applicable) overlaid on my video. | P0 |
| US-4 | As a lifter, I attach a free-text or voice "how it felt" note to any set. | P1 |
| US-5 | As a lifter, I view two sessions of the same exercise side-by-side with synced playback to verify a correction. | P1 |
| US-6 | As a competitor, I submit posing video or photos and receive feedback per pose (front double bi, lat spread, side chest, etc.). | P0 |
| US-7 | As a competitor, I compare the same pose across dates (photo overlay / A-B slider). | P1 |
| US-8 | As a lifter, I see a trend view: has my squat depth / knee valgus / lockout timing improved over 6 weeks? | P2 |
| US-9 | As a lifter, I export an analysis (video + annotations + metrics) to send to my coach manually. | P1 |
| US-10 | As a lifter, after a check-in I attach my coach's feedback to the submission she reviewed, so her judgment becomes ground truth for cue tracking and future calibration. | P1 |
| US-11 | As a lifter, I can mark an analysis as accurate or off-base, so false-positive behavior can be tuned even without coach input. | P2 |

US-9 + US-10 are the v1 bridge to coaching; they keep the coach relationship intact and accumulate labeled data without building Phase 2.

## 6. Functional Requirements

### 6.1 Capture & Import
- In-app camera: 1080p/60fps default (60fps materially improves pose tracking on fast concentrics), landscape and portrait supported, optional countdown timer and auto-stop after N seconds.
- Import from Photos library (video and stills).
- Exercise tagging at capture or import time: searchable exercise list seeded with the user's actual movements (belt squat, RDL, Nordic curl, Smith machine variants, pulley work, etc.) plus standard barbell/dumbbell lifts. Exercises carry metadata: movement pattern, camera angle guidance, key checkpoints.
- Posing mode: pose tagging from the standard mandatory poses (front double biceps, front lat spread, side chest, side triceps, rear double biceps, rear lat spread, abs & thighs, most muscular) plus free-form.
- Camera-angle guidance per exercise (e.g., "squat: 45° front-oblique or true side") shown as a ghost overlay; fixed garage-gym camera positions can be saved as presets.

### 6.2 Analysis Pipeline (see §7 for architecture)
- **Stage A — Pose extraction (on-device):** Vision framework body-pose detection on every frame (or every other frame at 60fps). Output: time series of joint positions, confidence scores.
- **Stage B — Kinematic computation (on-device):** rep segmentation, per-rep metrics: joint angles at key positions (e.g., hip/knee/ankle at squat bottom), ROM, eccentric/concentric tempo, bar/implement path approximation (wrist or tracked-point trajectory), left/right asymmetry, inter-rep consistency drift (fatigue signal). Checkpoint resolution = generic baseline def + user overlay (§7.5), with anthropometric parameters computed from the user's own pose data.
- **Stage C — Qualitative feedback (cloud LLM vision):** selected keyframes (rep bottom, sticking point, lockout, plus any frames flagged anomalous by Stage B) + Stage B metrics + resolved exercise definition + open cue list + user's "feel" note → structured coaching feedback. Posing path: keyframes/photos + symmetry/alignment metrics → per-pose critique. Prompt context includes the user's calibration overlay (cue vocabulary, prioritization) when present.
- Feedback output is structured: per-rep observations, set-level summary, 1–3 prioritized cues (never a wall of ten corrections), severity-ranked, with frame references so each observation is tappable → jumps to the frame.
- Confidence handling: if pose-tracking confidence is low (occlusion, bad angle, clothing), say so explicitly rather than confabulating feedback. Surface "re-shoot from X angle" guidance.
- Calibration-state framing: when the active user overlay is empty or thin for an exercise, feedback is explicitly labeled as based on general standards ("feedback sharpens as coach input is logged") rather than presented with false precision.

### 6.3 Feedback Presentation
- Video player with skeleton overlay toggle, metric overlays (live joint angle readout), rep markers on the scrubber.
- Feedback panel: prioritized cues with frame-linked evidence.
- Posing: annotated stills, symmetry guides (vertical center line, shoulder/hip level lines).

### 6.4 History, Comparison & Coach Feedback Capture
- Library organized by exercise/pose, filterable by date.
- Side-by-side synced playback (sync on rep start, not wall-clock).
- Photo A-B overlay slider for posing.
- **Cue tracking:** when feedback issues a cue, it persists as an open item on that exercise; subsequent analyses evaluate it explicitly and mark resolved/persisting. This is the close-the-loop mechanic and the app's differentiator.
- **Coach feedback attachment (US-10):** after a check-in, the user attaches the coach's feedback (text; voice memo transcribed on-device) to the specific submission reviewed. Verbatim capture is encouraged over paraphrase — it preserves cue vocabulary. Attached feedback:
  1. Becomes a labeled example, exportable to the calibration corpus (§7.5.4).
  2. Can open or resolve cues with coach authority (distinct from app-inferred status).
  3. Flags disagreements: if the app's analysis and the coach's feedback conflict (app said clean, coach flagged a fault, or vice versa), the pair is marked as a calibration discrepancy for review.
- **Accuracy marking (US-11):** any analysis can be marked accurate / off-base by the user; weaker signal than coach feedback but tunes false-positive behavior for users without a coach.
- This feature is also the structural bridge to Phase 2: coach accounts replace "user transcribes coach feedback" with "coach annotates directly" against the same entities.

### 6.5 Notes
- Free text or voice memo (transcribed on-device) attached to any submission; included in Stage C context.

## 7. Architecture

### 7.1 Recommendation: Hybrid (pose estimation + LLM reasoning)

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| LLM vision only | Coaching-quality language, flexible | Expensive on video; frame sampling loses tempo/path data; can't measure angles reliably; hallucination risk | Reject as sole engine |
| Pose estimation only | Free, on-device, private, frame-accurate, quantitative | Output is numbers, not coaching; rule-based feedback is brittle across exercise variety | Reject as sole engine |
| **Hybrid** | Quantitative ground truth constrains the LLM; LLM provides interpretation and cue language; cheap (only keyframes + JSON metrics go to cloud) | Two systems to maintain | **Recommended** |

### 7.2 Components
- **Client:** SwiftUI app. AVFoundation capture. Vision framework (`VNDetectHumanBodyPoseRequest`; evaluate `VNDetectHumanBodyPose3DRequest` on supported hardware for sagittal-plane angle accuracy). Core Motion not required v1.
- **Kinematics engine:** Swift package. Rep segmentation via vertical displacement of tracked landmark (e.g., hip y-coordinate for squat pattern, wrist for pulls) with smoothing + peak detection. Per-exercise checkpoint definitions stored as data (JSON), not code — adding an exercise should not require an app update. The same package is consumed by the dev-side calibration harness (§7.5.4) so device and harness produce identical metrics.
- **LLM service:** Anthropic Messages API with vision. Per-exercise system prompts define what good execution looks like and the cue vocabulary; user overlay content (coach vocabulary, priorities) is injected when present. Request payload: 4–8 keyframes (JPEG, downscaled), metrics JSON, resolved exercise definition, open cue list, user note. Response: structured JSON (observations[], cues[], confidence, frame_refs[]).
- **Backend:** v1 can be thin-to-none. Options: (a) direct API calls from device with key in Keychain — acceptable for personal tool; (b) lightweight proxy (small Cloudflare Worker / Vercel edge fn) if key protection or usage logging is wanted. Recommend (a) for v1, (b) before any TestFlight distribution.
- **Storage:** videos remain in app sandbox (or Photos with local identifiers); pose time series, metrics, analyses, overlays in SQLite/GRDB or SwiftData. CloudKit sync optional. Raw video never uploaded — only keyframes leave the device.

### 7.3 Posing-Specific Pipeline
- Photos: single-frame pose detection → symmetry metrics (shoulder/hip line tilt, limb angle L/R deltas, stance width relative to shoulder width) → LLM critique per tagged pose.
- Video: detect pose "hold" segments (low joint velocity windows), extract best frame per hold, then photo pipeline per hold.
- Honest limitation, stated in-app: automated analysis evaluates *pose mechanics* (symmetry, alignment, position correctness), not *physique presentation judgment* (conditioning, muscle maturity calls). The latter remains the coach's domain.

### 7.4 Latency Budget (target ≤60s for a 45s set)
- Pose extraction: ~1–2× realtime on A15+ → ≤45s, run during/immediately after recording where possible
- Kinematics: <2s
- Keyframe encode + LLM round trip: 10–20s
- Pipeline Stage A can begin streaming during capture to hide most of its cost.

### 7.5 Reference Provenance & Calibration Model

Form standards are authored content, not a given. The reference architecture is two-layered so that personalization never contaminates the shipped baseline.

#### 7.5.1 Layer 1 — Generic baseline (ships with the app)
- Literature-derived checkpoint definitions (NSCA, peer-reviewed biomechanics, published technique standards) with deliberately **wide** tolerances: population-level "defensibly correct form" any competent coach would endorse.
- Posing baseline derives from published NPC/IFBB judging criteria for the mandatory poses, scoped to mechanics (symmetry, alignment, position) only.
- **Anthropometric parameterization:** thresholds that vary with leverages (torso angle at squat depth, press ROM, hinge geometry) are expressed as functions of segment-length ratios, not constants. Segment ratios are measured automatically from the user's own pose data — this converts a large share of "personal calibration" into math every user receives on day one with zero history.
- Layer 1 is never modified by any individual user's calibration data. Changes to Layer 1 require their own multi-source eval (§7.5.5) to prevent drift toward any one user's standard.

#### 7.5.2 Layer 2 — Per-user calibration overlay
- A set of deltas applied over Layer 1 at analysis time: narrowed/shifted thresholds, coach cue vocabulary, fault prioritization weights, exercise-variant preferences.
- Resolution: `effective_def = layer1_def(anthropometrics) + user_overlay`.
- A user with an empty overlay gets pure Layer 1: fully functional, looser tolerances, generic coaching language, explicitly framed as such (§6.2).
- Overlay provenance is tracked: every delta references the labeled examples that justified it.

#### 7.5.3 Sources of overlay data
1. **Historical coach corpus (bulk, one-time, dev-side):** the owner's existing archive of (video + verbatim coach feedback) pairs. Used offline for initial threshold tuning and as the M1 eval set. Not an app feature in v1; potentially a future onboarding import flow.
2. **Ongoing coach feedback attachment (§6.4):** weekly check-ins continuously produce new labeled pairs through normal use. This is the universal calibration path for any future user with a coach.
3. **Accuracy marking (§6.4):** coachless signal; tunes false-positive behavior only.

#### 7.5.4 Calibration corpus & workflow
- **Corpus structure (repo, not app):**
```
/coach-history/
  /incline_db_press/
    2026-03-14.mov
    2026-03-14_feedback.txt      # verbatim coach comments
  /rdl/ ...
  manifest.csv  # date, exercise, video, feedback, outcome (cue later resolved? y/n/partial)
```
- The `outcome` column identifies before/after pairs for the same fault — the validation set for the cue-tracking mechanic itself.
- Target composition: 5–10 pairs per major lift, deliberately mixing flagged sets and "looked good" sets (negative examples gate the false-positive metric). Posing photo + critique pairs included on the same pattern.
- **Dev harness:** CLI (or notebook) wrapping the same Swift kinematics package as the device. Runs corpus videos through Stages A–B, compares metrics against coach labels, reports agreement/false-positive rates per exercise.
- **Calibration loop (deliberate, human-in-the-middle):** new labeled pairs accumulate in-app → periodically exported to corpus → thresholds/prompts re-tuned offline → full eval re-run → updated overlay (or Layer 1, with its own gate) shipped. The app never self-adjusts; every change is testable and reversible.

#### 7.5.5 Eval discipline
- The owner's corpus is the regression suite for **Layer 1 + owner overlay** combined.
- Layer 1 changes additionally require evaluation that does not rely solely on one user's data (literature cross-check at minimum; multi-user data when it exists), so the generic baseline is never accidentally tuned toward a single lifter's leverages or a single coach's preferences.
- Every prompt or threshold change re-runs the harness; agreement rate must not regress.

## 8. Data Model (sketch)

```
Exercise(id, name, pattern, baselineDef JSON,        -- Layer 1: checkpoints as
         cameraGuidance)                              --   functions of anthro params
UserProfile(id, anthropometrics JSON)                 -- segment ratios, auto-measured
UserOverlay(id, userId, exerciseId,                   -- Layer 2
            thresholdDeltas JSON, cueVocabulary JSON,
            priorityWeights JSON, provenanceRefs JSON)
Submission(id, userId, type: lift|pose, exerciseId/poseId,
           mediaRef, capturedAt, userNote, audioNoteRef)
PoseTimeSeries(submissionId, frameIdx, joints JSON, confidence)
RepMetrics(submissionId, repIdx, metrics JSON)
Analysis(id, submissionId, model, resolvedDefSnapshot JSON,
         observations JSON, cues JSON, confidence, createdAt)
CoachFeedback(id, submissionId, verbatimText, audioRef,
              attachedAt, discrepancyFlag bool,        -- conflicts w/ Analysis
              exportedToCorpus bool)
AccuracyMark(id, analysisId, verdict: accurate|off_base, note)
Cue(id, userId, exerciseId, text, sourceType: app|coach,
    sourceRef, status: open|resolved|persisting,
    resolvedBy: app|coach)
```

Notes:
- `userId` present throughout — single-user v1, but the model doesn't assume it.
- `resolvedDefSnapshot` on Analysis records exactly which effective definition produced the feedback (auditability across calibration rounds).
- Phase 2 entities (`Coach`, `Annotation`, `SharedAnalysis`) remain absent but FK-compatible; `CoachFeedback` is the v1 stand-in that Phase 2 upgrades in place.

## 9. Privacy & Security
- All video and pose data on-device by default; only downscaled keyframes + numeric metrics sent to the LLM API.
- The calibration corpus contains personal video and coach communications; it lives in private storage (local/private repo), never in the shipped app bundle. Only derived deltas (numbers, vocabulary) ship in the overlay.
- API traffic over TLS; key in Keychain.
- No third-party analytics v1.
- Health-adjacent note: no HR or health data integration in v1 (possible later via HealthKit; out of scope now).

## 10. Milestones

| Phase | Scope | Exit criteria |
|-------|-------|---------------|
| **M0 — Corpus assembly** | Collect & structure historical (video + verbatim feedback) pairs per §7.5.4; 5–10 pairs for the 3–4 highest-frequency lifts incl. negative examples; build manifest with outcome labels | Corpus complete enough to eval one lift end-to-end |
| **M1 — Pipeline proof** | One exercise (highest-frequency lift), import-only, dev harness UI. Vision → kinematics → LLM round trip. Tune Layer 1 + owner overlay against M0 corpus; benchmark 2D vs 3D pose | ≥70% agreement with coach labels across corpus; <10% false positives on "looked good" sets |
| **M2 — Capture + library** | In-app recording, exercise tagging, history list, video player with overlay, coach feedback attachment (US-10) | Daily-driver usable for one lift; first post-check-in feedback attached in-app |
| **M3 — Exercise breadth + posing** | Checkpoint defs for full movement list; anthropometric parameterization live; posing photo pipeline | All weekly check-in movements covered |
| **M4 — Close the loop** | Cue tracking incl. coach-authority resolution, discrepancy flagging, side-by-side comparison, A-B posing overlay, export package, corpus export of in-app labeled pairs | A cue can be issued, worked, and verified resolved entirely in-app; first deliberate recalibration round completed via §7.5.4 loop |
| **M5 (Phase 2)** | Coach accounts, direct annotation (upgrading CoachFeedback), cue-vocabulary calibration UI, onboarding corpus import for new users | — |

## 11. Success Metrics
- Time-to-feedback ≤60s p90.
- Agreement rate: % of app-issued cues the coach independently confirms at weekly check-in (target ≥70% by M3 — this is the trust metric).
- False-positive rate: cues the coach explicitly contradicts (<10%).
- Loop closure: median sessions from cue issued → marked resolved.
- Calibration health: discrepancy flags (§6.4) trending down across recalibration rounds.

## 12. Risks & Open Questions

| # | Risk / Question | Notes |
|---|-----------------|-------|
| R1 | Vision body-pose accuracy under barbell occlusion, low garage lighting, or baggy clothing | Mitigate: camera presets, lighting guidance, confidence gating; evaluate 3D pose request vs 2D |
| R2 | LLM feedback quality variance across exercises | Per-exercise prompt + checkpoint defs; M1 agreement-rate gate before scaling breadth |
| R3 | Rep segmentation on non-standard movements (Nordic curls, belt squat) | Per-exercise tracked-landmark config; allow manual rep marking fallback |
| R4 | Posing critique overreach | Hard-scope to mechanics; disclaim physique judgment |
| R5 | Baseline contamination: personal calibration leaking into Layer 1 | Two-layer model + separate eval gates (§7.5.5); Layer 1 changes require multi-source evidence |
| R6 | Overlay overfit: coach corpus tunes the owner's overlay so tightly that legitimate technique variation gets flagged | Keep overlay deltas provenance-linked; discrepancy flags surface overcorrection |
| ~~Q1~~ | ~~Does coach feedback history exist in exportable form?~~ **Resolved: yes.** Historical (video + feedback) pairs exist → M0 corpus assembly added | Verbatim transcripts preferred over paraphrase |
| Q2 | 2D vs 3D pose request: 3D improves sagittal angles but device/perf constraints apply | Benchmark in M1 |
| Q3 | iOS minimum version / device floor (affects 3D pose availability) | Propose iOS 17+, A15+ |
| Q4 | Anthropometric parameterization functions: which thresholds vary with leverage, and what's the functional form? | Derive candidates from biomechanics literature in M1; validate against owner corpus (tall-lifter data point) |
