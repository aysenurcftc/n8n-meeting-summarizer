

# Meeting Audio → Slack Summary (n8n)

An n8n workflow that watches a Google Drive folder for meeting recordings, transcribes them with Groq Whisper, summarizes them with an LLM (key points + action items), and posts the result to Slack. Errors are logged to Google Sheets and reported to an admin Slack channel.

<img width="1520" height="444" alt="image" src="https://github.com/user-attachments/assets/49069c76-8539-49c7-babc-a0e74c4acb53" />
## Flow

```
Drive Trigger → duplicate check (Sheets) → download audio → Groq Whisper
→ summarize (single call, or map-reduce if transcript > 15,000 chars)
→ Slack → mark as processed (Sheets)
                 ↘ on any error: log to Sheets + notify Slack admin channel
```

## Requirements

- n8n (self-hosted or cloud)
- Groq API key
- Google account (Drive + Sheets OAuth2)
- Slack app with a bot token (`xoxb-...`) and scopes `chat:write`, `channels:read` (add `groups:read` for private channels)

## Setup

1. In n8n: **Workflows → Import from file** → select `workflow.json`.
2. Create a Google Sheet with the header row `Meeting ID` | `Error`.
3. Create the credentials (Google Drive, Google Sheets, Groq, Slack) and select them on each node.
4. Set the Drive folder, the Sheet (3 nodes), and the two Slack channels (summary + errors).
5. Invite the bot to both channels: `/invite @your_bot`.
6. Activate the workflow.

## Notes

- The trigger polls once a day at 12:00; change it in the **Drive Trigger** node.
- Groq free tier accepts audio files up to about 25 MB.
- Designed for one new file per run; if several files arrive at once, process them one at a time.
- A failed file is also written to the Sheet, so it will be skipped on the next run. Delete its row to retry.
