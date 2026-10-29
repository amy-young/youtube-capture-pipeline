# YouTube Watcher + Daily Digest — Setup

Ongoing playlist-driven capture. Design rationale: `docs/design.md`.
Backlog (one-time) processing: `docs/setup-batch.md` + `workflows/batch-backlog-workflow.json`.

## Files in this build

| File | What it is |
|---|---|
| `workflows/watcher-workflow.json` | The n8n workflow — import this |
| `docs/synthesis-lenses.md` | Replacement JS for 2 nodes in the EXISTING doc pipeline (category-aware lenses) |
| `docs/design.md` | Full design doc |
| OB1 `synthesize-wiki.mjs` | Wiki compiler with a `content-brief` synthesizer (lives in your OB1 install, not this repo) |

## One-time setup checklist

### 1. YouTube (5 min, on your phone is fine)
- Create playlist `Capture` (private is fine — the API reads it)
- Keep `transcript` + subject playlists as-is
- Get each playlist's ID: open playlist → share/copy link → the `list=PL...` value

### 2. Google Sheet: `YouTube Watcher` (10 min)
Four tabs, exact lowercase names and headers:

- **playlists**: `playlist_id | name | mode`
  - `transcript` playlist → mode `autofile`
  - `Capture` → mode `digest`
  - each subject playlist → mode `tag` (its `name` becomes the category slug — use
    kebab-case names that match lens keys: `womens-sports`, `ai-systems`, `adrd-memory-care`)
- **trusted_channels**: `channel_name | channel_id | category`
  - e.g. `Some AI Creator | (optional) | ai-systems`
  - channel_name must match YouTube's display name exactly (it's matched case-insensitively)
- **ledger**: `video_id | title | channel | mode_applied | categories | processed_date | status`
- **resources**: `resource | type | url | note | video_id | video_title | date | status`

**Seed the ledger** so the watcher doesn't re-pull the backlog: paste the video_id
column from your backlog video list (the batch workflow's Sheet) into the ledger with status `done` for every
row already processed by the batch workflow.

### 3. Google Drive (1 min)
Create a `youtube-digest` folder (e.g. alongside your `_INBOX` folder) — copy its folder ID from the URL.

### 4. n8n credentials (5 min)
- **YouTube OAuth2 API** — new credential, same Google OAuth flow as Drive. Name: `YouTube account`.
- Google Sheets, Google Drive, and SSH credentials — reuse the ones from the batch workflow if created, else add them now.
- **Gmail OAuth2** — sends the digest email.

### 5. Import + placeholders
Import `watcher-workflow.json`. Replace:

| Placeholder | Value |
|---|---|
| `YOUR_SHEET_ID` | from the Sheet URL (5 nodes) |
| `YOUR_GSHEETS_CREDENTIAL_ID` | Sheets credential ID (5 nodes) |
| `YOUR_YOUTUBE_CREDENTIAL_ID` | YouTube credential ID (2 nodes) |
| `YOUR_DRIVE_CREDENTIAL_ID` | Google Drive credential ID (2 nodes) |
| `YOUR_GMAIL_CREDENTIAL_ID` | Gmail credential ID (1 node) |
| `YOUR_SSH_CREDENTIAL_ID` | SSH credential for the VPS host (1 node) |
| `YOUR_USER` | your user on the VPS host, in the yt-dlp command's deno + cookies paths (1 node) |
| `YOUR_OPENROUTER_API_KEY` | your OpenRouter API key (1 node) |
| `YOUR_INBOX_FOLDER_ID` | your Drive `_INBOX` folder ID (1 node) |
| `YOUR_DIGEST_FOLDER_ID` | folder ID from step 3 (1 node) |

### 6. Update the EXISTING doc pipeline (10 min)
Follow `docs/synthesis-lenses.md` — replace the JS in two nodes of your
document-processing workflow. This adds the category lenses and
classify-then-extract for untagged documents.
Then export both workflows fresh so your saved copies match what's live.

### 7. Test, then activate
1. Save a video to `Capture` and another to `transcript` (+ a subject playlist)
2. Run the watcher manually
3. Verify: digest email arrives + digest .md in the youtube-digest folder + resources
   rows appended; autofile video lands in `_INBOX` and the doc pipeline processes it
   with the right lens (check the synthesis has `**Category:**`)
4. Toggle Active. It runs daily at 5:30am; the digest email lands before the 6am target.

## The content loop (after capture is flowing)

Weekly (or on demand), compile a venture content brief:

```
cd /path/to/OB1 && node recipes/wiki-synthesis/scripts/synthesize-wiki.mjs --topic content-brief --scope category=womens-sports
cd /path/to/OB1 && node recipes/wiki-synthesis/scripts/synthesize-wiki.mjs --topic content-brief --scope category=ai-systems
```

Options: `--scope days=60` to widen the window (default 30). Output:
`content-brief-<category>.md` in the wiki output dir, with sections: What's
developing / Material bank / The discourse / Content opportunities.

Then in a content session: load the brief → `/content-ideas` or `/talking-points`
→ `/draft-blog` / `/draft-social` with the venture's audience profile. The same
ai-systems brief serves the training program, the AI community, and the portfolio blog.

Consider adding the two content-brief commands to `/weekly-review` alongside the
existing wiki compiler run.

## Digest email actions (daily habit)

- Repo/tool looks interesting → paste URL into a session → `/evaluate-tool`
- Video marked WORTH-FULL-CAPTURE → say "file <title> to OpenBrain" (Claude drops the
  transcript .md into `_INBOX`)
- Track resource decisions in the `resources` tab (status: new → evaluated → integrated/skipped)

## Costs (approximate)

- Mining pass: Gemini 2.5 Flash ~$0.005–0.01/video
- Autofile synthesis: existing pipeline cost (~$0.0055/doc)
- Content brief: one LLM call per run (sonnet-4-6 ≈ $0.05–0.15 depending on capture volume)
- YouTube API: playlistItems reads are ~1 quota unit each — daily polling of ~10
  playlists is far below the free 10,000/day quota

## Troubleshooting

**YouTube node returns nothing for a playlist:** check the playlist_id (must start
`PL`, from the `list=` URL param). Watch Later (`WL`) does NOT work — API limitation.

**Trusted channel not matching:** channel_name must equal the display name shown on
the video. Check the ledger row's `channel` value for the exact string and copy it.

**Digest email never arrives:** check the workflow ran (n8n executions) — if there
were no new videos, no email is sent by design.

**Same video processed twice:** ledger append failed on a prior run. Check the ledger
tab headers match exactly; the dedup reads `video_id`.
