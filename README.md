# HireSense AI

**Multimodal interview coach: it measures how you actually speak and look, then makes an LLM explain it to you.**

A browser-based mock-interview trainer that runs a live webcam + microphone pipeline during every answer, converts video and audio into numeric signals on-device, and feeds those signals into an LLM that returns scored, structured feedback. Progress is stored per session and charted over time.

- **Live app:** https://hiresensee.lovable.app
- **Stack:** React 18 + Vite 5 + TypeScript, Tailwind v3 + shadcn/Radix, Supabase (auth/DB/edge functions), Lovable AI Gateway (Gemini)
- **State:** working MVP. Core loop is real. Several things are heuristics, not ML — see [Honest Assessment](#honest-assessment-what-is-real-and-what-is-not).

---

## Table of contents

1. [The Problem](#the-problem)
2. [Why the Obvious Approaches Fail](#why-the-obvious-approaches-fail)
3. [The Aha Moment](#the-aha-moment)
4. [The Solution](#the-solution)
5. [Architecture](#architecture)
6. [How Each Signal Is Actually Computed](#how-each-signal-is-actually-computed)
7. [Tech Stack (and why each piece)](#tech-stack)
8. [Data Model](#data-model)
9. [What Broke During the Build](#what-broke-during-the-build)
10. [Honest Assessment](#honest-assessment-what-is-real-and-what-is-not)
11. [Known Limitations](#known-limitations)
12. [Not Built](#not-built-on-purpose-or-by-omission)
13. [Running It](#running-it)
14. [Repo Layout](#repo-layout)
15. [Design System](#design-system)
16. [Privacy](#privacy)

---

## The Problem

People prepare for interviews by reading lists of questions and talking at a wall. That practice is almost worthless because it removes the two things that actually decide a real interview:

1. **You can't hear yourself.** You don't know that you say "um" 34 times, that your pace collapses to 90 wpm when you're nervous, or that you trail off at the end of every answer.
2. **You can't see yourself.** You don't know that you look down-left when you're improvising, that your face goes flat when you're actually stressed, or that you broke camera contact for 40% of the answer.

Human practice partners give polite, uncalibrated feedback. Recording yourself and watching it back is painful, slow, and most people don't do it. Real interview coaching costs money most candidates won't spend on practice.

The gap: **nobody gives you measurable, per-answer feedback on delivery in the moment, for free, without another human involved.**

## Why the Obvious Approaches Fail

| Approach | Why it fails |
| --- | --- |
| Paste your transcript into an LLM | It grades your *content*, never your delivery. It has no idea you mumbled. |
| Upload the recording to a video model | Cost and latency scale with video length; most models can't give you a per-second confidence curve; you're shipping raw camera footage to a third party. |
| "AI interviewer" chatbots | They generate questions and conversation, not measurement. There's no signal extracted from your voice or face. |
| Self-review of a recording | Subjective, delayed, and people skip it. No trend line, no baseline. |
| Hiring-platform scoring tools | Not available to candidates, not transparent, not practice-oriented. |

The core constraint nobody talks about: **an LLM cannot watch your webcam.** It reads text. If you want it to critique delivery, something has to turn pixels and sound waves into numbers first.

## The Aha Moment

The first working version of this app *looked* fine and was completely wrong.

It generated questions, it accepted an answer, it returned a score and a wall of encouraging bullets. It also returned essentially the same feedback whether the user spoke for two minutes or sat in total silence with the camera covered.

Two root causes, both embarrassing and both invisible from the UI:

1. **The sensor was never on.** Face analysis was gated behind `isRecording && cameraOn`, and the record button didn't reliably start the media stream. So in practice the pipeline collected zero facial samples, and the model received `n/a` for every vision field.
2. **A silent fallback was hiding the failure.** When the model returned no structured tool call, the code quietly returned canned generic feedback instead of erroring. The user saw "feedback." The system had produced nothing.

That produced the reframe that shaped the rest of the build:

> **The model was never the problem. The model is the narrator. The product is the measurement layer.**

An LLM can only be as specific as the numbers you hand it. Feed it `null` and you get platitudes. Feed it `eye contact 41%, 22 filler words in 148 words, spectral centroid dropping, pace 96 wpm` and it can tell you exactly what went wrong and why.

So the architecture inverted. Instead of "record video, ask AI to judge," it became:

**measure continuously on-device → aggregate into a small numeric profile → hand that profile to the model → require structured output → never fall back silently.**

Everything after that was making the measurement layer honest and the failure modes loud.

## The Solution

A candidate picks a role, gets 20 AI-generated questions, and answers them one at a time on camera. While they talk:

- **Vision:** face-api.js runs a tiny face detector + expression net on the webcam feed every 600 ms. Face presence, centering, and expression probabilities accumulate into running averages.
- **Voice:** Web Speech API transcribes continuously (Chrome/Edge); Web Audio's `AnalyserNode` independently measures loudness (RMS), spectral centroid, and level variation every animation frame — so voice analysis survives even when transcription is unsupported.
- **Overlay:** live pills on the video (conf / eye / smile / engage), a face-tracked state, and a per-second timeline chart that draws confidence, attention, smile, and voice curves as you speak.

When you submit, the numeric profile plus the transcript go to an edge function, which forces Gemini into a `provide_feedback` function call: 0–100 score, 3–4 strengths, 3–4 improvements, 2–3 rewrites, four emotion/delivery scores, a speech verdict, and a facial/presence verdict. The prompt instructs the model to use the measured numbers and not to invent them, and to weight content ~40% / voice ~30% / facial presence ~30%.

Every answer is persisted with its full feedback object, so progress becomes a score-trend line, a four-axis skills radar, and a session history.

**The loop that matters:** answer → get a number and a reason → answer again → watch the number move.

## Architecture

```
                          BROWSER (on-device, no upload)
  ┌───────────────────────────────────────────────────────────────────┐
  │  getUserMedia({ video, audio })                                   │
  │        │                                                          │
  │        ├── <video> ──► face-api.js                                │
  │        │              tinyFaceDetector(224, thr 0.5)              │
  │        │              faceExpressionNet (7 classes)               │
  │        │              poll every 600 ms ──► FaceMetrics            │
  │        │                                                          │
  │        └── audio track ──┬──► Web Speech API (transcript, wpm,     │
  │                          │      fillers, pauses)                  │
  │                          └──► AudioContext + AnalyserNode(2048)    │
  │                                 RMS level, spectral centroid,     │
  │                                 variation (per rAF) ──► SpeechMetrics
  │                                                                       │
  │  FaceOverlay (live pills)   MetricsTimeline (1 Hz sample)           │
  └───────────────┬───────────────────────────────────────────────────────┘
                  │  { question, transcript, faceMetrics, speechMetrics }
                  ▼
        EDGE FUNCTIONS (Deno, Lovable Cloud)
        ├── generate-questions  → Gemini, forced return_questions tool
        └── analyze-answer      → Gemini, forced provide_feedback tool
                  │  structured JSON (no silent fallback)
                  ▼
        SUPABASE (RLS, per-user)
        ├── profiles            (auto-created by DB trigger)
        ├── interview_sessions  (role, status, overall_score)
        └── session_answers     (question, answer, score, feedback JSONB)
                  │
                  ▼
        Dashboard (stats)  ·  Progress (line + radar)  ·  History
```

Nothing about your face or voice leaves the browser as media. Only derived numbers and a text transcript cross the network.

## How Each Signal Is Actually Computed

This is the part most "AI interview app" READMEs skip. These are the real formulas in the code, so it's clear what is measurement and what is a guess.

### Facial (`useFaceAnalysis.tsx`)

Sampled every 600 ms. Per frame it records: face detected? is the face box centered (`|dx| < 0.25 && |dy| < 0.3`)? and the seven expression probabilities.

```
detRate      = detectedFrames / totalFrames
centeredRate = centeredFrames / detectedFrames

engagement   = detRate * 100
eyeContact   = centeredRate * 100
positivity   = min(100, (happy̅ * 0.7 + surprised̅ * 0.3) * 100)
confidence   = min(100, max(0, (happy̅*0.6 + neutral̅*0.5 − negative̅*0.6) * 100 * 1.4)
                        * (0.5 + detRate * 0.5))
negative̅     = (sad + angry + fearful + disgusted) / detectedFrames
```

**Read this honestly:** "eye contact" here means *your face stayed in the center of frame*. It is a presence proxy, not gaze estimation. If you stare at your own image on the left side of the screen, this drops. If you look at the camera but lean right, this also drops. It is useful, and it is not an eyeball tracker.

Fallback ladder if the model weights can't load from CDN: native `FaceDetector` API (Chrome's Shape Detection) → detection + centering only, expressions reported as `attentive` → if neither exists, the hook reports `unavailable` and the UI says facial analysis is off. It does not pretend to have data it doesn't.

### Voice (`useSpeechAnalysis.tsx`)

Two independent channels. Transcription from Web Speech; physical audio from `AnalyserNode` (`fftSize 2048`, smoothing 0.82).

```
level      = min(100, round(rms * 240))                  // per rAF
centroid   = Σ(mag_i · i) / Σ(mag_i) / (bins − 1)        // brightness of voice
speaking   = frames where level > 8

speakingRatio = speakingFrames / totalFrames
avgLevel      = mean(level)
avgVariation  = mean(|level_t − level_{t−1}|)
avgCentroid   = mean(centroid)

pace        : wpm < 110 → slow · 110–160 → moderate · > 160 → fast
tone        : soft       if speakingRatio < 0.12 or avgLevel < 6
              flat       if avgVariation < 2.5 and avgCentroid < 0.18
              energetic  if avgLevel > 22 or avgCentroid > 0.32 or avgVariation > 8
              steady     otherwise

paceScore   = wpm > 0 ? max(0, 100 − |138 − wpm| * 1.15)
                      : voiceDetected ? min(70, avgLevel*2.4 + speakingRatio*24) : 0
clarity     = clamp(paceScore − fillerRatio*200 + speakingRatio*8, 0, 100)
voiceConf.  = clamp(speakingRatio*45 + min(avgLevel,28)*1.5
                    + max(0, 18 − avgVariation*1.5)
                    + (wpm > 0 ? max(0, 20 − |138−wpm|*0.18) : 8)
                    − fillerRatio*120, 0, 100)
```

`138` wpm is the tuned "ideal" pace; the score falls off linearly in both directions. Fillers are matched with a fixed regex (`um|uh|like|you know|basically|actually|literally|sort of|kind of|i mean|hmm|er|ah`). Pauses are counted when transcription goes silent for more than 1500 ms.

**Read this honestly:** "tone" is derived from loudness, brightness, and jitter. It detects a quiet monotone delivery versus an animated one. It does not detect sincerity, warmth, nervousness, or accent, and it will happily call a loud monotone "steady."

### Fusion (`analyze-answer`)

The prompt states the weighting explicitly and forbids invented numbers:

```
content quality  ≈ 40%
delivery/voice   ≈ 30%
facial presence  ≈ 30%
```

The model is forced to answer through a `provide_feedback` tool call with a strict schema (`additionalProperties: false`). If no tool call comes back, the function returns a **502 and the UI shows an error** — it does not fabricate feedback. That behavior is a deliberate scar from the first version.

## Tech Stack

| Layer | Choice | Why (and the tradeoff) |
| --- | --- | --- |
| Framework | React 18 + Vite 5 + TypeScript | Fast HMR for a UI with this much state; strict types on the metric interfaces kept the hooks honest. |
| Styling | Tailwind v3 + shadcn/Radix + `class-variance-authority` | Radix gives accessible primitives without a component vendor; tokens live in CSS so theming stays centralized. |
| Motion | Framer Motion | Answer-card transitions, metric bars that animate to value, the pulsing REC ring. |
| Charts | Recharts (line + radar) + hand-rolled SVG | Recharts for trend/radar; the per-second timeline is raw SVG polylines because it re-renders every second and Recharts is too heavy for that. |
| Data fetching | TanStack Query + direct Supabase calls | Query for cache safety; the session flow is imperative enough that plain calls were clearer. |
| Vision | `face-api.js` (tinyFaceDetector + faceExpressionNet, TensorFlow.js) | Runs entirely in-browser at ~1.6 Hz — no media upload, no per-frame API cost. Weights load from a jsDelivr CDN with a second mirror as fallback. |
| Vision fallback | Native `FaceDetector` (Shape Detection API) | Chrome-only, detection-only, but keeps *some* signal alive when the CDN is unreachable. |
| Transcription | Web Speech API (`webkitSpeechRecognition`) | Free, zero infra, live interim results. Cost: Chrome/Edge only, network-backed, `en-US` hardcoded. |
| Audio physics | Web Audio `AnalyserNode` | Deliberately independent of speech recognition so loudness/centroid/variation still measure in browsers that can't transcribe. |
| AI | Lovable AI Gateway → `google/gemini-2.5-flash` (feedback), `google/gemini-3-flash-preview` (questions), OpenAI-compatible tool calling | Structured function-calling is non-negotiable — free-form prose can't be charted. |
| Backend | Supabase via Lovable Cloud: Postgres + RLS + triggers + Deno edge functions | Auth, per-user row security, and AI calls behind a server so no gateway key sits in the browser. |
| Tests | Vitest + Playwright (installed) | **Currently only a placeholder test.** See [Not Built](#not-built-on-purpose-or-by-omission). |

## Data Model

Three tables, all RLS-scoped to `auth.uid()`, no cross-user reads possible.

```sql
profiles            (id, user_id → auth.users, display_name, avatar_url, timestamps)
interview_sessions  (id, user_id, role, status in ('in_progress','completed'),
                     overall_score, total_questions, completed_at, timestamps)
session_answers     (id, session_id → interview_sessions, user_id, question_number,
                     question_text, answer_text, score, feedback JSONB, created_at)
```

- `handle_new_user()` (SECURITY DEFINER, `on_auth_user_created`) inserts a profile row on signup, so the dashboard never races a missing profile.
- `feedback` is stored as JSONB, which is what makes the progress radar possible: skill averages are computed by aggregating the stored emotion objects across all past answers.
- Indexes: `sessions(user_id)`, `sessions(created_at DESC)`, `answers(session_id)`.
- `updated_at` maintained by trigger on `profiles` and `interview_sessions`.

**Roles:** software-developer, product-manager, data-analyst, marketing, healthcare, education, design, engineering — 20 generated questions each, mixed roughly 30% warm-up / 50% medium / 20% curveball.

## What Broke During the Build

Not a highlight reel. The actual failure log, because every one of these changed the design.

| # | Symptom | Root cause | Fix / lesson |
| --- | --- | --- | --- |
| 1 | Blank screen: `Cannot read properties of null (reading 'useRef')` inside a Vite dep chunk | Two React instances in the dep graph — face-api.js pulls TensorFlow.js, which destabilized Vite's pre-bundled chunking; a stray `TooltipProvider` sat above `BrowserRouter` and lost context | Added an explicit `dedupe` list in `vite.config.ts` (react, react-dom, jsx runtimes, react-query) and removed the unused provider. Lesson: a heavy ML dependency in the browser needs dedupe configured *before* it breaks at runtime. |
| 2 | Feedback was vague and didn't change with actual performance | Silent fallback returned canned feedback when the model produced no tool call | Removed the fallback; a missing tool call is now a 502 the user sees. **Never let an error path look like a success path.** |
| 3 | "The AI doesn't detect my voice or my face" | Face analysis only ran while `isRecording && cameraOn`, so the metrics were empty until the user hit record — and many users never hit record | Vision now runs whenever the camera is on, so the bars move immediately; a warning appears when the mic isn't actually listening. Lesson: if a feature only works after an undiscoverable second click, it doesn't exist. |
| 4 | Model returned no structured output on `gemini-3-flash-preview` with forced tool choice | Preview model's function-calling reliability | Switched the feedback path to `gemini-2.5-flash`. Structured output is a model-selection criterion, not an afterthought. |
| 5 | Build broken: missing `@tanstack/query-core` | Transitive dep not present in the resolved install | Reinstalled; the dev server picks it up automatically. |
| 6 | Transcript never appeared in Safari/Firefox | Web Speech API doesn't exist there | Voice physics still run via Web Audio; the UI states the browser limitation and offers manual typing instead of failing quietly. |
| 7 | Face model weights failed to load | Single CDN mirror | Two weight URLs tried in order, then native `FaceDetector`, then an explicit `unavailable` state with a visible message. |

## Honest Assessment: what is real and what is not

**Genuinely real:**
- The camera and mic are truly live and truly measured. Numbers move because your face and voice changed.
- The transcript, WPM, filler count, pause count, loudness, and spectral centroid are measured, not simulated.
- Expression probabilities come from a trained model (face-api's expression net), not from a rule engine.
- Every score is stored and the trend line is a real aggregation of past sessions.
- RLS means one user genuinely cannot read another user's answers.

**Heuristics dressed as metrics — say this out loud:**
- **Eye contact is frame-centering, not gaze.** No iris tracking, no head-pose estimation.
- **Confidence, positivity, and engagement are weighted formulas over expression probabilities.** They are interpretable, not scientifically validated constructs.
- **Voice tone is loudness + brightness + jitter.** It cannot hear emotion, and it can't tell a nervous monotone from a bored one.
- **The 0–100 score is one LLM's judgment** of a numeric profile plus a transcript. It is not calibrated against real interview outcomes and it is not the same score two runs in a row. Treat it as directional feedback, not an assessment.
- **"Correct facial expressions" is not detected.** There is no ground truth for the right expression at a given moment; the app measures what your face did and lets the model comment on it.

**Deliberate design constraints:**
- The AI never sees your video or audio. If the numbers say nothing, the feedback says nothing — that's the point of the measurement layer.
- Failures surface as errors and warnings. There are no placeholder scores in this codebase.

## Known Limitations

- **Browser support is uneven.** Chrome/Edge gives full transcription + vision. Safari/Firefox get vision + audio physics with a typed or absent transcript. Firefox has no `FaceDetector` fallback either.
- **English only.** `recognition.lang = "en-US"` is hardcoded.
- **Network required for transcription** (Google's speech service) and for the CDN model weights.
- **No media recorded or stored.** There is no video replay, no timestamped annotations, no "watch yourself at 0:47." The timeline chart exists during recording and is gone after.
- **Vision accuracy depends on lighting, angle, and distance.** Tiny-face-detector at `inputSize 224` with `scoreThreshold 0.5` misses profiles, low light, and off-axis faces; a missed frame counts as "no face," which lowers engagement.
- **Short answers are punished by pace math.** With few words, WPM is noisy and `paceScore` is unstable.
- **One model, one prompt, no calibration.** Feedback quality inherits the model's moods; there's no ensemble, no self-critique pass, no A/B of prompts.
- **No accent, dialect, or non-native-speaker fairness testing.** Filler and pace heuristics can misjudge fluent non-native speakers.
- **No rate-limit or cost story for heavy use** beyond passing through 429/402 from the gateway with a readable message.
- **Accessibility is unaudited.** Live metric values are visual-only; there are no ARIA announcements for a screen-reader user, and color is one signal channel for tone (green/amber/red).
- **Mobile is responsive-web, not native.** Camera behavior on iOS Safari is the least-tested path in the app.

## Not Built (on purpose or by omission)

- **Dark/light toggle.** Dark-mode tokens exist in CSS, but there is no theme switch in the UI — `next-themes` is only pulled in by the toast component.
- **Google OAuth.** Email/password only; the provider isn't wired into the sign-in screen.
- **Personalized improvement plans** across sessions, and **shareable result cards** — both were listed as bonuses and neither exists.
- **Real tests.** `src/test/example.test.ts` asserts `true === true`. The metric formulas in `useSpeechAnalysis` are pure and trivially unit-testable, and they are not tested.
- **Video replay with annotations, speaker diarization, interview mode with a live AI interviewer, role-specific rubrics, admin/analytics surface.**

## Running It

```bash
npm install
npm run dev            # Vite dev server
npm run build          # production build
npm run lint           # eslint
npm run test           # vitest (placeholder suite)
```

Environment: `VITE_SUPABASE_URL` and `VITE_SUPABASE_PUBLISHABLE_KEY` are generated by the backend — don't edit `src/integrations/supabase/client.ts`. Edge functions need `LOVABLE_API_KEY` set server-side and are redeployed with the project's function deploy step; both `generate-questions` and `analyze-answer` read it from the function environment, never from the browser.

For the full experience use Chrome or Edge, allow camera **and** microphone, and click the record button before you start talking — vision runs from the moment the camera is on, but the timeline and transcript only collect while recording.

## Repo Layout

```
src/
  hooks/
    useFaceAnalysis.tsx     # tinyFaceDetector + expression net, CDN → native → unavailable ladder
    useSpeechAnalysis.tsx   # Web Speech + Web Audio; RMS/centroid/variation → tone, clarity, voiceConfidence
    useMetricsTimeline.tsx  # 1 Hz sampling of both metric streams
    useAuth.tsx             # session context over the auth client
  components/
    VideoRecorder.tsx       # <video> + stream lifecycle + blocked-camera state
    FaceOverlay.tsx         # live conf/eye/smile/engage pills, face-tracked state, reticle
    LiveMetrics.tsx         # metric bars + pace/words/fillers/pauses tiles
    MetricsTimeline.tsx     # hand-drawn SVG multi-series per-second chart
    FeedbackCard.tsx  EmotionIndicator.tsx  ScoreBadge.tsx  StatCard.tsx  RoleCard.tsx
    AppLayout.tsx  ProtectedRoute.tsx  NavLink.tsx
  pages/
    AuthPage.tsx  Dashboard.tsx  PracticePage.tsx  SessionPage.tsx  ProgressPage.tsx  NotFound.tsx
supabase/
  migrations/…sql            # profiles, interview_sessions, session_answers, RLS, triggers
  functions/
    generate-questions/      # Gemini → forced return_questions
    analyze-answer/          # Gemini → forced provide_feedback (no silent fallback)
```

`SessionPage.tsx` is the orchestrator: it owns the media stream, starts vision and voice, syncs the live transcript into the answer box, submits the fused profile, and persists the answer with its feedback.

## Design System

Deliberately not a generic SaaS gradient. Deep slate/navy base with a teal accent for measured success and amber for "fix this," so a color glance tells you which metric needs work.

```
--primary   navy 220 60% 20%      structural
--accent    teal 168 70% 40%      measured good
--warning   amber 38 92% 55%      needs work
--destructive red 0 72% 55%       failing / recording
--success   green 158 64% 42%     above threshold
```

Metric thresholds are shared across the overlay and the live panel: **≥70 success, ≥40 warning, below that destructive**. Space Grotesk for headings, Inter for body. All color, gradient, and shadow values are semantic tokens in `src/index.css` — no hardcoded hex in components, which is what keeps dark mode coherent.

## Privacy

No audio or video is uploaded or stored. What leaves the device is: your spoken words as a text transcript, and the derived metric numbers. Those are persisted per user in Postgres under row-level security, so only your own account can read them. Delete your account and the cascade removes your sessions and answers.

---

**The one-sentence version:** HireSense AI turns your webcam and mic into a measurement instrument, and uses an LLM only as the voice that reads the instrument back to you.
