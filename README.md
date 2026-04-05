# Claude Code Discord 配置文档

通过 Claude Code 原生 Discord 插件，将 Discord Bot 接入 Claude，运行在订阅额度下。

## 架构概览

```
Discord 用户 (DM / 服务器) → Discord Bot → Claude Code (--channels) → Claude API
```

- Bot 通过 Claude Code 的 `discord@claude-plugins-official` 插件连接
- Claude Code 以 `--channels` 模式运行，持续监听 Discord 消息
- 后台通过 launchd + tmux 自动管理，开机自启

---

## 文件结构

| 文件 | 说明 |
|------|------|
| `~/.claude/channels/discord/.env` | Bot Token |
| `~/.claude/channels/discord/access.json` | 访问控制（白名单/配对策略） |
| `~/.claude/channels/discord/channel_config.json` | Discord 频道自定义配置（模型、系统提示词） |
| `~/Library/LaunchAgents/ai.claude.discord.plist` | launchd 服务定义 |
| `~/.local/bin/claude-discord-channel` | 启动脚本 |
| `~/.claude/logs/discord.log` | 标准输出日志 |
| `~/.claude/logs/discord.err.log` | 错误日志 |

---

## 初次安装

### 1. 安装依赖

```bash
# Bun（插件运行环境）
curl -fsSL https://bun.sh/install | bash

# tmux（后台 TTY 管理）
brew install tmux
```

### 2. 安装 Claude Code Discord 插件

在 Claude Code 会话中运行：

```
/plugin install discord@claude-plugins-official
/reload-plugins
```

### 3. 创建 Discord Bot

1. 前往 [Discord Developer Portal](https://discord.com/developers/applications) → New Application
2. 进入 **Bot** 页面 → 开启 **Message Content Intent**
3. **Reset Token** 复制 Token（仅显示一次）
4. 进入 **OAuth2 → URL Generator**，勾选 `bot` scope，权限勾选：
   - View Channels / Send Messages / Send Messages in Threads
   - Read Message History / Attach Files / Add Reactions
5. 用生成的 URL 将 Bot 邀请到你的服务器

### 4. 配置 Token

```bash
# 写入 Token
mkdir -p ~/.claude/channels/discord
echo "DISCORD_BOT_TOKEN=你的Token" > ~/.claude/channels/discord/.env
chmod 600 ~/.claude/channels/discord/.env
```

### 5. 安装启动脚本

```bash
cat > ~/.local/bin/claude-discord-channel << 'EOF'
#!/bin/bash
/opt/homebrew/bin/tmux kill-session -t claude-discord 2>/dev/null
/opt/homebrew/bin/tmux new-session -d -s claude-discord \
  '/Users/你的用户名/.local/bin/claude --channels plugin:discord@claude-plugins-official'
sleep 4
/opt/homebrew/bin/tmux send-keys -t claude-discord "" Enter
while /opt/homebrew/bin/tmux has-session -t claude-discord 2>/dev/null; do
  sleep 5
done
EOF
chmod +x ~/.local/bin/claude-discord-channel
```

### 6. 配置 launchd 自启服务

参考 [`launchd/ai.claude.discord.plist`](launchd/ai.claude.discord.plist)，复制到 `~/Library/LaunchAgents/`，然后：

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.claude.discord.plist
```

### 7. 首次配对

第一次运行后，在 Discord 里 DM Bot，它会回复配对码。在 Claude Code 中运行：

```
/discord:access pair <配对码>
```

配对成功后锁定策略：

```
/discord:access policy allowlist
```

---

## 日常管理

### 查看 Bot 状态

```bash
tmux attach -t claude-discord   # 查看运行状态（Ctrl+B D 退出不关闭）
launchctl list | grep ai.claude.discord  # 查看服务状态
```

### 手动重启

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.claude.discord.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.claude.discord.plist
```

### 查看日志

```bash
tail -f ~/.claude/logs/discord.err.log
```

### 停止服务

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.claude.discord.plist
```

---

## Discord Slash Commands

Bot 支持以下 slash commands（服务器立即可用，DM 约需 1 小时同步）：

| 命令 | 说明 |
|------|------|
| `/status` | 查看当前模型和系统提示词配置 |
| `/model` | 切换 Claude 模型（sonnet / opus / haiku），重启后生效 |
| `/system` | 设置自定义系统提示词，立即对后续消息生效 |
| `/reset` | 清除所有自定义配置 |

配置保存在 `~/.claude/channels/discord/channel_config.json`。

---

## 访问控制

配置文件：`~/.claude/channels/discord/access.json`

```json
{
  "dmPolicy": "allowlist",
  "allowFrom": ["你的Discord用户ID"],
  "groups": {},
  "pending": {}
}
```

| 字段 | 说明 |
|------|------|
| `dmPolicy` | `pairing`（配对模式）/ `allowlist`（仅白名单）/ `disabled` |
| `allowFrom` | 允许发送消息的 Discord 用户 ID 列表 |

### 常用命令

```
/discord:access pair <code>       # 批准配对
/discord:access policy allowlist  # 锁定为白名单模式
/discord:access allow <user_id>   # 直接添加用户
/discord:access remove <user_id>  # 移除用户
```
