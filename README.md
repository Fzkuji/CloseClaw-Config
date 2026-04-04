<p align="center">
  <img src="logo.png" alt="CloseClaw-Config" width="128" />
</p>

# CloseClaw-Config

Route OpenClaw conversations through Claude Code (ACP) instead of direct Anthropic API calls.

## Background

Anthropic announced that **subscription credits no longer cover third-party API usage** (effective April 4, 2026). OpenClaw calls the Anthropic API directly, which means it's now billed as "extra usage."

However, **Claude Code still runs under subscription credits**. By routing OpenClaw messages through Claude Code via the ACP (Agent Client Protocol) runtime, you can continue using your subscription quota.

```
User (Discord/Telegram/etc.) → OpenClaw → ACP → Claude Code → response → OpenClaw → User
```

## Prerequisites

- [OpenClaw](https://github.com/openclaw/openclaw) installed and running
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed (`claude --version`)
- An active Anthropic subscription (Pro/Team/Enterprise)
- A messaging channel configured (Discord, Telegram, etc.)

## Step-by-Step Setup

### 1. Enable the ACPX plugin

```bash
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true
```

### 2. Enable ACP runtime

```bash
openclaw config set acp.enabled true
openclaw config set acp.backend acpx
openclaw config set acp.defaultAgent claude
openclaw config set acp.allowedAgents '["claude","codex"]'
```

### 3. Set permissions for non-interactive mode

ACP sessions run without a TTY, so permission prompts need to be auto-approved:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
```

> ⚠️ This auto-approves all file writes and shell commands. Use with caution.

### 4. Define an ACP agent in your config

Edit `~/.openclaw/openclaw.json` and add a new agent to `agents.list`:

```json
{
  "id": "claude-acp",
  "runtime": {
    "type": "acp",
    "acp": {
      "agent": "claude",
      "backend": "acpx",
      "mode": "persistent"
    }
  }
}
```

### 5. Bind your chat to the ACP agent

Add a binding entry in `bindings[]` to route your conversation through Claude Code:

#### Discord DM example

```json
{
  "type": "acp",
  "agentId": "claude-acp",
  "match": {
    "channel": "discord",
    "accountId": "default",
    "peer": {
      "kind": "dm",
      "id": "<YOUR_DM_CHANNEL_ID>"
    }
  },
  "acp": {
    "mode": "persistent"
  }
}
```

#### Telegram example

```json
{
  "type": "acp",
  "agentId": "claude-acp",
  "match": {
    "channel": "telegram",
    "accountId": "default",
    "peer": {
      "kind": "dm",
      "id": "<YOUR_TELEGRAM_USER_ID>"
    }
  },
  "acp": {
    "mode": "persistent"
  }
}
```

#### Discord guild channel example

```json
{
  "type": "acp",
  "agentId": "claude-acp",
  "match": {
    "channel": "discord",
    "accountId": "default",
    "peer": {
      "kind": "group",
      "id": "<CHANNEL_ID>"
    }
  },
  "acp": {
    "mode": "persistent"
  }
}
```

### 6. Restart the gateway

```bash
openclaw gateway restart
```

### 7. Verify

Send a message in your bound channel. Check logs for errors:

```bash
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep -i acp
```

You should see `acpx runtime backend ready` and no `Failed to spawn` errors.

## Quick Alternative: On-Demand Binding

Instead of editing config files, you can bind from chat:

```
/acp spawn claude --bind here
```

This binds the current conversation to Claude Code. Use `/acp close` to unbind.

## Finding Your Channel ID

- **Discord**: Enable Developer Mode (Settings → Advanced), right-click the channel → Copy Channel ID
- **Telegram**: Use `@userinfobot` or check the URL in Telegram Web

## Trade-offs

| | Direct API (default) | Claude Code ACP |
|---|---|---|
| **Response style** | General assistant | Coding-focused agent |
| **OpenClaw tools** | ✅ All available | ❌ Not available (web_search, browser, image, etc.) |
| **Latency** | Normal | Higher (extra ACP layer) |
| **Billing** | Extra usage (per-token) | Subscription credits |
| **Context** | Continuous conversation | Per-session (persistent mode helps) |

## Reverting

To go back to direct API mode:

1. Remove the ACP binding from `bindings[]`
2. Optionally disable ACP:
   ```bash
   openclaw config set acp.enabled false
   openclaw gateway restart
   ```

Or if you saved a backup:
```bash
cp ~/.openclaw/openclaw-backup.json ~/.openclaw/openclaw.json
openclaw gateway restart
```

## Disclaimer

This approach depends on Anthropic continuing to include Claude Code under subscription credits. If Anthropic changes their policy, this workaround may stop working. Consider using [OpenRouter](https://openrouter.ai/) as a more stable alternative.

## License

MIT
