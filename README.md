# OpenClaw on Pi: The Complete AI-First Setup Guide

I have a Raspberry Pi running 24/7 with a Claude-powered agent living inside my Discord. The thing that makes this different from every other "AI assistant" setup is that Discord becomes the operating system. Each channel is a domain — career, exercise, nutrition, dev-ops, journal — and the agent knows how to behave differently in each one. When I want to track something new, I create a channel and tell the agent what it's for. It writes its own cron job, sets up the schedule, and starts posting. I don't write the scripts. I don't configure the cron. I just say "post a stoic quote every morning in #philosophy" and it handles the rest — writes the script, sets the timer, deploys it. I can also say "create a new channel called "weather" for my city and tell me the weather each day at 5:00am. A $50 Pi replaces what used to require a full-stack app, a database, a notification service, and a frontend. Discord is the frontend. The Pi is the backend. The AI you plugin is the engineer. The full step-by-step setup guide is below if you want to build your own.

---

## What You're Building

A Raspberry Pi that runs 24/7 as your personal automation hub. An AI agent lives in your Discord servers, runs cron jobs, manages your career, tracks habits, and automates repetitive work. You talk to it in Discord. It talks back.

Two Discord servers:
- A **work** server — app development, career, dev-ops, content, learning
- A **family** server — shared planning, meals, budget, calendar, faith

The AI does most of the setup. You just need the hardware and accounts.

---

## What You Need

### Required
- Raspberry Pi 4 or 5 (4GB+ RAM recommended)
- microSD card (32GB+) with Raspberry Pi OS
- Internet connection (ethernet or wifi)
- Claude Code account (claude.ai/claude-code)
- Discord account
- GitHub account

### Recommended
- Gmail/Google Workspace account (for Docs, Sheets, Gmail automation)
- Tailscale account (free, for secure remote access)
- Domain or static IP (optional, Tailscale handles this)

---

## Part 1 — Flash and Boot the Pi

This section is for the human. The AI can guide you but can't physically insert an SD card.

1. Download Raspberry Pi Imager from raspberrypi.com
2. Flash **Raspberry Pi OS Lite (64-bit)** to your microSD
3. In Imager settings, enable SSH and set a username/password
4. Insert the card, plug in ethernet/power, and boot
5. Find the Pi's IP on your router or use: `ping raspberrypi.local`
6. SSH in: `ssh youruser@<pi-ip>`

Once you're SSH'd in, hand it to the AI.

---

## Part 2 — Base System Setup (AI Does This)

Tell your AI assistant to run these on the Pi:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl python3 python3-pip nodejs npm
```

Install Node.js (v20+):

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify:

```bash
node --version
npm --version
python3 --version
git --version
```

---

## Part 3 — Install Tailscale (Remote Access)

This lets you SSH into your Pi from anywhere without port forwarding.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the auth link it prints. Once connected, your Pi gets a stable Tailscale IP (e.g., `100.x.x.x`). Use that IP for all future SSH connections.

---

## Part 4 — Create Your Discord Servers

You need two servers. Do this in the Discord app or browser.

### Server 1: Work/Dev Server

Create a server. Name it whatever you want. Then create these channels:

- `#general` (default)
- `#agent` (where your AI lives)
- `#career`
- `#dev-ops`
- `#exercise`
- `#nutrition`
- `#journal`
- `#cron-jobs`

Common additions: `#app-ideas`, `#content`, `#learning`, `#stoic-quotes`, `#govcon`

### Server 2: Family Server

Create a second server. Invite family members. Channels:

- `#general`
- `#meal-planning`
- `#budget`
- `#calendar`
- `#faith`
- `#date-ideas`
- `#reading-lists`

---

## Part 5 — Create a Discord Bot

1. Go to discord.com/developers/applications
2. Click **New Application** — name it (e.g., "Q" or "Assistant")
3. Go to **Bot** tab → click **Add Bot**
4. Copy the bot token — save it somewhere safe
5. Under **Privileged Gateway Intents**, enable:
   - Message Content Intent
   - Server Members Intent
   - Presence Intent
6. Go to **OAuth2 → URL Generator**
   - Select scopes: `bot`
   - Select permissions: Send Messages, Read Message History, Embed Links, Attach Files
7. Copy the generated URL and open it in your browser
8. Add the bot to BOTH of your servers

---

## Part 6 — Get Your Discord IDs

Enable Developer Mode in Discord: **Settings → Advanced → Developer Mode → ON**

- Right-click each server → **Copy Server ID**
- Right-click each channel → **Copy Channel ID**

Save all of these. You'll put them in your config file.

---

## Part 7 — Set Up the Project (AI Does This)

On the Pi:

```bash
mkdir ~/automation && cd ~/automation
git init
npm init -y
npm install dotenv googleapis
```

Create the `.env` file (**NEVER** commit this):

```env
DISCORD_BOT_TOKEN=your_bot_token_here
# Optional Google OAuth (added later if using Gmail/Docs)
# GOOGLE_CLIENT_SECRET_JSON_PATH=./oauth/client_secret.json
```

Create `.gitignore`:

```
.env
oauth/
node_modules/
*.log
```

---

## Part 8 — Build the Config

Create `config/openclaw.json` with your server and channel IDs:

```json
{
  "gateway": {
    "port": 18789,
    "mode": "local"
  },
  "guilds": {
    "work": {
      "id": "YOUR_WORK_SERVER_ID",
      "channels": {
        "general": { "id": "CHANNEL_ID" },
        "agent": { "id": "CHANNEL_ID" },
        "career": { "id": "CHANNEL_ID" },
        "dev-ops": { "id": "CHANNEL_ID" },
        "exercise": { "id": "CHANNEL_ID" },
        "nutrition": { "id": "CHANNEL_ID" },
        "journal": { "id": "CHANNEL_ID" },
        "cron-jobs": { "id": "CHANNEL_ID" }
      }
    },
    "family": {
      "id": "YOUR_FAMILY_SERVER_ID",
      "channels": {
        "general": { "id": "CHANNEL_ID" },
        "meal-planning": { "id": "CHANNEL_ID" },
        "budget": { "id": "CHANNEL_ID" },
        "calendar": { "id": "CHANNEL_ID" },
        "faith": { "id": "CHANNEL_ID" }
      }
    }
  }
}
```

---

## Part 9 — Create Your First Cron Script

Create `scripts/cron-daily.sh`:

```bash
#!/bin/bash
# Usage: bash scripts/cron-daily.sh <topic>
# Topics: exercise, nutrition, journal, weather

source .env
TOPIC=$1

# Map topics to channel IDs (fill these in)
declare -A CHANNELS
CHANNELS[exercise]="YOUR_EXERCISE_CHANNEL_ID"
CHANNELS[nutrition]="YOUR_NUTRITION_CHANNEL_ID"
CHANNELS[journal]="YOUR_JOURNAL_CHANNEL_ID"

# Generate message based on topic
case $TOPIC in
  exercise)
    MSG="Daily workout reminder. Log your session in this channel."
    ;;
  nutrition)
    MSG="Daily nutrition check-in. What did you eat today?"
    ;;
  weather)
    WEATHER=$(curl -s "wttr.in/?format=3")
    MSG="$WEATHER"
    ;;
  *)
    echo "Unknown topic: $TOPIC"
    exit 1
    ;;
esac

# Post to Discord
CHANNEL_ID=${CHANNELS[$TOPIC]}
JSON=$(python3 -c "import json; print(json.dumps({'content': '$MSG'}))")

curl -s -X POST "https://discord.com/api/v10/channels/$CHANNEL_ID/messages" \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$JSON"
```

Make it executable: `chmod +x scripts/cron-daily.sh`

---

## Part 10 — Set Up Cron Jobs

Edit crontab: `crontab -e`

Example schedule:

```cron
# Exercise reminder at 7am
0 7 * * * cd ~/automation && bash scripts/cron-daily.sh exercise

# Nutrition check at 6pm
0 18 * * * cd ~/automation && bash scripts/cron-daily.sh nutrition

# Weather at 6:30am
30 6 * * * cd ~/automation && bash scripts/cron-daily.sh weather
```

---

## Part 11 — Optional: Google API Setup

If you want Gmail monitoring, Google Docs automation, or Sheets integration:

1. Go to console.cloud.google.com
2. Create a new project
3. Enable these APIs:
   - Gmail API
   - Google Sheets API
   - Google Docs API
   - Google Drive API
4. Create **OAuth 2.0 credentials** (Desktop app type)
5. Download the `client_secret` JSON file
6. Place it in: `oauth/client_secret.json`
7. Update your `.env`:
   ```
   GOOGLE_CLIENT_SECRET_JSON_PATH=./oauth/client_secret.json
   ```

Create `scripts/google-auth.js` for the OAuth flow:

```javascript
const { google } = require('googleapis');
const fs = require('fs');
const readline = require('readline');

const SCOPES = [
  'https://www.googleapis.com/auth/gmail.readonly',
  'https://www.googleapis.com/auth/gmail.send',
  'https://www.googleapis.com/auth/spreadsheets',
  'https://www.googleapis.com/auth/documents',
  'https://www.googleapis.com/auth/drive.file'
];

const creds = JSON.parse(
  fs.readFileSync(process.env.GOOGLE_CLIENT_SECRET_JSON_PATH || './oauth/client_secret.json')
);
const { client_id, client_secret, redirect_uris } = creds.installed;
const oauth2 = new google.auth.OAuth2(client_id, client_secret, redirect_uris[0]);

const url = oauth2.generateAuthUrl({ access_type: 'offline', scope: SCOPES });
console.log('Open this URL in your browser:\n', url);

const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
rl.question('Paste the auth code: ', async (code) => {
  const { tokens } = await oauth2.getToken(code);
  fs.writeFileSync('./oauth/token.json', JSON.stringify(tokens, null, 2));
  console.log('Tokens saved to oauth/token.json');
  rl.close();
});
```

Run it: `node scripts/google-auth.js`
(This requires a browser — do it from your laptop, then copy `token.json` to the Pi)

---

## Part 12 — Agent Context Files

Create channel-specific context files so the AI knows how to behave in each channel.

```bash
mkdir -p channels agent memory
```

Example `channels/exercise.md`:

> Post daily workouts. Track consistency. Be encouraging but not cheesy.
> Format: bullet list of exercises with sets x reps.

Example `channels/career.md`:

> Track job applications. Post new opportunities.
> Target roles: AI deployment, solutions engineering, technical customer success.
> Format: Company | Role | Link | Status

Example `agent/SOUL.md`:

> You are Q, a personal automation agent.
> Style: lowercase, direct, no fluff, stoic pragmatism.
> You live on a Raspberry Pi and communicate through Discord.
> Your job is to help the operator stay organized, ship code, and build good habits.

---

## Part 13 — Git and Deploy Workflow

From your laptop (where you do development):

```bash
git remote add origin https://github.com/yourusername/your-repo.git
git add . && git commit -m "Initial openclaw setup" && git push origin main
```

From the Pi (pull updates):

```bash
cd ~/automation && git pull origin main
```

Or automate the sync. After every push from your laptop, SSH to the Pi and pull:

```bash
ssh youruser@YOUR_TAILSCALE_IP "cd ~/automation && git pull origin main"
```

---

## Part 14 — Verify Everything Works

Run these checks:

```bash
# Test Discord posting
bash scripts/cron-daily.sh weather

# Test Google auth (if configured)
node scripts/google-auth.js

# Check cron is scheduled
crontab -l

# Check Pi is on Tailscale
tailscale status

# Verify git sync
git log --oneline -5
```

---

## Part 15 — Extending the System

Once the base is running, tell your AI to add:

### Career Automation
- Job search scripts that post to `#career`
- Resume builders that push to Google Docs
- Government contract scrapers (SAM.gov API)

### Content Pipelines
- Reddit scrapers for market research
- Image generation via Replicate API
- Social media uploaders

### Life Tracking
- Daily stoic quotes to a `#philosophy` channel
- Meal planning posts to the family server
- Weather reports every morning

### Gmail Monitoring
- Watch for important emails
- Post summaries to a Discord channel
- Auto-categorize by sender/subject

Each of these is a script in `scripts/` with a cron entry. The AI builds them. You review and approve.

---

## Cost Breakdown

| Item | Cost |
|------|------|
| Raspberry Pi 4/5 | $35–$80 (one-time) |
| Claude Code | Subscription (check claude.ai for pricing) |
| Discord | Free |
| Google APIs | Free tier covers personal use |
| Tailscale | Free for personal use |
| GitHub | Free |

**Total recurring:** just the Claude Code subscription.

---

## Final Notes

This system is designed to be **AI-first**. You describe what you want, the AI writes the scripts, sets up the cron jobs, and manages the config files. You just need to:

1. Provide the hardware (Pi + internet)
2. Create the accounts (Discord, GitHub, Claude Code, optional Gmail)
3. Approve what the AI builds

**The AI handles:** code, config, deployment, scheduling, and ongoing maintenance.

**You handle:** hardware, account creation, and saying "yes" or "no."

---

## License

MIT
