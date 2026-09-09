# Telegram Digest — a plugin/skill for Claude

> No API keys, no bots, no phone-number login needed. All the user has to do is name the channels they're interested in.

Builds a digest of new posts from public Telegram channels and publishes it as an artifact page. Remembers where it left off, so it never repeats itself.

For convenience, we recommend setting up a Scheduled Task to run once a day and send the result to your inbox via a connected email MCP.

## What you need

- Web-browsing access (`WebFetch`) — that's how the skill reads `t.me/s/<channel>`.
- A place to store and version the config: a document in a Claude Project (recommended) or a connected document store (Google Drive, etc.). Without it, the digest starts from scratch every time.
- Channels must be public (reachable at `t.me/s/<channel>` without signing in).

## Installation

**As a plugin via Claude Code** (recommended):

```
/plugin marketplace add tatianahub/telegram-digest-skill
```

then select `telegram-digest` from the list and install it.

**Manually, as a standalone skill** — if your environment doesn't support plugins: copy `skills/telegram-digest/SKILL.md` into your skills folder.

## First-time setup

Just tell Claude something like:

> Set up a Telegram digest for channels: it_secur, xakep_ru

From there the skill will:
1. ask where to store the config (by default it suggests a `telegram_channels.conf` document in the current project);
2. go through each channel and record the current starting point;
3. confirm that setup is complete.

Note: on first setup the skill does **not** pull the channel's full history — tracking starts from this point forward.

## Everyday use

Once set up, just ask:

> Build the digest

Every run produces a fresh artifact with the latest posts, and the config moves forward automatically.

## Add or remove a channel

> Add channel some_channel to the digest

The skill only adds the new channel — it leaves the rest of the config alone.

## Automatic delivery every morning (no manual request)

This is handled by the platform, not the skill:

1. Make sure setup (above) is already done and the config lives in a project document, not a temporary file.
2. In Claude Cowork, open **Scheduled** → create a task with a prompt like "Build the Telegram channel digest" and the cadence you want (e.g. daily at 8 AM).
3. Cowork will run the task on its own, using this skill — no need to type into chat manually.

Scheduled Tasks are currently available on paid Cowork plans; check current availability on your account. Conveniently, Scheduled Tasks can be set to run in the cloud without needing your device on — the digest gets built even if your machine is off.

## Why no authorization is needed

The usual way to get Telegram data is the Bot API (create a bot, add it to every channel) or an MTProto client with phone-number login and a stored session. This skill instead reads `t.me/s/<channel>` — the public web preview Telegram serves with no login at all, to any browser.

That gets you:

- **Zero setup friction** — name your channels and it works, no bot to create, no phone-number login.
- **No secrets to store** — no API token, session, or seed phrase that could leak.
- **Nothing to revoke** — the worst-case content of the config is a list of public post ids, not access to someone's account.

The trade-off: it only works with public channels that have their web preview enabled. Private channels and closed groups can't be read this way — those would need the Bot API with full authorization. But for most use cases, this is enough.

## Limitations

- Only works with public channels (the `t.me/s/...` page, no login).
- Doesn't store or publish content from private/closed channels.
- The config is plain text with post ids, no personal data; if you store it in a shared project document, keep in mind who else has access to it.

## Repository structure

```
.
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest
│   └── marketplace.json    # lets the repo act as its own marketplace
├── skills/
│   └── telegram-digest/
│       └── SKILL.md
├── LICENSE
└── README.md
```

## Feedback

If the digest gets something wrong or misses posts, just tell Claude what went wrong in chat — the behavior is defined by `SKILL.md` text and is easy to adjust. Pull requests welcome.
