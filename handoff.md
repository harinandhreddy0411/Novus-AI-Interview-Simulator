# Novus AI — Interview Simulator — Handoff (Updated)

Continuation doc for a teammate picking up the project. Covers what the product
is, the current state after the latest work session, what's working, what's
half-done, the exact open bugs, and the next steps in priority order.

---

## 1. What the project is

**Novus AI** is a client-side, AI-powered mock interview web app. The loop:

1. User logs in (Supabase auth) and picks a **role** (SWE / Data / Frontend / Behavioral).
2. The app asks questions from a **fixed, topic-tagged bank** for that role.
3. Each spoken answer is scored on three composites — **Correctness** (LLM rubric),
   **Communication** (delivery metrics), **Composure** (face/timing proxies).
4. A **one-tap self-report (1–5 confidence)** is collected after each answer.
5. The engine adaptively asks more questions on the **weakest topic**.
6. A **report view** shows overall score, a Chart.js radar of the 3 composites,
   per-topic bars, and a per-question breakdown.
7. Session results are **saved to Supabase** (per-user history).

Supporting modules (older, still present): AI study guides (DSA / System Design /
STAR), replicated OAs with a live Python runner (Piston API), and webcam
attention tracking (MediaPipe).

**Privacy angle (a selling point):** all face processing runs **on-device** in
the browser. Video never leaves the machine.

---

## 2. File inventory (current state)

Project folder (Windows, OneDrive-synced):
`C:\Users\rahul\OneDrive\Desktop\interview\`

```
interview/
├── index.html          Working — shell, all views, loads Chart.js + Supabase
├── styles.css          Working — dark theme + report view + self-report overlay
├── app.js              Working — main logic (see below)
├── scoring.js          Working — weighted scoring engine, fully integrated
├── fer.js              NEW — in-browser facial-expression inference (see bug #2)
├── create_sessions_table.sql   One-time SQL, already run in Supabase
├── fer_cnn_train.ipynb Colab notebook — trains the FER CNN (already run once)
├── models/
│   └── fer_tfjs/       The exported TF.js model (model.json + 3 .bin + class_names)
└── supabase/
    └── functions/
        └── gemini-proxy/
            └── index.ts   Edge Function source (already deployed to Supabase)
```

---

## 3. How to run it

- **Must be served over HTTP**, not `file://` (ES modules + camera/mic need it).
  In the `interview` folder address bar type `cmd`, then:
  `python -m http.server`  → open `http://localhost:8000`
- **Browser:** Chrome or Edge (speech uses `webkitSpeechRecognition`).
- **Login:** Supabase auth. Any email + 6+ char password auto-creates an account.
  "Confirm email" is OFF in Supabase Auth settings (already done).

---

## 4. What got DONE this session

1. **scoring.js fully integrated** into app.js. All raw metrics captured live:
   WPM, word count, filler rate, response latency, focus %, and (when the FER
   model loads) a tension proxy.
2. **Question bank expanded** from 2 roles to 4 real banks:
   `TECHNICAL` (SWE), `DATA`, `FRONTEND`, `HR` (behavioral) — 10 questions each,
   properly topic-tagged so the adaptive weak-topic selector works.
   `ROLE_OPTIONS` maps data→DATA and frontend→FRONTEND.
3. **Self-report (1–5 confidence)** overlay added after each answer. Stored on
   each answer object as `selfReport`. This starts the labelled dataset.
4. **Dedicated report view** (`view-report`) built with a Chart.js radar of the
   three composites, weakest-first topic bars, and per-question cards showing the
   self-report badge and a transcript snippet. Replaces the old inline text report.
5. **Gemini API key secured.** The hardcoded key is GONE from app.js. All Gemini
   calls now route through a **Supabase Edge Function** (`gemini-proxy`) that
   holds the key server-side as a secret.
6. **Session storage.** A `sessions` table was created (RLS on, owner-read).
   `renderInterviewReport` fires `saveSessionToSupabase()` (non-blocking) which
   posts the result through the same Edge Function.
7. **FER CNN trained and exported.** `fer_cnn_train.ipynb` was run in Colab on
   FER-2013 + MobileNetV2. Final accuracy ~58% (state-of-the-art for FER-2013 is
   ~65%, human agreement ~65%, so this is fine). Exported to TF.js into
   `models/fer_tfjs/`. `fer.js` was written to load it and compute the tension
   proxy from webcam frames during each answer.
8. **Critical bugs resolved.** The Supabase Auth login issues, FER model loading
   errors (`batchInputShape`), and Gemini proxy 404s have all been fixed. End-to-end testing is unblocked.

---

## 5. Supabase setup (already configured)

- **Project:** "Novus AI", ref `zzaqawcpqdbdymcugcfy`, region ap-southeast-2, FREE/Nano plan.
- **Auth:** "Confirm email" OFF, "Allow new signups" ON.
- **Edge Function `gemini-proxy`** is deployed. "Verify JWT with legacy secret" is OFF.
  - Handles two actions: `gemini` (proxies the model call) and `save_session`
    (verifies the user JWT, inserts into `sessions`).
  - **Secrets set** (Dashboard → Edge Functions → Secrets):
    - `GEMINI_API_KEY`
    - `SERVICE_ROLE_KEY`  (named without the `SUPABASE_` prefix because Supabase
      reserves that prefix — the function reads `Deno.env.get("SERVICE_ROLE_KEY")`)
- **`sessions` table** created via `create_sessions_table.sql` (RLS enabled).
- **API keys:** the app uses the **legacy anon key** (`eyJ...`) in app.js, not the
  new `sb_publishable_...` key — the new format wasn't working with
  `supabase-js@2`'s `signInWithPassword`. The legacy anon key is in
  Settings → API Keys → "Legacy anon, service_role API keys".

---

## 6. OPEN BUGS

*(No critical blockers! The previous bugs regarding Supabase Login 400 errors, FER model loading with Keras 3, and Gemini proxy 404s have all been successfully resolved.)*

---

## 7. Architecture / scoring notes (unchanged, still true)

- **Two-phase question selection:** Phase 1 asks one random question per topic
  (calibration); Phase 2 is weighted-random biased toward lowest-scoring topics.
- **scoring.js pipeline:** raw measurement → normalize each to 0–100 → weighted
  combine into 3 composites → weighted combine into 1 role-weighted overall.
  Missing inputs (e.g. no camera) are dropped and remaining weights renormalized.
- **All thresholds/weights in `CONFIG`** are honest placeholders to be calibrated
  from real data later, NOT validated constants.
- **Honesty stance (keep this for credibility):** the CNN classifies *expressions*,
  not emotions. Its output feeds a clearly-labelled **"Composure (proxy)"** —
  never a raw "anxiety" number. The self-report (1–5) is the real label to train
  a fusion model on later.

---

## 8. Next steps (priority order)

1. **Verify session save end-to-end** — finish an interview, confirm a row appears in
   Supabase → Table Editor → `sessions`.
2. **Polish the "history" view** — ensure the user's past `sessions` accurately reflect progress
   over time (the logic is partially implemented in `app.js`).
3. **Add the low-confidence flag** more prominently on the overall score when
   composure is computed from too few inputs (partial: per-answer "⚠ LOW DATA"
   badge already exists).
4. **Calibrate weights** once enough labelled sessions exist — fit `CONFIG`
   weights by regression against the self-report labels / outcomes.
5. **Rotate secrets before any public use.** Both the Gemini key and the Supabase
   service-role key were shared in chat during setup; generate fresh ones
   (Google AI Studio for Gemini; Supabase → Settings → API for the service role).

---

## 9. Security TODO (do not skip before real users)

- Rotate the Gemini API key and Supabase service-role key (see step 7 above).
- The anon key in app.js is fine to be public (that's its purpose) **as long as
  Row Level Security is on** for every table — confirm RLS is enabled on
  `sessions` (it is, via the SQL) and on any future tables.
