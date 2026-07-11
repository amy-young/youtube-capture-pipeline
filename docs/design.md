# YouTube Playlist Watcher — Design v2
*Drafted 2026-07-08, revised same day after capture-context discussion.*
*Companion to the batch backlog workflow in this folder.*

## The core insight

the owner saves videos from her TV and phone, mid-watch: "this is good, look into it later."
The save is a bookmark of intent, not a filing decision. So the system has two
distinct destinations, and the playlist chooses between them:

1. **Auto-file** — trusted content goes straight into OpenBrain (the existing path)
2. **Daily digest** — everything else gets mined (summary + resources) and surfaced
   in one email a day. the owner decides what happens next from the email, at her desk.
   Nothing auto-files.

## Capture tiers

| Tier | Trigger | Destination |
|---|---|---|
| **Auto-file** | Video added to the existing `transcript` playlist, OR published by a trusted channel (Nate Jones, etc.) and saved to any watched playlist | Full transcript → `_INBOX` → Gemini synthesis → OpenBrain (existing pipeline, unchanged) — AND the video also runs through the mining pass, so it appears in the daily digest (marked "[auto-filed to OpenBrain]") with its resources in the ledger (added 2026-07-09) |
| **Digest** | Video added to `→ Capture` playlist (the one-tap save from phone/TV) | Transcript + **video description** → Gemini mining pass → daily digest email + resource ledger + .md saved to a review folder (NOT imported to OpenBrain) |
| **Subject tag** | Video also in a subject playlist | Adds `category:` to frontmatter / digest entry. Subject playlists alone never trigger capture |

Answer to "do I need a Capture Review playlist": **one new playlist total** — `→ Capture`.
The old "review & extract into OpenBrain" tier is gone; the digest replaces it.

## What the mining pass extracts (per video)

Input: transcript (yt-dlp subs) + video description (yt-dlp `--dump-json` — this is
where GitHub links, tool links, and discount/docs URLs actually live; transcripts
rarely contain URLs).

Output per video:
1. **Summary** — 3–5 bullets, calibrated to the owner's interest profile
2. **Resources** — every GitHub repo, open-source tool, plugin, service, or doc
   mentioned. From the description: name + URL. From the transcript: tools mentioned
   by name without a link get flagged `(mentioned, no link — search: "X")`
3. **Suggested next action** — e.g. "evaluate-tool candidate", "relevant to content
   engine", "skip — sponsor content"
4. **Worth-full-capture verdict** — if the mining pass finds the video is dense with
   signal, it says so, and the digest offers: "reply/ask Claude to file this one to
   OpenBrain"

## The daily digest email

One email per day (~6am), only when there are new saves. Format:

```
Subject: YouTube digest — 3 videos, 7 resources (2026-07-09)

── 10 Best Claude Code Repos This Week — Chase AI ──
Summary: ...3-5 bullets...
Resources:
  • anthropics/claude-cookbooks — github.com/... 
  • ccusage (token tracker) — github.com/...
  • SuperClaude framework — (mentioned, no link — search: "SuperClaude")
Next action: 2 evaluate-tool candidates
Watch again: https://youtube.com/watch?v=...
──────
...per video...

Full digest file: _PROCESSED/youtube-digest/2026-07-09-digest.md
```

The same content is saved as a dated .md in `_PROCESSED/youtube-digest/` (Google
Drive, synced locally). That folder is **not** in the 7am OpenBrain import path —
deliberately. From the digest the owner can:
- paste a repo link into a session → `/evaluate-tool` takes it from there
- say "file video 2 to OpenBrain" → Claude drops the saved transcript .md into `_INBOX`
- ignore it → it ages out harmlessly

## Resource ledger

`resources` tab in the watcher Sheet: `resource | type | url | video_id | video_title | date | status`
(status: new / evaluated / integrated / skipped). Every mined resource lands here, so
"what were those repos from two weeks ago" is a sheet filter, not an email search.
Periodic review can happen in `/weekly-review`.

## Category-aware synthesis (autofile path)

The existing `_INBOX` synthesis prompt is hardcoded to the ADRD/research lens — wrong
for other domains. The pipeline reads `category:` from the .md frontmatter and applies
the matching extraction lens. Categories also come from a new `category` column on the
`trusted_channels` tab (creator → default category, no playlist juggling needed).

### Lens: `womens-sports` (confirmed with the owner 2026-07-08)

Two layers per video, kept distinct:

**Layer 1 — the material (facts):**
- Storylines: what's developing, who's involved, where it's headed
- Stats and records: specific numbers, milestones, firsts
- Quotes: verbatim, attributed, worth quoting in a post
- Angles: tensions, contrasts, underexplored connections

**Layer 2 — the discourse (opinion reference):**
- Creator's take: what THIS creator argues/believes about each storyline, attributed
  by name (this is reference material — the owner writes her own take against it)
- Current consensus: where the creator positions their take relative to the broader
  conversation ("most people think X, I think Y" moments are gold — capture both sides)
- Open debates: facets where creators disagree; note who holds which position

Layer 2 feeds the owner's opinion writing directly: her posts are her take on the
storylines, and knowing the existing takes lets her position against consensus
(pairs with `/opposite-start`, which maps consensus and inverts it).

### The lens template generalizes to every content venture

The two-layer structure isn't women's-sports-specific. Every venture the owner creates
content for needs the same shape:

- **Layer 1 — material:** domain facts worth building content on
- **Layer 2 — discourse:** attributed creator takes, consensus position, open debates
  (so the owner can write HER take against the existing conversation)

Only the Layer 1 field list changes per category:

| Category | Layer 1 fields |
|---|---|
| `womens-sports` | storylines, stats/records, quotes, angles |
| `ai-systems` | tools/repos/methods, workflow patterns, capability claims, build ideas |
| `adrd-memory-care` | findings, methodology, clinical/caregiver implications, quotables (extends existing prompt) |
| `legal` | legal developments (WA-flagged), process explanations, client questions/misconceptions, practice positioning — audience is potential clients, not lawyers (added 2026-07-08) |
| `travel`, `web-dev`, ... | fall to the GENERIC content lens (material + discourse, no domain fields) until defined — a lens is one prompt block, cheap to add |

Lens routing (revised 2026-07-08 — no fixed default): named lens > generic content
lens (any tagged category without one) > classify-then-extract for untagged files
(PDFs/DOCX) — Gemini picks the lens in the same call and declares the category on
its first output line, so manually dropped files get tagged too. ADRD is now just a
named lens (`adrd-memory-care`), not a default. Multiple playlists
can map to one category in the Sheet — playlist_id is the join key, the Sheet `name`
column is the category slug, so YouTube playlist titles never need renaming.

### Categories ≠ ventures (the mapping layer)

One capture category serves multiple ventures. The mapping lives in the CONTENT
ENGINE, not the pipeline — the pipeline only tags and extracts once:

| Capture category | Feeds ventures |
|---|---|
| `ai-systems` | AI Training Program (curriculum material), AI Community (discussion topics), Portfolio blog (posts about AI) |
| `womens-sports` | Women's sports blog |
| `adrd-memory-care` | Alz researcher site, Memory care directory, NACC awareness |
| `legal` | Collaborative law site, Lisa's legal site |
| `travel` | Travel app (when unparked) |

Flow: capture (tagged once) → OpenBrain → per-category **topic synthesizer** in the
OB1 wiki compiler (pattern: `current-state`) compiles a rolling content-sources brief
→ content engine generates per-venture output from the same brief, differentiated by
the venture's audience profile + voice guide (`ventures/context/audiences/`,
`profiles/`). The same ai-systems brief becomes a training module idea, a community
discussion prompt, and a blog post — three audiences, one capture.

Start with 3 lenses + 2 topic synthesizers (ai-systems, womens-sports; ADRD extends
the existing default). Add more only when a venture unparks.

## Components

### 1. Sheet: `PKA YouTube Watcher`
- Tab `playlists`: `playlist_id | name | mode` (mode: `autofile` / `digest` / `tag`)
- Tab `trusted_channels`: `channel_name | channel_id`
- Tab `ledger`: `video_id | title | channel | mode_applied | categories | processed_date | status`
- Tab `resources`: as above

### 2. n8n workflow: "YouTube Watcher + Daily Digest"
Single daily run (~5:30am), so the digest is one email, not a trickle:
1. Read config tabs + ledger
2. n8n **YouTube node** (OAuth, same Google account flow as Drive credential) →
   `playlistItems` for each watched playlist. API, not RSS — RSS fails on private
   playlists.
3. Build one row per new video; merge categories; resolve mode
   (trusted channel → autofile; `transcript` playlist → autofile; `→ Capture` → digest)
4. Loop (batch 1, 3s wait, same yt-dlp Execute Command pattern as batch workflow):
   - yt-dlp: subs + `--dump-json` metadata (description, channel, duration, upload date)
   - **Autofile path:** format .md → `_INBOX` → existing pipeline
   - **Digest path:** OpenRouter Gemini mining prompt → per-video block + resource rows
   - Ledger update either way
5. After loop: assemble digest blocks → one email + one .md to `_PROCESSED/youtube-digest/`
   → append resource rows to `resources` tab

### 3. Downstream — unchanged
7am import, entity extraction (hourly VPS cron), wiki compiler.

## Capture ergonomics (phone/TV reality)

- **Phone:** Share → Save to playlist → `→ Capture`. Reliable, two taps.
- **TV:** the TV app often only offers "Watch Later". **Watch Later is not readable
  via the YouTube API** (long-standing limitation) — the watcher cannot see it.
  Workaround: save to Watch Later on TV, then on the phone move items to `→ Capture`
  (Watch Later → three-dot → Save to playlist). Treat Watch Later as the TV inbox,
  `→ Capture` as the real trigger.

## What the owner sets up once
1. YouTube: create `→ Capture` playlist (keep `transcript` + subject playlists as-is)
2. n8n: YouTube OAuth2 credential
3. Create the Sheet (4 tabs); fill `playlists` + `trusted_channels`
4. Google Drive: create `_PROCESSED/youtube-digest/` folder
5. Import watcher workflow JSON (to be generated), fill Sheet ID + credential IDs

## Open items
- Watcher workflow JSON: generate after design confirmation
- Mining prompt interest profile: draft from USER.md + threads; store in a Sheet cell
  so it's editable without opening n8n
- Seed ledger from batch backlog `videos.csv` before activating, or the watcher
  re-pulls the backlog
- Later idea: "reply to digest email to trigger actions" — v2; for now actions happen
  in a Claude session with the digest file as context
