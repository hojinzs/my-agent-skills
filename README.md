# my-agent-skills

A Claude Code plugin marketplace for reusable agent skills.

## Available Plugins

| Plugin | Purpose |
| --- | --- |
| `codex-image-gen` | Generate and review images with Codex image generation. |
| `coolify-cli` | Manage Coolify deployments and resources from the CLI. |
| `hermes-tweet` | Use X/Twitter through Xquik. |
| `supertest` | Plan and run QA workflows with HTML reports. |

## Install

Add this marketplace in Claude Code:

```text
/plugin marketplace add hojinzs/my-agent-skills
```

Then install the plugin you need:

```text
/plugin install <plugin-name>@my-agent-skills
```

For example:

```text
/plugin install supertest@my-agent-skills
```
