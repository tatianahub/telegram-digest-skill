---
name: telegram-digest
description: "Builds a digest of new posts from open Telegram channels and publishes it as an artifact — no API key, bot, or account login required. State lives in a project doc (claude/telegram_channels.conf), so channels and positions survive between runs and the digest can be scheduled to run automatically. With no project doc to read or write it works as a one-off: no memory between sessions. Use for requests like 'make a digest', 'set up a Telegram channel digest', 'what's new in the channels', 'summarize Telegram channels'."
---

# Telegram Channel Digest

Reads new posts from public channels, builds a digest out of them, and remembers where it stopped so it doesn't repeat itself next time.

## 0. First run

The state lives in a project doc (section 1). Start every run by reading it: `project_read` on `claude/telegram_channels.conf`.

**It read back** — this is a normal run, go to section 2.

**It's missing** — branch on whether a human is in the room.

**Unattended run** (a scheduled task started this session; there is no live user message in it). Do **not** create the config and do **not** build a digest. A schedule could not have been set up before the first setup ran, so a missing doc means it existed and is gone: a write that failed, someone deleted it, a different project. Writing a fresh one here loses the position silently — the task keeps firing every day and reporting "no new posts", which is indistinguishable from a quiet day in the channels, so nobody notices for a week. Instead: send a notification saying the config is missing and no digest was built, and stop.

**Interactive run** (a person is here). This is setup:

1. Ask for the list of channels: usernames without `@` and without `https://t.me/`, one per line or comma-separated. Never invent a channel name — with no list, ask for one.
2. Ask which language to write the digest in — offer the language the user is writing in as the default, but let them pick something else.
3. For each channel, find the current maximum id using the method in section 2 (no need to analyze the posts yet).
4. Write the doc **immediately**: `project_write` on `claude/telegram_channels.conf`, with those ids as `last_post_id`. This is the one case where the write comes before a published digest.
5. State explicitly: "Setup done — tracking N channels from this point on, digest language: <language>. The first digest will be built on the next run." Everything published before setup counts as already read; don't try to build a digest out of history that isn't tracked.
6. If the user wants this to run automatically, point them to section 6.

**No project doc tools at all** (a plain chat with no `project_read`/`project_write`): say so plainly — this run will be a one-off, nothing will be remembered after the conversation ends, and there's no point asking where to keep a config. Then ask for channels and language, build a single digest, and skip section 5.

If the user asks to add or remove a channel in an already-configured setup, don't re-initialize everything: repeat steps 3–4 for the new channel only, and leave the other lines and the saved language untouched.

## 1. Config

The single source of truth is a **project doc** at `claude/telegram_channels.conf`, read with `project_read` and written with `project_write`:

```
# updated: 2026-09-09
# digest_language: English
channel_name=last_post_id
```

**Not a file in the working directory.** The working directory is ephemeral: every scheduled run starts in a fresh container, and anything written to disk goes away with it — including a config written by the previous run. Keep the config in a local file and section 0 will fire on every single run, re-record the current ids and produce an empty digest forever, all while looking like it works. A project doc outlives the session and is visible to every chat inside the project.

`digest_language` is set once during first-time setup (section 0) and reused on every run — including unattended scheduled runs, where there's no one to ask.

`last_post_id` is the id of the last post already included in a digest. New posts are all posts with `id > last_post_id`.

Never make up a `last_post_id` value — a missing doc is a signal (see section 0), not a reason to improvise.

Local files are still the right place for scratch work: the digest HTML you assemble and any throwaway scripts. Only the state belongs in the project doc.

## 2. Fetching posts

Only `WebFetch` against `https://t.me/s/<channel>/<id>`. No curl/wget/python requests — the web restrictions forbid it.

**The prompt is not optional.** Ask for permalinks, never for "ids":

> For EVERY post on this page, read its permalink URL exactly as written (format `https://t.me/<channel>/NNNN`) — do NOT infer or count IDs. Output one line per post: the ID, the time, and a 3–4 sentence summary keeping technical specifics, names, versions, CVE numbers. Then on the last line: `MAX: <largest id on page>`

Asked instead for "every post with an ID greater than N", the small model parsing the page doesn't read ids at all — it numbers the posts by counting and drifts 1–12 positions. Every id in the digest is then wrong, and so is the `last_post_id` written back at the end. Don't reword this prompt into the counting form.

**Quirks of these pages (verified over many runs):**

- The bare `https://t.me/s/<channel>` page with no anchor is unreliable: the parser regularly returns an id from the middle of the page instead of the maximum.
- An anchor `https://t.me/s/<channel>/<id>` shows roughly 10 posts before and 10 after it.
- **Don't use a deliberately huge anchor like `/99999`.** It does not reliably collapse to the newest posts — it returns an arbitrary older window. Anchor near an id you already know instead.
- **Windows go stale, often.** A given anchor frequently returns posts well below the recorded `last_post_id`, and a nearer anchor is sometimes fresher than a farther one. Expect most anchors in a run to lie; a single window is never evidence.
- Responses are cached 15 minutes per URL. Re-fetching the same URL won't refresh anything — change the anchor.
- Gaps in numbering (861, 862, 864…) are normal: deleted messages, service messages, and the extra permalinks an album generates. Don't treat them as errors and don't chase missing ids.

**Per-channel algorithm** (3–4 fetches typically, up to 7 for a disputed channel):

1. Anchor at `last_post_id + 2..4`.
2. A second anchor at `+10..20` — always, whatever step 1 returned.
3. A third anchor above the highest id seen so far — required before calling a channel silent. Two agreeing windows that both stop at or below `last_post_id` are not enough.
4. If any anchor reports `MAX > last_post_id`, walk forward from it in `+8..18` steps while the maximum keeps growing, then confirm that maximum with a second independent window. For a channel with only one or two new posts this confirmation is mandatory — a lone window showing a new id is wrong often enough to matter.
5. If the first window starts above `last_post_id + 1`, close the gap with an anchor between the recorded id and the lowest id you saw. If two such anchors come back stale, treat the missing ids as real gaps.
6. Batch 10–13 fetches per block rather than one at a time. If the proxy returns HTTP 429, don't repeat the same URL — take a different anchor.

**Two checks before trusting a number:**

- **Phantom anchor.** If the reported `MAX` equals the id in the URL you fetched and no post with that id appears in the body of the answer, it isn't a real maximum — the model guessed it from the URL. Confirm with another anchor.
- **Markup drift.** A window sometimes returns the right posts under the wrong ids, shifted by one. Before trusting it, check its content against what the config said those ids were last time. Content agreement beats a majority vote of windows.

Never move `last_post_id` on a single unconfirmed window, and never move it down: an id that goes backwards resurfaces old posts as "new" in the next issue.

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

**Only after the digest is built and published.** The one exception is first-time setup in section 0, where the write comes first.

For every channel with new posts, set `last_post_id = MAX`. If a fetch failed for some channel, leave its `last_post_id` untouched so posts aren't lost, and say so in your reply. Update the collection-date comment.

Write the whole file back with `project_write` on the same path. Project docs have no patch operation, only full replacement, so the payload must carry the entire file: every comment and every channel line, including the ones that didn't change. A line you leave out is deleted.

`project_write` has no version check either — last write wins. So read the config once at the start of the run, and don't hand-edit the doc while a run is in flight: the final write will overwrite it.

In a one-off run with no project doc (section 0), skip this step — there's nothing to update.

## 6. Scheduling automatic delivery

This is the main way to actually use this skill day to day — a one-off chat digest works, but it forgets everything afterward. The skill doesn't schedule itself; that's set up once on the platform side:

1. Make sure setup (section 0) is done and the project doc exists. Don't schedule anything before that: the first unattended run would hit a missing config and stop by the rule in section 0.
2. Create a scheduled task with a prompt like "Build the Telegram channel digest" and the cadence you want (e.g. daily at 8 AM). The user does this through the UI; the skill doesn't automate it for them.
3. Point the schedule at the project that holds the config. A run started in a different project reads a different doc, or none at all.

## 7. In your reply

One or two sentences: how many channels and posts were included, anything notably new, which channels were silent. The digest itself lives in the artifact — don't duplicate it in chat.
