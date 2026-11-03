# Project Brief: Point-and-Fix: a coding agent that checks its own fix by looking at the screen

> Working name. Rename freely.

## Context

- **Event:** Meta Global AI Developer Hackathon, **Indie Developer track**.
- **Deadline:** Project submission due **Wed Nov 4, 2026, 9:00 PM ET**. Code freeze target: **3:00 PM ET Nov 4**. The remaining time goes to the demo video and README.
- **Theme fit:** "Build for Your World". This is a developer tool for a problem I hit daily.
- **Required platform:** the project must use Meta's models (Meta Model API and/or open weights). No specific model or model count is required.
- **All code is written fresh for this event.** Do not import code from other projects.

### Judging (from the Participant Guide)

- **Stage 1 (pass/fail):** fits a problem statement and reasonably uses Meta models.
- **Stage 2 (0–10 per category):**

| Category | Weight | What it asks | How this plan answers it |
| --- | --- | --- | --- |
| Impact | 30% | Real-world potential, who benefits, could people use it | Every frontend developer hits visual bugs. Second audience: designers/PMs can circle a bug instead of writing a ticket. The target app is set by config, not hardcoded. |
| Use of Meta AI Tooling | 25% | Creative, beyond basic integration, surprising capabilities | Muse Spark as agent **and** as an independent vision verifier (structured output, multi-image cross-viewport check). Muse Voice Transcribe for spoken notes. SAM 3.1 only if its test earns a place. |
| Technical Depth & Execution | 20% | Sound, refined engineering, real craft | CDP matched-rules locator, three-signal verifier, cross-viewport regression check, sandbox, snapshot revert, tests. |
| Demo | 25% | Working, impressive, holds up live, video cool to watch | Real finger on a real phone, a caught failure and retry, live HMR change on the laptop. |

- **Stage 3:** the top 5 per track; finalists' videos are watched live and ranked with the same rubric.
- **Bonus prizes we go for:**
  - **Best Agentic Use-Case ($5K):** say so in the submission text. The case: a multi-step agent with its own verify-and-retry loop.
  - **Most Viral on X ($20K) and Community Vote ($10K):** post the demo video on X after submitting and submit the post link.
  - **Not targeted:** Best Build on Open Weight Models. It needs Baseten credits and a second model path, and tool calling on `muse-spark-open-1026` is unconfirmed. The base URL and model come from env, so an open-weights run stays possible after the freeze.

### Rules and submission (from the Participant Guide)

- The repository **must be public**. Check the git history for secrets before making it public.
- **Demo video: 3 minutes max.** It must show only features, code and functionality built during the hackathon. It is a primary judging artifact, together with the code and the live product.
- **Show what is new.** The building period is Oct 27 – Nov 4. The README and the video must say clearly that all code was written during the hackathon. Commit often, because the public git history is evidence.
- Solo is allowed. Submit on the CV platform by **Nov 4, 9:00 PM ET**. Late submissions are not considered.
- Avoid the "anti-projects" (e.g. simple image analyzers). Present this as an agent that edits code and verifies its fix, not as an image analyzer.
- **API credits:** claim the $150 through the form by Nov 4, 9 PM. A credit card on the account is required before any API call works. Set a spend-limit alert on the Billing page.
- **Help:** Discord #office-hours, 10–11 AM ET on weekdays. **Nov 3 is the last full day for technical troubleshooting.**

## Pitch

**Lead with the loop, not the phone.** A coding agent that **looks at the screen to check its own fix**: see → act → verify. The phone is how it is delivered: you circle the bug with a finger and say what's wrong, instead of describing it in text. The UI also works in a desktop browser.

**Who benefits:** frontend developers first. Second: a designer or PM circles the bug, and the developer approves a fix that comes with evidence. Nobody writes a bug ticket.

## Problem

Coding agents verify their work with tests and logs, but they rarely look at the result. For visual and UI bugs (misalignment, wrong color, overflow, a broken layout on mobile), this creates two problems:

1. **Describing the bug is slow and lossy.** "The second button in the card footer is 4px lower than the first" is harder to write than pointing at it.
2. **The agent can't confirm the fix.** It edits CSS, the tests pass, and the bug is still on screen. The developer has to check manually, which is the step they wanted to skip.

This is worse when the developer is away from their desk on a phone, where typing precise descriptions is painful and reviewing code diffs is hard.

## Solution

A mobile-first web app where the developer **circles a bug on a live screenshot of their app**. The system:

1. Takes the circled region plus a short note, typed or spoken (e.g. "align these").
2. Asks the browser which DOM elements and **which CSS rules** paint that region.
3. Lets the agent find and edit the responsible code.
4. **Re-renders the app, screenshots the same region, and verifies the fix** with pixel diff, DOM geometry and a vision model. If the fix failed, it reverts and retries (max 3 attempts).
5. **Checks that nothing else broke:** renders the other viewport and diffs everything outside the target region.
6. Returns a **before/after image pair, the evidence, any regression warnings and the diff**. The developer taps **Approve** or **Reject**.

## Core user flow (the demo)

1. The developer opens the **Point-and-Fix web UI** on their phone (not the target app). It shows a fresh screenshot of the target app, rendered on the developer's machine with Playwright.
2. They pick a viewport (Desktop or Mobile), circle the bug with a finger, and hold the mic button to say a short note (or type it).
3. The phone shows live progress: locating the code → editing → re-rendering → verifying → (reverting and retrying) → checking the other viewport.
4. The result card shows before/after crops side by side, the verdict and its evidence ("fixed": both buttons now have `top: 412px`), any regression warnings ("also changed: desktop header"), and the code diff.
5. Approve keeps the change. Reject reverts it.

Target: under 60 seconds for a simple CSS bug. **This is unverified.** Measure real model latency in milestone 1 (see Latency budget).

## Runtime setup and coordinates

```
Developer's machine
 ├─ Target app dev server   → TARGET_URL, default localhost:3000 (Vite, never exposed)
 ├─ Point-and-Fix server    → localhost:4378
 │    └─ Playwright (headless Chromium) opens localhost:3000 and takes screenshots
 └─ HTTPS tunnel (cloudflared) → exposes only Point-and-Fix

Phone → tunnel → Point-and-Fix UI → shows the screenshot PNG; the user draws on it
```

- The phone never loads the target app. The target app does not need to be deployed.
- Only the Point-and-Fix server is exposed, through `cloudflared tunnel --url http://localhost:4378`.
- **The server sets the viewport, not the phone.** Presets: Desktop 1280×800 (DPR 1), Mobile 390×844 (mobile user agent, `isMobile`, DPR 3).
- The phone displays the screenshot scaled to fit, with pinch-zoom. The selection box is sent as **normalized coordinates (0–1)** relative to the image.
- **Coordinate rule (unit-tested for both presets):**
  - DOM queries use `normalized × viewport CSS size` (e.g. 390×844).
  - Image crops use `normalized × image pixel size` (e.g. 1170×2532 on Mobile).
  - Never mix the two. The image size is not the CSS size when DPR > 1.
- The MVP uses a single viewport-sized screenshot (no full-page scroll).
- Before and after screenshots use the same viewport, the same URL and route, and a stable state (animations and transitions disabled through injected CSS, fonts loaded, network idle, and a wait until Vite HMR has applied the edit) so crops are comparable.

## Selection UX

- Selection happens **inside the Point-and-Fix UI**, not with iOS Markup or screenshot uploads.
- The screenshot is shown with a canvas overlay. The user draws a freehand loop with one finger. The app computes the loop's bounding box and sends it as normalized coordinates.
- A **Draw / Move** toggle separates drawing from pinch-zoom and panning.
- After drawing, show the resulting box and allow **Redo** before submitting.
- Use Pointer Events, and test on iOS Safari specifically.

## Architecture

```
phone (mobile web) ──HTTPS──▶ server (Node 22 + TypeScript, Hono)
                                 │
                                 ├─ Auth: startup token → HttpOnly cookie
                                 │
                                 ├─ Renderer: Playwright (headless Chromium): stable screenshots
                                 │
                                 ├─ Locator: box → overlapping DOM elements → CDP matched CSS rules
                                 │           (file + line ranges)
                                 │
                                 ├─ Agent: Muse Spark tool calling
                                 │     tools: list_files · read_file · search · edit_file
                                 │     (the agent never renders; it ends its turn when done)
                                 │
                                 ├─ Snapshots: original bytes of every edited file (Reject / retry)
                                 │
                                 ├─ Verifier: pixel diff gate → DOM geometry/style delta → vision verdict
                                 │
                                 ├─ Regression check: other viewport + outside-region diff → warnings
                                 │
                                 ├─ Transcriber: WAV from the phone → Muse Voice Transcribe → note text
                                 │
                                 └─ Job runner: one job at a time; owns the attempt loop (max 3),
                                                timeouts, and SSE progress events
```

### Attempt loop (owned by the harness, not the model)

```
for attempt in 1..3:
  restore originals (snapshot store)            # every attempt starts clean
  run agent (with previous failure feedback)     # edits files, then ends its turn
  if no edit was made → failure "no edit made"; continue
  re-render, screenshot, re-query the same elements
  verdict = verify(before, after)
  if verdict.fixed → run regression check, stop, show result (with any warnings)
  feedback = failed diff + verdict.explanation
final failure → restore originals, show "not fixed" card with all attempts
```

The "before" screenshots for **both** viewports are taken when the job starts, so the regression check has a baseline.

### Key technical decisions

- **Locator: ask the browser which rule painted the pixel.**
  1. List the visible elements whose bounding rect overlaps the box. Rank them by overlap and keep the top ~8, with selector path, tag, classes, text, `data-*` and key computed styles (display, position, margin, padding, flex/grid props, color, font, overflow).
  2. For each element, call CDP `CSS.getMatchedStylesForNode`. It returns the matched rules with their stylesheet and source line ranges, which map to the CSS files in `TARGET_DIR`.
  3. Send these to the agent as candidates, together with the image inputs.
  - [ ] **Check in milestone 1:** how Vite dev-mode `<style>` tags report their source (`data-vite-dev-id`, source maps). If the mapping is unreliable, serve the CSS through `<link>` tags in dev, or fall back to grepping the selectors.
- **What the model sees:** the crop padded by ~30% (alignment depends on neighbors), plus the full viewport screenshot with the box drawn on it. Only the box is sent, not the freehand loop.
- **Verifier: three independent signals. The agent never grades itself.**
  1. **Pixel diff gate:** if no pixels change in the padded region → `fixed: false` ("your edit had no visible effect"). No model call.
  2. **DOM delta:** bounding boxes and computed styles of the same elements, before vs after. Shown on the result card as evidence (e.g. `top: 418 → 412`).
  3. **Vision verdict:** a separate call that gets only the before crop, the after crop, the user's note and the DOM delta. It does **not** see the agent's reasoning or explanation.
  - `fixed: true` requires the pixel gate to pass **and** the vision verdict to say fixed.
  - Vision output uses `response_format: json_schema`: `{ "fixed": boolean, "confidence": 0-1, "explanation": string }`. A truncated, failed or timed-out response becomes `fixed: false` with an error message.
- **Cross-viewport regression check (a warning, not a blocker).** After a verified fix:
  1. Pixel-diff the **same** viewport outside the padded target region.
  2. Render the **other** viewport and pixel-diff the whole screen against its baseline.
  3. If either diff passes a small threshold (ignore anti-aliasing noise), send the before/after pairs to the vision model in **one multi-image call** with the question "is this change an unintended side effect?".
  4. Show the result on the card as a warning with the crops. The developer decides.
- **Spoken notes.** The phone records with Web Audio (PCM), downsamples to 16 kHz mono and encodes WAV in the browser (about 40 lines, no dependency, no ffmpeg). The server forwards it to `POST /v1/asr/transcribe` and fills the note field. The user can edit the text before submitting. If transcription fails, typing still works.
- **Reject and retries use a snapshot store, not git.** On the first edit of a file in a job, save its original bytes (in memory, mirrored to `.point-and-fix/snapshots/` in case the server crashes). Retry and Reject restore them. Approve clears them. One job at a time, enforced by a server-side lock.
- **Agent output is untrusted.** `tool_choice` only supports `"auto"`, so the model may answer in text without editing anything. That counts as "no edit made". Tool arguments are not guaranteed to match the schema, so validate every argument in code before running it (especially paths, for the sandbox).
- **SAM 3.1 is optional and off the critical path.** In milestone 1, run a **hard 30-minute test**: one screenshot of a button plus a box prompt. If it isolates the element better than the DOM rect, use the mask only to **outline the target on the result card**. If not, drop it and explain why in the README.

## Meta Model API (verified against dev.meta.ai/docs, Nov 3)

| Fact | Value | Source |
| --- | --- | --- |
| Base URL / auth | `https://api.meta.ai/v1`, `Authorization: Bearer <key>`. OpenAI-compatible. | /docs/sdks |
| Client | `openai` npm package (no official Meta Node SDK) | /docs/sdks |
| Model | `muse-spark-1.3` (standard; **not** `-contributor`, which trains on your data and is limited to 150 RPM). Read from `META_MODEL` (default `muse-spark-1.3`); base URL from `META_BASE_URL` (default `https://api.meta.ai/v1`). | /docs/models |
| Image input | base64 `data:` URL, png/jpeg/webp; 50 MB per image; max 50 images per request; ~1,300–1,500 tokens for a ~1280px image | /docs/image-understanding |
| Tool calling | OpenAI-style `tools` / `tool_calls`; parallel calls supported; `tool_choice` only `"auto"`; args not guaranteed to match the schema | /docs/tool-calling |
| Structured output | `response_format: { type: "json_schema" }`, constrained decoding; no recursive schemas; nesting depth ≤ 10 | /docs/structured-output |
| SAM 3.1 | `sam-3.1` through `/v1/responses`; box/point prompts as tokens in `input_text` (pixels); masks as base85 tokens; decode with `@meta-sam/parser` | /docs/sam/* |
| Voice | WAV PCM16 mono 16/24 kHz only (webm/m4a not documented) | /docs/speech-to-text |
| Rate limits | Spark 1.3: 3,000 RPM / 4M TPM. Credits: $150 per participant. | /docs/pricing-rate-limits |
| Vision latency | **Not published. Measure it.** | — |

### Latency budget

- In milestone 1, time one real vision call and one tool-call round trip.
- Per-model-call timeout: **45 s**. Total job budget: **3 min**. Hitting either shows a clear error card. No silent hangs.
- If a single call takes more than ~15 s, cut the retry cap from 3 to 2.

## Scope

### Focus: visual/UI bugs only

- The MVP handles **visual/UI bugs only** (layout, spacing, alignment, color, overflow, responsive issues). Visual verification adds value here that tests don't.
- If a request isn't visually verifiable (the pixel gate fails because nothing visible was supposed to change), the result card must say "not visually verified". Never report a false `fixed: true`.

### In scope (MVP, must work end to end)

- Mobile web UI: screenshot view, Desktop/Mobile viewport toggle, freehand selection with Draw/Move toggle, note input, live progress over SSE, result card with before/after + evidence + regression warnings + diff, Approve/Reject.
- Server: render, screenshot, locator (DOM + matched CSS rules), agent loop with sandboxed file tools, snapshot store, verifier, retry loop (max 3), cross-viewport regression check, timeouts.
- **Target from config:** `TARGET_URL` (default `http://localhost:3000`) and `TARGET_DIR` (default `./demo-app`). Nothing hardcodes `demo-app/`. It is tested on `demo-app/` only.
- **Demo target app:** Vite + React + **plain `.css` files** (no Tailwind, no CSS-in-JS, because they break the matched-rules locator) in `demo-app/`, with **4–5 planted visual bugs**:
  1. Misaligned buttons in a card footer.
  2. Wrong color token.
  3. Text overflow.
  4. Broken flex wrap at the Mobile viewport (the responsive bug for the demo).
  5. **The trap:** the obvious rule is overridden by a more specific rule elsewhere, so the naive edit changes nothing on screen. The pixel gate catches it, and the retry fixes the real rule. This makes the "failed attempt caught and retried" demo reproducible. Test that it fails on the first attempt most of the time.
- Auth: at startup the server creates a random token and prints the full URL (optionally as a QR code) in the terminal. The first visit swaps the token for an HttpOnly, Secure cookie. Every API route and SSE stream requires the cookie.
- File tools sandboxed to `TARGET_DIR`. No arbitrary shell commands.

### Planned after the MVP (Nov 4, 11 AM – 1 PM, hard 2-hour timebox)

- **Voice note via Muse Voice Transcribe** (see "Spoken notes"). Start only when all planted bugs pass. If it isn't working by 1 PM, drop it.

### Dependencies (approved)

- Server/UI: `playwright`, `hono`, `@hono/node-server`, `openai`, `diff`, `typescript`, `tsx`, `vitest`, `esbuild` (bundles the vanilla-TS phone UI).
- Demo app: `vite`, `@vitejs/plugin-react`, `react`, `react-dom`.
- Conditional: `@meta-sam/parser`, only if the SAM test passes.
- Any other dependency needs approval first.

### Cut line (decide at Nov 3, 8 PM)

If one bug is not fixed end to end by **Nov 3, 8 PM**, apply these cuts in order until back on schedule:

1. Freehand loop → drag a rectangle.
2. SSE → polling.
3. QR code → printed URL only.
4. SAM → dropped.
5. Voice → dropped.
6. Cross-viewport regression check → same-viewport outside-region diff only.
7. 5 planted bugs → 3 (keep the trap and the responsive bug).

**Never cut:** the verify loop, the trap-bug retry, Reject/revert, the sandbox.

### Stretch (only after voice is done and before code freeze)

- History of past fixes.

### Out of scope

- Non-visual bugs (logic, data, API, crashes). See "Why visual only" below.
- Multiple agent harnesses, routing between models, or integration with other CLIs.
- Support for arbitrary repositories in the wild. The target is configurable, but only `demo-app/` is tested.
- **A hosted public instance.** It would run an agent that edits files with my API key and credits, which brings cost and abuse risk, and deployment time I don't have. Judges get the video and a one-command local run. The README explains why.
- iOS Markup or screenshot uploads; phone _camera photos_ of a screen.
- Full-page (scrolling) screenshots.
- Interaction bugs that need clicks or state changes before the screenshot.
- Team or multi-user features.

### Why visual only

- The verify loop is the differentiator, and it only gives real evidence when the result is visible.
- A general bug fixer competes directly with established coding agents and loses on "whether it works".
- "Whether it works" rewards a narrow tool that works every time over a broad one that works sometimes.
- "What's next" in the README extends the same loop to anything that renders.

## Quality bar

- TypeScript strict mode. Simple, readable code over clever abstractions.
- Tests for the safety-critical paths:
  - File-tool sandbox: no path traversal, no escape through symlinks, malformed tool arguments rejected.
  - Snapshot store: Reject restores the exact prior bytes. Each retry starts from the originals.
  - Attempt loop: stops at max attempts; "no edit made" counts as a failed attempt; the job timeout fires.
  - Verifier: pixel gate short-circuits to `fixed: false`; a failed or truncated model response becomes `fixed: false`.
  - Coordinates: normalized → CSS pixels and → image pixels, for both presets.
  - Auth: requests without the cookie are rejected, SSE included.
  - Regression check: a change outside the region produces a warning; anti-aliasing noise below the threshold does not.
  - WAV encoder: correct header, sample rate and length for a known PCM input.
- Unit tests use a fake model client. No live API calls in tests.
- Clear errors in the UI when the model API fails or times out.
- No secrets in the repo. API key via env var `META_API_KEY`. Ship `.env.example` with a placeholder.

## Acceptance criteria

1. Every planted bug in `demo-app/` can be fixed through the phone flow with zero typing beyond a short note.
2. For at least 4 of 5 planted bugs, the verifier correctly reports `fixed: true` after the fix.
3. **No false positives:** for each planted bug, an attempt that changes nothing visible (or the wrong thing) gets `fixed: false`.
4. The trap bug's naive first attempt is caught by the verifier and triggers a retry that succeeds.
5. Reject restores the exact prior file state (byte-for-byte).
6. Selection works on iOS Safari at both viewport presets.
7. A planted change that also alters the other viewport (e.g. editing a shared rule) shows a regression warning on the result card.
8. If voice ships: a spoken note recorded in iOS Safari is transcribed into the note field.
9. A fresh clone runs with `npm install && npm start` plus `META_API_KEY`:
   - `postinstall` installs `demo-app/` dependencies and runs `playwright install chromium`.
   - `npm start` starts the demo app and the server.
   - The README lists cloudflared as the only extra prerequisite.

## Demo plan (3:00 max video)

**Recording setup:** don't use iPhone Mirroring (it means drawing with a trackpad, which undercuts the "from your phone" pitch).

- Connect the iPhone by cable → QuickTime → File → New Movie Recording → choose the iPhone as the camera source. This gives a live window of the phone screen.
- Place that window beside the demo app (open in a Mac browser, updating through Vite HMR) and the code diff, and record the whole Mac screen in a single take, drawing with a real finger on the phone.
- Optional: a 5-second real-camera shot of a hand circling a bug on the phone, as proof it's real.

**Script:**

1. Problem (15s): coding agents can't see whether their fix worked, and visual bugs are hard to describe. "Built during the hackathon" on screen.
2. Live demo (90s): circle the **trap bug** and say "align these buttons" → the agent's first edit has no visible effect → the verifier catches it ("no visible effect") → the retry fixes the real rule → the laptop app visibly changes → the before/after card with evidence lands on the phone → Approve.
3. Viewport toggle (20s): switch to Mobile and fix the flex-wrap bug without deploying. The card confirms the desktop layout is unchanged.
4. Who it's for (10s): "A designer can circle this too. The developer just approves the evidence."
5. How it works and what's hard (30s): asking the browser which CSS rule painted the pixel, stable rendering, the three-signal verifier, the regression check, which Meta models do what.

Total ≈ 2:45. **Hard limit 3:00.**

**Hygiene:**

- Don't cut waiting time. If slow, speed it up and label it on screen ("2× speed").
- Do a full dry run (tunnel, model latency, iOS Safari) before recording.

## Milestones (Nov 3–4, ET)

| By            | Milestone                                                                                                                                                                                                 |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Nov 3, 11 AM  | API credits claimed and a card added (blocks every API call). Questions posted in #office-hours before it closes at 11 AM.                                                                                |
| Nov 3, 2 PM   | API calls verified (vision + tool calling + json_schema) and latency measured. Demo app with planted bugs. Playwright screenshots at both viewports. Vite matched-rules mapping checked. SAM 30-min test done. |
| Nov 3, 8 PM   | Phone selection → locator → agent loop fixes one bug end to end (no verifier yet). **Cut-line decision.**                                                                                                 |
| Nov 4, 2 AM   | Verifier (all three signals) + attempt loop with snapshot revert. Before/after result card on mobile.                                                                                                     |
| Nov 4, 11 AM  | All planted bugs pass, including the trap retry. Cross-viewport regression check. Safety-path tests. Approve/Reject. Auth.                                                                               |
| Nov 4, 1 PM   | Voice notes working on iOS Safari, or dropped (hard timebox).                                                                                                                                             |
| Nov 4, 3 PM   | **Code freeze.** Stretch goals only if everything above is done.                                                                                                                                          |
| Nov 4, 3–6 PM | Dry run, record the demo, write the README and the submission text (mention Best Agentic Use-Case).                                                                                                       |
| Nov 4, 8 PM   | Repo made public (history checked for secrets). Video link checked from a logged-out browser.                                                                                                             |
| Nov 4, 9 PM   | Submitted on the CV platform: public repo, README, 3-min demo video.                                                                                                                                      |
| After submit  | Post the demo on X; submit the post link for Most Viral (engagement counts until Nov 8) and share it for the Community Vote.                                                                             |

## README must include (for judges)

- One-line pitch led by the verify loop, a GIF of the circle → verified-fix flow, and setup steps (including the tunnel).
- A "Built during the hackathon" statement: all code was written Nov 3–4, 2026 (see the git history). No code from earlier projects.
- Exactly which Meta models are used, and for what (Muse Spark 1.3: agent, vision verifier and regression judge; Muse Voice Transcribe: spoken notes, if shipped; SAM 3.1: target outline, if kept, or why it was dropped).
- A short "what's hard about this" section: region-to-rule mapping through CDP, stable rendering for comparison, the three-signal verifier, the cross-viewport regression check, and why the agent never grades itself.
- An honest list of limitations (viewport-only screenshots, tested on `demo-app/` only, plain CSS only, no interaction bugs, no hosted instance and why).
- Config: `META_API_KEY` (required), `META_MODEL`, `META_BASE_URL`, `TARGET_URL`, `TARGET_DIR`.
- "What's next": the same see → act → verify loop applies to any visual output, such as charts, dashboards, email templates, generated PDFs and canvas rendering.
