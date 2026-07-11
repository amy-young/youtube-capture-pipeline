# YouTube Transcript Pipeline — Setup

Pulls transcripts from YouTube videos, formats as .md, and drops into Google Drive `_INBOX` where the existing document pipeline synthesizes them via Gemini and imports to OpenBrain.

## Architecture

```
Manual Trigger
  → Google Sheets (read pending rows)
  → Loop one at a time:
      yt-dlp on VPS → strip VTT timestamps → format .md
      → Upload to _INBOX (existing pipeline takes over)
      → Mark row "processed" in Sheet
  → Wait 3s between items (rate limit)
```

## Prerequisites

**Already done:**
- yt-dlp installed on VPS (confirmed)
- Google Drive credential in n8n: `YOUR_DRIVE_CREDENTIAL_ID`
- _INBOX folder ID: `YOUR_INBOX_FOLDER_ID`

**Still needed:**
- Google Sheets credential in n8n (one-time)
- Google Sheet populated from `videos.csv`

---

## Step 1 — Create the Google Sheet

1. Go to Google Drive → New → Google Sheets
2. Name it: `PKA YouTube Backlog`
3. Row 1 headers (exact, case-sensitive):
   `video_id` | `title` | `channel` | `category` | `status`
4. Import `videos.csv` (File → Import → Upload → Replace current sheet)
5. Copy the Sheet ID from the URL:
   `https://docs.google.com/spreadsheets/d/THIS_PART_HERE/edit`

**Status column values the workflow uses:**
- empty = pending (will process)
- `processed` = done
- `error` = yt-dlp failed (no transcript available)
- `skip` = excluded (REMOVE category, pre-filled)
- `review` = needs your decision before processing (pre-filled for REVIEW category)

---

## Step 2 — Add Google Sheets credential in n8n

1. n8n → Settings → Credentials → New
2. Type: **Google Sheets OAuth2 API**
3. Follow OAuth flow with you@example.com
4. Name it exactly: `Google Sheets account`
5. Save → copy the credential ID from the URL

---

## Step 3 — Import and configure the workflow

1. n8n → Workflows → Import from file → upload `workflow.json`
2. Open the imported workflow
3. Replace all placeholders:

| Placeholder | Replace with |
|---|---|
| `REPLACE_WITH_SHEET_ID` | Sheet ID from Step 1 (appears in 3 nodes) |
| `REPLACE_WITH_GSHEETS_CREDENTIAL_ID` | Credential ID from Step 2 (appears in 3 nodes) |

---

## Step 4 — Test with one video

Before running the full batch:

1. In the Sheet, set all rows to `status = skip` temporarily
2. Find one row you want to test — clear its status (leave blank)
3. Run the workflow manually
4. Verify:
   - A .md file appears in `G:\My Drive\PKA-System\_INBOX\`
   - The existing pipeline synthesizes it (email notification arrives)
   - The row in the Sheet shows `processed`
5. Once confirmed, clear `skip` from the rows you want to process

---

## Step 5 — Run in batches

The workflow processes one video every ~3 seconds. For 224 pending videos, a full run takes ~11 minutes. You can stop mid-run and resume — it skips already-processed rows.

**Recommended batch order** (process by category, reset skips):
1. PKA → AI Systems (56 videos) — highest priority
2. PKA → Claude & Tools (89 videos)
3. PKA → Ventures (38 videos)
4. PKA → Content & Brand (20 videos)
5. PKA → Web Dev (13 videos)
6. PKA → Learning (7 videos)
7. PKA → Travel (2 videos)
8. REVIEW (12 videos) — decide individually before un-skipping

---

## Troubleshooting

**yt-dlp returns NO_TRANSCRIPT_FOUND:**
Video has no auto-captions (common for older or non-English videos). Row marked `error` — skip it.

**yt-dlp errors with "Sign in to confirm age" or similar:**
Add cookies to the command. In n8n Execute Command node, change the command to add `--cookies-from-browser chrome` — but this requires Chrome to be installed on the VPS (unlikely). Alternative: use a cookies.txt file exported from your browser and placed on the VPS at `/tmp/yt-cookies.txt`, then add `--cookies /tmp/yt-cookies.txt`.

**Google Sheets update fails:**
The update node matches rows by `video_id` column. If the column header doesn't exactly match `video_id`, the update silently fails. Check the Sheet headers are lowercase with underscores.

**Workflow stops mid-run:**
Just re-trigger manually. The filter skips `processed` and `error` rows — it picks up where it left off.
