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
