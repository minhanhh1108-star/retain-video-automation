# Retain Agency Video - Compliance Gate

Adapted from the rules already enforced on the image pipeline
(`retain-agency-design` skill, `facebook-page-post-scheduler` skill's
`retain-agency-config.md`) and from Tram AI's Gate A/B pattern. Applies to
every video script before narration/build.

## Gate A - topic screening (apply before committing to a topic)

Reject a candidate topic outright, no exceptions, if it:
- Requires stating or implying "cho thue tai khoan quang cao" / "ban tai
  khoan quang cao Facebook" (renting/selling ad accounts) in any public-facing
  text (script, on-screen text, or caption). This is a Facebook-prohibited
  practice; stating it publicly risks the Page being flagged. Use only
  indirect phrasing about the service, e.g. "giai phap quang cao Facebook",
  "dong hanh cung quang cao cua ban", "tu van & toi uu quang cao Facebook".
  The real phrase stays internal-only (this repo, chat with Quang), never in
  published content.
- Covers "Nick Facebook / Via" or "Clone Facebook" content (workarounds for
  platform restrictions using fake paperwork) - these two topics are
  explicitly off-limits for this Page (policy violation), independent of
  whether they appear "educational".
- Makes a guaranteed-results claim ("cam ket tang X% doanh thu", "chac chan
  ra don") - Retain's tone is professional/informative, not a sales pitch
  with fabricated guarantees.
- Is criminal, political, or health/miracle-cure content unrelated to
  advertising (should not arise given the fixed pillar/news scope, but if a
  news-mode WebSearch result drifts into this territory, reject it and pick
  another).

## Gate B - script self-check (JSON decision before TTS/build)

Grade the drafted script and answer in this exact shape:
```json
{"decision": "APPROVE|REWRITE|HUMAN_REVIEW|REJECT", "risk_level": "GREEN|YELLOW|ORANGE|RED|BLACK", "violations": [], "claims_to_verify": [], "reason": "..."}
```
- GREEN/YELLOW + APPROVE -> continue (self-fix any flagged phrasing first if
  REWRITE was the initial read, then re-grade once).
- ORANGE/RED/BLACK or a REWRITE that cannot be auto-fixed cleanly ->
  HUMAN_REVIEW/REJECT: STOP, do not narrate/build/publish this run, note the
  reason in the run summary.

Specific checks Gate B must run, beyond the banned-phrase/guarantee checks in
Gate A:
- **D_case_study pillar**: the script AND on-screen text must explicitly say
  this is an illustrative example (e.g. "vi du minh hoa", "khong phai so lieu
  cua mot khach hang cu the") whenever a concrete number/formula result is
  shown - never presented as if it were a real client's data.
- **News-mode topics**: the two-sided framing (if any) must be traceable to
  something actually in the sourced article/announcement - do not invent a
  controversy or exaggerate a platform change that wasn't reported.
- **No abbreviations in narration text** - spell out full terms in the
  spoken lines (on-screen text/kicker labels may still show short forms like
  "CPA", "ROAS" since those are the actual industry terms being taught, not
  arbitrary abbreviations of Vietnamese words).

## Gate B4 - AI/synthetic media (added 2026-09-15, news-mode slot)

Applies whenever a video sources a real external news item (news-mode slot)
and considers using an AI-generated image to illustrate it:
- An AI-generated image may illustrate a CONCEPT freely - a diagram, an
  icon, an abstract graphic representing "ad targeting" or "algorithm
  change", for example.
- An AI-generated image must NEVER recreate a real event or a real person
  photorealistically as if it were an actual photograph of that event or
  person. If a real photo is needed (e.g. a real executive announcement, a
  real campaign visual), use the real photo with credit, or use an explicit,
  clearly-stylized concept illustration instead - never a fabricated "photo".
- When in doubt whether a generated image reads as a real photo vs. a concept
  graphic, default to the concept-graphic treatment (icon, chart, abstract
  shape) rather than risk a photorealistic fabrication.

## Gate B7 - originality / not mass-produced (added 2026-09-15)

Every video must carry at least one distinct angle, analysis, or presentation
choice - never just restate a headline or a single article's summary
verbatim. This is the single highest platform-policy risk for any automated
multi-video-per-day channel (Meta/YouTube "inauthentic / mass-produced"
content detection), and applies to every slot, not only news-mode:
- News-mode: add a specific "what this means for a Vietnamese small/medium
  advertiser" angle, a concrete implication, or a comparison to a prior
  platform change - do not just narrate the announcement.
- Any slot: vary the visual treatment (chart type, layout, invented icon set)
  video to video rather than reusing the exact same template feel back to
  back, even when the underlying HyperFrames frame structure is similar.
- Before finalizing a script, ask: "does this video say something a plain
  headline summary would not?" - if no, add the missing angle before
  proceeding to narration/build.

## Tone and brand voice

Chuyen mon - Ro rang - Dang tin cay (professional, clear, trustworthy). No
hard-sell language, no clickbait exaggeration. Matches the existing image
pipeline's established voice - video should read as the same channel, not a
different personality.

## Standard caption sign-off block

Append to every published video's Facebook caption, after 2 blank lines:
```
-----------------
Retain Agency - Dong Hanh Cung Quang Cao Cua Ban
Hotline/Zalo: 0972 382 983
Website: Retain.vn
Email: info.retain@gmail.com
```
