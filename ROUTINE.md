# Retain Agency - Video Automation - ROUTINE (canonical policy/architecture)

This file is the single source of truth for the video production + publishing
pipeline for the Retain Agency Facebook Page. Any cloud routine prompt that
references this repo should treat this file (and RETAIN-COMPLIANCE-GATE.md) as
authoritative over its own embedded summary of the steps.

## Goal

Produce and publish a short vertical (1080x1920) faceless-explainer video
sourced from real, recent Vietnamese-market marketing/advertising news, fully
automated, 2 times per day (07:00 / 19:00 Asia/Ho_Chi_Minh), published to BOTH
the Retain Agency Facebook Page (Reel) AND the Retain Agency YouTube channel
(Short), with no human in the loop for a routine run that passes all gates.

**Schedule change (2026-09-15):** this replaces the earlier 3x/day plan (07:00
/ 12:00 / 19:00, 2 pillar-education slots + 1 news slot, Facebook only). The
pillar-education rotation is retired for this routine; every slot is now
news-mode. `video-content-plan-state.json`'s `pillars_rotation` /
`next_pillar_index` fields are frozen as of this change (kept in the file for
history, no longer advanced) - only `topics_used`/`posts_log` and the new
`youtube_posts_log`-equivalent tracking stay live. See "YouTube publishing"
below for the setup this change depends on before it can go live for real.

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
  - `publish_youtube` - **NOT YET BUILT (blocked on OAuth setup, see "YouTube
    publishing" section below).** Once credentials exist, mirrors the same
    pattern as Tram AI/Tin Tuc So's YouTube action: `videos.insert` (Data API
    v3, `multipart/form-data` resumable-ish upload via `UrlFetchApp`) using a
    refresh token stored in this script's own Script Properties
    (`YT_CLIENT_ID`, `YT_CLIENT_SECRET`, `YT_REFRESH_TOKEN`, `YT_CHANNEL_ID`),
    exchanged for a short-lived access token at the start of every call (do
    not cache the access token across routine runs). Body shape (planned):
    `{"action":"publish_youtube","video_url":"<raw github url>","title":"...","description":"...","tags":[...],"video":"<slug>"}`.
    Same `video_posts_log` Sheet tab records both platforms' post IDs for one
    `video` slug (two columns: `facebook_post_id`, `youtube_video_id`) so a
    partial failure (one platform succeeds, the other fails) is visible and
    not silently retried as a full duplicate.
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

## Content rotation (news-mode only, both daily slots - changed 2026-09-15)

State lives in `video-content-plan-state.json` in this repo (NOT on Quang's
Mac - the cloud routine has no access to `/Users/quang/Desktop/Retain Agency/`).
Read it, decide, then git commit the updated state back at the end of a
successful run (mirrors how COMPLIANCE.md / media files are committed).

- **Both the 07:00 and 19:00 slots run news mode.** There is no pillar slot
  anymore - `pillars_rotation`/`next_pillar_index` are frozen (kept for
  history only, see the Goal section's 2026-09-15 note).
- WebSearch for one real, recent (last few days, prefer last 24-48h so the
  07:00 and 19:00 runs on the same day do not converge on the same story)
  piece of Vietnamese-market marketing/advertising news: Meta/Facebook Ads
  platform changes, TikTok Ads, Google Ads Vietnam, e-commerce advertising
  trends, a notable Vietnamese brand campaign, or a relevant regulatory/policy
  update (e.g. Nghi dinh quang cao, data-privacy rules affecting ad targeting)
  - prefer sources with clear local relevance: Vietnamese tech/marketing
  outlets (Advertising Vietnam, Brands Vietnam, Vietcetera, ICTnews, VnExpress
  Kinh doanh) or the Vietnamese coverage of a global platform announcement.
  Do not invent a platform change, statistic, or trend - every factual claim
  in the script must trace to the sourced article.
- Cross-check against `topics_used` AND the last 3-4 days of `posts_log` for
  both slots so the 07:00 and 19:00 videos on the same day, and consecutive
  days, do not restate the same story - if the only strong candidate story was
  already covered within the last 3 days, either find a distinct angle on it
  (not just a restatement - see the originality/B7 compliance clause) or pick
  a second, less prominent but still real story instead of skipping the slot.
- Apply the same Gate A/B screening as before, PLUS the two clauses added to
  `RETAIN-COMPLIANCE-GATE.md` on 2026-09-15 specifically for this news-only
  mode: AI/synthetic-media handling (real photo with credit or explicit
  concept illustration, never a fabricated photorealistic recreation of a real
  event/person) and originality (must add a distinct angle, not just restate
  the headline).
- After a successful publish, append to `topics_used`/`posts_log` (recording
  BOTH `facebook_post_id` and `youtube_video_id` for the run, or whichever one
  succeeded if the other platform failed - see "YouTube publishing" below),
  increment `next_video_number`, and commit `video-content-plan-state.json`
  back to this repo (`git pull --rebase` first to avoid races between the two
  daily runs).

## YouTube publishing (added 2026-09-15, DONE as of 2026-09-15 - action built, tested, live)

`publish_youtube` is built and smoke-tested (test upload confirmed on the
correct "Retain Agency" channel, Unlisted, then deleted by Quang). Script
Properties `YT_CLIENT_ID`, `YT_CLIENT_SECRET`, `YT_REFRESH_TOKEN` are set on
the "Retain Agency - Theo doi Fanpage" Apps Script project (same OAuth
consent screen as Tram AI's own YouTube action, status "En production" - no
Testing-mode 7-day refresh-token expiry to worry about for this app).

Body shape (implemented): `{"action":"publish_youtube","video_url":"<raw
github url>","title":"...","description":"...","tags":[...],"video":"<slug>",
"privacy":"public"}` - **always pass `"privacy":"public"` explicitly for a
real routine run** (the action's own default, if omitted, is `unlisted` -
that default exists only so a manual test call can't accidentally go public;
every real routine publish must set it explicitly or the video stays hidden
forever). Returns `{"youtube_video_id":"...","youtube_url":"..."}`.

The two-platform cron IS live (both daily routines call `publish_facebook`
AND `publish_youtube` for the same render). See "Routine environment
bootstrap" below for the one real gap this surfaced on first run: the cloud
container has no HyperFrames tooling pre-installed.

## Routine environment bootstrap (added 2026-09-16, root-caused a failed run)

**The RemoteTrigger cloud container starts with NOTHING HyperFrames-related
installed** - no `hyperframes-creative`/`hyperframes-audio` skills, no GSAP,
no project scaffold, not even the `hyperframes` CLI itself. This is normal
(it is a fresh sandbox every run, per-run installs are expected, same as
Tram AI's own routine), but a routine prompt that only says "dung video bang
HyperFrames" in prose - trusting the agent to already know how - will
correctly conclude the tooling is missing and STOP rather than install it.
This happened on the first real 19:00 run (2026-09-15): the agent read this
file, correctly identified every locked-in design rule, but never attempted
`npx hyperframes` because nothing told it to.

**Fix: every routine prompt must explicitly spell out the bootstrap
commands**, not just reference the workflow by name:
```bash
npx hyperframes init "/tmp/<slug>" --non-interactive --example=blank --skill=faceless-explainer
```
If that reports the skill is missing/stale: `npx hyperframes skills update
faceless-explainer` first, then retry `init`. Build the actual project under
`/tmp/<slug>` (NOT inside the git-cloned repo directory - the repo only
receives the FINAL rendered `video.mp4`/`thumbnail.jpg`, copied over after
render, per "Infrastructure" above). After `npm run render`, copy
`/tmp/<slug>/renders/{video.mp4,thumbnail.jpg}` into this repo's
`videos/<slug>/`, then commit/push/verify-200/publish as already documented.


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


## Lessons imported from the "Tin Tuc So" sibling channel (2026-09-15)

A full technical guide for a sibling news channel ("Tin Tuc So", repo `quangnv-cloud/tin-tuc-so`,
8 real videos analyzed) was reviewed against this project. Same brand architecture (Montserrat
latin+vietnamese self-host, 7-act vertical 1080x1920, GSAP timeline-per-frame, HyperFrames). Where
their proven practice is stricter or more mature than what Retain currently does, adopt it going
forward - these are now standing rules, not suggestions to revisit later.

### No emoji, ever - CSS/SVG shapes only

The first Retain video used Unicode glyphs for icons (`&#128172;` speech-bubble, `&#10003;`
checkmark) in the CTA frame. These happened to render correctly on this Mac's local Chromium
render because a system emoji font was present - but a bare cloud render sandbox (where the
eventual RemoteTrigger routine runs) may have no emoji-capable font installed at all, silently
dropping the glyph. Tin Tuc So's rule, proven across 9 real projects with zero exceptions: **build
every icon as a CSS shape or inline SVG**, never an emoji character. Reusable recipes:
- Checkmark: a rotated `::after` box using `border-top` + `border-right` (see the "up" icon in
  their CTA template), or a short 2-segment SVG path.
- Speech bubble (comment CTA icon): a rounded-corner square with one corner sharp
  (`border-radius: 12px 12px 12px 4px` - the sharp corner reads as the bubble's tail) plus 2 short
  horizontal bars (`::before` + `box-shadow`) simulating lines of text.
- Warning triangle: pure CSS triangle (`border-left`/`border-right` transparent + `border-bottom`
  solid) with a `::after` "!" centered on it.
Retroactively replace the two emoji glyphs already in `07-cta.html` with CSS equivalents before
the next render.

### Brand Anchor (logo + name) belongs at the ROOT composition, not duplicated per frame

Retain's first video nests a small masthead (`<img class="f{N}-wm-logo">` + "RETAIN" text) inside
EVERY frame file (2 through 7) - 6 copies of near-identical markup, each re-running its own fade-in
reveal on every scene cut. Tin Tuc So's proven pattern is architecturally simpler and more correct:
one `#brand-anchor` block lives directly in `index.html` (a sibling of the 7 `.clip`/`.scene` act
divs), starts at `opacity: 0`, and the ONLY animation on it is a single
`tl.set('#brand-anchor', { opacity: 1 }, <hook's data-duration>)` in the root script - it appears
once, right when the Hook frame ends, and stays visible unchanged for the rest of the video. Adopt
this for the next Retain video: move the logo+wordmark (small corner) and the "Nguon: ..."-style
badge (if Retain ever needs one) into `index.html` itself, delete the per-frame duplicates, and
keep only the Hook frame's own full masthead treatment (which legitimately needs its own build-in
reveal, per the existing Hook design) and the CTA frame's closing full-size signature (which is a
deliberate SECOND, later brand beat, not the same element).

### Vendor GSAP locally, do not rely on a CDN `<script src>`

Every one of Tin Tuc So's 9 real projects (including their reference template) ships
`assets/vendor/gsap.min.js` - copied from `node_modules/gsap/dist/gsap.min.js` after `npm i gsap`
- specifically because a common CDN host is blocked at their cloud sandbox's network egress. Retain
currently loads GSAP from `cdnjs.cloudflare.com` via `<script src>` in every frame file. This has
worked fine locally and on Tram AI's cloud routine so far, but it is an unverified assumption for
Retain's own future RemoteTrigger routine - vendor GSAP locally (`assets/vendor/gsap.min.js`,
committed to the repo) before that routine goes live, removing the external-network dependency
entirely rather than hoping the CDN stays reachable.

### Vertical-fill QC: measure pixels, do not eyeball a compressed screenshot

The existing "no empty bottom third" rule stays, but the verification METHOD upgrades. Tin Tuc So
hit two real bugs that eyeballing missed or mis-caught:
1. A ring-progress data frame was genuinely too small (620px) and left the bottom ~35% empty -
   only caught by measuring the actual pixel row of the last visible content against the
   background color on a frame extracted from the RENDERED .mp4 (not the Studio preview thumbnail).
2. A leaderboard frame's content looked fine on paper but its GSAP reveal hadn't finished by the
   sample timestamp (sampled at act-MIDDLE), so the extracted frame under-reported real content -
   fixed by re-sampling near the act's END instead, and separately, a suspected clipped closing
   logo on the CTA frame turned out to be a false alarm caused by a compressed preview image, only
   resolved by measuring the real render with PIL.
**New standing QC step**: when checking vertical fill on any future Retain frame, extract the
sample from a timestamp near the END of that frame's own `data-duration` (not the middle), and when
in doubt whether content reaches the safe zone, measure it (a quick Python/PIL script scanning rows
against the `#121212` background) rather than trusting a compressed screenshot by eye. Target for
Retain's 1920px canvas: last content element's bottom edge lands between roughly 73% and 87% of
frame height (matches Tin Tuc So's observed 1400-1680px band on their own 1920px canvas).

### BGM mood: prefer genuinely calm/ambient, do not fight a busy track with volume alone

Tin Tuc So's own brand rule exists BECAUSE of user feedback identical to what Retain's user gave on
the first BGM choice ("too much rhythm, overpowers the voice"): their fix was not a lower volume on
a busy track, it was picking a fundamentally calmer track at the SOURCE. Their Lyria prompt:
`"calm ambient news underscore, soft synth pads, sparse, minimal pulse, no drums, instrumental
only"`, with a mandatory negative-prompt excluding `drums, heavy beat, aggressive percussion, busy
rhythm, loud, driving, energetic, buildup, drop` (in addition to vocals/lyrics). Retain reused Tram
AI's energetic `news-broadcast.mp3` and had to compress + drop its mix volume to 0.06-0.08 to make
it work - functional, but fighting the wrong track. **For the next Retain video's BGM, prefer a
genuinely sparse/ambient bed with no drum hits** (generate via Lyria with the negative-prompt above
once `GEMINI_API_KEY` is available in the build environment, or source a comparably calm track)
rather than reusing an energetic bed and taming it after the fact.

### Wire up `carve.mjs` for real ducking - stop treating it as a future TODO

Tin Tuc So runs this in production on every video, not as an aspiration:
```
node ~/.claude/skills/hyperframes-audio/scripts/carve.mjs --comp index.html --strength 0.4
```
(lower than the tool's own default of 0.8, because their BGM is already mixed quiet - the same
logic applies to Retain's quiet 0.06-0.08 BGM). This writes `data-fx-carve` / `data-fx-chain` /
`data-automation` onto the `<audio id="...bgm...">` element automatically - never hand-write those
3 attributes. It requires every voice `<audio>` to carry `data-audio-group="voiceover"` (already
straightforward to add). **Run carve.mjs after every timing or audio change** - a stale carve
result desyncs from the new timing. Use this instead of (or in addition to) the interim flat-volume
fix on the next Retain video.

### Animate focal numbers with a count-up, even inside a bar/leaderboard layout

Retain's "Support" frame (CTR/CPC bar comparison) shows "+27%"/"-26%" as static text next to
animated bar fills. Tin Tuc So's convention, observed with zero exceptions across every "data
moment" act regardless of visual metaphor (single big number, leaderboard, ring, line-chart): the
number itself is ALWAYS a GSAP-tweened plain JS object (`{v: 0}`, never tweening a DOM element
directly) with `onUpdate` writing `Math.round(v).toLocaleString('vi-VN')` into the target span -
even when it sits next to a bar/track that is separately animating its own fill. Add this to the
next Retain video's focal stat number(s) for a livelier data moment, not just static text beside a
moving bar.

### Compliance gate additions worth mirroring for Retain's news-mode slot

`RETAIN-COMPLIANCE-GATE.md` currently covers the banned-phrase list, case-study labeling, and tone.
Tin Tuc So's gate (same news-sourcing risk profile as Retain's news-mode slot) adds two clauses
Retain's doc does not yet have - both apply directly whenever a Retain video sources a real
external news item:
- **AI/synthetic media**: an AI-generated image may illustrate a concept (a diagram, an icon, an
  abstract graphic) freely, but must NEVER recreate a real event or a real person photorealistically
  as if it were an actual photograph of that event - use a real photo (with credit) or an explicit
  concept illustration instead, never a fabricated "photo".
- **Originality / not mass-produced**: every video must carry at least one distinct angle, analysis,
  or presentation choice - never just restate a headline. This is flagged as the single highest
  platform-policy risk for any automated multi-video-per-day channel (Meta/YouTube "inauthentic /
  mass-produced" detection).
Add both clauses to `RETAIN-COMPLIANCE-GATE.md` before the news-mode slot starts running
unattended.
