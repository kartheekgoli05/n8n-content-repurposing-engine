# Content Repurposing Engine (n8n)

An n8n workflow that takes one piece of source material (a blog post URL, a call transcript, a case study) and turns it into a month of LinkedIn, Instagram and Facebook posts, then publishes them on schedule.

The point is to write from material you already have instead of asking a model to invent topics. Every post traces back to a specific detail in the source.

## How it works

There are two separate chains in the one workflow.

**1. Fill the queue (form trigger)**

```
Form (URL or pasted text, post count, start date)
  -> fetch + extract article text if a URL was given
  -> Claude: pull out 8 / 12 / 16 distinct ideas, each tied to a concrete detail
  -> loop over ideas
       -> Claude: write a LinkedIn, Instagram and Facebook version of that idea
       -> assign a date (Mon / Wed / Fri across the next weeks) and a per-platform time
  -> insert rows into the content_queue Data Table
  -> email a summary of what got queued
```

**2. Publish (every 30 minutes)**

```
Schedule -> read content_queue -> keep rows due now
  -> publish one at a time with a gap between posts (upload-post.com API)
  -> mark each row done or failed
  -> email a daily summary
```

Prompt rules worth noting: ideas can't overlap, no facts that aren't in the source, formats are varied (how-to, opinion, mistake, example...), and each platform gets a real rewrite rather than the same text with different hashtags. If Claude returns something that isn't valid JSON, the row is queued with a `[REVIEW NEEDED]` tag instead of failing the run.

## Setup

1. Import `content-repurposing-engine.json` into n8n.
2. Create a Data Table called `content_queue` with columns `sourceTitle`, `angle`, `platform`, `content`, `scheduledFor`, `scheduledTime`, `status`, and put its ID in place of `REPLACE_WITH_content_queue_TABLE_ID` (4 nodes).
3. Anthropic credential: Header Auth, name `x-api-key`.
4. Publisher credential: Header Auth, name `Authorization`, value `Apikey YOUR_KEY`.
5. Gmail credential, and your address in both Gmail nodes.
6. Edit the BRAND block in `ExtractIdeas` and the brand line in `WriteVariants`.
7. Posting times are in `ScheduleAcrossMonth` (LinkedIn 09:00, Facebook 13:00, Instagram 18:30 by default).

There's no per-post approval step in the publisher, since you review everything in the queue email. If you want one, put a Wait node (resume on webhook) before `PublishQueued`.

## Stack

n8n (form trigger, HTTP Request, HTML extract, Code, Data Tables, Gmail), Anthropic Messages API, upload-post.com.
