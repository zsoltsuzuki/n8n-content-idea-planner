# Daily Content Idea Planner (n8n)

A scheduled n8n workflow that plans a day's content ideas based on how busy the calendar is. Every morning it counts today's Google Calendar events, picks the least recently used topics from a Google Sheet, asks an LLM (via Groq) for one concrete video idea per topic, logs the result, updates the topic rotation, and puts the ideas into the calendar as events.

Built as a demo project to practise workflow automation, API integration and LLM prompting.

## How it works

```
Schedule Trigger (07:00, Europe/Budapest)
  -> Google Calendar: Get Many (today's events)
  -> Summarize: count events
  -> Google Sheets: Get Rows ("ideas" sheet)
  -> Sort: by "last used", ascending (never used first)
  -> Limit: 1 idea on busy days (3+ events), otherwise 3
  -> Basic LLM Chain + Groq Chat Model: one idea per topic
  -> Edit Fields: split LLM output into title / description
       -> Google Sheets: Append Row ("log" sheet)
            -> Google Sheets: Update Row ("ideas" sheet, last used = today)
       -> Google Calendar: Create Event (18:00, 19:00, 20:00; 30 min each)
```

## Design decisions

- **Timezone:** the workflow uses the IANA timezone `Europe/Budapest` instead of a fixed UTC offset, so the 07:00 run survives daylight saving changes. n8n defaults to `America/New_York` if nothing is set.
- **Time window anchored to the day, not the run:** events are filtered from `$today` to `$today.plus({ day: 1 })`, so the result does not depend on when the workflow runs.
- **Empty days:** n8n stops a branch when a node returns 0 items. The Calendar node has *Always Output Data* enabled, and the Summarize node counts only the `id` field, so a day with no events returns `0` instead of silently ending the run.
- **Topic rotation without an LLM:** topics are selected deterministically (least recently used first). Dates are stored as `yyyy.MM.dd` with leading zeros, so text sorting equals chronological sorting.
- **Input and output separated:** the `ideas` sheet is the input pool, the `log` sheet is the append-only output. Keeping them apart prevents the workflow from reading its own output back as input.
- **Item linking:** after the LLM step the topic is no longer in the current item, so it is referenced from the Limit node with `$('Limit').item`, which returns the item paired with the current one (not just the first).
- **LLM output format:** the prompt asks for exactly two plain-text lines (`TITLE:` and `DESCRIPTION:`), no markdown, so the output can be split reliably and fits into a Sheets cell and a calendar title. Temperature is 0.7 to allow some variety.

## Google Sheets structure

Spreadsheet with two tabs:

| Sheet | Columns |
|-------|---------|
| `ideas` | `topic`, `Category`, `last used` |
| `log` | `date`, `topic`, `category`, `title`, `description`, `event_count` |

`topic` values in `ideas` must be unique, because the Update Row step matches on that column.

## Setup

1. Run n8n locally with Docker:
   ```bash
   docker run -it --rm -p 5678:5678 \
     -e GENERIC_TIMEZONE=Europe/Budapest -e TZ=Europe/Budapest \
     -v n8n_data:/home/node/.n8n n8nio/n8n
   ```
2. Google Cloud Console: create a project, enable the Google Calendar, Google Sheets and Google Drive APIs, configure the OAuth consent screen (External, Testing, add yourself as a test user), and create a *Web application* OAuth client with n8n's redirect URL (`http://localhost:5678/rest/oauth2-credential/callback`).
3. In n8n, create a **Google Calendar OAuth2 API** and a **Google Sheets OAuth2 API** credential with the same client ID and secret.
4. Create a Groq API key at console.groq.com and add a **Groq** credential in n8n.
5. Import `workflow.json`, select your own credentials, spreadsheet and calendar in each node, then activate the workflow.

Credentials are not included in the exported workflow.

## Limitations

- Runs only while the n8n instance is running; a missed 07:00 run is not repeated later.
- With the OAuth app in Testing mode, Google refresh tokens expire after about 7 days and the credentials need a new sign-in.
- Running the workflow twice on the same day counts its own previously created events, so the second run treats the day as busy.
- The LLM output is in Hungarian; the model used occasionally makes spelling or word-choice mistakes, so the ideas are drafts, not final copy.

## Project history

The first version of this idea generated video ideas separately with the OpenAI API, the results were placed into Google Sheets manually, and n8n created calendar events from the sheet rows on a daily schedule. This version moves idea generation into the scheduled workflow and adds calendar-based volume control, topic rotation and logging.
