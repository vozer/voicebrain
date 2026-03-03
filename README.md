# VoiceBrain

A personal AI voice assistant that runs on Telegram. Send voice notes, text messages, or photos — VoiceBrain understands your intent and routes to the right service automatically.

**This is a template** — fork it, customize the tools, swap the APIs, and make it yours. The entire assistant is a single n8n workflow (no code deployment needed). Extend it with AI coding tools like Cursor, Claude, or Codex.

Built with [n8n](https://n8n.io) + [OpenAI](https://openai.com) (GPT-4o-mini) + 9 integrated tools.

## What It Does

| You say... | VoiceBrain does... |
|---|---|
| "Buy milk and eggs" | Adds items to your [Bring!](https://www.getbring.com/) shopping list |
| "Create an issue about the font bug in rollersite" | Creates a GitHub issue in the right repo (fuzzy-matches project names) |
| "Remember to call Juan tomorrow" | Saves a note in [Memos](https://www.usememos.com/) |
| "What's the weather in Madrid?" | Searches the web via [Brave Search](https://brave.com/search/api/) |
| "Show my open tasks" | Lists all memos tagged `#task #open` |
| "Mark those tasks as done" | Replaces `#open` with `#done` in the memo |
| "Search my notes about meeting" | Full-text search across all memos |
| "Write this as an email in my style" | Uses Pinecone style memory to match your writing voice |
| Voice note with multiple topics | Handles each topic separately (multi-task) |

## Architecture

```
Telegram Bot (@your_bot)
  ↓
n8n: voicebrain-v2 (23 nodes)
  ↓
Message Trigger → Route Chat Input → [Voice/Photo/Text]
  → Parse Input (dedup + chat history)
  → VoiceBrain Agent (GPT-4o-mini, 9 tools)
  → Format Reply → Send Reply

Tools:
  ├── create_memo        → Memos API (notes, tasks, ideas)
  ├── add_to_bring       → Bring! Shopping API
  ├── create_github_issue → GitHub API (private repo support)
  ├── search_memos       → Memos API (full-text search)
  ├── web_search         → Brave Search API (real-time web search)
  ├── update_memo        → Memos API (append/replace/mark_done/replace_tag)
  ├── get_open_tasks     → Memos API (#task #open filter)
  ├── send_buttons       → Telegram inline keyboard (disambiguation)
  └── Pinecone Vector Store → Writing style retrieval
```

## Where to Host

VoiceBrain runs on any machine that can run Docker containers. Choose what fits your setup:

| Option | Cost | Privacy | Best For |
|--------|------|---------|----------|
| **Raspberry Pi / Home Server** | ~$0/month (hardware only) | Full control, data stays home | Privacy-first, always-on home setup |
| **[Hostinger VPS](https://www.hostinger.com/vps-hosting)** | ~$5/month | Your own server | Reliable uptime, easy setup |
| **[Hetzner Cloud](https://www.hetzner.com/cloud/)** | ~$4/month | Your own server | EU data residency, great value |
| **[n8n Cloud](https://n8n.io/cloud/)** | ~$20/month | n8n manages infra | Zero maintenance, managed service |
| **[Railway](https://railway.app/)** | ~$5/month | PaaS | Fast deploy, auto-scaling |
| **Any VPS** (DigitalOcean, Linode, etc.) | ~$5-10/month | Your own server | Familiar provider |

### Raspberry Pi / Home Server Setup

If you run this from home (like the author does):

```bash
# On your Raspberry Pi / mini PC
docker compose up -d

# Use a dynamic DNS service (e.g., DuckDNS, Cloudflare Tunnel)
# to expose n8n's webhook endpoint to Telegram
```

> **Note**: You need a public URL for the Telegram webhook. Options:
> - Cloudflare Tunnel (free, recommended)
> - DuckDNS + port forwarding
> - ngrok (free tier available)

## Prerequisites

### Core Services

| Service | Purpose | Required | Alternatives |
|---------|---------|----------|-------------|
| [n8n](https://n8n.io) | Workflow engine | Yes | — |
| [Memos](https://www.usememos.com/) | Self-hosted note server | Yes | [Notion API](https://developers.notion.com/), [Obsidian + API](https://obsidian.md/), [Joplin API](https://joplinapp.org/), any REST note service |
| [Bring!](https://www.getbring.com/) | Shared shopping list | No | [Todoist API](https://developer.todoist.com/), [AnyList](https://www.anylist.com/), any list app with API |
| [Pinecone](https://www.pinecone.io/) | Style memory (vectors) | No | [Qdrant](https://qdrant.tech/) (self-hosted), [Weaviate](https://weaviate.io/), [Chroma](https://www.trychroma.com/) |

### API Keys Required

| Key | Where to get it | Cost |
|-----|----------------|------|
| Telegram Bot Token | [@BotFather](https://t.me/BotFather) | Free |
| OpenAI API Key | [platform.openai.com](https://platform.openai.com) | ~$0.01-0.05/conversation |
| GitHub PAT | [github.com/settings/tokens](https://github.com/settings/tokens) | Free |
| Bring! Email/Password | Your Bring! account | Free |
| Brave Search API Key | [brave.com/search/api](https://brave.com/search/api/) | Free (2K queries/month) |
| Pinecone API Key | [pinecone.io](https://www.pinecone.io/) | Free tier available |

## Installation

### 1. Set Up Services

```bash
docker network create voicebrain-net

# Memos — self-hosted note server
docker run -d --name memos --network voicebrain-net \
  -p 5230:5230 -v memos-data:/var/opt/memos \
  neosmemo/memos

# n8n — workflow automation
docker run -d --name n8n --network voicebrain-net \
  -p 5678:5678 -v n8n-data:/home/node/.n8n \
  -e MEMOS_URL=http://memos:5230 \
  -e MEMOS_TOKEN=your_memos_token \
  -e GITHUB_PAT=your_github_pat \
  -e BRING_EMAIL=your@email.com \
  -e BRING_PASSWORD=your_password \
  -e BRAVE_SEARCH_KEY=your_brave_key \
  n8nio/n8n
```

### 2. Import the Workflow

1. Open n8n at `http://localhost:5678`
2. Go to **Workflows** → **Import from File**
3. Import `workflow/voicebrain-v2.json`
4. Configure credentials:
   - **Telegram Bot API**: your bot token from BotFather
   - **OpenAI API**: your API key
   - **Pinecone API**: your API key (optional)
5. Update credential references in each node that shows a warning

### 3. Set Up Memos

1. Open Memos at `http://localhost:5230`
2. Create an account
3. Go to **Settings** → **Access Tokens** → create a token
4. Add the token as `MEMOS_TOKEN` environment variable in n8n

### 4. Set Up Pinecone (Optional)

For writing style memory:

1. Create a Pinecone index named `voicebraincommand`
   - Dimensions: 1024
   - Metric: cosine
2. Load your writing samples into the index
3. Add your Pinecone API key as a credential in n8n

### 5. Activate

1. Activate the workflow in n8n
2. Send `/start` to your Telegram bot
3. Start talking!

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `MEMOS_URL` | Yes | Memos API base URL (e.g., `http://memos:5230`) |
| `MEMOS_TOKEN` | Yes | Memos personal access token |
| `GITHUB_PAT` | Yes | GitHub fine-grained PAT (needs Issues R/W) |
| `BRING_EMAIL` | Yes | Bring! account email |
| `BRING_PASSWORD` | Yes | Bring! account password |
| `BRAVE_SEARCH_KEY` | No | Brave Search API key (free tier) |
| `OPENAI_API_KEY` | Yes | OpenAI API key (used by n8n AI nodes) |

## How the Tools Work

### Memos (create_memo, search_memos, update_memo, get_open_tasks)
[Memos](https://www.usememos.com/) is a lightweight, self-hosted note server. VoiceBrain uses it as primary storage — notes, tasks, ideas, and reminders all saved as memos with hashtag-based organization (`#task`, `#open`, `#done`, `#idea`, etc.).

### Shopping (add_to_bring)
[Bring!](https://www.getbring.com/) is a shared shopping list app. Say "buy milk, eggs, and bread" and VoiceBrain adds all three items. Shared with family members who also use the app.

### GitHub Issues (create_github_issue)
Creates issues in your GitHub repositories. Fuzzy-matches project names — say "rollersite" and it finds the right repo. Supports private repos via fine-grained PAT. Automatically filters out archived repos.

### Web Search (web_search)
Real-time web search via [Brave Search API](https://brave.com/search/api/). Ask "What's the latest on n8n updates?" and get summarized results. Free tier gives 2,000 queries/month.

### Pinecone (style memory)
[Pinecone](https://www.pinecone.io/) stores vector embeddings of your writing samples. When VoiceBrain needs to write something in your style (emails, messages, posts), it retrieves similar text and uses it as tone/vocabulary context.

### Inline Buttons (send_buttons)
When VoiceBrain needs clarification (e.g., "which project for this issue?"), it sends Telegram inline keyboard buttons for quick disambiguation.

## Conversation Memory

VoiceBrain maintains a rolling conversation history (last 40 messages) in n8n workflow static data. This enables multi-turn conversations:

```
You: "Create an issue about the login bug"
Bot: "Which project?" [buttons: madrid-alive, rollersite, abonoteatro]
You: [clicks madrid-alive]
Bot: "Created issue #5 in madrid-alive: Login bug"
```

No external database needed — memory lives in n8n's static data and resets on `/start`.

## Telegram Commands

| Command | Description |
|---------|-------------|
| `/start` | Welcome message + clear chat history |
| `/help` | List all capabilities |
| `/tasks` | List open tasks (shortcut for natural language) |
| `/search <term>` | Search notes (shortcut for natural language) |

## Customizing VoiceBrain

This workflow is designed to be extended. Common customizations:

### Swap the Note Backend
Replace the Memos tool code with API calls to Notion, Obsidian, Joplin, or any service with a REST API. The AI Agent just needs a tool that accepts text and returns a confirmation.

### Add New Tools
Create a new **Code Tool** node in n8n, connect it to the AI Agent, and describe what it does. The AI will learn to use it from the description.

### Change the AI Model
Swap GPT-4o-mini for GPT-4o (better quality, higher cost), Claude, or any model supported by n8n's AI nodes.

### Add More Integrations
Ideas for additional tools:
- **Calendar** — Google Calendar API for scheduling
- **Email** — Send emails via Gmail/SMTP
- **Home automation** — Home Assistant API
- **Music** — Spotify API for playback control
- **Weather** — OpenWeatherMap for forecasts
- **Translation** — DeepL API
- **PDF/Document** — Summarize uploaded documents

## Security & Privacy

> **Important**: VoiceBrain processes personal data (voice transcriptions, notes, shopping lists, GitHub issues). Consider these points before deploying:

- **Self-hosted by design** — Memos and n8n run on your infrastructure. Your notes never leave your server (unless you use cloud-hosted alternatives)
- **API keys** — stored as n8n environment variables, never in the workflow JSON
- **OpenAI** — voice transcriptions and text are sent to OpenAI for processing. Review their [data usage policy](https://openai.com/policies/api-data-usage-policies)
- **Telegram** — messages transit through Telegram's servers. Use for personal/non-sensitive data
- **Bring!** — your email/password authenticate directly with Bring!'s API
- **Pinecone** — writing samples are stored as vector embeddings in Pinecone's cloud (use self-hosted Qdrant if this concerns you)
- **No analytics** — VoiceBrain does not track usage or send telemetry
- **Credential rotation** — rotate API keys periodically, use fine-grained tokens with minimal permissions

For maximum privacy, run everything on a home server (Raspberry Pi) with a Cloudflare Tunnel, and use self-hosted alternatives (Qdrant instead of Pinecone, local Whisper instead of OpenAI).

## Key Technical Notes

- **toolCode `query` variable**: n8n Code Tool nodes receive AI Agent input via `query`, not `$input`
- **Input schema**: Tools needing structured JSON must have `specifyInputSchema: true` with a `jsonSchemaExample`
- **Bring! API headers**: Requires `X-BRING-API-KEY`, `X-BRING-CLIENT: android`, `X-BRING-APPLICATION: bring`
- **Telegram formatting**: HTML mode (`<b>bold</b>`, `<i>italic</i>`), no markdown links — Telegram doesn't support `[text](url)`
- **n8n partial updates**: `n8n_update_partial_workflow` replaces the entire `parameters` object — always include all fields

## License

MIT — use it, fork it, extend it, sell it. No attribution required (but appreciated).
