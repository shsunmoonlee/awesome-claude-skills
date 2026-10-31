---
name: transcribe-audio
description: Transcribe audio or video from a YouTube link, podcast or platform URL, direct file URL, or upload with the transcribe.so API, quoting the price first and capping the spend, then read the speaker-labelled transcript.
---

# Transcribe Audio

Turns a recording (a YouTube video, a podcast episode, a Zoom or Loom recording, a direct audio or video link, or a local file) into a speaker-labelled transcript with timestamps, using the transcribe.so HTTP API. The skill always checks the price before it starts a paid job, sets a spend ceiling on every job, and uses a retry key so a lost response never charges twice.

## When to Use This Skill

- You have a link to a video, podcast, interview or meeting and want the transcript
- You need to know who said what, and when, in a recording
- You want subtitles (SRT or VTT) for a video
- You want to search a recording for a quote and get the exact timestamp

## What This Skill Does

1. **Quotes the price**: asks the API what the job will cost before creating it (quotes cost nothing)
2. **Creates the job safely**: sends a spend ceiling and an idempotency key, so a retry never double-charges
3. **Waits for the result**: polls a bounded wait endpoint until the job finishes
4. **Reads the transcript**: returns the full text, speaker-labelled segments, subtitles, or search hits

## Setup

1. Sign in at https://transcribe.so (new accounts get a $1 signup credit; pay as you go is $1 per audio hour).
2. Create an API key in Settings > API keys. Keys look like `tsk_live_...`.
3. Make the key available to Claude, for example as the `TRANSCRIBE_API_KEY` environment variable. Never print the key.

Every request sends `Authorization: Bearer $TRANSCRIBE_API_KEY` to the base URL `https://transcribe.so`.

## Instructions

### Rules

- Check the price before creating a job. Stay within the budget the user gave you; ask before going over it.
- Send an `Idempotency-Key` header (a stable UUID) on every create. Reuse the same key if you retry the same request.
- Send `max_charge_usd` on every create. If the server answers 402 `max_charge_exceeded`, nothing was charged; go back to the user instead of raising the ceiling yourself.
- `status: "queued"` from create means success. Do not submit the job again.
- Never pass a local file path as a URL. Upload the file first (see below).

### 1. Quote (no charge)

```bash
curl -s https://transcribe.so/api/v1/quotes \
  -H "Authorization: Bearer $TRANSCRIBE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"source":"youtube","url":"https://www.youtube.com/watch?v=VIDEO_ID"}'
```

Show the price to the user if they have not already approved a budget.

### 2. Create the job

```bash
curl -s https://transcribe.so/api/v1/transcriptions \
  -H "Authorization: Bearer $TRANSCRIBE_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 5f0c1a2e-0000-4000-8000-000000000001" \
  -d '{"source":"youtube","url":"https://www.youtube.com/watch?v=VIDEO_ID","max_charge_usd":2.00,"client_reference":"5f0c1a2e-0000-4000-8000-000000000001"}'
```

Sources:

- `youtube`: a YouTube video URL
- `platform_url`: a public page on a supported platform (Apple Podcasts, SoundCloud, Vimeo, Twitch, Loom and similar)
- `external_url`: a direct http(s) link to an audio or video file
- `upload`: a file you uploaded first, `{"source":"upload","upload_id":"...","duration_seconds":123}`

To upload a local file: `POST /api/v1/uploads` with `{"filename","content_type","file_size"}` returns `{upload_id, upload_url}`. `PUT` the raw bytes to `upload_url` with the same `Content-Type`, then create the job with that `upload_id`.

### 3. Wait

```bash
curl -s "https://transcribe.so/api/v1/transcriptions/JOB_ID/wait?timeout=45" \
  -H "Authorization: Bearer $TRANSCRIBE_API_KEY"
```

Call it again while `status` is `queued` or `processing` (`_timed_out: true` means call again). Stop on `completed`. `failed` and `cancelled` are final: report the error to the user and do not create a new job without asking.

### 4. Read the result

- Full transcript: `GET /api/v1/transcriptions/JOB_ID/transcript?format=md` (or `txt`)
- Subtitles: `GET /api/v1/transcriptions/JOB_ID/subtitles?format=srt` (or `vtt`)
- Search the transcript for a quote, with speaker and timestamp: `GET /api/v1/transcriptions/JOB_ID/search`

The exact request and response shapes are in https://transcribe.so/api/v1/openapi.yaml.

## How to Use

### Basic Usage

```
Transcribe https://www.youtube.com/watch?v=VIDEO_ID and give me the transcript with speaker labels.
```

### Advanced Usage

```
Transcribe this podcast episode, spend at most $1, then find every place the guest talks about pricing and give me the quote, the speaker and the timestamp for each one.
```

## Example

**User**: "Transcribe this interview and make SRT subtitles: https://www.youtube.com/watch?v=VIDEO_ID"

**Output**:
```
Quote: $0.42 for 25 minutes of audio. Created job 4821 (queued).
Completed after 2 waits.

[00:00:04] Speaker 1: Thanks for joining us today.
[00:00:07] Speaker 2: Happy to be here.
...

Subtitles saved to interview.srt (312 cues).
```

**Credit:** Based on the transcribe.so agent skills (https://github.com/shsunmoonlee/transcribe-agent)

## Tips

- Quotes cost nothing, so quote first even when you expect a small charge.
- Send a `client_reference` with every create; `GET /api/v1/transcriptions?client_reference=...` finds the job if a create call timed out.
- Instead of polling, you can pass a public `callback_url` on create to receive a signed completion webhook.
- If your client supports MCP, the same features are available as tools from the remote MCP server at https://transcribe.so/mcp (OAuth or Bearer API key).

## Common Use Cases

- Transcribing podcasts and interviews for show notes and quotes
- Making subtitles for YouTube and course videos
- Finding who said what, and when, in a long meeting or panel recording
- Turning lectures and webinars into searchable text
