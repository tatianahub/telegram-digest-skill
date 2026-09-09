---
name: telegram-digest
description: "Builds a digest of new posts from open Telegram channels and publishes it as an artifact — no API key, bot, or account login required. Must be run inside a Claude Project (or another location with persistent storage) so it can remember where it left off. On first run it checks for that, asks which channels to track and which language to use, then remembers everything for next time. Use for requests like 'make a digest', 'set up a Telegram channel digest', 'what's new in the channels', 'summarize Telegram channels'."
---

# Telegram Channel Digest

Reads new posts from public channels, builds a digest out of them, and remembers where it stopped so it doesn't repeat itself next time.

## 0. First run

If the config (see section 1) isn't found yet, this is the first run. Do the following:

1. **Check the environment first.** If you're not running inside a Claude Project (or another location with persistent documents/connected storage), say so upfront, before asking anything else: without a Project the config can't survive between sessions, so every run would start from scratch. Ask the user to open this chat inside a Project (or point to a connected document store like Google Drive) before continuing. Don't proceed to channel setup until there's a real place to store the config.
2. Ask for the list of channels: usernames without `@` and without `https://t.me/`, one per line or comma-separated.
3. Ask which language to write the digest in (don't just infer it silently) — offer the language the user is writing in as the default, but let them pick something else. Save the answer in the config header (see section 1) so it's remembered for every future run, including unattended Scheduled Task runs where no one is there to answer.
4. Confirm where to store the config — by default suggest "in the current project, document `telegram_channels.conf`".
5. For each channel: `WebFetch https://t.me/s/<channel>/99999`, grab the maximum id from the page (no need to analyze the posts yet).
6. Write the full config: `channel_name=MAX` for each channel, plus the chosen digest language and an initialization-date comment on top.
7. State explicitly: "Setup done — tracking N channels from this point on, digest language: <language>. The first digest will be built on the next run" — don't try to build a digest from history that doesn't exist.

If the user asks to add/remove a channel from an already-configured setup, don't re-initialize everything: for a new channel, repeat steps 5–6 for it only, leave the other lines and the saved language untouched.

## 1. Config

Format (independent of where the file physically lives):

```
# updated: 2026-09-09
# digest_language: English
channel_name=last_post_id
```

`digest_language` is set once during first-time setup (section 0) and reused on every run — including unattended Scheduled Task runs, where there's no one to ask.

`last_post_id` is the id of the last post already included in a digest. New posts are all posts with `id > last_post_id`.

Where to look: first check the location the user named during setup (see section 0). If it's a project document, read via `project_read` / write via `project_write`. If it's a file, use regular file tools. Never make up a `last_post_id` value — if there's no config, that's a sign of a first run (see section 0), not a reason to improvise.

## 2. Fetching posts

Only `WebFetch` against `https://t.me/s/<channel>/<id>`. No curl/wget/python requests — the web restrictions forbid it.

**Important quirks of these pages (verified in practice):**

- The bare `https://t.me/s/<channel>` page without an anchor gives unreliable results: the small model parsing the page regularly returns an id from somewhere in the middle instead of the actual maximum. Never rely on it.
- An anchor `https://t.me/s/<channel>/<id>` shows ~10 posts before and ~10 after the anchor.
- An anchor known to be larger than the latest id (e.g. `https://t.me/s/<channel>/99999`) collapses to the newest posts. **This is the main way to find the current maximum.**
- Responses are cached for 15 minutes per URL. Re-fetching the same URL won't refresh the data — change the anchor.
- Gaps in numbering (861, 862, 864…) are normal: deleted and service messages. Don't treat this as an error and don't try to reach missing ids.

**Per-channel algorithm:**

1. `WebFetch https://t.me/s/<channel>/99999` — find the current `MAX` and see the latest posts at the same time.
2. If `MAX == last_post_id` — no new posts, the channel goes into the "no new posts" list.
3. Otherwise walk forward from `last_post_id`: `WebFetch https://t.me/s/<channel>/<cursor>`, starting with `cursor = last_post_id`. Each fetch returns a batch of posts; set `cursor` = the maximum id from the response and repeat until you reach `MAX`.
4. Active channels can accumulate dozens of posts a day — that's 3–5 iterations. This is expected; see it through.

**WebFetch prompt** — ask for id and content together so you don't have to go back:

> List every post on this page with ID greater than `<last_post_id>`. For each: the numeric post ID, the time, and a detailed summary of what the post says (3–5 sentences, keep technical specifics, names, versions, links mentioned). Then state the largest ID on the page. Format at end: MAX: `<number>`

Fetch channels in parallel — 5–6 `WebFetch` calls in one block, not one at a time.

## 3. The digest

Language: use whatever was set during first-time setup (`digest_language` in the config, section 1). If it's ever missing for some reason, ask instead of guessing. Tone: informal, easy to scan, pleasant to read. Keep technical details — don't water them down.

**Structure:**

1. **By channel** — each channel as its own block, in the order from the config. Inside, each new post as its own paragraph.
2. **Post summary** — medium level of detail: not a one-liner, not a full retelling either. Aim for 3–5 sentences. What happened, why it matters, specifics (versions, names, numbers).
3. **Link at the end of the post summary**, format: `([number](https://t.me/s/channel/number))`
4. **Channels with no new posts** — as a list at the end, one line.
5. **Terms** — the final section. Only genuinely hard-to-understand things: acronyms, niche technologies, non-obvious names. Use judgment about what's worth explaining; don't explain common knowledge.

Don't group by topic and don't filter by category — cover everything significant that happened in the channels.

## 4. Publishing

The digest is an HTML artifact.

- If an artifact-styling skill is available (e.g. `artifact-design`), load it before laying out the page. If not available, build a clean, self-contained HTML page with no external dependencies.
- Write the file to the working directory, then use `Artifact`/`present_files` with its path.
- Title with the date, e.g. "Digest for September 8".
- Channels work well as collapsible blocks with a post count; a short summary up top (how many channels, how many posts).
- Every run is a new artifact (a new file path). A given day's digest is never overwritten.

## 5. Updating the config

**Only after the digest is built and published.** If a fetch failed for some channel, leave its `last_post_id` untouched so posts aren't lost, and say so in your reply.

For every channel with new posts, set `last_post_id = MAX`. Update the collection-date comment. Write the whole file/document back the same way you read it (see section 1) — to the same location determined during setup.

## 6. Automatic scheduled delivery (optional)

The skill doesn't run anything on a schedule by itself — scheduling is set up once on the platform side (Scheduled Tasks in Claude Cowork). If the user asks how to get the digest automatically every morning without a manual request:

1. Make sure setup (section 0) is already done and the config lives somewhere that survives between sessions (a project document, not a temporary file in the working directory — temp files don't persist between Scheduled Task runs).
2. Explain that they need to create a Scheduled Task in the Cowork interface with a prompt like "Build the Telegram channel digest" and the desired cadence — the user does this themselves through the UI; the skill doesn't automate it for them.

## 7. In your reply

One or two sentences: how many channels and posts were included, anything notably new, which channels were silent. The digest itself lives in the artifact — don't duplicate it in chat.
