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
    default voice changed 2026-09-10 per explicit user request to match
    Tram AI's voice - "Khanh Lam" (northern accent), `RCmOaM1iiIH5xX3QXjIF`,
    the SAME voice_id as Tram AI, same ElevenLabs API key. (Originally
    launched with a separate Retain-only voice, "Thanh Ngoc - Warm & Trusted
    Expert" / `Na15FlRRkMEDtEW4nVVP` - no longer used; pass `voice_id`
    explicitly in every `generate_narration` call rather than relying on a
    server-side default.) Body: `{"action":"generate_narration","text":"...","voice_id":"RCmOaM1iiIH5xX3QXjIF"}`.
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
  font **Montserrat** (changed 2026-09-10 from Public Sans - see "Font" below
  for why). Logo lives in this repo at `assets/logo.png` (copied from
  `/Users/quang/Desktop/Retain Agency/Logo.png` - the routine cannot read
  Quang's Mac, so it must use the repo copy).
- Pick a HyperFrames preset that fits a dark, premium/professional look
  (browse `hyperframes-creative` frame-presets at build time - do not
  hardcode one here; `blue-professional` is Tram AI's look, not Retain's).
- Structure: **7 visual frames** (changed 2026-09-10 - see "Hook pacing"
  below): a dedicated Hook frame (~5-7s) always precedes the Context frame,
  even though both can share one continuous voice take.
- Apply the same 4 design upgrades proven on Tram AI, adapted to Retain's
  content types:
  1. Kicker (short all-caps label) on every frame naming its role in the
     narrative.
  2. Small fixed masthead watermark (Retain logo + wordmark) top-right from
     frame 2 onward; a full-size masthead reappears at the closing/CTA frame.
     **Always the real `assets/logo.png` `<img>`, never a hand-drawn CSS
     substitute** - see "Logo" below.
  3. If (and only if) a real photo is available (typically only in news-mode,
     from the sourced article) - reuse it in exactly one additional content
     frame as a small rounded card with a photo credit, beyond frame 1. When
     no real photo/HeyGen access is available, prefer a simple invented
     chart/icon (bar comparison, count-up stat, small line icon) over dense
     paragraphs of text on any frame presenting a statistic.
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

### Font (locked in 2026-09-10, do not revert to a naive fetch)

The first published video shipped with visibly broken Vietnamese diacritics
(acute/grave/circumflex marks rendering as a detached box/flag shape on
letters like "tot", "ve"). Root cause: `@font-face` was self-hosted from a
Google Fonts `css2` fetch made with a **modern** `curl` User-Agent, which
returns ONE merged variable-font woff2 file covering every weight but only
the `latin` unicode-range subset - the `vietnamese` subset block (needed for
precomposed characters like U+1EA0-1EF9) is silently missing, so the browser
falls back to a different font for accented glyphs mid-word. `npm run check`
does not catch this (it is not a lint-detectable failure - the font "loads"
fine, it just doesn't cover the codepoints used).

**Fix, now standard for every Retain video:**
1. Fetch with an OLD Chrome User-Agent (e.g. `Mozilla/5.0 (Windows NT 6.1)
   AppleWebKit/537.36 (KHTML, like Gecko) Chrome/55.0.2883.87 Safari/537.36`)
   against `https://fonts.googleapis.com/css2?family=Montserrat:wght@400;500;600;700;800&display=swap`
   - this returns separate static files per weight AND per subset
   (`latin`, `latin-ext`, `vietnamese`, `cyrillic`, ...).
2. Keep only the `latin` and `vietnamese` blocks per weight (10 blocks for 5
   weights), download each woff2, self-host under `assets/fonts/`.
3. Write all 10 `@font-face` rules (same `font-family`/`weight`, each with its
   own `unicode-range` copied verbatim from the fetched CSS) into every frame
   file. Never drop the `unicode-range` line - without it the two rules for
   the same weight conflict and only the last-declared one wins for every
   character.
4. Reference fonts with a project-root-relative path (`assets/fonts/...`),
   consistent with how `hyperframes-core` resolves all asset paths.
5. Visually zoom into rendered frames containing Vietnamese tone marks
   (acute/grave/hook/tilde/dot-below) and circumflex/breve letters before
   considering a font swap done - this class of bug is invisible in
   plain-text review of the HTML source.

Font family is now **Montserrat** (matches Bot Ban Hang / Tram AI's proven
system) instead of Public Sans - do this same latin+vietnamese self-host
procedure for whatever font a future rebrand picks; the procedure is the
point, not the specific family name.

### Logo (locked in 2026-09-10)

Never build the Retain mark out of CSS (a rounded square + letter "R", a
colored dot, etc.) as a stand-in "for now" - the first published video did
exactly this, and it visibly does not match the real hexagon mark (black
fill, gold border, gold serif "R") once a side-by-side comparison is made.
Always render the actual file: `<img src="assets/logo.png" alt="Retain
Agency" />` at whatever size the masthead slot needs (small corner ~24px,
closing masthead ~64px), `object-fit: contain`, small `border-radius` to
soften the corner without clipping the hexagon. There is no acceptable
placeholder for a real, already-available brand asset - if the logo file is
ever missing from the repo, stop and flag it rather than drawing a
substitute.

### Hook pacing (locked in 2026-09-10, overrides the earlier "merged, no
dedicated Hook frame" note above for any hook+context pair longer than ~10s)

The first published video merged the Hook line (script line 1) into Frame
1's audio with Context (script line 2), producing one 18s frame with a
single static reveal, then nothing changing on screen for the remaining
~17s while the voiceover kept talking. This reads as a dead/frozen opening
and was the single biggest viewer-facing complaint on the first publish.

**Fix:** always give the Hook its own frame (target 5-7s), separate from
Context, even when both lines are read as one continuous ElevenLabs take.
Do NOT re-run TTS twice or introduce a pause in the read - cut the single
audio file into two pieces at the natural sentence-boundary timestamp
(available for free from the `/v1/text-to-speech/{id}/with-timestamps`
character-level alignment already used for the single-continuous-take
method) with ~0.05-0.12s of padding on each side. The voice stays one
uninterrupted take; only the on-screen frame changes at that boundary. If a
future script's Hook+Context naturally reads under ~10s total, merging into
one frame remains acceptable (matches the original Tram AI finding); split
whenever the combined read is materially longer than that.

### BGM level (locked in 2026-09-10, tighter than the general Tram AI note)

The Tram AI-wide BGM guidance ("~0.11 relative volume, 0.06 too quiet, 0.12+
risks overpowering on a busier track" - see the equivalent Tram AI
production-rules note) undersells the risk for a track like
`news-broadcast.mp3`, which has real drum/bass transient hits reaching
0dB even though its long-run average loudness (`ffmpeg -af volumedetect`)
measures the SAME as the narration's average loudness. A flat 0.13 gain
(within the "documented safe" range) was still reported as overpowering the
voice on the first publish - the transient peaks punch through a simple
linear-gain mix even when the smoothed average sits far enough below the
voice.

**Fix:** for any bed with audible drum/bass hits (not a smooth pad/ambient
bed), (1) lightly compress/limit the bed file itself before mixing
(`ffmpeg -af "acompressor=threshold=-18dB:ratio=6:attack=8:release=250,alimiter=limit=-6dB"`)
to tame the transients, and (2) keep the `audio_meta.json` bgm `volume` at
**0.06-0.08**, not the general 0.11 figure. `/hyperframes-audio`'s
voiceover-carve mechanism (`data-fx-carve` on the bed track, sidechained to
the voice) is the more correct long-term fix for this exact "busy bed fights
voice" problem and is worth wiring in properly for a future revision - a
lower static volume plus light compression is the interim fix, not the
final word on it.

### Text-box height (reinforcing the existing Layer-1 rule below with a concrete miss)

A headline container height was copy-pasted from an earlier single-line
design (130px) onto a new two-line headline at the same font-size, and
`npm run check`'s layout pass did NOT catch it - the checker flags a text
box overflowing its OWN declared bounds in some cases, but does not reliably
catch that overflowing text visually collides with a DIFFERENT sibling
element positioned below it (here: the headline's second line rendering on
top of a card's rounded border). Whenever a headline's line-count is
uncertain (switching fonts, tightening/loosening copy, changing font-size),
recompute its container height against the FORMULA below for the ACTUAL
render, and always extract a real screenshot of that specific frame's
settled state to confirm the next element below it has clear space - do not
trust "the lint check passed" as proof there is no overlap with a sibling.

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
