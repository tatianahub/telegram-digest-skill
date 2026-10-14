# Telegram Digest — a plugin/skill for Claude

> No API keys, no bots, no phone-number login. Just name the channels you want to follow.

I spent a long time looking for a simple way to get a Telegram digest that works without a Telegram MCP server and keeps running while my computer is off. I couldn't find one, so I built this.

The skill builds a digest of new posts from public Telegram channels and publishes it as an artifact page.

For convenience, set up a scheduled task that runs once a day and sends the result to your inbox through a connected email MCP (for example, Gmail).

## What you need

- Claude Cowork (desktop app, macOS or Windows) on a paid plan (Pro, Max, Team, or Enterprise).
- Public Telegram channels: each one must be reachable at `t.me/s/<channel>` without signing in.

## Installation

Plugins (and marketplace installs) work only in Cowork and Claude Code, not in regular chat on claude.ai.

**Cowork (desktop app):**

Settings → Customize → Plugins → Add → Add Marketplace → enter `tatianahub/telegram-digest-skill` (or the full GitHub URL) → install `telegram-digest` from the list.

## First-time setup

1. Create a separate project in Claude and give it a clear name, for example `Scheduled digest`. The digest keeps its channel list and reading position in a doc inside this project, so every digest run (manual or scheduled) must happen in this same project.
2. Open a Cowork chat in that project and tell Claude something like:

   > Set up a Telegram digest

From there the skill will:

1. Ask which channels to track (usernames without `@` or `https://t.me/`, e.g. `it_secur, xakep_ru`) and which language to write the digest in.
2. Record each channel's current starting point in the project doc and confirm that setup is complete.

> **Note:** on first setup the skill does **not** pull the channels' full history. Tracking starts from this point forward.

## Building a digest

You can stop here and simply come back to this chat whenever you want a digest of everything posted since the last run. Just ask:

> Build the digest

Every run produces a fresh artifact with the latest posts.

## Running on a schedule (recommended)

To get the digest automatically, ask in the same project:

> Build Scheduled every day at 20.30 gmt +3

If you have an email MCP connected (e.g. Gmail), ask for delivery as well:

> Build Scheduled every day at 20.30 gmt +3 and result sent to <your email>@gmail.com

A few things to know:

- Run setup first. A scheduled run that can't find the project doc stops and reports that the config is missing instead of building an empty digest.
- The scheduled task must be attached to the same project as the digest config. Check this once in the task's settings.
- Claude may shift a schedule set exactly on the hour or half hour a few minutes earlier (e.g. 20:15 instead of 20:30) to avoid peak-time delays. Ask for the exact time if it matters.
- If nothing new was posted, you still get a short "no new posts" message.

## Add or remove a channel

In the project where the digest is set up, just write:

> Add channel some_channel to the digest

> Remove channel some_channel from the digest

New channels start tracking from their latest post; other channels keep their positions.

## Why no authorization is needed

The usual way to get Telegram data is the Bot API (create a bot, add it to every channel) or an MTProto client with phone-number login and a stored session. This skill instead reads `t.me/s/<channel>`, the public web preview that Telegram serves to any browser with no login at all.

That gets you:

- **Zero setup friction:** name your channels and it works. No bot to create, no phone-number login.
- **No secrets to store:** no API token, session, or seed phrase that could leak.
- **Nothing to revoke:** the worst-case content of the config doc is a list of public post ids, not access to someone's account.

The trade-off: it only works with public channels that have their web preview enabled. Private channels and closed groups can't be read this way; those would need the Bot API with full authorization. For most use cases, this is enough.

## Repository structure

```
.
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest
│   └── marketplace.json    # lets the repo act as its own marketplace
├── skills/
│   └── telegram-digest/
│       ├── SKILL.md
│       └── page-template.html   # fixed page design for the digest artifact
├── LICENSE
└── README.md
```

## Feedback

If the digest gets something wrong or misses posts, just tell Claude what went wrong in chat. The behavior is defined by the `SKILL.md` text and is easy to adjust. Pull requests are welcome.
