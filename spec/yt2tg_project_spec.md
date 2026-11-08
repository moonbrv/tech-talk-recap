# yt2tg: project spec

Nov 8, 2026 · @Dima

## Overview

yt2tg makes a structured recap of a YouTube video and posts it to a Telegram channel. A local Go CLI gets the audio. A Cloudflare service, written with Effect and deployed with Alchemy, makes the transcript and the recap. The transcript keeps the language of the video. The recap is always in English. The MVP has no UI: you send all requests with the CLI or with Postman, and each request must have a bearer token.

**In scope for the MVP**

- Go CLI `yt2tg` (already written): it reads video data, downloads the audio, cuts it into 10-minute mp3 chunks and uploads them.
- Cloudflare service: a protected HTTP API, a Workflow, a D1 database, an R2 bucket, Workers AI and a Telegram post.
- A Postman collection and a step-by-step upload procedure.
- Volume: approximately 5 videos per day. One video can be up to 5 hours long.

**Out of scope for the MVP**

- A web UI, user accounts, more than one API token.
- Download of audio in the cloud (YouTube blocks datacenter IP addresses).
- More than one Telegram channel, edits of posts after publication.

**Writing rules for tasks (ASD-STE100)**

All tasks in this spec use ASD-STE100 Simplified Technical English. When you add or change a task, obey these rules:

1. Write one instruction in one sentence. Use the imperative: "Make", "Add", "Do a test of".
2. Write procedural sentences with 20 words or fewer. Write descriptive sentences with 25 words or fewer.
3. Use the active voice. Do not use the passive voice in instructions.
4. Use one word for one meaning. For example, use "chunk" for one audio part, not "segment" or "piece".
5. Write a code name, a file name or a command in `code` format. These are technical names and you can use them.
6. Start a warning with "WARNING:" and a note with "NOTE:". Put the warning before the step that it is about.

## Architecture

The audio download stays on your computer, because YouTube blocks requests from datacenter IP addresses. All other work runs on Cloudflare. One Alchemy stack in TypeScript declares all Cloudflare resources.

&#91;embedded content: yt2tg architecture · local CLI, Cloudflare service, Telegram\]

The Worker accepts the upload and starts a workflow. The workflow does each slow step as a durable task with retries.

| Part | Technology | Job |
| --- | --- | --- |
| CLI | Go 1.22+, `yt-dlp`, `ffmpeg`, `deno` | Reads title, description and duration, downloads audio, cuts 16 kHz mono mp3 chunks, uploads them |
| HTTP API | Cloudflare Worker, Effect `HttpApi` | Checks the token, stores the rows and the chunks, starts the workflows |
| Pipeline | Cloudflare Workflows: `VideoPipeline` | Transcribes, saves the SRT, makes the recap (call A), posts, cleans up |
| Segments job | Cloudflare Workflows: `TranscriptSegmentsJob` | Makes `transcriptSegments` (call B). A failure here does not stop the recap |
| Database | D1 | Tables `videos`, `transcripts` and `recaps`, joined on the video id |
| Audio storage | R2 | Chunks, kept for 1 day or less |
| Speech to text | Workers AI, `@cf/openai/whisper-large-v3-turbo` | One call for each 10-minute chunk |
| Recap | Workers AI, `RECAP_MODEL` (default `@cf/openai/gpt-oss-120b`, 128k context) | Call A: all recap fields except `transcriptSegments` |
| Speaker labels | Workers AI, `SEGMENTS_MODEL` (default `@cf/meta/llama-3.3-70b-instruct-fp8-fast`, JSON mode) | Call B: one speaker or `null` for each SRT cue |
| Infrastructure | Alchemy v2 (`alchemy`, `@alchemy.run/cloudflare`) | Declares and deploys all resources from `alchemy.run.ts` |

## Repository layout

One Git repository holds two parts: `cli/` (Go) and `service/` (TypeScript, Bun, Effect, Alchemy). The two parts share only the HTTP contract in this spec.

```text
yt2tg/
├── README.md                      # how to install, configure, deploy
├── cli/                           # Go CLI (existing code, moved here)
│   ├── go.mod                     # module github.com/<you>/yt2tg/cli
│   ├── main.go  deps.go  ytdlp.go  audio.go  api.go
│   ├── yt2tg_test.go
│   └── README.md
├── service/
│   ├── package.json               # alchemy, @alchemy.run/cloudflare, effect
│   ├── tsconfig.json
│   ├── alchemy.run.ts             # the Stack: D1, R2, Worker, workflows, AI
│   ├── .env.example               # API_TOKEN, TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID
│   ├── migrations/
│   │   └── 0001_init.sql          # videos, transcripts, recaps
│   ├── src/
│   │   ├── worker.ts              # Effect Worker: HTTP API entry point
│   │   ├── workflow.ts            # VideoPipeline: transcribe, recap (call A), post
│   │   ├── segmentsWorkflow.ts    # TranscriptSegmentsJob: call B
│   │   ├── resources.ts           # Database, AudioBucket declarations
│   │   ├── api/
│   │   │   ├── Api.ts             # HttpApi, VideosGroup, endpoints
│   │   │   ├── Auth.ts            # bearer token middleware
│   │   │   └── Errors.ts          # tagged errors with HTTP status
│   │   ├── domain/
│   │   │   ├── Video.ts           # Video, VideoStatus, CreateVideo
│   │   │   └── Recap.ts           # VideoRecap, ModelRecapOutput, RecapSectionKey
│   │   ├── services/
│   │   │   ├── VideoRepo.ts       # D1: videos
│   │   │   ├── TranscriptRepo.ts  # D1: transcripts
│   │   │   ├── RecapRepo.ts       # D1: recaps
│   │   │   ├── AudioStore.ts      # R2 access
│   │   │   ├── Transcriber.ts     # Whisper on Workers AI
│   │   │   ├── Recapper.ts        # call A: one-pass and long-transcript paths
│   │   │   ├── SpeakerLabeler.ts  # call B
│   │   │   └── Telegram.ts        # Bot API client
│   │   └── lib/
│   │       ├── srt.ts             # build and parse SRT, timestamped lines
│   │       ├── timestamps.ts      # parse loose timestamps, format, deep links
│   │       ├── grounding.ts       # quote and stat checks against the transcript
│   │       ├── prompts.ts         # prompt text and PROMPT_VERSION
│   │       ├── hashtags.ts        # HASHTAG_ALIASES, normalizeHashtags
│   │       ├── telegramFormat.ts  # HTML post, escape, 4096 limit
│   │       └── uuidv7.ts
│   └── test/
│       └── fixtures/              # audio, SRT and recap golden set
└── postman/
    └── yt2tg.postman_collection.json
```

NOTE: Keep the `.env` file out of Git. Add `service/.env` to `.gitignore`.

## Data model

D1 has three tables: `videos`, `transcripts` and `recaps`. The two last tables use the video id as their primary key, so a join on `video_id` is cheap. R2 holds the audio chunks for one day only. D1 is SQLite, so the UUIDv7 is `TEXT` (made in TypeScript) and JSON columns are `TEXT` with a `json_valid` check.

```sql
-- migrations/0001_init.sql
CREATE TABLE videos (
  id                  TEXT PRIMARY KEY,                 -- UUIDv7
  youtube_id          TEXT NOT NULL UNIQUE,             -- 11 characters
  video_url           TEXT NOT NULL,                    -- https://www.youtube.com/watch?v=<youtube_id>
  title               TEXT NOT NULL,
  channel             TEXT,
  description         TEXT,                             -- optional, speaker hints
  speakers            TEXT CHECK (speakers IS NULL OR json_valid(speakers)),  -- optional JSON array of names
  language_hint       TEXT,                             -- optional ISO 639-1 code for Whisper
  duration_sec        INTEGER,
  status              TEXT NOT NULL DEFAULT 'uploading'
    CHECK (status IN ('uploading','transcribing','recapping',
                      'publishing','done','failed')),
  attempt             INTEGER NOT NULL DEFAULT 1,       -- part of the workflow instance id
  chunk_count         INTEGER,
  telegram_message_id INTEGER,
  error               TEXT,
  created_at          TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  updated_at          TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);
CREATE INDEX videos_status_idx ON videos (status);

CREATE TABLE transcripts (
  video_id    TEXT PRIMARY KEY REFERENCES videos(id) ON DELETE CASCADE,
  language    TEXT NOT NULL,                            -- ISO 639-1, the language of the video
  srt         TEXT NOT NULL,                            -- one cue for each Whisper segment
  cue_count   INTEGER NOT NULL,
  created_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  updated_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);

CREATE TABLE recaps (
  video_id            TEXT PRIMARY KEY REFERENCES videos(id) ON DELETE CASCADE,
  recap               TEXT NOT NULL CHECK (json_valid(recap)),   -- VideoRecap without transcriptSegments
  transcript_segments TEXT CHECK (transcript_segments IS NULL OR json_valid(transcript_segments)),
  segments_status     TEXT NOT NULL DEFAULT 'idle'
    CHECK (segments_status IN ('idle','updating','failed')),
  segments_error      TEXT,
  model               TEXT NOT NULL,                    -- RECAP_MODEL used for call A
  prompt_version      INTEGER NOT NULL,
  created_at          TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  updated_at          TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);

-- One trigger of this form for each table: videos (key id), transcripts and recaps (key video_id).
CREATE TRIGGER videos_updated_at AFTER UPDATE ON videos
FOR EACH ROW WHEN NEW.updated_at = OLD.updated_at
BEGIN
  UPDATE videos SET updated_at = strftime('%Y-%m-%dT%H:%M:%fZ','now') WHERE id = NEW.id;
END;
```

**Video status values**

| Status | Set by | Meaning |
| --- | --- | --- |
| `uploading` | `POST /videos` | The row exists. The chunks are not all in R2. |
| `transcribing` | `POST /videos/:id/complete` | `VideoPipeline` started. Whisper runs on each chunk. |
| `recapping` | `VideoPipeline` | The SRT is in `transcripts`. Call A makes the recap. |
| `publishing` | `VideoPipeline` | The recap is in `recaps`. The service sends the Telegram post. |
| `done` | `VideoPipeline` | The post is in the channel. `telegram_message_id` has a value. |
| `failed` | `VideoPipeline` | A step failed after all retries. `error` has the cause. |

`recaps.segments_status` is separate from the video status. `idle` with `transcript_segments` empty means "not made yet". `idle` with a value means "ready". `updating` means that call B runs. `failed` means that call B failed, and `segments_error` has the cause. A failure of call B does not change the video status.

**Recap shape**

All times are integer seconds from the start of the video. The service uses these values in the `t` query parameter of the YouTube link.

```ts
type Seconds = number;

type RecapSectionKey =
  | 'summary' | 'keyTakeaways' | 'topics' | 'hashtags' | 'chapters'
  | 'quotes' | 'stats' | 'mentions' | 'transcript';

interface VideoRecap {
  summary: string;
  keyTakeaways: string[];
  topics: string[];
  hashtags: string[];             // project addition, see below
  chapters: { title: string; startSec: Seconds; endSec: Seconds }[];
  quotes: {
    text: string;
    translation: string | null;   // project addition, see below
    speaker: string | null;
    startSec: Seconds;
    featured?: boolean;
    hidden?: boolean;
  }[];
  stats: {
    value: string;
    description: string;
    startSec: Seconds | null;
    hidden?: boolean;
  }[];
  mentions: {
    people: string[];
    companies: string[];
    books: string[];
    concepts: string[];
  };
  transcriptSegments: { speaker: string | null; startSec: Seconds | null; text: string }[];
  hiddenSections?: RecapSectionKey[];
}
```

| Field | Intent |
| --- | --- |
| `summary` | A short prose overview of the video: what it is about and what the viewer gets from it. 2–4 sentences. |
| `keyTakeaways` | The main lessons, one sentence each. These are the "if you watch nothing else" points. |
| `topics` | Short topic tags (for example "Postgres indexing" or "hiring"). Use them for filtering and search. |
| `hashtags` | Up to 7 hashtags for the Telegram post, made from `topics` and `summary`. Users search for posts on similar topics with them. Each item is one lowercase word without `#`. Short forms are preferred: "database" gives `db`, "machine learning" gives `ml`, "database indexing" gives `db` and `indexing`. |
| `chapters` | A time-coded outline of the video. Each chapter links to its start time in the video. |
| `quotes` | Notable quotes, copied word for word from the transcript, with the speaker and start time. `featured` and `hidden` are set by the user, never by the model. |
| `stats` | Numbers the speakers actually said (`value` is "40%", `description` is what that number measures). `hidden` is set by the user. |
| `mentions` | People, companies, books, and concepts named in the video, for a "referenced in this video" list. |
| `transcriptSegments` | The transcript split into segments, each with a speaker and a start time. A separate step creates it (see the generation rules). |
| `hiddenSections` | Whole sections the user has hidden. If it is absent or empty, show every section that has content. |

**Project additions to the shape**

| Field | Rule |
| --- | --- |
| `quotes[].translation` | The English translation of `text`. It is `null` when the video is in English. `text` stays word for word in the language of the video. |
| `hashtags` | Tags for the Telegram post. The Telegram post shows them in place of `topics`. See rule 8 in "Recap generation". |
| Language | All other recap text is in English: `summary`, `keyTakeaways`, `topics`, `hashtags`, chapter titles, stat descriptions and `concepts`. Names in `people`, `companies` and `books` use their usual English spelling. `transcriptSegments[].text` stays in the language of the video. |

**Where each part is stored**

| Part | Table and column |
| --- | --- |
| SRT transcript (language of the video) | `transcripts.srt` |
| All recap fields except `transcriptSegments` | `recaps.recap` |
| User choices (`featured`, `hidden`, `hiddenSections`) | `recaps.recap`, in place |
| `transcriptSegments` | `recaps.transcript_segments` |

Size check: a 2-hour SRT is approximately 0.25 MB and its `transcript_segments` approximately 0.3 MB. Each is in a different column, and both are far below the D1 limit of 2 MB for one row.

**R2 key layout**

- `audio/<video id>/<n>.mp3`, where `<n>` has three digits: `000`, `001`, `002`.
- `VideoPipeline` deletes the chunks after the post. A lifecycle rule deletes all objects under `audio/` after 1 day, as a backup.

## Recap generation

Two model calls make the recap. Call A makes all fields except `transcriptSegments` in one pass for most videos. Call B adds a speaker to each SRT cue. Code, not the model, does all time conversion, checks and user fields. Most videos are 40–60 minutes. A 2-hour video is rare but must work.

**Generation rules**

1. **Stay grounded.** Every item must be supported by the transcript. An empty array is better than invented or padded items. Quotes are copied word for word. Stats include only numbers that a speaker said.
2. **Timestamps are parsed by code, not converted by the model.** Give the model the transcript with one line for each SRT cue, in the fixed form `[HH:MM:SS] text`. Make the timestamp from the cue start time. Tell the model to copy timestamps exactly as they appear. In the model output schema, timestamp fields are plain strings with a description. Do not use regex or pattern constraints: providers drop them or reject the whole response. After generation, parse each timestamp to seconds and accept loose forms (`5:30`, `1:05:30`, `[00:05:30]`, `00:05:30.500`). If a required time (a chapter boundary or a quote start) cannot be parsed, use `0`. If an optional time cannot be parsed, use `null`.
3. **Use two model calls.** Call A makes every field except `transcriptSegments`. Call B makes `transcriptSegments`. Call B keeps the SRT cue order and cue boundaries: it must not merge cues, summarize them or make new time intervals. If call B fails, the recap is still usable. Track its status separately (`idle` / `updating` / `failed`) and allow a retry.
4. **The model never sets user choices.** The model output schema has no `featured`, `hidden` or `hiddenSections` fields. After generation, set `featured` to `false`.
5. **Speakers.** Whisper gives no speaker labels. Set `speaker` only when the transcript identifies who speaks. Give the model the title, the description and the optional `speakers` list as hints. Else set `speaker` to `null`. Never guess a speaker.
6. **Deep links.** Show each time as `MM:SS` when the video is shorter than 1 hour, else as `HH:MM:SS`. Link it to `https://www.youtube.com/watch?v=<youtube_id>&t=<startSec>s`.
7. **Language.** Write the recap in English for all input languages. Keep `quotes[].text` word for word in the language of the video, and put the English text in `quotes[].translation`. Keep the SRT and `transcriptSegments` in the language of the video.
8. **Hashtags.** Make `hashtags` after `summary` and `topics`, from the content of these two fields. Give 7 tags or fewer. Make each tag one English word or one usual short form. Use the short form when one is usual: "database" gives `db`, "machine learning" gives `ml`. Split a topic of more than one word into separate tags: "database indexing" gives `db` and `indexing`. Use the same tag for the same idea in all videos, so that users can find similar posts. Give the model the short forms from `HASHTAG_ALIASES` and the 50 most used tags in the prompt (see "Known hashtags"). After generation, code replaces each known word with its short form (see "Hashtag aliases").

**Call A: model output schema**

The model returns this object. Code converts it into `VideoRecap`.

```ts
interface ModelRecapOutput {
  summary: string;            // English, 2-4 sentences
  keyTakeaways: string[];     // English, 3-7 items
  topics: string[];           // English, 3-8 short tags
  hashtags: string[];         // English, 1-7 single words or short forms, from summary and topics, no "#"
  chapters: {
    title: string;            // English
    start: string;            // timestamp copied exactly from a transcript line
    end: string;              // timestamp copied exactly from a transcript line
  }[];
  quotes: {
    text: string;             // word for word, language of the video
    translation: string | null; // English, null if the video is in English
    speaker: string | null;
    start: string;            // timestamp copied exactly from a transcript line
  }[];                        // 0-8 items
  stats: {
    value: string;            // as said, for example "40%"
    description: string;      // English
    start: string | null;     // timestamp copied exactly, or null
  }[];                        // 0-15 items
  mentions: { people: string[]; companies: string[]; books: string[]; concepts: string[] };
}
```

The prompt asks for one chapter for each 5–10 minutes of video: 5–8 chapters for 45 minutes, 10–16 chapters for 2 hours. The prompt also has the title, the channel, the description, the `speakers` list, the transcript language, `duration_sec` and the known hashtags.

**Call A: input size**

| Estimated input | Path | Typical video |
| --- | --- | --- |
| 60,000 tokens or fewer | One pass: the full transcript in one call | All videos up to 2 hours in English. Most videos up to 2 hours in other languages. |
| More than 60,000 tokens | Long path: map and reduce | Very long videos, or 2 hours in a language that uses many tokens |

- Estimate tokens as the number of characters divided by 3. This is a high estimate for English and for Cyrillic text.
- Long path, map: cut the transcript lines into windows of 30,000 estimated tokens or fewer, at cue boundaries. Run call A on each window with the same schema. Each window gives candidate chapters, quotes, stats and mentions with timestamps copied from its lines.
- Long path, reduce: one call gets the window outputs and the known hashtags, not the transcript. It writes `summary`, `keyTakeaways`, `topics` and `hashtags`, and it selects chapters, quotes and stats from the candidates. It copies the selected items without change.
- Both paths go through the same checks below, so a long video has the same grounding as a short video.

**Call A: checks after generation (code)**

1. Decode the output with the `ModelRecapOutput` schema. If the decode fails, call the model again one time with the decode error. If it fails again, fail the task, so that the workflow retries it.
2. Parse all timestamps (rule 2). Clamp each time to the range from 0 to `duration_sec`.
3. Chapters: sort them by `startSec` and remove duplicates of the same `startSec`. Keep the model `endSec` only if it is after `startSec` and not after the next chapter start. Else set `endSec` to the next chapter start, or to `duration_sec` for the last chapter.
4. Quotes: compare each quote with the transcript text, without case, punctuation or extra spaces. Remove a quote that is not in the transcript. If the model time is more than 30 seconds from the found cue, use the start of that cue.
5. Stats: keep a stat only if each number in `value` occurs in the transcript. When `startSec` has a value, the number must occur within 2 minutes of it.
6. Speakers: keep a speaker name only if it occurs in the hints or in the transcript text. Else set `null`.
7. Topics and mentions: trim them, remove empty items, and remove duplicates without case.
8. Set `translation` to `null` when the transcript language is `en`. Set `featured` to `false` on each quote.
9. Hashtags: remove `#`, make each tag lowercase, and remove all characters that are not letters, digits or `_`. Replace each tag that is a key in `HASHTAG_ALIASES` with its value. Then remove empty tags, tags with no letter and duplicates. Keep the first 7 tags.

**Hashtag aliases**

`src/lib/hashtags.ts` has the constant map `HASHTAG_ALIASES`. It makes sure that one idea always gets the same tag, so that a search by tag finds all similar posts.

- A key is a tag after the cleanup in check 9: lowercase, with `_` between words. A value is one tag.
- Code applies the map after the cleanup and before the duplicate check. Thus `database` and `db` in one recap give one `db` tag.
- The prompt gets the list of values, so that the model uses these short forms first.
- When two posts use different tags for the same idea, add an entry to the map and deploy again.

NOTE: A new entry changes only new posts. The service does not edit old posts.

The first entries of the map:

| Key | Value |
| --- | --- |
| `database`, `databases` | `db` |
| `machine_learning`, `machinelearning` | `ml` |
| `artificial_intelligence` | `ai` |
| `large_language_model`, `large_language_models`, `llms` | `llm` |
| `javascript` | `js` |
| `typescript` | `ts` |
| `kubernetes` | `k8s` |
| `postgresql` | `postgres` |
| `continuous_integration` | `ci` |
| `user_experience` | `ux` |

**Known hashtags**

The alias map fixes only the words in the map. To make the model use tags that earlier posts use, the prompt also gets the 50 most used tags.

- Before call A, a task with the name `known-hashtags` reads the tags from `recaps.recap` with this query:

```sql
SELECT j.value AS tag, COUNT(*) AS uses
FROM recaps, json_each(recaps.recap, '$.hashtags') AS j
WHERE recaps.video_id != ?1                    -- the current video, for a retry
GROUP BY j.value
ORDER BY uses DESC, tag ASC
LIMIT 50;
```

- The user message of call A gets the list, with this instruction: "Use a tag from this list when it agrees with the video. Add a new tag only when no tag in the list agrees."
- The list is a hint only. Never add a tag because it is in the list. Every tag must agree with `summary` and `topics`.
- An empty list is correct, for example for the first video. Then the prompt has no list.
- Check 9 applies to these tags too, so the list cannot bring back a word that has an alias.

NOTE: The query reads all rows of `recaps`. At 5 videos each day, this is less than 2,000 rows each year, which is sufficiently fast.

**Call B: speaker labels**

1. If the hints have no names and `mentions.people` is empty, do not call the model. Write one segment for each cue with `speaker: null`.
2. Else send the cues in batches of 300 lines in the form `<i>\t[HH:MM:SS] text`, with the hints and the last 10 labels of the previous batch.
3. The model returns `{ labels: { i: number; speaker: string | null }[] }`. It returns no text and no times.
4. Check that each `i` of the batch occurs one time. Else fail the batch. Apply rule 6 of the checks to each name.
5. Make each segment from its SRT cue: `text` from the cue, `startSec` as the cue start rounded down, `speaker` from the label.

Call B cannot merge, change or summarize text, because it never returns text.

**Telegram post**

The post is the only display in the MVP. It has, in this sequence: the title and the channel, `summary`, `keyTakeaways`, `chapters` with deep links, and `hashtags` on the last line, each with `#` before it (for example `#db #indexing`). The post does not show a section that is in `hiddenSections` or that is empty. The post has 4096 characters or fewer. To make it shorter, remove chapters from the end, then takeaways from the end.

## API contract and security

The API has five endpoints, and each one requires the header `Authorization: Bearer <API_TOKEN>`. The Go CLI already uses the first three. The new optional fields do not break the CLI.

| Method and path | Request body | Success | Errors |
| --- | --- | --- | --- |
| `POST /videos` | JSON, see below | `201 {id, status}` | `400`, `401`, `409 {id, status}` |
| `PUT /videos/:id/chunks/:n` | mp3 bytes, `Content-Type: audio/mpeg` or `application/octet-stream` | `204` | `400`, `401`, `404`, `409`, `413`, `415` |
| `POST /videos/:id/complete` | JSON: `chunk_count` | `202 {id, status}` | `400`, `401`, `404`, `409` |
| `GET /videos/:id` | none | `200`, see below | `401`, `404` |
| `POST /videos/:id/segments/retry` | none | `202 {id, segments_status}` | `401`, `404`, `409` |

**`POST /videos` body**

| Field | Required | Rule |
| --- | --- | --- |
| `video_url` | yes | Must be `https://www.youtube.com/watch?v=` plus `youtube_id` |
| `youtube_id` | yes | 11 characters, `^[A-Za-z0-9_-]{11}$` |
| `title` | yes | 1 to 300 characters |
| `duration_sec` | yes | Integer, 1 to 36000 |
| `channel` | no | 300 characters or fewer |
| `description` | no | 5000 characters or fewer. A hint for speakers and names |
| `speakers` | no | Array of 10 names or fewer, each 100 characters or fewer. A hint for host and guest names |
| `language_hint` | no | ISO 639-1 code, for example `en` or `uk`. Whisper gets it as `language` |

**`GET /videos/:id` response**

```json
{
  "id": "0193…",
  "youtube_id": "dQw4w9WgXcQ",
  "title": "…",
  "status": "done",
  "transcript": { "language": "uk", "cue_count": 812 },
  "recap": { "summary": "…", "keyTakeaways": [], "topics": [], "hashtags": [], "chapters": [], "quotes": [], "stats": [], "mentions": {} },
  "segments_status": "idle"
}
```

- `transcript` and `recap` are `null` until they exist.
- Add `?include=srt` to get `transcript.srt`. Add `?include=segments` to get `recap.transcriptSegments`. You can use both: `?include=srt,segments`.

**Rules for each endpoint**

- `POST /videos`: If `youtube_id` exists with status `failed`, set status to `uploading`, add 1 to `attempt`, update the optional fields, and return `409` with status `uploading`. The CLI then uploads again. For all other existing rows, return `409` with the current status.
- `PUT /videos/:id/chunks/:n`: Accept `n` from 0 to 99. Accept a body from 1 byte to 25 MB. Accept this request only when status is `uploading`. A second `PUT` of the same `n` replaces the object.
- `POST /videos/:id/complete`: Make sure that R2 has the objects `000` to `chunk_count - 1`. If one is missing, return `409` with the list of missing numbers. Then set status to `transcribing` and start `VideoPipeline` with the instance id `<id>-<attempt>`.
- `POST /videos/:id/segments/retry`: Return `409` if the recap does not exist or `segments_status` is `updating`. Else set `segments_status` to `updating` and start `TranscriptSegmentsJob` with the instance id `<id>-segments-<unix ms>`.
- All errors: return JSON `{ "error": "<tag>", "message": "<text>" }`.

**Security**

- The service has one token, `API_TOKEN`. Make it with `openssl rand -base64 32` (32 random bytes).
- An Effect `HttpApiMiddleware` with `HttpApiSecurity.bearer` checks the token on every endpoint. It uses a constant-time comparison.
- A missing or incorrect token gives `401` with the header `WWW-Authenticate: Bearer`. The response does not tell which of the two it was.
- The Worker reads the token with `Config.Redacted("API_TOKEN")`. The token never goes into logs or error messages.
- Cloudflare gives HTTPS on `*.workers.dev`. Do not add an HTTP route.
- NOTE: To change the token, change `.env`, deploy again, and change the Postman environment and `YT2TG_TOKEN`.

## Postman procedure: upload a video

This procedure sends one video to the service with four kinds of request. It takes approximately 5 minutes for a 1-hour video (6 chunks).

**Before you start**

- The service is deployed (task T33). You have its URL and the `API_TOKEN` value.
- `yt2tg doctor` shows `[ok]` for `yt-dlp` and `ffmpeg`.

**A. Make the chunk files**

1. Open a terminal.
2. Type `yt2tg -no-upload https://www.youtube.com/watch?v=<id>` and push Enter.
3. Wait until the CLI shows the list of chunk files.
4. Write down the title, the channel and the duration from the line under "Fetching video info".
5. Make sure that the folder `./<youtube id>/` has the files `chunk_000.mp3`, `chunk_001.mp3` and more.
6. Count the chunk files. This number is `chunk_count`.

**B. Set up Postman (one time only)**

1. In Postman, select **Import** and select `postman/yt2tg.postman_collection.json`. If you do not have the file, do steps 2 to 6.
2. Select **Environments**, then **+**. Type the name `yt2tg prod`.
3. Add the variable `baseUrl`. Set it to the service URL, without a slash at the end.
4. Add the variable `token`. Set the type to **secret**. Set it to the `API_TOKEN` value.
5. Add the variable `videoId`. Keep it empty.
6. Make a collection with the name `yt2tg`. On its **Authorization** tab, set **Auth Type** to **Bearer Token** and **Token** to `{{token}}`.
7. Select the `yt2tg prod` environment in the top-right menu.

WARNING: Do not put the token in a request URL or in a shared workspace. Keep it in the secret environment variable only.

**C. Register the video**

1. Make a request in the collection. Set the method to `POST` and the URL to `{{baseUrl}}/videos`.
2. On the **Body** tab, select **raw** and **JSON**.
3. Type the body with the values from step A4:

```json
{
  "video_url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
  "youtube_id": "dQw4w9WgXcQ",
  "title": "The video title",
  "channel": "The channel name",
  "duration_sec": 3600,
  "description": "Optional. Paste the YouTube description.",
  "speakers": ["Optional host name", "Optional guest name"],
  "language_hint": "en"
}
```

NOTE: `description`, `speakers` and `language_hint` are optional. Remove them if you do not know them. Speaker names make the speaker labels better.

4. On the **Scripts** tab, in **Post-response**, type this line:

```javascript
pm.environment.set("videoId", pm.response.json().id);
```

5. Select **Send**.
6. Make sure that the status is `201 Created`. If the status is `409`, read the `status` field: `uploading` means continue, all other values mean that the video is already in the service.

**D. Upload each chunk**

1. Make a request. Set the method to `PUT` and the URL to `{{baseUrl}}/videos/{{videoId}}/chunks/0`.
2. On the **Headers** tab, add `Content-Type` with the value `audio/mpeg`.
3. On the **Body** tab, select **binary**. Select **Select File** and select `chunk_000.mp3`.
4. Select **Send**. Make sure that the status is `204 No Content`.
5. Change the last number in the URL to `1`. Select `chunk_001.mp3`. Select **Send**.
6. Do step 5 again for each chunk file. The number in the URL must agree with the number in the file name.

NOTE: If a `PUT` fails, send the same request again. The service replaces the old object.

**E. Start the processing**

1. Make a request. Set the method to `POST` and the URL to `{{baseUrl}}/videos/{{videoId}}/complete`.
2. On the **Body** tab, select **raw** and **JSON**. Type `{ "chunk_count": 6 }`, with your number from step A6.
3. Select **Send**. Make sure that the status is `202 Accepted`.
4. If the status is `409`, read the list of missing chunks in the response. Upload these chunks (part D), then do this part again.

**F. Monitor the result**

1. Make a request. Set the method to `GET` and the URL to `{{baseUrl}}/videos/{{videoId}}`.
2. Select **Send** each minute. The `status` field changes from `transcribing` to `recapping` to `publishing` to `done`.
3. When the status is `done`, open the Telegram channel. Make sure that the post is there.
4. If the status is `failed`, read the `error` field. To try again, do part C again, then parts D and E.

NOTE: The response field `segments_status` changes separately from `status`. If it is `failed`, the recap is still correct. To make the transcript segments again, send `POST {{baseUrl}}/videos/{{videoId}}/segments/retry` with no body. Make sure that the status is `202 Accepted`.

**If you get an error**

| Status | Cause | Action |
| --- | --- | --- |
| `401` | The token is missing or incorrect. | Examine the collection **Authorization** tab and the `token` variable. |
| `404` | `videoId` is empty or incorrect. | Send the part C request again. |
| `409` on `PUT` | The status is not `uploading`. | Send `GET`. Do part C again if the status is `failed`. |
| `413` | The file is larger than 25 MB. | Make the chunks again with `-chunk 5m`. |
| `415` | The `Content-Type` header is incorrect. | Set it to `audio/mpeg`. |

## Tasks

The MVP has 34 tasks in 6 phases. Each task is small enough for one pull request. Do the tasks in the sequence of their numbers, unless "Needs" gives a different sequence. Each task has steps and a result. A task is complete only when all items in "Result" are true.

### Phase 0: Repository

- [ ] **T01. Make the repository.** Needs: none.
  1. Make a Git repository with the name `yt2tg`.
  2. Move the existing Go files into `cli/`.
  3. In `cli/go.mod`, set the module path to `github.com/<you>/yt2tg/cli`.
  4. Add a `.gitignore` with `service/.env`, `service/node_modules/`, `service/.alchemy/` and `cli/yt2tg`.

  Result: `cd cli && go test ./...` passes. `go install ./cli` from the root makes the `yt2tg` binary.
- [ ] **T02. Make the service package.** Needs: T01.
  1. Make the folder `service/`. Run `bun init` in it.
  2. Install `alchemy`, `@alchemy.run/cloudflare` and `effect` at the latest version.
  3. Set `strict: true`, `exactOptionalPropertyTypes: true` and `noUncheckedIndexedAccess: true` in `tsconfig.json`.
  4. Add the scripts `deploy`, `dev`, `destroy`, `typecheck` and `test` to `package.json`.

  NOTE: Alchemy v2 uses Effect 4. Its modules have paths such as `effect/http-api/HttpApi`. Use the paths in the Alchemy docs.

  Result: `bun run typecheck` passes with no errors.
- [ ] **T03. Connect Alchemy to Cloudflare.** Needs: T02.
  1. Run `bunx alchemy profile edit --add Cloudflare` and sign in.
  2. Make `.env.example` with the names `API_TOKEN`, `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID`, and no values.
  3. Copy `.env.example` to `.env` and add the values (see "Configuration and secrets").

  Result: `bunx alchemy profile show` shows the Cloudflare account. `.env` is not in `git status`.

### Phase 1: Infrastructure

- [ ] **T04. Declare the resources.** Needs: T03.
  1. In `src/resources.ts`, declare `Database = Cloudflare.D1.Database("Database", { migrations: "./migrations" })`.
  2. Declare `AudioBucket = Cloudflare.R2.Bucket("Audio")`.
  3. In `alchemy.run.ts`, make `Alchemy.Stack("yt2tg", { providers: Cloudflare.providers(), state: Cloudflare.state() }, …)`.
  4. In the stack, `yield*` the Worker. Return the Worker `url`.
  5. Make a temporary Worker that returns the text `ok`.

  Result: `bun run deploy` shows a URL. A `GET` to this URL returns `ok`.
- [ ] **T05. Write the first migration.** Needs: T04.
  1. Make `migrations/0001_init.sql` with the SQL in "Data model".
  2. Deploy.

  Result: `wrangler d1 execute <db> --remote --command "PRAGMA table_info(videos)"` shows 16 columns, and the tables transcripts and recaps exist. A second deploy does not apply the migration again.
- [ ] **T06. Add an expiry rule to the R2 bucket.** Needs: T04.
  1. Find the lifecycle option in the Alchemy R2 `Bucket` API reference.
  2. If the option exists, set a rule: delete objects with the prefix `audio/` after 1 day.
  3. If the option does not exist, add the rule in the Cloudflare dashboard. Write the procedure in `README.md`.

  Result: The bucket settings show the rule.

### Phase 2: Domain and services

- [ ] **T07. Write the domain schemas.** Needs: T02.
  1. In `src/domain/Video.ts`, write `VideoStatus` as a `Schema.Literal` of the six status values.
  2. Write `CreateVideo` with the fields and rules in "`POST /videos` body".
  3. Write `Video` with all columns of `videos`. Decode `speakers` from a JSON string.
  4. In `src/domain/Recap.ts`, write `RecapSectionKey`, `VideoRecap` and `ModelRecapOutput` as in "Data model" and "Recap generation".
  5. In `ModelRecapOutput`, write each timestamp field as a string with a description. Do not add a pattern.
  6. Make sure that `ModelRecapOutput` has no `featured`, `hidden` or `hiddenSections` field.
  7. Make a JSON Schema from `ModelRecapOutput` with the Effect `Schema` JSON Schema function. The prompt uses it.
  8. In `src/lib/uuidv7.ts`, write a UUIDv7 function with `crypto.getRandomValues`.

  Result: Tests decode one correct and three incorrect values for each schema. The JSON Schema has no `pattern` keyword.
- [ ] **T08. Write the timestamp and SRT functions.** Needs: T02.
  1. In `src/lib/timestamps.ts`, write `parseTimestamp(text)`. It returns whole seconds, rounded down, or `null`.
  2. Make it accept `5:30`, `1:05:30`, `[00:05:30]`, `00:05:30.500` and `00:05:30,500`.
  3. Write `formatTime(sec, durationSec)`. It gives `MM:SS` when `durationSec` is less than 3600, else `HH:MM:SS`.
  4. Write `deepLink(youtubeId, sec)`. It gives `https://www.youtube.com/watch?v=<youtube_id>&t=<sec>s`.
  5. In `src/lib/srt.ts`, write `toSrt(chunks)`, `parseSrt(text)` and `toTimestampedLines(cues)`.
  6. Make `toTimestampedLines` give one line `[HH:MM:SS] text` for each cue. It always uses `HH:MM:SS`.

  Result: Tests cover each loose form, an incorrect form, a time of more than 1 hour, and `parseSrt(toSrt(x))` equal to `x`.
- [ ] **T09. Write the grounding functions.** Needs: T08.
  1. In `src/lib/grounding.ts`, write `normalize(text)`. It removes case, punctuation and extra spaces. It keeps the letters of all alphabets.
  2. Write `findQuote(quote, cues)`. It returns the index of the cue where the quote starts, or `null`.
  3. Write `statIsGrounded(value, startSec, cues)`. It looks for each number of `value` in the cue text, within 2 minutes of `startSec` when it has a value.
  4. Write `speakerIsGrounded(name, hints, transcriptText)`.

  Result: Tests cover an exact quote, a quote across two cues, a Cyrillic quote, a quote with changed words, a correct stat and an invented stat.
- [ ] **T10. Write the D1 repositories.** Needs: T05, T07.
  1. Make `VideoRepo`, `TranscriptRepo` and `RecapRepo` as Effect services. Use `Cloudflare.D1.QueryDatabase(Database)` or `SQL.D1` from `@alchemy.run/cloudflare/D1/SQL`.
  2. In `VideoRepo`, add `insert`, `findById`, `findByYoutubeId`, `setStatus`, `resetForRetry`, `markPublished` and `markFailed`.
  3. In `TranscriptRepo`, add `save` and `find`. `save` replaces an existing row.
  4. In `RecapRepo`, add `save`, `find`, `setSegmentsStatus`, `saveSegments` and `topHashtags(limit, excludeVideoId)`. `save` replaces an existing row.
  5. Make `topHashtags` use the query in "Known hashtags".
  6. Use bound parameters in all queries. Do not put values into SQL text.
  7. Return a typed `VideoNotFound` error when a row does not exist.

  Result: Each function has a test against local D1. A delete of a `videos` row also deletes its `transcripts` and `recaps` rows. `topHashtags` sorts the tags by use count, does not count the excluded video and gives an empty list when `recaps` is empty.
- [ ] **T11. Write the `AudioStore` service.** Needs: T04.
  1. Make `AudioStore` with `Cloudflare.R2.ReadWriteBucket(AudioBucket)`.
  2. Add `putChunk(videoId, n, stream, length)`. Use the key `audio/<videoId>/<n>.mp3` with three digits.
  3. Add `listChunks(videoId)`, `getChunk(videoId, n)` and `deleteAll(videoId)`.

  Result: A test puts three chunks, lists them, reads one and deletes all.

### Phase 3: HTTP API

- [ ] **T12. Define the HTTP API.** Needs: T07.
  1. In `src/api/Errors.ts`, write tagged errors: `BadRequest` (400), `Unauthorized` (401), `VideoNotFound` (404), `Conflict` (409), `PayloadTooLarge` (413), `UnsupportedMediaType` (415).
  2. In `src/api/Api.ts`, write `VideosGroup` with the five endpoints in "API contract".
  3. Add `VideosGroup` to `YtApi` with `HttpApi.make`.

  Result: `bun run typecheck` passes. The five endpoints show in the generated OpenAPI document.
- [ ] **T13. Protect the API with a bearer token.** Needs: T12.
  1. In `src/api/Auth.ts`, write a `HttpApiMiddleware` with `HttpApiSecurity.bearer`.
  2. Read the token with `Config.Redacted("API_TOKEN")` in the Worker construction phase.
  3. Compare the tokens in constant time. Compare SHA-256 digests with `crypto.subtle.timingSafeEqual`.
  4. On failure, return `401` with `WWW-Authenticate: Bearer`.
  5. Add the middleware to `YtApi`, so that all endpoints use it.

  WARNING: Do not log the token or the `Authorization` header.

  Result: A request with no token, an empty token and an incorrect token each gets `401`. A request with the correct token gets past the middleware.
- [ ] **T14. Write the `POST /videos` handler.** Needs: T10, T13.
  1. Decode the body with `CreateVideo`. On failure, return `400`.
  2. Find the row with `findByYoutubeId`.
  3. If no row exists, insert a row with a new UUIDv7 and the optional fields. Return `201 {id, status}`.
  4. If the row has status `failed`, call `resetForRetry`, update the optional fields, and return `409 {id, status: "uploading"}`.
  5. Else, return `409 {id, status}`.

  Result: Tests cover a new video with and without the optional fields, a `failed` video and a `done` video. The Go CLI test against the service passes.
- [ ] **T15. Write the `PUT` chunk handler.** Needs: T11, T14.
  1. Check that `n` is an integer from 0 to 99. Else return `400`.
  2. Check that `Content-Type` is `audio/mpeg` or `application/octet-stream`. Else return `415`.
  3. Check that `Content-Length` is from 1 to 26214400 (25 MB). Else return `413`.
  4. Check that the video exists and its status is `uploading`. Else return `404` or `409`.
  5. Stream the request body into `AudioStore.putChunk`. Do not read the full body into memory.

  Result: A 3 MB upload returns `204`, and the object is in R2. The same `PUT` again replaces the object.
- [ ] **T16. Write the `complete` handler.** Needs: T15, T19.
  1. Decode `{ chunk_count }`. It must be an integer from 1 to 100.
  2. Check that the status is `uploading`. Else return `409`.
  3. List the chunks. If a number from 0 to `chunk_count - 1` is missing, return `409` with the missing numbers.
  4. Set `chunk_count` and the status `transcribing`.
  5. Start `VideoPipeline` with `create({ id: "<id>-<attempt>", params: { videoId } })`.
  6. Return `202 {id, status}`.

  Result: With all chunks, the workflow instance starts. With one chunk missing, the response names it, and the status stays `uploading`.
- [ ] **T17. Write the `GET /videos/:id` handler.** Needs: T14.
  1. Read `videos`, `transcripts` and `recaps` in one query with two `LEFT JOIN` clauses on the video id.
  2. Return the response in "`GET /videos/:id` response". Return `transcript` and `recap` as `null` when they do not exist.
  3. Add `transcript.srt` only for `include=srt`. Add `recap.transcriptSegments` only for `include=segments`.
  4. Return `recap` as a JSON object, not as a string.

  Result: Tests cover a video with no recap, a video with a recap, and each `include` value. An unknown id gives `404`.
- [ ] **T18. Write the `segments/retry` handler.** Needs: T17, T25.
  1. Return `404` if the video does not exist.
  2. Return `409` if the recap does not exist or `segments_status` is `updating`.
  3. Set `segments_status` to `updating` and clear `segments_error`.
  4. Start `TranscriptSegmentsJob` with the instance id `<id>-segments-<unix ms>`.
  5. Return `202 {id, segments_status}`.

  Result: A retry after a failed call B makes `transcript_segments` and sets `segments_status` to `idle`. A second retry during `updating` gives `409`.

### Phase 4: Workflow

- [ ] **T19. Write the `VideoPipeline` frame.** Needs: T10, T11.
  1. In `src/workflow.ts`, write `VideoPipeline` with `Cloudflare.Workflow<VideoPipeline>()("VideoPipeline", …)`. Its input is `{ videoId }`.
  2. Put all I/O in `Cloudflare.Workflows.task("<name>", effect, { retries })`. Give each task a stable name.
  3. Write one task for each status change. The sequence is `transcribing`, `recapping`, `publishing`, `done`.
  4. Catch the final error outside the tasks. In a task with the name `mark-failed`, call `markFailed` with the error tag and message.
  5. In the Worker construction phase, `yield* VideoPipeline` to bind it.

  NOTE: Code outside a task runs again on each replay. Do not put D1, R2, AI or fetch calls outside a task.

  Result: A workflow with empty steps moves a test video from `transcribing` to `done`. A step that always fails sets `failed` and an `error` text.
- [ ] **T20. Write the `Transcriber` service and the transcribe tasks.** Needs: T19.
  1. Bind Workers AI with `Cloudflare.Workers.AI()` and provide `Cloudflare.Workers.AIBinding`.
  2. For each chunk `n`, write a task with the name `transcribe-<n>`.
  3. In the task, read the chunk from R2 and encode it as base64.
  4. Call `ai.run("@cf/openai/whisper-large-v3-turbo", { audio, vad_filter: true })`. Add `language` when `language_hint` has a value.
  5. Return only the segments (`start`, `end`, `text`), the detected language and the chunk duration.
  6. Set the retries to 3, with a 10-second delay and exponential backoff.

  NOTE: Find the exact output field names in the model `schema-output.json`. The return value of a task must be less than 1 MiB. One 10-minute chunk gives approximately 15 KB of segments.

  Result: A 25-minute test file gives 3 tasks. Each task returns segments with times from 0 to its chunk duration.
- [ ] **T21. Save the transcript.** Needs: T08, T20.
  1. In a task with the name `save-transcript`, call `toSrt` with the chunk outputs.
  2. For chunk `n`, add the sum of the durations of chunks 0 to `n - 1` to each time. Remove empty segments.
  3. Make one SRT cue for each Whisper segment. Do not merge segments.
  4. Set `language` to the language that most chunks detected. Use `language_hint` if no chunk detected a language.
  5. Write the SRT, `language` and `cue_count` with `TranscriptRepo.save`. Set the status `recapping`.

  NOTE: Do not return the SRT from the task. Its size can be more than 1 MiB.

  Result: The SRT of a 25-minute test file opens in VLC with no error. Its times agree with the audio within 1 second.
- [ ] **T22. Write call A, the one-pass path.** Needs: T07, T21.
  1. In `src/lib/prompts.ts`, write the call A prompt and `PROMPT_VERSION = 1`.
  2. In `src/lib/hashtags.ts`, write `HASHTAG_ALIASES` with the entries in "Hashtag aliases".
  3. Put generation rules 1, 2, 4, 5, 7 and 8, the item counts and the JSON Schema of `ModelRecapOutput` in the system message.
  4. Put the list of `HASHTAG_ALIASES` values in the system message.
  5. In a task with the name `known-hashtags`, call `RecapRepo.topHashtags(50, videoId)`.
  6. Put the title, channel, description, `speakers`, language, `duration_sec`, the known hashtags and the timestamped lines in the user message.
  7. In `src/services/Recapper.ts`, call `ai.run(RECAP_MODEL, …)` with a temperature of 0.2 and 8000 output tokens or more. Use the request format on the model page of the Workers AI catalog.
  8. Take the JSON object from the response text and decode it (check 1 in "Recap generation").
  9. Run the call in a task with the name `recap-a`, with 2 retries.

  Result: A 45-minute fixture and a 2-hour English fixture both decode on the one-pass path. A prompt test shows the known hashtags when the list has items, and no list when it is empty.
- [ ] **T23. Write call A, the long path.** Needs: T22.
  1. Estimate the input tokens as the number of characters divided by 3.
  2. If the estimate is more than 60,000, use the long path. Put the limit in a constant, so that a test can change it.
  3. Cut the timestamped lines into windows of 30,000 estimated tokens or fewer. Cut only between cues.
  4. Run one task `recap-map-<w>` for each window. Use the call A prompt with the text "part \<w> of \<total>".
  5. Run one task `recap-reduce`. Give it the window outputs and the known hashtags, not the transcript.
  6. Tell the reduce call to copy chapters, quotes and stats from the window outputs without change.

  Result: A test with a limit of 5,000 forces the long path on a 45-minute fixture. Its recap passes all checks of T24.
- [ ] **T24. Check and save the recap.** Needs: T09, T22.
  1. Write a pure function `finalizeRecap(output, cues, video, language)`. It does checks 2 to 9 in "Recap generation".
  2. In `src/lib/hashtags.ts`, write `normalizeHashtags(tags)` for check 9. `finalizeRecap` calls it.
  3. Make it return `VideoRecap` without `transcriptSegments`, and a list of the items it removed.
  4. In a task with the name `save-recap`, save the recap with `RECAP_MODEL` and `PROMPT_VERSION`.
  5. Log the number of removed items for each section. Do not log the transcript.
  6. Set the status `publishing`.

  Result: Tests cover each check: chapter sort and end times, an invented quote, a quote with an incorrect time, an invented stat, a guessed speaker, duplicate topics, `featured: false`, and hashtags with `#`, capitals, spaces, duplicates or more than 7 items. A test shows that `database` and `db` in one recap give one `db` tag.
- [ ] **T25. Write call B, the `TranscriptSegmentsJob` workflow.** Needs: T08, T09, T10.
  1. In `src/segmentsWorkflow.ts`, write `TranscriptSegmentsJob` with the input `{ videoId }`.
  2. In the task `segments-start`, set `segments_status` to `updating`.
  3. If there are no hint names and `mentions.people` is empty, make all segments with `speaker: null`. Do not call the model.
  4. Else run one task `labels-<b>` for each batch of 300 cues. Call `SEGMENTS_MODEL` with JSON mode and the schema `{ labels: { i, speaker }[] }`.
  5. Check that each `i` of the batch occurs one time. Apply the speaker check to each name.
  6. In the task `segments-save`, make each segment from its SRT cue and its label. Call `saveSegments` and set `segments_status` to `idle`.
  7. On a final error, set `segments_status` to `failed` and write `segments_error`. Do not change the video status.

  Result: The segment count is equal to `cue_count`. Each segment text is equal to its cue text. A forced failure sets `failed` and the video stays `done`.
- [ ] **T26. Write the `Telegram` service and the publish task.** Needs: T24.
  1. Read `TELEGRAM_BOT_TOKEN` with `Config.Redacted` and `TELEGRAM_CHAT_ID` with `Config.string`.
  2. In `src/lib/telegramFormat.ts`, make the HTML post as in "Telegram post". Use `formatTime` and `deepLink` for each chapter.
  3. Put `#` before each item of `hashtags`. Put all hashtags on the last line of the post, with one space between them.
  4. Do not show a section that is in `hiddenSections` or that is empty.
  5. Escape `&`, `<` and `>` in all text from YouTube and from the model.
  6. Make sure that the post has 4096 characters or fewer. Remove chapters from the end, then takeaways from the end. Do not cut inside an HTML tag.
  7. Send it with `sendMessage` and `parse_mode: "HTML"`. If Telegram returns `429`, wait for `parameters.retry_after` seconds and try again.
  8. Save `message_id` with `markPublished`.

  WARNING: Do not log the bot token. The token is part of the Telegram API URL.

  Result: Tests cover escape, the 4096 limit, a hidden section, chapter links in `MM:SS` and in `HH:MM:SS`, and the hashtag line. A test post shows in a test channel.
- [ ] **T27. Write the last pipeline tasks.** Needs: T25, T26.
  1. Write a task with the name `cleanup`. It calls `AudioStore.deleteAll(videoId)`.
  2. Set the status to `done` after the cleanup.
  3. Write a task with the name `start-segments`. It starts `TranscriptSegmentsJob` with the instance id `<id>-segments-<attempt>`.

  Result: After a full run, R2 has no objects under `audio/<videoId>/`. The status is `done`, and `segments_status` goes to `idle` with segments.
- [ ] **T28. Make the recap test set.** Needs: T24.
  1. Select three videos: an English talk of approximately 45 minutes, a non-English interview of approximately 60 minutes, and an English video of approximately 2 hours.
  2. Save their SRT files in `test/fixtures/`, so that the test does not need Whisper.
  3. Write the script `bun run eval:recap`. It runs call A and `finalizeRecap` on each fixture.
  4. Make the script check that the recap decodes, chapters start at 0 and end at `duration_sec`, and the chapter count agrees with the prompt.
  5. Make the script check that the English fields have no letters from the video alphabet when the video is not in English.
  6. Make the script print the removed items, so that you can examine them.

  Result: The script passes for all three fixtures. Fewer than 20% of quotes and stats are removed by the checks.

### Phase 5: CLI

- [ ] **T29. Send the optional fields from the CLI.** Needs: T14.
  1. In `ytdlp.go`, read `description` from the `yt-dlp -J` output.
  2. Cut `description` to 5000 characters. Do not cut inside a UTF-8 character.
  3. Add the flag `-speakers "Name One,Name Two"`. Send it as the `speakers` array.
  4. Add the flag `-lang <code>`. Send it as `language_hint`.
  5. Do not send a field that has no value.

  Result: A CLI test with the fake service shows the four new fields in the request. A request without the flags has no `speakers` and no `language_hint`.
- [ ] **T30. Make sure that the CLI agrees with the service.** Needs: T17, T29.
  1. Run `yt2tg <url>` against the `dev` stage with a 5-minute video.
  2. Stop the CLI during the upload. Run it again. Make sure that it continues with the same id.
  3. Run it again after the status is `done`. Make sure that it stops and downloads nothing.
  4. Set a failed status by hand. Run it again. Make sure that it uploads again.
  5. If a step does not agree with "API contract", change the service, not the CLI.

  Result: The four checks pass. The CLI `README.md` shows the service URL format and the new flags.

### Phase 6: Delivery

- [ ] **T31. Make the Postman collection.** Needs: T18.
  1. Make the collection and the environment as in "Postman procedure", parts B to F.
  2. Add the post-response script that sets `videoId`.
  3. Add the request `POST {{baseUrl}}/videos/{{videoId}}/segments/retry`.
  4. Export the collection to `postman/yt2tg.postman_collection.json`.
  5. Make sure that the exported file has no token value.

  Result: A new Postman workspace can import the file and do the procedure.
- [ ] **T32. Write the integration test.** Needs: T27.
  1. Start the local service with `bun run dev`.
  2. Use a 90-second test mp3 in `service/test/fixtures/`. Make it with `ffmpeg` from text-to-speech audio with two named speakers.
  3. Send the requests in sequence, with `speakers` set. Use a Telegram test channel.
  4. Wait until the status is `done` or `failed`. The time limit is 5 minutes.
  5. Wait until `segments_status` is `idle` with segments, or `failed`.

  Result: `bun test` passes. The test channel has one new post. `GET ?include=segments` returns segments.
- [ ] **T33. Deploy to production.** Needs: T28, T30, T31, T32.
  1. Run `bunx alchemy deploy --stage prod`.
  2. Do the Postman procedure with one short video.
  3. Run `yt2tg <url>` with one 45-minute video and one 2-hour video.
  4. Look at the Workers AI usage in the Cloudflare dashboard. Make sure that it agrees with the cost estimate.

  Result: Three posts show in the production channel. All videos have the status `done` and a recap that passes the checks.
- [ ] **T34. Write the README.** Needs: T33.
  1. Write how to install the CLI, `yt-dlp`, `ffmpeg` and `deno`.
  2. Write how to deploy the service and set the secrets.
  3. Add a link to the Postman procedure.
  4. Write how to change the API token, `RECAP_MODEL` and `SEGMENTS_MODEL`.
  5. Write how to add an entry to `HASHTAG_ALIASES`.

  Result: A person with no knowledge of the project can deploy it and post one video.

## Configuration, cost and open questions

The service needs three secrets and three variables. Alchemy reads them from `service/.env` at deploy time.

| Name | Kind | Where to get it | Read with |
| --- | --- | --- | --- |
| `API_TOKEN` | secret | `openssl rand -base64 32` | `Config.Redacted` |
| `TELEGRAM_BOT_TOKEN` | secret | @BotFather, `/newbot` | `Config.Redacted` |
| `TELEGRAM_CHAT_ID` | variable | `getUpdates` after a post in the channel (`-100…`) | `Config.string` |
| `RECAP_MODEL` | variable | Default `@cf/openai/gpt-oss-120b` (128,000-token context, 16,384 output tokens) | `Config.string` |
| `SEGMENTS_MODEL` | variable | Default `@cf/meta/llama-3.3-70b-instruct-fp8-fast` (on the JSON mode list) | `Config.string` |

On your computer, the CLI reads `YT2TG_API_URL` (the Worker URL) and `YT2TG_TOKEN` (the same value as `API_TOKEN`).

**Cost estimate.** The Workers Paid plan is $5 per month. Whisper large-v3-turbo is $0.0005 per audio minute, and the first approximately 210 minutes each day are free. For call A, a 1-hour English video is approximately 15,000 input and 4,000 output tokens. At the listed $0.35 and $0.75 per million tokens, that is approximately $0.01 for each video. Five 1-hour videos each day cost approximately $1.30 per month for Whisper and $1.50 per month for call A. Call B costs less than call A. D1, R2 and Workflows stay in the included quota at this volume.

**Open questions**

- [ ] How is a transcript handled when it is longer than the model context? The spec has a proposal: one pass up to 60,000 estimated tokens, else map and reduce. Confirm the limits after T28.
- [ ] Where are the user-set fields (`featured`, `hidden`, `hiddenSections`) stored and edited? The spec stores them in `recaps.recap`. No endpoint or UI edits them in the MVP.
- [ ] Are recaps made again when the transcript changes? If they are, are the user choices kept?
- [ ] Which Workers AI model to use for `RECAP_MODEL`? The default is `gpt-oss-120b`. It is not on the JSON mode list, so the spec decodes and checks its output in code.
- [ ] Does the Alchemy R2 `Bucket` have a lifecycle option (T06)?
- [ ] Is a `POST /videos/:id/retry` endpoint necessary, or is the `failed` reset in `POST /videos` sufficient?
- [ ] Must the Telegram post show quotes, stats or mentions? When a user sets `featured` later, must the post change with `editMessageText`?
- [ ] `quotes[].translation` is an addition to the given recap shape. Keep it, or use a different field?
- [ ] Must the first chapter start at 0 and the last chapter end at `duration_sec`? T28 checks this now.
- [ ] Which language is the transcript language when a video has more than one language? T21 uses the language of most chunks.

## Sources

- [Alchemy: Effect HTTP API on Cloudflare](https://alchemy.run/cloudflare/apis/effect-http-api/)
- [Alchemy: Workers](https://alchemy.run/cloudflare/compute/workers/)
- [Alchemy: Workflows](https://alchemy.run/cloudflare/compute/workflows)
- [Alchemy: D1](https://alchemy.run/cloudflare/data/d1)
- [Alchemy: R2](https://alchemy.run/cloudflare/data/r2)
- [Alchemy: Workers AI](https://alchemy.run/cloudflare/ai/workers-ai)
- [Alchemy: Secrets and env](https://alchemy.run/cloudflare/security/secrets-env)
- [Alchemy: Setup](https://alchemy.run/cloudflare/setup)
- [Cloudflare: Workers AI pricing](https://developers.cloudflare.com/workers-ai/platform/pricing/)
- [Cloudflare: Workers AI JSON mode](https://developers.cloudflare.com/workers-ai/json-mode/)
- [Cloudflare: whisper-large-v3-turbo](https://developers.cloudflare.com/workers-ai/models/whisper-large-v3-turbo)
- [gpt-oss-120b on Workers AI: context and price listing](https://pi.dev/models/cloudflare-ai-gateway/workers-ai-cf-openai-gpt-oss-120b)
- [Cloudflare: D1 limits](https://developers.cloudflare.com/d1/platform/limits/)
