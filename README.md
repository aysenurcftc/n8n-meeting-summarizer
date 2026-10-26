# Meeting Audio Summary (n8n + Groq)

Upload a meeting recording (English, Turkish or mixed) to a Google Drive folder. It is transcribed with Groq Whisper, summarized with `gpt-oss-120b`, saved as a Google Doc and posted to Slack.

Built for **short recordings on Groq's free tier**.

<img width="1520" height="444" alt="Workflow" src="https://github.com/user-attachments/assets/49069c76-8539-49c7-babc-a0e74c4acb53" />


## Requirements

- n8n (self-hosted or cloud)
- Groq API key (free tier is enough)
- Google account (Drive + Sheets OAuth2)
- Slack app with a bot token and scopes `chat:write`, `channels:read` (add `groups:read` for private channels)

## Setup

1. **Import** `meeting.json` and `error-handler.json` into n8n.
2. **Credentials:** connect Google Drive, Google Sheets, Groq and Slack on the nodes that show a warning.
3. **Replace the placeholders** (`YOUR_...`) by selecting them in the nodes:
   - *Drive Trigger* → folder where recordings are uploaded
   - All Google Sheets nodes → the status sheet
   - *Send Summary* → summary channel, *Notify Error* + *Alert Admin Channel* → error channel
4. **CONFIG** (top of the *Validate File* node): set `summariesFolderId` to the Drive folder for summary docs. It must **not** be the watched recordings folder.
5. Invite the Slack bot to both channels: `/invite @your_bot`.
6. Main workflow → *Settings* → *Error Workflow* → `Meeting Summary - Error Handler`.
7. Activate the main workflow.

### Status sheet

Row 1 headers (exact spelling):

`File ID | File Name | Status | Attempts | Last Error | Updated At`

| Status | Meaning |
|---|---|
| `processing` | Running now. If stuck > 2 h (crash), it is retried automatically |
| `done` | Summary sent |
| `failed` | Transient error (rate limit, timeout, 5xx). Retried every 2 h, max `maxAttempts` |
| `failed_permanent` | Not retried (unsupported/too large file, empty audio, out of attempts) |

**Re-run a file manually:** set `Status` to `failed` and `Attempts` to `0`.

## Flow

```
Drive Trigger (every 15 min) ─┐
Retry Schedule (every 2 h) ───┴─> Loop Over Files (one file at a time)
   -> Validate File -> Check Status -> Decide Action -> Route
        process -> Mark Processing -> Download -> Whisper -> Prepare Transcript
                   -> short: Summarize (Direct)
                   -> long : Split -> Map -> Aggregate -> Reduce
                   -> Validate Summary -> Save Summary Doc -> Slack -> Mark Done
        reject  -> Build Error
        skip    -> next file
   any error -> Build Error -> Slack alert -> Log Error To Sheet -> next file
```

## Slack message

```
:memo: <Meeting topic>
<file name> · <duration> · <language>

Key Points / Decisions / Action Items
• Task — Owner · deadline      (owner/deadline only if mentioned)

:page_facing_up: Full transcript & summary  (link to the Google Doc)
```

The Google Doc is named `YYYY-MM-DD - <topic>` and contains the summary plus the full transcript.

## Error handling

1. **Node level:** API and Google Sheets nodes retry 3× before failing.
2. **Workflow level:** every error goes to *Build Error*, which classifies it:
   - *transient* (429, 5xx, timeout) → `failed`, auto-retried
   - *permanent* (400/404/413/415/422, validation, empty transcript) → `failed_permanent`
   - A failed chunk never produces a partial summary; the whole file fails.
3. **Crash level:** `error-handler.json` catches anything unhandled. A file that keeps crashing is stopped after `maxAttempts`.

## Configuration

All tunables are in the `CONFIG` block of the *Validate File* node:
`maxAttempts`, `maxFileSizeMB`, `directThreshold`, `chunkSize`, `chunkOverlap`, `minTranscriptChars`, `summaryLanguage`, `summariesFolderId`, `n8nBaseUrl`.

## Limitations

- Max file size 25 MB (Groq free tier), about 1 h of mono 64 kbps mp3.
- Long meetings may hit Groq's free-tier tokens-per-minute limit in the chunk step.
