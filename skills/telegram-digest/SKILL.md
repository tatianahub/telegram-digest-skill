---
name: telegram-digest
description: "Builds a digest of new posts from open Telegram channels and publishes it as an artifact — no API key, bot, or account login required. In Claude Cowork it saves a config file and remembers channels between runs, so it can be scheduled to run automatically. In a regular chat it works as a one-off: no memory between sessions. Use for requests like 'make a digest', 'set up a Telegram channel digest', 'what's new in the channels', 'summarize Telegram channels'."
---

# Telegram Channel Digest

Reads new posts from public channels, builds a digest out of them, and remembers where it stopped so it doesn't repeat itself next time.

## 0. First run

If the config (see section 1) isn't found yet, this is the first run.

**Step 1 — check whether you can save a real file.** If you have working file tools (Cowork: `bash_tool`/`create_file`), you can remember channels between runs — go to step 2. If not (a regular chat, no file access), say so plainly: this run will be a one-off, nothing will be remembered after this conversation ends, and there's no point asking where to store a config. Then just ask for channels and language and build a single digest — skip the rest of this section.

**Step 2 — set up persistent tracking (Cowork only):**

1. Ask for the list of channels: usernames without `@` and without `https://t.me/`, one per line or comma-separated.
2. Ask which language to write the digest in — offer the language the user is writing in as the default, but let them pick something else.
3. For each channel: `WebFetch https://t.me/s/<channel>/99999`, grab the maximum id from the page (no need to analyze the posts yet).
4. Write the config file (see section 1 for format) to the working directory.
5. State explicitly: "Setup done — tracking N channels from this point on, digest language: <language>. The first digest will be built on the next run" — don't try to build a digest from history that doesn't exist.
6. If the user wants this to run automatically, point them to section 6 (Scheduled Tasks).

If the user asks to add/remove a channel from an already-configured setup, don't re-initialize everything: repeat steps 3–4 for the new channel only, leave the other lines and the saved language untouched.

## 1. Config

A plain file, `telegram_channels.conf`, in the working directory:

```
# updated: 2026-09-09
# digest_language: English
channel_name=last_post_id
```

`digest_language` is set once during first-time setup (section 0) and reused on every run — including unattended Scheduled Task runs, where there's no one to ask.

`last_post_id` is the id of the last post already included in a digest. New posts are all posts with `id > last_post_id`.

Read and write it with regular file tools. Never make up a `last_post_id` value — if there's no config file, that's a sign of a first run (see section 0), not a reason to improvise.

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

**Only if you have a config file (Cowork) and only after the digest is built and published.** If a fetch failed for some channel, leave its `last_post_id` untouched so posts aren't lost, and say so in your reply. In a one-off chat run (no config file), skip this step — there's nothing to update.

For every channel with new posts, set `last_post_id = MAX`. Update the collection-date comment. Write the file back to the same working directory.

## 6. Scheduling automatic delivery (Cowork)

This is the main way to actually use this skill day to day — a one-off chat digest works, but it forgets everything afterward. The skill doesn't schedule itself; that's set up once on the platform side:

1. Make sure setup (section 0) is done in Cowork and a config file exists.
2. In the Cowork interface, open **Scheduled** → create a task with a prompt like "Build the Telegram channel digest" and the desired cadence (e.g. daily at 8 AM). The user does this through the UI; the skill doesn't automate it for them.

## 7. In your reply

One or two sentences: how many channels and posts were included, anything notably new, which channels were silent. The digest itself lives in the artifact — don't duplicate it in chat.
