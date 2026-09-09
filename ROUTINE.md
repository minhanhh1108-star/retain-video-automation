# Retain Agency - Video Automation - ROUTINE (canonical policy/architecture)

This file is the single source of truth for the video production + publishing
pipeline for the Retain Agency Facebook Page. Any cloud routine prompt that
references this repo should treat this file (and RETAIN-COMPLIANCE-GATE.md) as
authoritative over its own embedded summary of the steps.

## Goal

Produce and publish a short vertical (1080x1920) faceless-explainer video as a
Facebook Reel on the Retain Agency Page, fully automated, 3 times per day
(07:00 / 12:00 / 19:00 Asia/Ho_Chi_Minh), with no human in the loop for a
routine run that passes all gates.

## Infrastructure (separate from Tram AI on purpose)

- Video/thumbnail hosting: this repo (`retain-video-automation`, public) at
  `videos/<slug>/{video.mp4,thumbnail.jpg}`, fetched by Facebook via GitHub
  raw URL (`https://raw.githubusercontent.com/minhanhh1108-star/retain-video-automation/main/videos/<slug>/...`).
- Publishing + narration proxy: Apps Script project "Retain Agency - Theo doi
  Fanpage" (the SAME project that already runs `publishQueue_`/
  `checkQueueHealth_` for the image pipeline) - a `doPost`/`doGet` Web App was
  added on top of it (2026-09-09), namespaced with a `video_` prefix on every
  new function/variable so it can never collide with the existing image-queue
  code. Actions:
  - `generate_narration` - ElevenLabs TTS proxy (same API key as Tram AI,
    different default voice: "Thanh Ngoc - Warm & Trusted Expert",
    `Na15FlRRkMEDtEW4nVVP`). Body: `{"action":"generate_narration","text":"..."}`.
  - `publish_facebook` - publishes a Facebook Reel via the Graph API
    `video_reels` 3-phase flow (start -> upload bytes -> finish), reusing the
    EXISTING `getConfig_()`/`getPageToken_()` helpers already in that script
    (no separate Facebook credentials for video). Body:
    `{"action":"publish_facebook","video_url":"<raw github url>","caption":"...","thumbnail_url":"<raw github url, optional>","title":"...","video":"<slug>"}`.
    Dedup/log lives in a new Sheet tab `video_posts_log` inside the SAME bound
    Spreadsheet ("Retain Agency - Theo doi Fanpage") - separate from
    `Hang cho dang`/`Bai dang` used by the image pipeline.
- Same calling convention as every Apps Script Web App: POST returns a 302
  redirect even on success - read the `Location` header, then GET that URL to
  get the real JSON. Example:
  ```bash
  EXEC="<retain video exec url>"
  curl -sS -D /tmp/h.txt -o /dev/null -X POST "$EXEC" -H "Content-Type: application/json" -d '<json body>'
  LOC=$(grep -i '^location:' /tmp/h.txt | sed 's/^location: //I' | tr -d '\r')
  curl -sS "$LOC"
  ```
- Do NOT use the image-pipeline's Sheet queue (`Hang cho dang`) or Drive
  folder for video - video publishes directly and immediately
  (`published`-equivalent via `video_state: PUBLISHED`), it does not go
  through the 29-day-ahead scheduling limit workaround the image pipeline
  uses. There is no pre-scheduling for video; each routine run produces and
  posts the same run.

## Content rotation (2 pillar slots + 1 news slot per day)

State lives in `video-content-plan-state.json` in this repo (NOT on Quang's
Mac - the cloud routine has no access to `/Users/quang/Desktop/Retain Agency/`).
Read it, decide, then git commit the updated state back at the end of a
successful run (mirrors how COMPLIANCE.md / media files are committed).

- **07:00 and 12:00 slots -> pillar mode.** Take the next pillar in
  `pillars_rotation` at `next_pillar_index` (wrap around after F). Pick one
  topic not yet in `topics_used` (and, as a soft check, not recently used in
  `recent_image_topics_snapshot_2026_09_09` either, to avoid the image and
  video feeds feeling repetitive in the same week) - for `A_thuat_ngu_nang_cao`
  pull from `glossary_upcoming_suggestions` (move it to `glossary_done` after
  use); for `B_cap_nhat_nen_tang` and `F_xu_huong`, WebSearch is REQUIRED (real,
  recent Meta Ads / Facebook Marketing API news) - do not invent a platform
  change or trend; for `D_case_study`, the video MUST state on-screen and in
  the script that it is an illustrative example, not a real client's numbers
  (same rule as the image pipeline).
- **19:00 slot -> news mode.** WebSearch for one real, recent (last few days)
  piece of digital-marketing / Meta Ads / Facebook advertising news (English
  or Vietnamese sources both fine - prefer Meta Business newsroom, Search
  Engine Journal, Social Media Today, or a Vietnamese tech/marketing outlet if
  directly relevant). Apply the same Gate A/B screening as pillar content
  before committing to a topic.
- After a successful publish, append to `topics_used`/`posts_log`, advance
  `next_pillar_index` (pillar slots only - news-mode runs do not consume a
  pillar slot), increment `next_video_number`, and commit
  `video-content-plan-state.json` back to this repo (`git pull --rebase` first
  to avoid races between the three daily runs).

## Compliance

See `RETAIN-COMPLIANCE-GATE.md` for the hard rules (banned phrase, no fake
guarantees, case-study labeling, tone). Apply a Gate B self-check (JSON
decision, same shape as any compliance gate: `{decision, risk_level,
violations, claims_to_verify, reason}`) on the script before narration/build,
exactly like the image pipeline's editorial rules and like Tram AI's Gate B.
GREEN/YELLOW + APPROVE -> continue. ORANGE/RED/BLACK or REWRITE-that-can't-be-
auto-fixed -> STOP, do not narrate/build/publish, note it in the run summary.

## Video design (HyperFrames)

- Brand tokens (`tokens.json`): colors `["#121212","#EED688","#F68822","#FFFFFF"]`,
  font `Public Sans`. Logo lives in this repo at `assets/logo.png` (copied
  from `/Users/quang/Desktop/Retain Agency/Logo.png` - the routine cannot read
  Quang's Mac, so it must use the repo copy).
- Pick a HyperFrames preset that fits a dark, premium/professional look
  (browse `hyperframes-creative` frame-presets at build time - do not
  hardcode one here; `blue-professional` is Tram AI's look, not Retain's).
- Structure: 6 visual frames, Hook (script line 1) is voice-only and merged
  into frame 1's audio (`ffmpeg concat` -> one intro audio file), no dedicated
  Hook frame - same technique validated on Tram AI to avoid a dead/empty
  opening 2-3s.
- Apply the same 4 design upgrades proven on Tram AI, adapted to Retain's
  content types:
  1. Kicker (short all-caps label) on every frame naming its role in the
     narrative.
  2. Small fixed masthead watermark (Retain logo + wordmark) top-right from
     frame 2 onward; a full-size masthead reappears at the closing/CTA frame.
  3. If (and only if) a real photo is available (typically only in news-mode,
     from the sourced article) - reuse it in exactly one additional content
     frame as a small rounded card with a photo credit, beyond frame 1.
  4. Any frame with a central statistic (a percentage, a formula result, a
     cost figure) animates it with a GSAP count-up tween - never a duplicate
     giant faint background numeral behind it (tried once on Tram AI, reads
     as a bug/duplication - do not repeat that mistake).
- CTA frame: adapt the "hieu ung chon lua" structure (eyebrow + question
  headline + 2 opposing cards + pill CTA button + closing masthead) to
  Retain's voice - e.g. contrasting a common mistake vs. the correct
  approach for a tip/myth-busting topic, or "dung/sai" framing for glossary.
- Facebook caption posted alongside the video: short title + 2-4 sentences +
  the Retain standard sign-off block (Hotline/Zalo/Website/Email - see
  `RETAIN-COMPLIANCE-GATE.md`).

## QC (embed the 9-step process proven on Tram AI, in full)

Layer 1 (while writing HTML): a coordinate-positioned child element MUST be
nested inside its actual parent panel in the HTML - never a sibling with a
hand-added offset (this exact bug caused text-over-photo overlap once). Any
text block that can wrap to >=2 lines gets a container height of
`(expected_lines + 1) x (font_px x line_height)`, never an eyeballed guess,
with >=24px clearance to the next block. `grep '<div id='` after each frame
with a split panel to confirm correct nesting before moving on.

Layer 2 (after render, extended Gate D): `npm run check` must be clean.
Extract at least 3 timestamps per frame that has >=2 stacked blocks (start,
mid-reveal, end), not just 1 frame-level screenshot. Before declaring PASS,
ask: "if I were a first-time viewer, is anything overlapping?" - only answer
yes to PASS after actually looking closely at every extracted frame.

Layer 3 (when a QC issue IS found): fix every frame with a similar risk in
the same pass, not one at a time waiting for the next issue to surface. Re-
read the edited section after any multi-line Edit/Write before moving to the
next frame.

## Safety / stop conditions

Do not publish if: Gate B is not GREEN/YELLOW-APPROVE, Gate D fails (silence
detected + transcript mismatch, or a visible overlap/contrast bug found by
the 8.5d image QC), or the GitHub raw URL does not return HTTP 200 after one
30s retry. In every stop case, write a clear reason to the run summary and do
not touch `video-content-plan-state.json`'s rotation counters - leave them
exactly as they were so the next run retries the same slot cleanly.
