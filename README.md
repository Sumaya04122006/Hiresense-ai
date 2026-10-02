# HireSense AI

**Practice interviews with live feedback on how you sound and look while you answer, then get an AI review of the whole session.**

**Live demo:** https://hireesense.lovable.app

---

## The Problem

Most interview prep tools only look at **what** you say. Interviewers also judge **how** you say it: pace, volume, long pauses, eye contact, fidgeting.

- Mock interviews with real people are expensive and hard to schedule.
- Recording yourself and watching it back is slow and awkward, and nobody gives you a score.
- Text-only AI tools (paste your answer into ChatGPT) miss delivery completely.
- You get feedback **after** the habit is already built, never while you're doing it.

## The Solution

HireSense AI runs a mock interview in your browser:

1. Pick a role, or type in a custom one.
2. Answer up to **20 role-specific questions** out loud, on camera.
3. While you speak, **live indicators** show your volume, speaking pace (WPM), pauses, whether your face is in frame, head movement, and look-aways.
4. Each answer is transcribed.
5. At the end, an LLM reviews the **whole session**: transcripts plus the measured audio and video signals. You get a score, strengths, weaknesses, delivery notes, concrete suggestions, and a rewritten version of your weakest answer.

## The "Aha" Moment

You see **"Pace: Fast"** or **"Looking away"** come up *while you're mid-sentence*, and you fix it on the spot. Then the final report links those delivery habits to what you actually said, across all your answers, not just one.

## What Makes It Useful

- **Real-time, on-device analysis.** Audio and video are analyzed in the browser. No video is uploaded or processed on a server.
- **Evaluates the whole session.** Feedback is based on patterns across every answer, not one cherry-picked response.
- **Feedback based on signals.** The evaluation prompt tells the model not to invent emotions or confidence levels the measured data doesn't support.
- **No account needed.** Open the link and start.
- **Privacy.** Media is not stored. Audio is sent only to be transcribed, then thrown away.

## Supported Roles (20 curated questions each)

- Frontend Engineer
- Backend Engineer
- Product Manager
- Data Scientist
- UX Designer
- Sales Development Rep
- Custom role (free text)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | TanStack Start (React 19, SSR) + Vite 7 |
| Styling | Tailwind CSS v4, shadcn/ui |
| Audio analysis | Web Audio API (`AnalyserNode`, RMS volume) |
| Video analysis | MediaPipe Tasks Vision `FaceLandmarker` (WASM/GPU, in-browser) |
| Recording | `MediaRecorder` (WebM) |
| Backend | Lovable Cloud (Supabase Edge Functions, Deno) |
| AI | Lovable AI Gateway → `google/gemini-2.5-flash` |
| Hosting | Lovable (Cloudflare edge) |

### Edge Functions

| Function | Purpose |
|---|---|
| `generate-question` | Writes a question for custom roles |
| `transcribe-audio` | Turns recorded answer audio into a transcript (Gemini multimodal) |
| `evaluate-answer` | Returns a structured evaluation via forced tool-calling: `{ score, content, delivery, improvedAnswer, suggestions }` |

---

## How It Works

```text
Browser                                         Cloud
-------                                         -----
Camera ─► MediaPipe FaceLandmarker ─► face %, movement, look-aways ─┐
Mic    ─► Web Audio AnalyserNode   ─► volume, WPM est., pauses     ─┤
Mic    ─► MediaRecorder (webm)  ───────────────► transcribe-audio ──┤
                                                                    ▼
                         per-answer record { transcript, audio, video summary }
                                                                    ▼
                    after last answer ──────────► evaluate-answer (LLM, 1 call)
                                                                    ▼
                                              Feedback page (score + report)
```

**No GPT calls during answering.** All live feedback is local math. The LLM is only called for transcription and the final evaluation.

---

## Honest Limitations (no sugar coat)

- **WPM is estimated**, not counted. It comes from time spent speaking × a syllable heuristic, not from live speech recognition.
- **"Look-away" means head position**, not eye tracking. It fires when your nose landmark leaves the center of the frame.
- **Speaking detection is a volume threshold.** Background noise can count as speech.
- **Transcription uses a general-purpose LLM** (Gemini 2.5 Flash), not a dedicated speech-to-text model like Whisper. Accuracy is good but not best-in-class.
