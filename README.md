# Telegram Digest — a plugin/skill for Claude

> No API keys, no bots, no phone-number login needed. All the user has to do is name the channels they're interested in.

Builds a digest of new posts from public Telegram channels and publishes it as an artifact page. Remembers where it left off, so it never repeats itself.

For convenience, we recommend setting up a Scheduled Task to run once a day and send the result to your inbox via a connected email MCP.

## What you need

Two ways to use this, with different requirements:

**For an automatic daily digest** — needs [Claude Cowork](https://support.claude.com/en/articles/13345190) (desktop app, macOS or Windows) on a paid plan (Pro, Max, Team, or Enterprise), with the conversation attached to a Project. The state lives in a doc inside that project, and that — not file access — is what lets the skill remember channels between runs: every scheduled run starts in a fresh cloud container, so a config written to disk would already be gone by the next one.

**For a one-off digest in regular chat** — works on any plan, including Free. [Skills](https://support.claude.com/en/articles/12512180) are available on Free too — just enable Code execution and file creation in Settings → Capabilities. The catch: with no project doc to write to, nothing is remembered between conversations, so you'd ask for a fresh digest each time.

Either way, channels must be public (reachable at `t.me/s/<channel>` without signing in).

## Installation

Plugins (and marketplace installs) only work in Cowork and Claude Code — not in regular chat on claude.ai. Pick the path that matches your setup:

**Claude Code (terminal):**

```
/plugin marketplace add tatianahub/telegram-digest-skill
```

then `/plugin install telegram-digest@telegram-digest-skill`.

**Cowork (desktop app):** Customize → Plugins → Add → Add Marketplace → enter `tatianahub/telegram-digest-skill` (or the full GitHub URL) → install `telegram-digest` from the list.

**Regular chat on claude.ai (including Free):** plugins don't work here — install the skill directly instead. Go to Settings → Skills, enable Code execution and file creation if it isn't already, then upload the `skills/telegram-digest/` folder as a ZIP (it contains `SKILL.md` and `page-template.html` — the skill needs both). Remember: outside a project there's nowhere to keep the state, so this only gives you one-off digests with no memory between conversations (see "What you need" above).

## First-time setup

Just tell Claude something like:

> Set up a Telegram digest for channels: it_secur, xakep_ru

From there the skill will:
1. read the project doc that holds its state — and say plainly if there is none, so you know this run won't be remembered;
2. ask which channels to track and which language to write the digest in;
3. record each channel's current starting point in the project doc and confirm setup is complete.

Note: on first setup the skill does **not** pull the channel's full history — tracking starts from this point forward.

## Everyday use

Once set up, just ask:

> Build the digest

Every run produces a fresh artifact with the latest posts, and the config moves forward automatically.

## Add or remove a channel

> Add channel some_channel to the digest

The skill only adds the new channel — it leaves the rest of the config alone.

## Automatic delivery every morning

This is the recommended way to actually use the skill day to day, and it's handled by the platform, not the skill:

1. Make sure setup (above) is done and the project doc exists. Don't create the schedule before that: the first unattended run would find no config and stop rather than build an empty digest.
2. In Claude Cowork, open **Scheduled** → create a task with a prompt like "Build the Telegram channel digest" and the cadence you want (e.g. daily at 8 AM). Point it at the same project that holds the config — a run started elsewhere reads a different doc, or none.
3. Cowork will run the task on its own, using this skill — no need to type into chat manually.

Scheduled Tasks are currently available on paid Cowork plans; check current availability on your account. Conveniently, they can be set to run in the cloud without needing your device on — the digest gets built even if your machine is off. That works precisely because the state lives in a project doc rather than on your disk.

## Why no authorization is needed

The usual way to get Telegram data is the Bot API (create a bot, add it to every channel) or an MTProto client with phone-number login and a stored session. This skill instead reads `t.me/s/<channel>` — the public web preview Telegram serves with no login at all, to any browser.

That gets you:

- **Zero setup friction** — name your channels and it works, no bot to create, no phone-number login.
- **No secrets to store** — no API token, session, or seed phrase that could leak.
- **Nothing to revoke** — the worst-case content of the config doc is a list of public post ids, not access to someone's account.

The trade-off: it only works with public channels that have their web preview enabled. Private channels and closed groups can't be read this way — those would need the Bot API with full authorization. But for most use cases, this is enough.

## Limitations

- Only works with public channels (the `t.me/s/...` page, no login).
- Doesn't store or publish content from private/closed channels.
- The config is a plain-text project doc with post ids, no personal data.

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

If the digest gets something wrong or misses posts, just tell Claude what went wrong in chat — the behavior is defined by `SKILL.md` text and is easy to adjust. Pull requests welcome.
