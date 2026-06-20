---
name: hermes-tweet
description: |
  Use Hermes Tweet when Hermes Agent needs X/Twitter search, public reads,
  account context, monitors, trends, media, giveaway draws, or explicitly
  approved posting/actions through Xquik.
version: 0.1.6
author: Xquik
license: MIT
metadata:
  hermes:
    tags: [hermes-agent, xquik, twitter, x, social-media, automation]
    related_skills: [social-listening, launch-monitoring, creator-research]
---

# Hermes Tweet

Hermes Tweet is a native Hermes Agent plugin for X/Twitter automation through Xquik. It installs as the `hermes-tweet` Python package and exposes the `hermes-tweet` toolset.

## Install

```bash
hermes plugins install Xquik-dev/hermes-tweet --enable
```

If Hermes installed the plugin but left it disabled, run:

```bash
hermes plugins enable hermes-tweet
```

Set `XQUIK_API_KEY` in the Hermes runtime environment or `~/.hermes/.env`. Do not paste API keys into prompts, logs, issues, PR comments, or tool arguments.

Keep account-changing actions disabled unless the session intentionally needs them:

```bash
export HERMES_TWEET_ENABLE_ACTIONS=false
```

## Tool Flow

1. Use `tweet_explore` to find a catalog-listed `/api/v1/...` endpoint.
2. Use `tweet_read` for catalog entries marked `GET` and `action:false`, including safe monitor, extraction, media, and draw reads.
3. Use `tweet_action` only for catalog entries that require writes, private account changes, webhooks, or explicit `action:true` operations.

## When To Use

- Search X/Twitter for current public signal.
- Read tweet details, replies, profiles, followers, media, and trends.
- Monitor launches, support mentions, creators, brands, or communities.
- Audit giveaways, follower eligibility, replies, and draw evidence.
- Draft or publish posts only after the operator approves the exact action.
- Run the same enabled toolset from Hermes Desktop, TUI, CLI, remote gateway, or cron sessions.

## Safety Rules

- Never ask for or reveal API keys, passwords, cookies, signing keys, or TOTP secrets.
- Never pass credentials in tool arguments.
- Do not guess endpoint paths. Use `tweet_explore`.
- Keep `tweet_action` disabled for unattended or read-only workflows.
- For remote gateway profiles, install and configure Hermes Tweet on the remote Hermes host where plugin tools execute.

## Verify

```bash
hermes plugins list
hermes tools list
```

Confirm `hermes-tweet` is enabled, `tweet_explore` appears without `XQUIK_API_KEY`, and authenticated read tools appear after the key is configured.

## Links

- GitHub: https://github.com/Xquik-dev/hermes-tweet
- PyPI: https://pypi.org/project/hermes-tweet/
- Hermes Agent: https://github.com/NousResearch/hermes-agent
