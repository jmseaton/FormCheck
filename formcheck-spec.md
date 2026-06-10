# FormCheck — Product & Engineering Spec

**Version:** 0.1 (Draft)
**Date:** 2026-06-10
**Owner:** Jared
**Status:** Draft for review

---

## 1. Problem Statement

Remote bodybuilding coaching operates on a weekly feedback loop. Exercise execution and posing videos are recorded during the week, reviewed at check-in, and feedback arrives days after the rep was performed. By then, the proprioceptive memory of the lift is gone — it's difficult to connect "what it felt like" to "what the coach saw," and equally difficult to verify that incorporated feedback actually changed the movement.

**Core insight:** the value isn't just faster feedback — it's feedback delivered while the kinesthetic memory is still fresh, plus an objective record that closes the loop between coach cue → attempted correction → verified change.

## 2. Goals

1. Record (or import) exercise execution video and receive structured form feedback within ~60 seconds of submission.
2. Same pipeline for bodybuilding posing — accepting video or photos.
3. Maintain a longitudinal library: track the same exercise/pose over time, compare sessions side-by-side, and verify whether a coaching cue was successfully incorporated.
4. Let the user attach subjective notes ("felt heavy off the chest," "lost lat tension at lockout") to pair the feel with the footage.

## 3. Non-Goals (v1)

- **Real-time feedback on live video.** Analysis is post-hoc on submitted media only.
- Coach-in-the-loop features (sharing analyses, coach annotations, calibrating feedback to coach's specific cues). **Deferred to Phase 2** — but v1 data model must not preclude it.
- Programming/periodization advice, load recommendations, or nutrition.
- Android / web clients.
- Multi-user or social features.

## 4. Users

- **v1:** Single user (owner). Advanced trainee, technically sophisticated, training in a home garage gym (fixed camera positions are feasible and should be exploited).
- **Phase 2:** Remote coach as a second persona (reviewer/annotator).

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

US-9 is the v1 bridge to coaching; it keeps the coach relationship intact without building Phase 2.

## 6. Functional Requirements

### 6.1 Capture & Import
- In-app camera: 1080p/60fps default (60fps materially improves pose tracking on fast concentrics), landscape and portrait supported, optional countdown timer and auto-stop after N seconds.
- Import from Photos library (video and stills).
- Exercise tagging at capture or import time: searchable exercise list seeded with the user's actual movements (belt squat, RDL, Nordic curl, Smith machine variants, pulley work, etc.) plus standard barbell/dumbbell lifts. Exercises carry metadata: movement pattern, camera angle guidance, key checkpoints.
- Posing mode: pose tagging from the standard mandatory poses (front double biceps, front lat spread, side chest, side triceps, rear double biceps, rear lat spread, abs & thighs, most muscular) plus free-form.
- Camera-angle guidance per exercise (e.g., "squat: 45° front-oblique or true side") shown as a ghost overlay; fixed garage-gym camera positions can be saved as presets.

### 6.2 Analysis Pipeline (see §7 for architecture)
- **Stage A — Pose extraction (on-device):** Vision framework body-pose detection on every frame (or every other frame at 60fps). Output: time series of joint positions, confidence scores.
- **Stage B — Kinematic computation (on-device):** rep segmentation, per-rep metrics: joint angles at key positions (e.g., hip/knee/ankle at squat bottom), ROM, eccentric/concentric tempo, bar/implement path approximation (wrist or tracked-point trajectory), left/right asymmetry, inter-rep consistency drift (fatigue signal).
- **Stage C — Qualitative feedback (cloud LLM vision):** selected keyframes (rep bottom, sticking point, lockout, plus any frames flagged anomalous by Stage B) + Stage B metrics + exercise definition + user's "feel" note → structured coaching feedback. Posing path: keyframes/photos + symmetry/alignment metrics → per-pose critique.
- Feedback output is structured: per-rep observations, set-level summary, 1–3 prioritized cues (never a wall of ten corrections), severity-ranked, with frame references so each observation is tappable → jumps to the frame.
- Confidence handling: if pose-tracking confidence is low (occlusion, bad angle, clothing), say so explicitly rather than confabulating feedback. Surface "re-shoot from X angle" guidance.

### 6.3 Feedback Presentation
- Video player with skeleton overlay toggle, metric overlays (live joint angle readout), rep markers on the scrubber.
- Feedback panel: prioritized cues with frame-linked evidence.
- Posing: annotated stills, symmetry guides (vertical center line, shoulder/hip level lines).

### 6.4 History & Comparison
- Library organized by exercise/pose, filterable by date.
- Side-by-side synced playback (sync on rep start, not wall-clock).
- Photo A-B overlay slider for posing.
- "Cue tracking": when feedback issues a cue, it persists as an open item on that exercise; subsequent analyses evaluate it explicitly and mark resolved/persisting. This is the close-the-loop mechanic and the app's differentiator.

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
- **Kinematics engine:** Swift package. Rep segmentation via vertical displacement of tracked landmark (e.g., hip y-coordinate for squat pattern, wrist for pulls) with smoothing + peak detection. Per-exercise checkpoint definitions stored as data (JSON), not code — adding an exercise should not require an app update.
- **LLM service:** Anthropic Messages API with vision. Per-exercise system prompts define what good execution looks like and the cue vocabulary. Request payload: 4–8 keyframes (JPEG, downscaled), metrics JSON, exercise definition, open cue list, user note. Response: structured JSON (observations[], cues[], confidence, frame_refs[]).
- **Backend:** v1 can be thin-to-none. Options: (a) direct API calls from device with key in Keychain — acceptable for personal tool; (b) lightweight proxy (small Cloudflare Worker / Vercel edge fn) if key protection or usage logging is wanted. Recommend (a) for v1, (b) before any TestFlight distribution.
- **Storage:** videos remain in app sandbox (or Photos with local identifiers); pose time series, metrics, analyses in SQLite/GRDB or SwiftData. CloudKit sync optional. Raw video never uploaded — only keyframes leave the device.

### 7.3 Posing-Specific Pipeline
- Photos: single-frame pose detection → symmetry metrics (shoulder/hip line tilt, limb angle L/R deltas, stance width relative to shoulder width) → LLM critique per tagged pose.
- Video: detect pose "hold" segments (low joint velocity windows), extract best frame per hold, then photo pipeline per hold.
- Honest limitation, stated in-app: automated analysis evaluates *pose mechanics* (symmetry, alignment, position correctness), not *physique presentation judgment* (conditioning, muscle maturity calls). The latter remains the coach's domain.

### 7.4 Latency Budget (target ≤60s for a 45s set)
- Pose extraction: ~1–2× realtime on A15+ → ≤45s, run during/immediately after recording where possible
- Kinematics: <2s
- Keyframe encode + LLM round trip: 10–20s
- Pipeline Stage A can begin streaming during capture to hide most of its cost.

## 8. Data Model (sketch)

```
Exercise(id, name, pattern, checkpointDef JSON, cameraGuidance)
Submission(id, type: lift|pose, exerciseId/poseId, mediaRef, capturedAt, userNote, audioNoteRef)
PoseTimeSeries(submissionId, frameIdx, joints JSON, confidence)
RepMetrics(submissionId, repIdx, metrics JSON)
Analysis(id, submissionId, model, observations JSON, cues JSON, confidence, createdAt)
Cue(id, exerciseId, text, sourceAnalysisId, status: open|resolved|persisting, resolvedBy)
```
Coach entities (Phase 2: `Coach`, `Annotation`, `SharedAnalysis`) deliberately absent but FK-compatible.

## 9. Privacy & Security
- All video and pose data on-device by default; only downscaled keyframes + numeric metrics sent to the LLM API.
- API traffic over TLS; key in Keychain.
- No third-party analytics v1.
- Health-adjacent note: no HR or health data integration in v1 (possible later via HealthKit; out of scope now).

## 10. Milestones

| Phase | Scope | Exit criteria |
|-------|-------|---------------|
| **M1 — Pipeline proof** | One exercise (pick the highest-frequency lift), import-only, CLI-ish debug UI. Vision → kinematics → LLM round trip. | Feedback on a real set that matches what the coach would flag, ≥70% of the time across 10 test sets |
| **M2 — Capture + library** | In-app recording, exercise tagging, history list, video player with overlay | Daily-driver usable for one lift |
| **M3 — Exercise breadth + posing** | Checkpoint defs for full movement list; posing photo pipeline | All weekly check-in movements covered |
| **M4 — Close the loop** | Cue tracking, side-by-side comparison, A-B posing overlay, export package for coach | A cue can be issued, worked, and verified resolved entirely in-app |
| **M5 (Phase 2)** | Coach accounts, annotation, cue-vocabulary calibration | — |

## 11. Success Metrics
- Time-to-feedback ≤60s p90.
- Agreement rate: % of app-issued cues the coach independently confirms at weekly check-in (target ≥70% by M3 — this is the trust metric).
- False-positive rate: cues the coach explicitly contradicts (<10%).
- Loop closure: median sessions from cue issued → marked resolved.

## 12. Risks & Open Questions

| # | Risk / Question | Notes |
|---|-----------------|-------|
| R1 | Vision body-pose accuracy under barbell occlusion, low garage lighting, or baggy clothing | Mitigate: camera presets, lighting guidance, confidence gating; evaluate 3D pose request vs 2D |
| R2 | LLM feedback quality variance across exercises | Per-exercise prompt + checkpoint defs; M1 agreement-rate gate before scaling breadth |
| R3 | Rep segmentation on non-standard movements (Nordic curls, belt squat) | Per-exercise tracked-landmark config; allow manual rep marking fallback |
| R4 | Posing critique overreach | Hard-scope to mechanics; disclaim physique judgment |
| Q1 | Does coach feedback history exist in exportable form (texts, check-in docs)? If so, it can seed per-exercise prompts now without Phase 2 | TBD |
| Q2 | 2D vs 3D pose request: 3D improves sagittal angles but device/perf constraints apply | Benchmark in M1 |
| Q3 | iOS minimum version / device floor (affects 3D pose availability) | Propose iOS 17+, A15+ |
