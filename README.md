# OpenClaw Secure Installation on Mac Studio — Troubleshooting Guide

> **Date:** March 14, 2026
> **System:** Mac Studio (Apple Silicon)
> **Method:** Docker (for maximum isolation)

---

## Prerequisites

- **Docker Desktop for Mac — Apple Silicon** (free for personal use)
  - Download from [docker.com](https://www.docker.com/products/docker-desktop/)
  - Choose "Download for Mac – Apple Silicon" (first option)
  - Install: open the `.dmg`, drag Docker to Applications, launch it
  - Wait for the whale icon in the menu bar to show "Docker Desktop is running"

- **Node.js 24** (recommended) or Node 22 LTS (22.16+) — the Docker setup handles this inside the container

---

## Step-by-Step Installation

### Step 1: Clone the OpenClaw repo

```bash
git clone https://github.com/openclaw/openclaw.git ~/Documents/openclaw
cd ~/Documents/openclaw
```

### Step 2: Run the Docker setup script

```bash
bash docker-setup.sh
```

This creates two folders on your Mac:
- `~/.openclaw/` — configuration, memory, API keys
- `~/openclaw/workspace/` — files the agent can access

### Step 3: Walk through the onboarding wizard

The script will prompt you for:

| Prompt | Recommended Choice |
|--------|--------------------|
| Model provider | Skip for now (configure DeepSeek manually later) |
| Gateway binding | **Loopback** (critical for security) |
| Skills | Skip for now |
| Hooks | Skip for now |

### Step 4: Switch to pre-built Docker image

The source-built image has a known bug where the Telegram plugin fails with `Cannot find module`. Switch to the pre-built image to fix this.

Edit your `.env` file:

```bash
vim ~/Documents/openclaw/.env
```

Add this line:

```
OPENCLAW_IMAGE=alpine/openclaw:latest
```

Then pull and restart:

```bash
cd ~/Documents/openclaw
docker compose pull
docker compose up -d
```

---

## Step 5: Configure DeepSeek as Model Provider

### 5a. Get a DeepSeek API key

Sign up at [platform.deepseek.com](https://platform.deepseek.com) and generate an API key.

### 5b. Set the environment variable

> **Tip:** If you have multiple API keys for different machines/projects, use a descriptive variable name (e.g., `DEEPSEEK_API_KEY_MY_MACHINE`) instead of the generic `DEEPSEEK_API_KEY`. Just make sure the name matches **exactly** in all three places: `.zshrc`, `openclaw.json`, and `.env`.

```bash
echo 'export DEEPSEEK_API_KEY_MY_MACHINE="sk-your-actual-key-here"' >> ~/.zshrc
source ~/.zshrc
```

Verify:
```bash
echo $DEEPSEEK_API_KEY_MY_MACHINE
```

### 5c. Add the `.env` file for Docker Compose

Docker containers don't inherit your shell environment. You **must** add the key to the `.env` file in the project directory. This file **must** also include `OPENCLAW_CONFIG_DIR` and `OPENCLAW_WORKSPACE_DIR` or the container will fail.

```bash
cd ~/Documents/openclaw
cat > .env << EOF
OPENCLAW_CONFIG_DIR=/Users/$(whoami)/.openclaw
OPENCLAW_WORKSPACE_DIR=/Users/$(whoami)/openclaw/workspace
DEEPSEEK_API_KEY_MY_MACHINE=sk-your-actual-key-here
OPENCLAW_IMAGE=alpine/openclaw:latest
EOF
```

> **Critical:** If you only put the API key in `.env` and omit the other variables, Docker Compose fails with `invalid spec: :/home/node/.openclaw: empty section between colons`.

### 5d. Add DeepSeek provider to the config

```bash
vim ~/.openclaw/openclaw.json
```

Add the `models` block at the **top level** (sibling to `agents`, not inside it):

```json
"models": {
  "mode": "merge",
  "providers": {
    "deepseek": {
      "baseUrl": "https://api.deepseek.com",
      "apiKey": "${DEEPSEEK_API_KEY_MY_MACHINE}",
      "api": "openai-completions",
      "models": [
        {
          "id": "deepseek-chat",
          "name": "DeepSeek Chat (V3.2)",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 128000,
          "maxTokens": 8192
        },
        {
          "id": "deepseek-reasoner",
          "name": "DeepSeek Reasoner (V3.2)",
          "reasoning": true,
          "input": ["text"],
          "contextWindow": 128000,
          "maxTokens": 65536
        }
      ]
    }
  }
}
```

### 5e. Update the agents section

Make sure the model names include the `deepseek/` provider prefix:

```json
"agents": {
  "defaults": {
    "model": {
      "primary": "deepseek/deepseek-reasoner"
    },
    "models": {
      "deepseek/deepseek-reasoner": {},
      "deepseek/deepseek-chat": {}
    },
    "workspace": "/home/node/.openclaw/workspace"
  }
}
```

> **Why `deepseek-reasoner` over `deepseek-chat`?** OpenClaw needs around 16k tokens of context. `deepseek-chat` caps at 8k which isn't enough. `deepseek-reasoner` handles the larger context and is better for multi-step tasks.

### 5f. Restart

```bash
cd ~/Documents/openclaw
docker compose stop
docker compose up -d
docker compose ps
```

---

## Step 6: Harden the Configuration

Edit the config file:

```bash
vim ~/.openclaw/openclaw.json
```

> **Note:** The file is `openclaw.json` (not `.json5`). Use double quotes for all keys/values.

Add or adjust these settings inside the top-level `{}`:

```json
"gateway": {
  "bind": "loopback",
  "controlUi": {
    "allowedOrigins": ["http://127.0.0.1:18789"],
    "allowInsecureAuth": true
  }
},
"exec": {
  "ask": "on"
},
"fileSystem": {
  "allowedPaths": [
    "~/openclaw/workspace"
  ],
  "blockedPaths": [
    "~/.ssh",
    "~/.aws",
    "~/.config",
    "~/.*",
    "~/Documents",
    "~/Desktop",
    "~/Downloads"
  ]
},
"tools": {
  "profile": "minimal",
  "elevated": {
    "allowFrom": []
  }
}
```

---

## Step 7: Set up Telegram

### 7a. Create a bot with BotFather

1. Open Telegram, search for **@BotFather** (verify the blue checkmark)
2. Send `/newbot`
3. Choose a display name (e.g., "My AI Assistant")
4. Choose a username ending in `bot` (e.g., `my_openclaw_bot`)
5. BotFather replies with a token like `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`

> **Security:** Keep this token secret. If it leaks, use `/revoke` in BotFather to regenerate.

### 7b. Add the token to your config

```bash
vim ~/.openclaw/openclaw.json
```

Add or update the `channels` section at the top level:

```json
"channels": {
  "telegram": {
    "enabled": true,
    "botToken": "123456789:ABCdefGHIjklMNOpqrsTUVwxyz",
    "dmPolicy": "pairing"
  }
}
```

### 7c. Restart

```bash
cd ~/Documents/openclaw
docker compose restart
```

### 7d. Verify Telegram is running

```bash
docker compose logs --tail 10 | grep -i telegram
```

You should see something like:
```
[telegram] [default] starting provider (@your_bot_name)
```

If you instead see `Cannot find module` errors, you're still on the source-built image. Go back to Step 4 and switch to the pre-built image.

### 7e. Pair your account

1. In Telegram, search for your bot's username (e.g., `@my_openclaw_bot`)
2. Tap on it and send `/start`
3. The bot replies with a pairing code
4. Approve it:

```bash
cd ~/Documents/openclaw
docker compose exec openclaw-gateway openclaw pairing approve telegram <CODE>
```

5. Send a regular message to your bot — it should reply using DeepSeek

---

## Step 8: Store API keys safely

Never commit API keys. The `.env` file should be in `.gitignore`. Restrict permissions:

```bash
chmod 600 ~/Documents/openclaw/.env
```

## Step 9: Open the Control UI

Go to `http://127.0.0.1:18789/` in your browser. Paste your gateway token when prompted.

## Step 10: Run the security audit

```bash
cd ~/Documents/openclaw
docker compose exec openclaw-gateway openclaw security audit --deep
```

---

## Using OpenClaw

### Chat

- **Web UI:** `http://127.0.0.1:18789/` — type in the chat box
- **Telegram:** message your bot from your phone

### Useful slash commands

| Command | What it does |
|---------|-------------|
| `/status` | Show model, context usage, token count, estimated cost |
| `/usage full` | Append token/cost footer to every response |
| `/usage cost` | Show cost summary from session logs |
| `/context detail` | Per-file, per-tool token breakdown |
| `/model deepseek/deepseek-chat` | Switch to a cheaper model mid-chat |
| `/compact` | Summarize and trim long sessions to save tokens |
| `/stop` | Abort the current task |
| `/verbose` | Show detailed tool output |

### Customize personality

Edit the markdown files in your workspace:

```bash
vim ~/.openclaw/workspace/SOUL.md      # Agent personality, tone, values
vim ~/.openclaw/workspace/USER.md      # Who you are, your preferences
vim ~/.openclaw/workspace/IDENTITY.md  # Agent name, avatar
```

Or ask the agent: "Read BOOTSTRAP.md and walk me through the setup" — it'll interview you and build the files.

Changes take effect on the next message, no restart needed. Keep SOUL.md under 2,000 words to avoid wasting tokens.

### Monitor token usage

- `/status` in chat for a quick snapshot
- `/usage full` to see cost per response
- Check [platform.deepseek.com](https://platform.deepseek.com) for actual billing

### Cost-saving tips

- Use `/compact` to trim long sessions — context accumulation is the biggest cost driver
- Use `deepseek-chat` for simple tasks, `deepseek-reasoner` for complex ones
- Be careful with Heartbeat (proactive wake-ups) — each trigger is a full API call with full context

---

## Troubleshooting

### Container stuck in restart loop

**Symptom:**
```
STATUS: Restarting (1) 55 seconds ago
```

**Diagnose:**
```bash
cd ~/Documents/openclaw
docker compose logs --tail 30
```

**Common causes and fixes:**

| Cause | Fix |
|-------|-----|
| Missing/invalid API key | Check key in `~/.openclaw/openclaw.json` and `.env` |
| Malformed config (JSON syntax error) | Validate JSON, check for missing commas or brackets |
| Port 18789 in use | Run `lsof -i :18789` and kill the conflicting process |
| Control UI origin error | Add `allowedOrigins` to gateway config (see below) |
| Env variable missing in container | Add it to the `.env` file (see below) |

### `Gateway failed to start: non-loopback Control UI requires gateway.controlUi.allowedOrigins`

**Fix:** Add to your config (`~/.openclaw/openclaw.json`):

```json
"gateway": {
  "bind": "loopback",
  "controlUi": {
    "allowedOrigins": ["http://127.0.0.1:18789"]
  }
}
```

Then restart:
```bash
cd ~/Documents/openclaw
docker compose stop
docker compose up -d
```

### `Unknown model: anthropic/deepseek-reasoner`

The model name is wrong. DeepSeek is its own provider, not under Anthropic. Make sure:

1. You have a `models.providers.deepseek` block in your config
2. The agent's primary model uses the `deepseek/` prefix: `"deepseek/deepseek-reasoner"`
3. Not `"anthropic/deepseek-reasoner"` or just `"deepseek-reasoner"`

### `Environment variable "DEEPSEEK_API_KEY" is missing or empty`

Docker containers don't inherit your shell's environment variables. You need to pass them via the `.env` file.

**Fix:** Add the variable to `~/Documents/openclaw/.env`:

```
DEEPSEEK_API_KEY_MY_MACHINE=sk-your-actual-key-here
```

Also make sure the variable name in `~/.openclaw/openclaw.json` matches exactly:

```json
"apiKey": "${DEEPSEEK_API_KEY_MY_MACHINE}"
```

### `invalid spec: :/home/node/.openclaw: empty section between colons`

The `.env` file is missing `OPENCLAW_CONFIG_DIR` and/or `OPENCLAW_WORKSPACE_DIR`. Docker Compose needs these to mount volumes correctly.

**Fix:** Make sure your `.env` file has **all** required variables:

```bash
cd ~/Documents/openclaw
cat > .env << EOF
OPENCLAW_CONFIG_DIR=/Users/$(whoami)/.openclaw
OPENCLAW_WORKSPACE_DIR=/Users/$(whoami)/openclaw/workspace
DEEPSEEK_API_KEY_MY_MACHINE=sk-your-actual-key-here
OPENCLAW_IMAGE=alpine/openclaw:latest
EOF
```

### `Cannot restart container: container is exited`

The container has stopped completely, so `docker compose restart` won't work.

**Fix:** Use `up` instead:

```bash
cd ~/Documents/openclaw
docker compose up -d
```

### Telegram plugin: `Cannot find module '../../../src/infra/outbound/send-deps.js'`

This is a known bug in the source-built Docker image. The pre-built image works fine.

**Fix:** Switch to the pre-built image by adding to your `.env` file:

```
OPENCLAW_IMAGE=alpine/openclaw:latest
```

Then:

```bash
cd ~/Documents/openclaw
docker compose pull
docker compose up -d
```

Verify the fix:

```bash
docker compose logs --tail 10 | grep -i telegram
```

You should see `starting provider (@your_bot_name)` instead of the error.

### Control UI says "pairing required"

Docker's NAT bridge makes the gateway see an external IP instead of localhost, so it skips auto-approval.

**Fix:**

1. List pending device requests:
```bash
cd ~/Documents/openclaw
docker compose exec openclaw-gateway openclaw devices list
```

2. Find the pending request ID in the output table (the value in the "Request" column).

3. Approve it:
```bash
docker compose exec openclaw-gateway openclaw devices approve <request-id>
```

4. Refresh `http://127.0.0.1:18789/` in your browser.

> **Note:** The Docker service name is `openclaw-gateway` (not `openclaw`). Verify with:
> ```bash
> docker compose ps --format "{{.Service}}"
> ```

### Control UI says "disconnected (1006): no reason"

The container likely crashed. Check:

```bash
docker compose ps
docker compose logs --tail 20
```

Fix whatever error the logs show, then:

```bash
cd ~/Documents/openclaw
docker compose stop
docker compose up -d
```

### Getting the gateway token

```bash
grep -i token ~/.openclaw/openclaw.json
```

Or open the file directly and look for the `"token"` field inside the `"gateway"` → `"auth"` section.

### Config file location

The config file is `openclaw.json` (not `openclaw.json5`):

```bash
# Wrong
vim ~/.openclaw/openclaw.json5    # empty file

# Correct
vim ~/.openclaw/openclaw.json
```

To see all files in the config directory:
```bash
ls ~/.openclaw/
```

### `code` command not found (VS Code)

Open VS Code manually, press `Cmd + Shift + P`, type "shell command", select **"Shell Command: Install 'code' command in PATH"**. Close and reopen terminal.

Or just use vim:
```bash
vim ~/.openclaw/openclaw.json
```

### CLI command syntax differences

| What you might try | What actually works |
|---|---|
| `docker compose exec openclaw ...` | `docker compose exec openclaw-gateway ...` |
| `openclaw-cli gateway token` | `grep -i token ~/.openclaw/openclaw.json` |
| `openclaw-cli pairing list` | `openclaw-cli pairing list --channel <channel>` |
| `openclaw-cli devices list` | `docker compose exec openclaw-gateway openclaw devices list` |
| `docker compose restart` (when exited) | `docker compose up -d` |

### Environment variable still shows after removing from `.zshrc`

It may be set in another file:

```bash
grep -r "DEEPSEEK_API_KEY" ~/.zshrc ~/.zprofile ~/.bash_profile ~/.bashrc ~/.env 2>/dev/null
```

Or it's cached in the current terminal session. Open a **new terminal window** and check again.

---

## Security Checklist

- [ ] Gateway bound to **loopback** (127.0.0.1)
- [ ] `allowedOrigins` set to `["http://127.0.0.1:18789"]`
- [ ] Consent mode enabled (`exec.ask: "on"`)
- [ ] Filesystem restricted to workspace only
- [ ] Docker sandbox enabled with `network: "none"`
- [ ] Tool profile set to `minimal`
- [ ] Elevated access disabled (`allowFrom: []`)
- [ ] DM pairing enabled for Telegram
- [ ] API keys stored in `.env` with `chmod 600`
- [ ] `.env` file added to `.gitignore`
- [ ] Gateway token rotated periodically
- [ ] No ClawHub skills installed without source code review
- [ ] Security audit passing: `openclaw security audit --deep`
- [ ] Using pre-built image (`alpine/openclaw:latest`) for stable Telegram support

---

## Always-On Setup (Mac Studio)

**What happens when Mac sleeps:** Docker pauses → OpenClaw freezes → Telegram bot goes silent.

**What happens when Mac wakes:** Docker resumes automatically → OpenClaw picks up where it left off → bot responds again. No data loss, no manual steps needed.

If you're okay with the bot going silent during sleep, leave everything as default — it recovers on its own.

If you want the bot available 24/7, run one command:

```bash
sudo pmset -a sleep 0 displaysleep 5
```

This disables system sleep but turns off the display after 5 minutes. Mac Studio uses ~6-8W at idle.

To undo later:

```bash
sudo pmset -a sleep 1
```

---

## Ongoing Maintenance

```bash
# Update
cd ~/Documents/openclaw
docker compose pull
docker compose up -d

# Health check after updates
docker compose exec openclaw-gateway openclaw doctor

# View live logs
docker compose logs -f

# Backup config
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak

# Backup .env
cp ~/Documents/openclaw/.env ~/Documents/openclaw/.env.bak
```

---

## Useful References

- Official docs: https://docs.openclaw.ai/install
- Docker setup: https://docs.openclaw.ai/install/docker
- Security guide: https://docs.openclaw.ai/gateway/security
- Telegram channel: https://docs.openclaw.ai/channels/telegram
- DeepSeek integration: https://github.com/dazeb/openclaw-deepseek-integration
- Model providers: https://docs.openclaw.ai/concepts/model-providers
- Token usage: https://docs.openclaw.ai/reference/token-use
- SOUL.md guide: https://docs.openclaw.ai/guides/soul-md
- GitHub: https://github.com/openclaw/openclaw
