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
3. **The post link** lives in the mono column on the left (`.post-tag`), not at the end of the text — see the template in section 4.
4. **Channels with no new posts** — as a list at the end, one line.
5. **Terms** — the final section. Only genuinely hard-to-understand things: acronyms, niche technologies, non-obvious names. Use judgment about what's worth explaining; don't explain common knowledge.

Don't group by topic and don't filter by category — cover everything significant that happened in the channels.

## 4. Publishing

The digest is an HTML artifact. **The page design is fixed** — it's a recognizable issue that comes out every day, not a new design task.

**How to do it:**

1. Read `page-template.html` next to this file. It contains the reference CSS and placeholder markup: all text in angle brackets (`<channel>`, `<id>`, `<HH:MM>`, `<N>`) is a placeholder for this run's data.
2. Copy the template to the working directory as `digest-<YYYY-MM-DD>.html` and replace the placeholders with today's issue. No angle-bracket placeholders may remain in the published page; remove the hint comments too. The template's visible labels (masthead title, tally labels, section headings, footer) are in English — translate them into `digest_language`.
3. Publish with `Artifact`/`present_files` using the file path. On first publish use the 📰 favicon and `news` as the icon, where the tool supports them.

**Don't touch the CSS.** Tokens, fonts and selectors are part of the signature look: a cool grey palette with a single blue accent, Archivo + Source Serif 4 + JetBrains Mono, a single 760px column, a sticky masthead with the date and counters. No need to load `artifact-design` or any other styling skill — the design decisions are already made and recorded in the template.

**What changes from issue to issue:** `<title>` and `.masthead-meta`, `h1`, the `.standfirst` intro, the `.tally` cells, the `<details class="chan">` blocks, the `.silent` and `.glossary` sections, the footer.

**Reading time** is the fourth `.tally` cell. Calculate it from the finished file, before publishing:

```bash
python3 -c "
import re
t=open('digest-<YYYY-MM-DD>.html').read()
t=re.sub(r'<style.*?</style>','',t,flags=re.S)
t=re.sub(r'<!--.*?-->','',t,flags=re.S)
t=re.sub(r'<[^>]+>',' ',t)
t=re.sub(r'\s+',' ',t).strip()
print(len(t)//1300+1)
"
```

Stripping `<style>` and comments is mandatory — otherwise the CSS gets counted and the result is three times too high. 1300 characters per minute is the pace of careful reading of technical text; an issue with 47 posts comes out at ~14 minutes. Write the result with a tilde: `~14`. If the file doesn't exist yet and you need an estimate — 20 seconds per post plus 2 minutes for the intro, glossary and caveats.

**Markup rules that are easy to break:**

- Channel blocks are `<details class="chan" open>`. Exactly `open`: the page must read in full right after loading; collapsing is only a convenience.
- A post is a `.post` grid: `.post-tag` on the left with the number link and time, `.post-body` on the right. Wrap the first sentence of the summary in `<span class="lead">`.
- A post whose content couldn't be retrieved gets `.post-body.unavailable` with an explanation of why.
- Channel order follows the config. Within a channel, posts go in ascending id order, except when a channel clearly has a top story: it can be moved to the top of the block.
- If a channel has a caveat (failed fetch, id mismatch, duplicates), add a `.note` paragraph in the "No new posts" section. No caveats — remove the paragraph entirely.

Every run is a new artifact (a new file path). A given day's digest is never overwritten.

## 5. Updating the config

**Only if you have a config file (Cowork) and only after the digest is built and published.** If a fetch failed for some channel, leave its `last_post_id` untouched so posts aren't lost, and say so in your reply. In a one-off chat run (no config file), skip this step — there's nothing to update.

For every channel with new posts, set `last_post_id = MAX`. Update the collection-date comment. Write the file back to the same working directory.

## 6. Scheduling automatic delivery (Cowork)

This is the main way to actually use this skill day to day — a one-off chat digest works, but it forgets everything afterward. The skill doesn't schedule itself; that's set up once on the platform side:

1. Make sure setup (section 0) is done in Cowork and a config file exists.
2. In the Cowork interface, open **Scheduled** → create a task with a prompt like "Build the Telegram channel digest" and the desired cadence (e.g. daily at 8 AM). The user does this through the UI; the skill doesn't automate it for them.

## 7. In your reply

One or two sentences: how many channels and posts were included, anything notably new, which channels were silent. The digest itself lives in the artifact — don't duplicate it in chat.
